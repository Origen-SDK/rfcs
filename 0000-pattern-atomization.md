- Start Date: 2018-05-18
- RFC PR: (leave this empty)
- Implementation PR: (leave this empty)

# Summary

Add the ability to define pattern 'atoms' in pattern source code in Origen, so that pattern splitting can
be more easily handled in an efficient manner.

# Motivation

In a test program/project with 1000s of test patterns, each with unique functionality,
it is often desired to reduce the overall count of test patterns required to save on test program
load and validation time, as well as save on tester memory and to help with making software file
management easier.

Most of the patterns have many of the same parts in common, so a common strategy is to split these
patterns apart into common file chunks, that can be re-assembled into a variety of unique
pattern bursts.

The most common parts of a pattern that is typically separated is the startup and shutdown, both
top-level device startup/shutdown as well as those for any sub_block IPs therein.
Origen handles startups and shutdowns in separate functions that does permit easily omitting them if desired.

However, in reality, in order to ensure that the Origen registers and pins are in the correct state between
the end of startup and the main pattern body, as well as between the main pattern body and the shutdown, 
many users choose to still execute the entire pattern in Origen, only inhibiting vector output for the parts of 
the pattern not desired.  This method is ineffecient from a pattern generation time perspective, having to run
a pattern essentially 3x if you wanted to split it into 3 sub-patterns.  This process gets significantly more 
complex and inefficient if you wanted to split the pattern in more than 2 places.

Not only that, but the more you split the pattern the more chances of having a different register state across
each split boundary, so these partial patterns become less and less swappable with each other, basically causing a 
limit to how much you can split patterns effectively.

What we need is a wrapper that can be done around blocks of code that magically handles the register conflicts for us.
A wrapper that lets us know if a pattern section touches a register and if so helps to restore it back to its original
state.  It would also allow for improved optionality if we say want to split a pattern for one tester type (ATE) but 
not for another (origen_sim).

# Detailed design


This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

### Pattern Source
Ideally we want to do something like this in our pattern infrastructure:

~~~ruby
atomize(options) do |atom|
   block_select(blah)
end
atomize(options) do |atom|
   erase(blah)
end
~~~

which could be keyed off of `dut.atomize?` or `tester` to determine whether to actually implement the atomic pattern splitting or not.


Origen will then keep track of whether a register is dirtied or not.

The register model will need to be updated to handle the dirty/clean concept. 
i.e. you would write to register normally, then when you wanted to freeze it you would save the clean state:

~~~ruby
reg.save_clean
~~~

Then subsequent writes would 'dirty' the register: 

~~~ruby
reg.write(data) # doesn't matter if writing out to pattern or not, the register has been dirtied in origen
~~~

Then when you wanted to restore the 'clean' state you'd do:

~~~ruby
reg.clean  # only in origen
reg.clean! # in origen and out to test stimulus (pattern)
~~~

And this would restore clean register state again.

If you had a dirty register and wanted to update the clean state again, just run 'save_clean' again:

~~~ruby
reg.save_clean
~~~

If you wanted to clear the clean data back to reset state, without resetting the current state of the register (why don't know):
 
~~~ruby
reg.reset_clean
~~~

Basically now every register has 2 bit collections associated with it-- a clean/save state and a currently active state.

Each bit collection has the same features as previous register model, i.e. :

~~~ruby
reg.bit.store_clean
reg.bit.reset_clean
reg.bits.overlay_clean
~~~

Etc.

Actually every bit field would have to have clean/dirty state on it and a backup value to store if dirtied.

# Drawbacks

Depending on how it is implemented, updating Origen to support this feature may add complexity and slow down how the register
register model operates. By design any updates to the register model has to be backwards compatible.

This atomization feature could be handled at the application level, but given that it is likely something desired by many
users it makes sense to include some support in Origen itself to handle.


### Implementation

The underlying register model need not be modified.  Since the goal is to have essentially 2 copies of every register so that each
and every feature has a normal and a 'clean' version, a simple implementation could be simply having a seprate register collection
object that represents all clean registers, a mirror of the main register model. But from the user POV it would seem all part of the
same register.

The goal would be to minimize performance impact to users who choose not to use this feature, and only have a minimal perforance
hit for users that do.

# Unresolved questions

- How is the 'clean' state different than the 'reset' state?

  They are similar.  'Reset' could imply more the state of the register
when reset it pulled (and could be changed to reflect changes in NVM that might impact the reset state).  'Clean' could imply 
more a state where we like the current value of the register for general operation, but understand that there may need
to be a temporary modification for a particular special operation.

- How to handle cases where the 'clean' state is determined dynamically via an overlay?