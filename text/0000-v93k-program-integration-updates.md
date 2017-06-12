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

### Limits File object

Addition of limits file class, similar to variables_file class.   Test methods add the limits they
require to an instantion of this class.

Later upon program sheet generation, the limits file is generated via a template.

### Result

Upon setting `tester.create_limits_file = true`, a CSV limits file is generated and placed in a subdirectory called `testtable/limits`. Per current program modularity convention, each sub-module will have a CSV file in this directory.  Each test flow will then reference a limits MFH file that calls the CSV files.

E.g. Limits MFH:

~~~
hp93000,testtable_master_file,0.1

testerfile limits/group_submodule1_limits.csv
testerfile limits/group_submodule2_limits.csv
testerfile limits/group_submodule3_limits.csv`
~~~

The format of the CSV is as follows:
~~~
"Suite Name","Pins","Test name","Test number","Lsl","Lsl_typ","Usl_typ","Usl","Units","Bin_s_num","Bin_s_name","Bin_h_num","Bin_h_name","Bin_type","Bin_reprobe","Bin_overon","Test_remarks"
"test1","","               ","1 ","1    ","GE","LE","1    ","","10","","3","","","",""
"test2","","test2_seqlbl   ","2 ","1    ","GE","LE","1    ","","10","","3","","","",""
"test3","","test3_seqlbl   ","3 ","1    ","GE","LE","1    ","","10","","3","","","",""
"test4","","test4_seqlbl   ","4 ","1    ","GE","LE","1    ","","10","","3","","","",""
"test4","","test4_labelname","5 ","1.55 ","GE","LE","1.65 ","","10","","3","","","",""
"test5","","test5_seqlbl   ","6 ","1    ","GE","LE","1    ","","10","","3","","","",""
"test5","","test5_labelname","7 ","1.375","GE","LE","1.525","","10","","3","","","",""
"test6","","test6_seqlbl   ","8 ","1    ","GE","LE","1    ","","10","","3","","","",""
"test6","","test6_labelname","9 ","4.5  ","GE","LE","5.5  ","","10","","3","","","",""
"test7","","test7_seqlbl   ","10","1    ","GE","LE","1    ","","10","","3","","","",""
"test7","","test7_labelname","11","1    ","GE","LE","1    ","","10","","3","","","",""
~~~

Here it is again in tabular form for clarity:

|Suite Name|Pins|Test name|Test number|Lsl|Lsl_typ|Usl_typ|Usl|Units|Bin_s_num|Bin_s_name|Bin_h_num|Bint_type|Bin_reprobe|Bin_overon|Test_remarks|
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|
|test2||test2_seqlbl   |2 |1    |GE|LE|1    ||10||3|
|test3||test3_seqlbl   |3 |1    |GE|LE|1    ||10||3|
|test4||test4_seqlbl   |4 |1    |GE|LE|1    ||10||3|
|test4||test4_labelname|5 |1.55 |GE|LE|1.65 ||10||3|
|test5||test5_seqlbl   |6 |1    |GE|LE|1    ||10||3|
|test5||test5_labelname|7 |1.375|GE|LE|1.525||10||3|
|test6||test6_seqlbl   |8 |1    |GE|LE|1    ||10||3|
|test6||test6_labelname|9 |4.5  |GE|LE|5.5  ||10||3|
|test7||test7_seqlbl   |10|1    |GE|LE|1    ||10||3|
|test7||test7_labelname|11|1    |GE|LE|1    ||10||3|






NOTE that functional tests require LSL/USL limit set to GE/LE a value of 1.   Parametric tests require in addition the parametric test limits desired.

# Drawbacks

Don't really see any per se in adding this code other than additional program generation time.  Default
operation is as is today, so backwards compatible.

Mostly drawbacks I can see having to do with additional complexities in the test program dealing
with limits and bins being defined outside of the test flow.

# Unresolved questions

- SmartBuild additional required features?
- Pattern compiler optionality (pin config, v2b options, various dir involved)
