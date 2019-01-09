- Nov, 6th 2017
- RFC PR: (leave this empty)
- Implementation PR: (leave this empty)

# Summary

Create an ATE levels generator framework that will work across tester platforms, using the V93K as a POC

# Motivation

Directly supports getting Origen to v1.0 and checks off another test program collateral

# Detailed design

The first issue is to define what the source of truth is when making ATE levels.  ATE levels are determined by
the following components:

1. Device power domain specifications
2. ATE guardband based on ATE specific power supply and PEC accuracies
3. Test operation guardbands based on company/product specific business logic (i.e. relaxing at wafer sort due to contact resistance)

How can we store this information currently in Origen?

1. Origen::Specs can be applied to the DUT, sub_blocks, and power domains
2. Origen::PowerDomains can store nominal voltage and maximum voltage rating but currently does not include min, max specs
3. Origen::Parameters has a very flexible API for storing all sorts of data but it has no domain specific logic regarding ATE levels

The question we need to answer is where does the ATE levels generator go for these bits of information?  Let's examine each of the types of information
and sketch out a possible API.

1. Power domian specs: The first need is to add min and max spec limits to the model.

~~~ruby
$dut.power_domain :vdd do |d|
  d.description = 'Core power domain'
  d.nominal_voltage = 1.0.V
  d.mvr = 1.35.V
end
$dut.power_domains(:vdd).spec :vdd, :dc do |s|
  s.description = 'Core power domain voltage'
  s.min = 0.8.V
  s.max = 1.2.V
end
~~~

I showed this to my team and they were not fans, as it seems like duplication and a chance for error.  Using the API is also cumbersome:

~~~shell
peologin01 $ $dut.power_domains(:vdd).nominal_voltage # => 1.0
peologin01 $ $dut.power_domains(:vdd).specs(:vdd).min # => 0.8
peologin01 $ $dut.power_domains(:vdd).specs(:vdd).min # => 1.2
~~~

But I think some simple min, max methods could allow users to still use Origen::Specs and define power domains like this:

~~~ruby
$dut.power_domain :vdd do |d|
  d.description = 'Core power domain'
  d.nominal_voltage = 1.0.V
  d.mvr = 1.35.V
  d.min = 0.8.V
  d.max = 1.2.V
end
~~~

OK, so the next part to handle is how and where to store the power domain settings for each test operation.  We have an OrigenSourceFileCreator
module that interrogates dut.power_domains and dut.test_flow to create a bunch of test condition inspried parameter sets.

~~~ruby
module PPEKit
  class Product
    def define_levels_params
      define_params :pnom do |p|
        p.vddno1.oper1 = 0.9
        p.vddno1.oper2 = 0.9
        p.vddno1.oper3 = 0.9
        p.vddno2.oper1 = 0.9
        p.vddno2.oper2 = 0.9
        p.vddno2.oper3 = 0.9
        p.vddno3.oper1 = 0.9
        p.vddno3.oper2 = 0.9
        p.vddno3.oper3 = 0.9
~~~

Now this is where it starts to get sticky. Where do we expect folks to keep this data?  Origen::Parameters is very flexible but has no domain knowledge.
Before I continue on with the RFC I think everyone invoved needs to provide feedback on what I have so far.

# Drawbacks

TBD

# Unresolved questions

- where to store level sets info
- where to store level eqn info (API similar to Origen timings?)
- how to implement ATE platform specific guardbands.  I believe we should start modeling the actual supplies and PECs and allow users
  to model their tester config to whatever detail is necessary to implement guardbands.
