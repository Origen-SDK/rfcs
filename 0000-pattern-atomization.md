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

TBD: Why are we doing this? What use cases does it support? What is the expected
outcome?

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

# Drawbacks

Why should we *not* do this? Please consider the impact on teaching Origen,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

There are tradeoffs to choosing any path, please attempt to identify them here.

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?