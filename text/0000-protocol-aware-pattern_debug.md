- Start Date: 2018-03-05
- RFC PR: (leave this empty)
- Implementation PR: (leave this empty)

# Summary

Origen currently produces vector-level ATE patterns. When debugging silicon using these patterns, the user
must  decipher fails at the vector-level and make vector edits if they want to change the pattern behavior.
This RFC describes how to provide the user with register-level debug functionality (and in a way that they can get for
free from Origen), while still producing efficient vector-level patterns for production.

# Motivation

Jiang Liu (@jiangliu23) recently proposed something along these lines for the UltraFLEX platform and he created an initial proof of
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
* It would be hard to automatically convert an Origen protocol implementation into a native ATE one. Meaning that there would be a lot of work to re-implement the available Origen protocols in the ATE language, and potentially then repeat that for all supported platforms.
* Some PA implementations are actually not even all that good for debug, for example they don't allow you to single step through the code at runtime. The attraction of such a system must therefore be that it provides you with a way to natively write your patterns at transaction level in the first place, but that's no advantage at all if you are using Origen.
* The PA APIs can cost additional license fees to enable.
* Real patterns are more than just protocol transactions, e.g. often involving some direct pin interaction in-between transactions, so at best the official PA approach may only be a partial solution anyway.
* Most examples of PA seem to focus on the physical protocol, for example JTAG. While a step-up from vector-level, most engineers don't think in JTAG and they would still have mental translation to align to the register-level that they think in. While it is no doubt possible to create true register-level protocols using native PA APIs, examples of this seem to be few and far between.

# Detailed design

Given the following example Origen pattern code:

~~~ruby
# Source for my_pat

ss "Launch some command"
my_reg.write!(0x1234_5678)

ss "Verify that the command has completed and passed"
tester.wait time_in_us: 10

dut.pin(:done).assert!(1)
dut.pin(:done).dont_care

my_reg2.fail.read!(0)
~~~

We will continue to generate a vector-level representation exactly the same as we have it today. e.g.

~~~
// ###############################################################
// # Launch some command
// ###############################################################
repeat 6                                                         > tp0                          0 X X 1 ;
repeat 2                                                         > tp0                          0 X X 0 ;
repeat 2                                                         > tp0                          0 X X 1 ;
repeat 2                                                         > tp0                          0 X X 0 ;
// [JTAG] Write IR: 0xB
repeat 2                                                         > tp0                          0 1 X 0 ;
                                                                 > tp0                          0 0 X 0 ;
                                                                 > tp0                          0 1 X 1 ;
// [JTAG] /Write IR: 0xB
                                                                 > tp0                          0 1 X 1 ;
                                                                 > tp0                          0 1 X 0 ;
                                                                 > tp0                          0 1 X 1 ;
repeat 2                                                         > tp0                          0 1 X 0 ;
// [JTAG] Write DR: 0x1
                                                                 > tp0                          0 1 X 0 ;
repeat 33                                                        > tp0                          0 0 X 0 ;
                                                                 > tp0                          0 0 X 1 ;
// [JTAG] /Write DR: 0x1

//....
~~~

Additionally, most (all?) tester drivers will support a switch to enable PA debugging, initially this will be developed for
V93K (SMT8) and UltraFLEX.
When this switch is enabled in the target environment:

~~~ruby
OrigenTesters::V93K.new pa_debugging: true
~~~

then Origen will generate both the regular vector pattern and also a PA version. This PA version will be in the form of a VB/Java/C++ function, something like this (read this as pseudo-code, not sure if it is valid VB, Java, or C++!):

~~~c
// Function name will match the pattern name
function my_pat() {
  // ###############################################################
  // # Launch some command
  // ###############################################################
  origen.write_register(0x1000_0000, 0x1234_5678);  // my_reg  (we will at least generate comments like this, if not a full register table)
  
  // Wait for 10 us
  origen.cycle(1000);
  
  origen.assert_pin("done", 1);

  origen.read_register(0x1000_0200, 0x0000_0000, 0x0000_0080);  // my_reg2  (last arg is a compare mask)
}
~~~

