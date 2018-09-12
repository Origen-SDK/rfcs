- Start Date: 2018-09-05
- RFC PR: (leave this empty)
- Implementation PR: (leave this empty)

# Summary

Add an `app/` directory to Origen applications to bring more structure to everything
that currently resides in `lib/` and make things easier to find/modify by new users.

# Motivation

I was watching a demo of another tool recently and it really stood out to me how much
more accessible it was to new users by having dedicated directories for test engineering
concepts like levels and timing.

Origen has a defined application directory structure, but the bulk of what constitutes
an app ends up as a free for all within the `lib/` directory.

Timing is a good example, Origen recently 
[added an API for this](https://origen-sdk.org/origen/guides/pattern/timing/#Defining_Timesets),
however where should that be used and where should a new user find an application's
timing definition(s)? - in the model, in the controller, who knows?!


# Detailed design

Here is the proposed top-level directory structure for Origen applications. It adds an
`app/` directory, moves the existing `lib`, `patterns`, `program` (renamed to `flows`) and
`templates` directories to inside of `app/`, then adds some new folders like `timing` and
`parameters`.

The top-level directory structure is otherwise unchanged, but this now contains things
that are more to do with the infrastructure of running the app, and what would be considered
the app itself (i.e. the bulk of the code the user's will write) is now in the app directory.

~~~text
.
├── app/               // New folder, contains almost all Ruby code
│   ├── controllers/
│   ├── flows/         // New name for the current program dir
│   ├── levels/        // Don't really have a levels API yet, if we do it will be used here
│   ├── lib/           // A lib dir still exists in here is a catch-all, but mainly to host
│   │   └── my_app.rb  // the app's top-level load file
│   ├── models/
│   ├── parameters/
│   ├── patterns/
│   ├── templates/
│   └── timing/
├── config/
├── doc/
├── environment/
├── lbin/
├── lib/               // If an app has a lib dir it will be added to the load path like
├── log/               // it is today, but new apps would not have this at all
├── output/
├── simulation/
├── spec/
├── target/
├── tmp/
└── waves/
~~~

### Backwards Compatibility

This change will be implemented to be fully backwards compatible, the orginal directories will
continue to be supported in parallel and with no plans to phase out support for them.

### How Will Code be Loaded

#### Source Files

Origen will be updated to understand that patterns, program source files and templates can
also be found in `app/patterns`, `app/flows` and `app/templates`.
From a user perspective, you will be able to interchangeably put them there or else in their
existing locations and they will be found by the appropriate Origen command automatically.

#### Models, Controllers and Misc Ruby Classes (in app/lib)

`app/lib` will be added to the Ruby LOAD_PATH (same as
an app's lib directory is today). This new dir is therefore directly equivalent to 
the existing top-level `lib/` dir and code would be loaded/required from it in the same
way - i.e. just give a relative reference like `require 'my_app/my_file'` and it will be loaded
regardless of whether it is in `lib/` or `app/lib`.

`app/models` and `app/controllers` will also be handled much the same, they will essentially
just be aliases for 'app/lib' but using these will help to organize the code a bit more clearly.

So far, conventions (from documentation and examples in the wild) have encouraged users to
create model/controller pairs with something like the following directory structure:

~~~text
lib/my_app/my_dut.rb
lib/my_app/my_dut_controller.rb
lib/my_app/blocks/my_ip.rb
lib/my_app/blocks/my_ip_controller.rb
~~~

These files would be loaded by an application via:

~~~ruby
require 'my_app/my_dut'
require 'my_app/my_dut_controller'
require 'my_app/blocks/my_ip'
require 'my_app/blocks/my_ip_controller'
~~~

and their Ruby class naming would be:

~~~ruby
module MyApp
  class MyDut

  end
end

module MyApp
  class MyDutController

  end
end

module MyApp
  module Blocks
    class MyIp

    end
  end
end

module MyApp
  module Blocks
    class MyIpController

    end
  end
end
~~~

Origen will make the association between the model and the controller based on how they have
been named (the controller class should be called `MODELNAMEController`).

Under the new structure, the code will be exactly the same, but it would be put into these
directories instead:

~~~text
app/models/my_app/my_dut.rb
app/models/my_app/blocks/my_ip.rb
app/controllers/my_app/my_dut_controller.rb
app/controllers/my_app/blocks/my_ip_controller.rb
~~~

Otherwise, they will be required in exactly the same way and their internal Ruby class naming
structure will be the same as before.

It would perhaps have been nice to drop 'my_app' from the path and namespace when moving to the
new structure, however that would come with some significant downsides:

* It would almost certainly mean that Origen would have to hi-jack and override Ruby's `require`
  and `load` functions. There are probably a lot of corner cases to worry about from doing that and
  it seems too risky, at least in combination with the other changes being discussed here.
* Renaming of class names and directory structure could break applications if plugins converted
  over to the new structure and changed some of the naming. Therefore it is better to go with a system
  like this which is purely a code organizational change, rather than a logical one.

Also a new generate model feature is going to be added (discussed below) which will deal with
the extra bit of typing/responsibility that is put on the user to maintain the application's
namespace - in short, a generator will create these for you anyway.


#### Timing, Parameters and Levels

The levels API doesn't exists yet, but assuming we have one in future it will be handled as
described here and which will apply initially to the Timing and Parameters APIs which do exist.

In contrast to the models and controllers which get loaded by Ruby, these directories will
not be added to Ruby's LOAD_PATH and instead will be loaded by Origen as part of the target
loading process.

These files will not be architected as Ruby classes or modules, instead they will be written in terms
of adding to the current DUT, this is best shown by example.

A timing file will look something like this:

~~~ruby
# app/timing/application.rb

dut.timeset :func do |t|
  t.wave :tck do |w|
    w.drive 0, at: 0
    w.drive :data, at: 50
  end
end
~~~

For all of these types of resources the convention will be to load a file called `application.rb`
first, this will be considered global, then it will look for a file named after the current target.
So the contents of these directories will look something like this:

~~~text
.
└── app/               
    └── timing/
         ├── application.rb 
         ├── dut1.rb
         └── dut2.rb
~~~

By default, Origen will only automatically load files like this for the current application.

If a plugin wishes to provide resources like this to an application, then a mechanism needs to be
available to load those too. It is proposed to add a method on the application instance to do this:

~~~ruby
Origen.app.load_definitions
~~~

Which by default would load only the `application.rb` files, then you can supply a target name (though
in reality it is just a name and doesn't need to correspond to a target), and optionally decide
to only load a subset:

~~~ruby
Origen.app.load_definitions :my_defs, only: [:timing, :levels]
~~~

Having such an API means that plugins can take care of loading this themselves, e.g. during their
model `initialize` method:

~~~ruby
Origen.app!.load_definitions :my_defs    # note the use of app!
~~~

or, they can advise their user's how to do it from their application:

~~~ruby
Origen.app(:my_plugin).load_definitions  # note the app name means this refers to the plugin's defintiions
~~~

In fact, I propose that the automatic loading of resources is only done for top-level applications. If the current
application is a plugin within its own development workspace, then the application code will still have to do the loading.
This will
ensure that it works the same way standalone and as a plugin, which should avoid any un-necessary surprises when it is
added to an application.

### Adding Generators

Origen has actually contained a code generation API for some time (heavily ripped-off from
Ruby on Rails), however it has not been documented and has only really been used with `origen_app_generators`.

As part of this change it is proposed to enable the `origen new` command within application workspaces for
the purpose of generating new resources. e.g.

~~~text
origen new model blah    # Will generate a stub for the given model and a controller for it

origen new timing blah
~~~

### Steps Involved in Implementing This

* Update Origen core (maybe some impact on OrigenTesters too) to make it look for code in the
  new locations in addition to the existing ones.
* Add the `load_definitions` method and integrate into the target loading process.
* Update Origen App Generators to create new apps that follow these new conventions.
* Add the 'origen new' resource generator command.
* Document it all.

# Drawbacks

Creates more churn, but this should be mitigated by being strictly backwards compatible - i.e.
existing apps don't need to change and/or adopt this unless they want to.

# Unresolved questions

Can't think of any.
