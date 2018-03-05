- Start Date: 2018-03-05
- RFC PR: (leave this empty)
- Implementation PR: (leave this empty)

# Summary

Origen currently produces vector-level ATE patterns; when debugging silicon using these patterns, the user
must  decipher fails at the vector-level and make vector edits if they want to change the pattern behavior.
This RFC describes how to provide the user with register-level debug functionality (and in a way that they can get for
free from Origen), while still producing efficient vector-level patterns for production.

# Motivation

@jiangliu23 recently proposed something along these lines for the UltraFLEX platform and he created an initial proof of
concept within NXP.
This RFC builds on that work by describing how we can incorporate these ideas into Origen to provide our users with similar
functionality across all of our supported ATE platforms, and without the user having to make any major changes to their application
code in order to benefit from it.

More generally, protocol-aware (PA) is a buzzword these days and users are increasingly expecting to be able to use it. This proposal aims to
provide all the benefits of PA while actually avoiding some significant downsides that come with some of the ATE native implementations
of PA.

In fact, this proposal does not use any of the native vendor PA implementations. Some of us in the Origen community have talked and
thought about how Origen can make use of these features over the past few years, but ultimately none of them actually seemed to be particularly appealing.

Reasons for this include:

* For whatever reason, they just seem to be very complicated to use and not very accessible. Generally, the Origen way is to use simpler ATE features where possible since the code is easier for engineers to understand and maintain. For example, rather than complicated PA APIs, this RFC is implemented by vector overlay/capture which is much more widely understood.
* It would be hard to automatically convert an Origen protocol implementation into a native ATE one. Meaning that there would be a lot of work to re-implement these protocols in the ATE language, and potentially then repeat that for all supported platforms.
* Some PA implementations are actually not even all that good for debug, for example they don't allow you to single step through the code at runtime. The attraction of such a system must therefore be that it provides you with a way to natively write your patterns at transaction level in the first place, but that's no advantage at all if you are using Origen.
* The PA APIs can cost additional license fees to enable.
* Real patterns are more than just protocol transactions, e.g. often involving some direct pin interaction in-between transactions, so at best the official PA approach may only be a partial solution anyway.

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
