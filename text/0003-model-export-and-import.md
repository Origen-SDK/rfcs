- Start Date: 2017-10-26
- RFC PR: https://github.com/Origen-SDK/rfcs/pull/4
- Implementation PR: https://github.com/Origen-SDK/origen/pull/162

# Summary

Add an official model import and export API to Origen core. 

# Motivation

The CrossOrigen plugin provides the ability to render a model into Ruby/Origen format, however it is not
complete (e.g. doesn't include pin information) and there are also various other incarnations of similar
exporters within company-internal plugins.

Rather than doing further development on the CrossOrigen exporter, it is proposed to move this responsibility
to Origen core, so that it can be maintained in sync with the Origen model APIs that it will target, and to
resolve a number of issues that currently exist within the ecosystem:

* Many applications which import data from 3rd party sources do the conversion at runtime, mainly during
  the target initialization process. While this works OK when importing small amounts of data, it has
  become evident that this pattern doesn't scale well and can result in slow target low times which can
  generally slow down a lot of Origen operations e.g. the program generator will re-load the target
  many times. Having an easy and official way to export to Origen format, means that all importers, like
  those provided by CrossOrigen, can change over to always exporting direct to Origen format so that the
  conversion from 3rd party format becomes an off-line activity.
* Sub-blocks are currently instantiated immediately as they are declared. It has been reported that for
  some large devices, e.g. with 20k registers or so, it can take ~2s to load the sub-blocks and registers
  from Ruby format. Registers are already lazily instantiated and there is not much we can do to improve
  that further. The 2s therefore, is largely just the time it takes to read in all the register files even
  though it does not actually build the registers. However, if the registers are all contained within multiple
  sub-blocks, then moving to a lazy-instantiation for sub-blocks (i.e. only instantiate the sub-block when it
  is first accessed by an application) will mean that reading all the register data will not happen during
  model initialization. This should make target load times instant as far as human perception is concerned.
* Pins currently have to be converted from the 3rd party source at runtime, this will be addressed so that
  pin data can be treated the same as register/sub-block data.

# Detailed design

### Top-Level API

The API should be very simple, two methods called `export` and `import` will be added to `Origen::Model`,
meaning that any Origen model object can be exported.

The export method will take a single argument, which is a name.
All exports will be saved to a default location which will be: 
`vendor/lib/models/<app_namespace>/<name>/`.

The corresponding `import` method will take the same name argument.

Models will be able to call the `import` method multiple times to load model data from different sources.

```ruby
# All importers, like those provided by CrossOrigen, should build up a model from the data it is importing
# and then call export on that model at the end:
imported_model.export('device_1_pin_data')

# The application model can then call import during initialization:
module MyApp
  class MyDUT
    include Origen::TopLevel
    
    def initialize(options = {})
      import 'device_1_pin_data'
      import 'device_1_registers_from_xml'
      import 'device_1_test_registers'
    end
  end
end
```

It is possible that multiple import may contain references to the same sub-block(s), e.g. one might contain the user registers for a given sub-block(s) and another might define the test registers for them.
The import method must be designed to be aware of and handle this case.

### Lazy Loading Sub-blocks

The mechanism for lazy loading sub-blocks will follow the same kind of approach used by registers.
That is, whenever `sub_block` is encountered a lightweight placeholder object will be created instead, thus
preventing the additional files describing the sub-blocks registers from being loaded.
When an application first references this sub-block, the placeholder will trigger the loading of the
sub-block and it will transparently transform into the final sub-block object.
The time to load a single sub-block is relatively quick (not really perceptable) and any one pattern or flow
will usually only interact with a small sub-set of all of the available sub-blocks/registers.

The placeholder will follow this structure:

~~~ruby
class SubBlockPlaceholder
  def initialize(name, *args)
    # The arguments provided by the application will be stored into instance variables
    @name = name
    @attributes = args
  end
  
  # When the application tries to use this sub-block for the first time, create it
  def method_missing(method, *args, &block)
    materialize.send(method, *args, &block)
  end
  
  # An instantiate_sub_block method will be added to Origen::Model, this will do exactly what the
  # current sub_block definition method does, and will replace the placeholder object with the
  # created sub_block
  def materialize
    # Create the sub_block on the owner object here
  end
end  
~~~

For reference, here is the placeholder class used for registers: https://github.com/Origen-SDK/origen/blob/master/lib/origen/registers.rb#L133

It contains a number of additional methods over and above the basic ones mentioned above. This is to handle a few common methods that are called on the register internally by Origen and to stop it instantiating the full register if the request can be fulfilled without doing so. Similar methods may or may not emerge for the sub-block placeholder during implementation.

### File Structure

The most important thing in the file structure is that the registers for each sub-block are in different files and that those files must only be required when their sub-block is instantiated.
It is also recommended that nested sub-blocks should be split into different files, ultimately with the goal that when an application calls something like `dut.sub_block1.sub_block2.some_register`, then the minimum possible sub-blocks and registers are instantiated to keep things snappy.

It must not be assumed that you are importing into the top-level DUT, a sub-model could equally be importing model data. Therefore the code must be designed that all references are relative to the importing object.

Here is a proposed structure, though the implementer has latitude to deviate if required:

```ruby
# vendor/lib/models/my_app/device_1_from_xml.rb
module MyApp
  module Device1FromXML
    def self.extended(model)
      # Pins, can be instantiated normally, don't believe that a few thousand pins will slow things down
      # much though we an address that later if required
      model.add_pin :pin_a
      model.add_pin :pin_b
      
      # This will be creating a placeholder, not the final instantiation, and we can make the placeholder
      # keep a note of the file that the sub-block definition lives in so that we can hold off requiring
      # it right now
      model.sub_block :some_block,
                      file: "my_app/device_1_from_xml/some_block" # relative to vendor/lib/models

      model.sub_block :some_other_block, file: "my_app/device_1_from_xml/some_other_block"
    end
  end
end

# vendor/lib/models/my_app/device_1_from_xml/some_block.rb
module MyApp
  module Device1FromXML
    module SomeBlock
      def self.extended(model)
        model.add_reg :int, 0x10, size: 16, reset: 0x0, description: 'Interrupt Register' do |reg|
          reg.bit , :en, access: :rw, description: 'Interrupt Register'
          reg.bit , :mode, access: :rw, description: 'Interrupt Register'
        end
        
        # Additional registers and sub-block definitions as required...
      end
    end
  end
end

# Example of the materialize method in the sub-block placeholder
def materialize
  # Create the sub_block unless it already exists (e.g. from a previous import)
  owner.sub_block @name unless owner.respond_to?(@name)
  require @file
  owner.send(@name).extend module_name_from_file(@file) # e.g. returns MyApp::Device1FromXML::SomeBlock,
                                                        # Ruby will automatically invoke the extended
                                                        # method when this is called
end

# Example of the import method added to Origen::Model, this is very simple
def import(name)
  require "vendor/lib/models/#{Origen.app.namespace}/#{name}"
  extend name.camelize.constantize # Ruby will automatically invoke the self.extended method
end
```

# Drawbacks

Can't think of any.

# Unresolved questions

None.
