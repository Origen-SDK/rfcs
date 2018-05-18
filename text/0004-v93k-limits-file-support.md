- Start Date: 2017-06-09
- RFC PR: https://github.com/Origen-SDK/rfcs/pull/3
- Implementation PRs: https://github.com/Origen-SDK/origen_testers/pull/72 and https://github.com/Origen-SDK/origen_testers/pull/73 

# Summary

Updates to the origen_testers API to support the use of a separate test limits file   
to permit broader program integration support when generating a V93K test program
for a particular IP component.

Updates include features such as:

- Generate limits table CSV that includes test limits based on test suite name.
- Suppress test limits included in test flow file for these tests.
- Multibin support added as the limits CSV is where bin #s (and other bin-related descriptions) are defined.
- Test mode support -- multiple columns to permit unique limits per test mode (as defined by user)
- Ability to support multiple limits per test as needed for parametric tests that have both a functional and a parametric limit.


# Motivation

To simplify test program integration between a V93K IP sub-program and the SoC test program
into which the IP will be integrated.  Intially support features above.

# Detailed Design

Initial development mainly of an option to permit indicating whether you want separate limits file or 
not during program generation.   

### API

~~~ruby
tester.create_limits_file = true

tester.limitfile_test_modes = %w(mode1 mode2 mode3)  # currently called limitfile_pims_events but prefer this name
~~~

#### create_limits_file

If set to non-false, then generate separate limits file and suppress test limits included in test flows.

Also means that multibin support will be added to any tests, meaning the test will look like this in the test_flow:

~~~
    run_and_branch(testsuite1)
    then
    {
    }
    else
    {
      multi_bin;
    }
~~~
**Default:** false; means generate limits as part of the test flow.

#### limitfile_test_modes
This is an array of strings that serves to indicate which test modes are desired within the limits file.  A 'test mode' can be anything the user wants it to be (test insertion, temperature, VDD setting, etc.).

The test mode is associated with the limit value, limit type and limit units columns only.  Ideally, it would permit to assign the unique values themselves within the API, but today this feature only serves to duplicate the same limit data (value, type and units) per test mode.

This feature adds an additional line (second line) to the limits file with **Test mode** in the first column, columns 2-4 empty, and names of test modes in subsequent columns.  Example shown below.

### Limits File object

Addition of limits file class, similar to variables_file class.   Test methods add the limits they
require to an instantion of this class.

Later upon program sheet generation, the limits file is generated via a template.

### Result

#### Create Limits File
Upon setting `tester.create_limits_file = true`, a CSV limits file is generated and placed in a subdirectory called `testtable/limits`. Per current program modularity convention, each sub-module will have a CSV file in this directory.  Each test flow will then reference a limits MFH file that calls the CSV files.

E.g. Limits MFH:

~~~
hp93000,testtable_master_file,0.1

