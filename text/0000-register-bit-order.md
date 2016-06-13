- Start Date: (1016-06-13)
- RFC PR: ()
- Origen Issue: ()

# Summary

The PowerPC format of registers lists bits from MSB to LSB, numbered in reverse
of the current Origen register model, from bit 0 (MSB) to bit 31 (LSB) in a typical
32-bit register.  This request is for the Origen core register model/API to
support the PPC bit order when defining, assigning values, and reading registers
with the Origen API, including the assignment of values to bit collections inside
the register models.

# Motivation

As the NXP AMP buiseness unit has begun using Origen to generate tester patterns, 
we found the bit-order format of many registers is the reverse of the Origen model.
This prevents us from using the full capability of the register model, in accessing
values in bit collections or individual bit values after writing known hex values.

Example of reverse bit-order register:
	myreg.write(0x0001)
	myreg[0].data # => 1
	myreg[31].data  # => 0
	
# Detailed design

It is desired to assign a bit order property when declaring the register.
Here is an example of a declaration of a PPC formated register in Origen (using
reverse bit ordering).....with bit_order property passed in reg (optional) to 
tell Origen to use reverse bit ordering for this register.

reg :SIUL2_MIDR1, 0x4, size: 32, bit_order: reverse do |reg|
	bit 0..15,  :PARTNUM, res:0b0101011101110111, access: :ro
	bit 16,		:ED
	bit 17..21	:PKG
	bit 22.23	:reserved
	bit	24..27  :MAJOR_MASK
	bit 28..31	:MINOR_MASK
end

Example of reverse bit-order register:
	myreg.write(0x0001)
	myreg[0].data # => 1
	myreg[31].data  # => 0

# How We Teach This

A small addition to the register guide could easily explain the option to
reverse the bit order, and explain that this is only needed if your device
uses reverse bit ordering such as in most PowerPC register definitions.

Origen guides would not need to be re-organized, and only one example might
need to be supplied to show the user how the feature works.

# Drawbacks

Existing Origen register usage should not change or effect any user.  Only
the users that require the reverse register bit order will use it.  Some
up front discussions should include the CRR team to understand how this
proposal fits in with CRR and how they plan to represent PPC formatted registers.

# Alternatives

So far, the only alternative is to force the data in the register to be reversed
after writing, so that register reads produce the correct data by bit index.

# Unresolved questions

What is the best way to tell the Origen reg model to use reverse bit ordering ?
How is the CRR format going to deal with PPC formatted registers ?  We will
need to update Rosetta Stone for CRR parser if there is CRR support.
