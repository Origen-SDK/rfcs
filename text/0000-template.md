- Start Date: 9-6-2017
- RFC PR: (leave this empty)
- Origen Issue: (leave this empty)

# Summary

This RFC looks to push Origin further upstream in the product life cycle by
creating the concept of a Test.  It will help answer the question 'Did we
test it completely?".

# Motivation

Ideally, everyone would use Origen to create their test patterns. In reality, there are 
existing tool suites that do a good job that cannot, should not be replaced for a particular
generation of products.  However, Origen can still provide value by managing the overall
test list for a product.  Test lists define what needs to be tested for each IP and the top
level device.  A test list if made up of tests, which is really just a stimulus of some sort
with an ID.  It is not a test pattern, a test exists prior to simulation, emulation, and first silicon.
By iterating around the device and IP tests, test program flows, patterns, pattern lists, 
and other tracking documents can be created.  In the end, making tests part of the model should 
soundly anwer the question: 'Did we test it completely?' (correctness is an entirely separate subject). 

# Detailed design

Tests would be scoped the same as registers, pins, specs, etc., available to Origen.top_level
and any class that includes Origen::Model.  Given that each company will never converge on
test level metadata, the Test model will be relatively simple:

~~~ruby
test :my_test_id, do |t|
  t.description 'my test description'
  t.conditions  [:minvdd, :maxvdd, :bin1_1300Mhz, :bin2_1200Mhz]
  t.platforms   [:v93k, :j750]
  t.meta1       'dkwew'
  t.meta2       'jkjejkf'
end
self.tests      # => [:my_test_id]
self.has_tests? # => false
~~~

I see only one required argument and three defined args (:conditions, :platforms and :description), anything else
defined in the block is test level metadata that must be important to the author, so it will be
added without audit.  The test conditions and platforms attributes can be filled in as the info becomes 
available.  They must be arrays of Symbols and the symbols should correlate as follows:

- conditions => parameter sets defined with the owning parent ($dut or a SubBlock)
- platforms  => ATE test environments set by 'origen e <platform>'

The test conditions can be described using the existing Origen::Parameters::Set class.

~~~ruby
define_params :minvdd do |params|
  params.conditions.lev_equ_set = 1
  params.conditions.lev_spec_set = 2
  params.conditions.vdd = 0.8.V
end
~~~

By tying the tests to the test conditions described by the parameter sets, it will make iteration
easy and better sync the model and the test program content.

The test description should be able to be defined in the block as above or
be defined as comments directly above the test definition (identical to the register description API).
The platforms attribute could be used to direct which test environments are applicable to the test.

# Drawbacks

Don't see any.

# Unresolved questions

None
