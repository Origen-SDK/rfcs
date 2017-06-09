- Start Date: 2017-06-09
- RFC PR: (leave this empty)
- Origen Issue: (leave this empty)

# Summary

Updates to the origen_testers API to support several features to permit broader program 
integration support when generating a V93K test program for a particular IP component.

Updates include features such as:

- Support for limits table outside of test flow file
- Multibin support
- Multiport support

# Motivation

To simplify test program integration between a V93K IP sub-program and the SoC test program
into which the IP will be integrated.  Intially support features above. Will file separate PR 
for specific SmartBuild features as they arise.

# Detailed design

Initial development mainly of an option to permit indicating whether you want separate limits file or 
not during program generation.   

### API:

~~~ruby
tester.create_limits_file = true
tester.multiport = "portname"
~~~

#### create_limits_file 

If set to non-false, then generate separate limits file and suppress limits included in test flows.

Perhaps also allow for limits file name or other optionality passed in here
also.  

Default: false, means generate limits as part of the test flow.

#### multiport

If set to non-false, then generate multiport burst type pattern names.  CPP code templates included
outside of origen_testers but API permits query.

Allow to pass in port name would like to be used so it can be included in pattern compile AIV file.

#### Other features added here as result of ongoing RFC discussion...

### Limits File object

Addition of limits file class, similar to variables_file class.   Test methods add the limits they
require to an instantion of this class.

Later upon program sheet generation, the limits file is generated via a template.

# Drawbacks

Don't really see any per se in adding this code other than additional program generation time.  Default
operation is as is today, so backwards compatible.

Mostly drawbacks I can see having to do with additional complexities in the test program dealing
with limits and bins being defined outside of the test flow.

# Unresolved questions

- SmartBuild additional required features?
- Pattern compiler optionality (pin config, v2b options, various dir involved)