testerfile limits/group_submodule1_limits.csv
testerfile limits/group_submodule2_limits.csv
testerfile limits/group_submodule3_limits.csv`
~~~

The format of the CSV (without test mode support) is as follows:
~~~
"Suite name","Pins","Test name","Test number","Lsl","Lsl_typ","Usl_typ","Usl","Units","Bin_s_num","Bin_s_name","Bin_h_num","Bin_h_name","Bin_type","Bin_reprobe","Bin_overon","Test_remarks"
"functest1","","test1_functestname","1","1","GE","LE","1","","10","","3","","","","",""
"functest2","","test2_functestname","2","1","GE","LE","1","","10","","3","","","","",""
"functest3","","test3_functestname","3","1","GE","LE","1","","10","","3","","","","",""
"paratest4","","test4_functestname","4","1","GE","LE","1","","10","","3","","","","",""
"paratest4","pin1","test4_paratestname","5","1.55","GE","LE","1.65","V","10","","3","","","","",""
"paratest5","","test5_functestname","6","1","GE","LE","1","","10","","3","","","","",""
"paratest5","pin1","test5_paratestname","7","1.375","GE","LE","1.525","V","10","","3","","","","",""
"paratest6","","test6_funtestname","8","1","GE","LE","1","","10","","3","","","","",""
"paratest6","pin1","test6_paratestname","9","4.5","GE","LE","5.5","V","10","","3","","","","",""
"paratest7","","test7_functestname","10","1","GE","LE","1","","10","","3","","","","",""
"paratest7","pin1","test7_paratestname","11","6","GE","LE","7","V","10","","3","","","","",""
~~~

Here it is again in tabular form for clarity:

|Suite name|Pins|Test name|Test number|Lsl|Lsl_typ|Usl_typ|Usl|Units|Bin_s_num|Bin_s_name|Bin_h_num|Bin_h_name|Bin_type|Bin_reprobe|Bin_overon|Test_remarks|
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|
|functest1||test1_functestname|1 |1    |GE|LE|1    ||10||3| | | | | | 
|functest2||test2_functestname|2 |1    |GE|LE|1    ||10||3| | | | | | 
|functest3||test3_functestname   |3 |1    |GE|LE|1    ||10||3| | | | | | 
|paratest4||test4_functestname   |4 |1    |GE|LE|1    ||10||3| | | | | | 
|paratest4|pin1|test4_paratestname|5 |1.55 |GE|LE|1.65 |V|10||3| | | | | | 
|paratest5||test5_functestname|6 |1    |GE|LE|1    ||10||3| | | | | | 
|paratest5|pin1|test5_paratestname|7 |1.375|GE|LE|1.525|V|10||3| | | | | | 
|paratest6||test6_functestname   |8 |1    |GE|LE|1    ||10||3| | | | | | 
|paratest6|pin1|test6_paratestname|9 |4.5  |GE|LE|5.5  |V|10||3| | | | | | 
|paratest7||test7_functestname   |10|1    |GE|LE|1    ||10||3| | | | | | 
|paratest7|pin1|test7_paratestname|11|6    |GE|LE|7    |V|10||3| | | | | | 

##### Noteable Requirements
- All test suites require bin #s to be filled in by the Test Suite hard and soft bin values set in the test flow.
- All test suites require a functional limit, in which:
   - **Lsl_type= GE** (greater than equal to)
   - **Usl_type = LE** (less than or equal to)
   - **Usl = Lsl = 1** as functional tests return only a 0 or 1 value.
- Parametric tests, in addition to the functional limit, also require a parametric limit.  For this reason, the ability to support multiple limits per test suite is required in the implementation.  For today the bin #s for the parametric limit match that of the functional limit for a parametric test.


#### Test Mode Option
If the `tester.limitfile_test_modes` option is used, then the Limits file will look as follows:

|Suite name|Pins|Test name|Test number|Lsl|Lsl_typ|Usl_typ|Usl|Units|Lsl|Lsl_typ|Usl_typ|Usl|Units|Bin_s_num|Bin_s_name|Bin_h_num|Bin_h_name|Bin_type|Bin_reprobe|Bin_overon|Test_remarks|
|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|--|
|Test mode||||MODEA|MODEA|MODEA|MODEA|MODEA|MODEB|MODEB|MODEB|MODEB|MODEB| | | | |||||
|functest1||test1-functestname|1 |1    |GE|LE|1    ||1|GE|LE|1||10||3| | | | | | 
|functest2||test2_functestname|2 |1    |GE|LE|1    ||1|GE|LE|1||10||3| | | | | | 
|functest3||test3_functestname |3 |1    |GE|LE|1    ||1|GE|LE|1||10||3| | | | | |
|paratest4||test4_functestname  |4 |1    |GE|LE|1    ||1|GE|LE|1||10||3| | | | | |
|paratest4|pin1|test4_paratestname|5 |1.55 |GE|LE|1.65 |V|1.55|GE|LE|1.65|V|10||3| | | | | | 
|paratest5||test5_functestname  |6 |1    |GE|LE|1    ||1|GE|LE|1||10||3| | | | | | 
|paratest5|pin1|test5_paratestname|7 |1.375|GE|LE|1.525|V|1.375|GE|LE|1.525|V|10||3| | | | | | 
|paratest6||test6_functestname   |8 |1    |GE|LE|1    ||1|GE|LE|1||10||3| | | | | | 
|paratest6|pin1|test6_paratestname|9 |4.5  |GE|LE|5.5  |V|4.5|GE|LE|5.5|V|10||3| | | | | | 
|paratest7||test7_functestname |10|1    |GE|LE|1    ||1|GE|LE|1||10||3| | | | | | 
|paratest7|pin1|test7_paratestname|11|6    |GE|LE|7    |V|6|GE|LE|7|V|10||3| | | | | | 

# Drawbacks

Not aware of any drawbacks.  Default operation is as is today, so backwards compatible.

# Unresolved questions
None

# Future improvements
These are possible improvements not currently necessary.

- Provide unique Lsl/Lsl_typ/Usl_typ per test mode (if used)
- Provide unique bin # between the parametric and functional limit of a parametric type test (if desired)

