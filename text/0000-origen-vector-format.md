- Start Date: 2018-01-5
- RFC PR: (leave this empty)
- Implementation PR: (leave this empty)

# Summary

Defines an Origen-specific pattern vector format, similar in principle to
.atp, .avc, .stil, etc.

# Motivation

A couple of use cases have emerged recently where it has been necessary to save
the output of an Origen pattern generation operation into a some file-based format and
then re-play it again later:

* The simulation capture and replay feature being added to OrigenSim requires
  an offline way of storing pin data sequences. See
  [this PR](https://github.com/Origen-SDK/origen_sim/pull/6) for details.
* An application that relies on Windows-based 3rd party tools to generate patterns,
  wanted to transfer the generated patterns to a Linux environment for simulation. The
  pattern could not be fully generated in the Linux environment due to missing the Windows
  dependencies, so it needs to be generated in Windows and the 're-played' on Linux.

# Detailed design

This is somewhat being presented as a fait accompli since an initial implementation is
already in the above OrigenSim PR, but there is no problem with making changes to that
if anything emerges from this review.

File extension: .org   

It's snappy and looks a bit like .origen.

Each line in the file represents a 'vector' where in this case the term is loosely
defined as the operations that should occur before the next cycle(s).

The simplest vector is this, which means do 1 cycle (x3):

~~~
1
1
1
~~~

Lines are terminated by an end of line, and the number of cycles is always the last
thing in the line.

Repeats are as you would expect, this is equivalent to the above and there is no limit on
the repeat size:

~~~
3
~~~

To represent pin data, there is no concept of a header to define a fixed pin order within
the file, as is common with ATE pattern vector formats.
Instead it will more closely align to how Origen works, and that is to record pin state
changes rather than their state at every vector.

Pin state changes have the following record format: [pin_id, operation, \*args]

Each line can have multiple pin operations before the cycle, separate by a semi-colon.

Here to drive the TDI pin to 1 before waiting 3 cycles:

~~~
tdi,drive,b1;3
~~~

The data argument here can be in any format accepted by `Origen::Value`, most commonly
this will likely be either a binary or hex string:

~~~
b101010100
hFF
~~~

Another example spread across a few vectors:

~~~
tdi,drive,b1;3
tdo,drive,b1;2
tdo,drive,b0;tdi,drive,b1;1
tdi,drive,b0;10
done,assert,b0;1
porta,assert,b11110010;portb,assert,hff5e;1
reset,drive,b0;1
~~~

The end of the pattern is simply indicated by the end of the file.

The above is perhaps not as human-readable as a nicely aligned conventional vector file,
but it's not too bad and real life examples are likely to be more repetive which would
make them easier to scan by eye.

Primarily, this format is chosen because it is easy to consume and generate with Origen,
and it is resilient to changes in the underlying application structure - e.g. to remove some
unused pins from the output.

A parser/consumer for this can be as easy as:

~~~ruby
File.open("my_pattern.org", "r") do |f|
  f.each_line do |line|
    *operations, cycles = *(line.split(';'))
    operations.each do |pin_id, operation, data|
      dut.pin(pin_id).send(operation, Origen::Value.new(data))
      cycles.to_i.cycles
    end
  end
end
~~~

# Drawbacks

Do we need *another* vector format?!!

The intention here is not really to introduce a new vector format that is going to
be used anywhere outside of the Origen eco-system.
Both the generator and consumer will always be Origen processes and it makes sense
to have a format available that we can control to optimize what is effectively
internal information exchange.

Using something like STIL for the sake of standards alignment would make parsing and
generation more difficult, carry significant risk of not being able to fully express
everything we want to, yet provide no real benefits.

# Unresolved questions

This is not intended to be a final spec, just to lay down the basics to bootstrap the
format.

It is expected that more syntax will be added over time as application use cases
dictate.