In principle, we end up with a 'protocol aware' pattern which looks a lot like the original Origen source code.

### Selecting the PA Pattern

The way this PA pattern is implemented will mean that it will run much slower than its vector-based counterpart, so you never want to
use it in production.

The intended debug workflow is this:

* Tests are generated by default to use the vector-based patterns, same as today
* When debugging on the tester, you find that a specific test is failing and you don't understand why and want to play around with it. You then enable a switch to make that particular test use the PA function instead, allowing you to insert breakpoints to step through at the register-level, and add/modify transaction data easily.
* Once you get the test working (by making some pattern changes), you can leave the modified PA function in play until you update the changes in the Origen source and re-generate everything.

Conceptually this 'switch' will just do this at the test instance / test method level:

~~~c
if (PA_DEBUG) {
  my_pat();    // Call the PA function
} else {
  execute_pattern("my_pat");  // Execute the vector pattern
}
~~~

### Origen Std Lib

The [Origen Standard Library](https://github.com/Origen-SDK/origen_std_lib) is going to pick up most of the heavy lifting and will be undergoing some significant development to enable this.

It is expected to provide the following:

* Test instance / method implementations for all of the basic test types, e.g. functional, dc measurement, freq measurement. These will have the PA debug option built in.
* The APIs required to implement the PA pattern functions. This is read/write_register, wait, pin state assertions and drives, and generally everything supported by Origen's pattern API, e.g. match waits, etc.

### Read / Write Register Implementation

Users wishing to use this feature will need to generate two additional vector patterns from their application, read_register and write_register, which as the name suggests will each contain a single register read and write transaction respectively.

The read_register pattern should have overlay markers on the address bits and capture markers on the read data. The write_register pattern should have overlay markers on both the address and data vectors.

The PA APIs can then easily create the associated read and write functions very easily by overlaying into these patterns.

The real beauty in this approach is that all of the protocol information is encapsulated back at the Origen pattern source level. The new OrigenStdLib APIs don't need to understand the underlying protocol at all, they simply overlay and capture from the regular vector patterns which have been marked up by the Origen patgen. This means that this PA debug feature will instantly support any and all protocols supported by Origen.

### Origen Link

Astute readers will note that when the OrigenStdLib PA APIs are available on the tester, it will provide everything needed to make the tester a fully compliant [OrigenLink](http://origen-sdk.org/OrigenLink/) target.

All that is required then is to implement the necessary TPC client to connect to the Origen application process, and you will be able to actually step through and debug at the true Origen source level if you want to.

While not specifically part of this RFC, it is important that we ensure the route we take here is compatible with this vision and the required OrigenLink addon will likely follow not too long after this is implemented.

# Drawbacks

The PA patterns here will be flat patterns, meaning that if all of your patterns contain a reference to a common subroutine in their source code, then in the generated PA patterns they will all contain duplicate copies of the code generated by that subroutine.
That means that if there was a bug in the output from that subroutine, then in the PA patterns you can not fix it in one place - you would need to find/fix the issue on one pattern, backport it to Origen and then re-generate to fix the rest.

Having the PA patterns nicely organized into sub-routines that remove duplication is a benefit that you would get from writing your patterns natively in the tester API in the first place, but generally it is hard to replicate the code structure when transitioning from an original source to a 3rd party format.

Future iterations of this might introduce some Origen APIs to allow the user to express that certain sections should be treated as subroutines when generating PA patterns, however that is not being included in v1.

# Unresolved questions

Need to work out exactly how the code should be structured to enable the switch from vector pattern to PA pattern to be made easily by the user in order to debug a particular test. Dynamically calling functions as a variable name is generally not supported by compiled languages, so it is unlikely to work as easily as in the example above.

Origen should pick up most of the heavy lifting to generate the required read_register and write_register patterns, ideally doing it without the user having to do anything. Exactly how is TBD.
