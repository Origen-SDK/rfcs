- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Origen Issue: (leave this empty)

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

### File Structure

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
    owner.instantiate_sub_block(@name, @attributes)
  end
end  
~~~

# Drawbacks

Can't think of any.

# Unresolved questions

None.
