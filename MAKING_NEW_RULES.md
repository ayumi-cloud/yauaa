Detecting new useragent patterns
===============================================================
When you find a useragent for which one or more of the fields are wrong there is the need to change the patterns and rules
that are used by this system for classifying these attributes.
In order to write rules this first described how the system works and what tools have been created to make writing new rules easier.

Base problem: They all lie
==========================
When looking at useragents it is clear that almost all of them include the name of predecessors/competitors
with which they are supposed to be compatible with.

So in general there is a ranking in the patterns; some are more true than others.

Solution overview
=================
The way this system solves all of this is by employing several steps:

1. The user agent string is parsed into a tree using Antlr4.
2. This tree is matched against a set of "Matchers"
   A matcher is
   * a set of patterns that must be present in the tree
   * a set of field/value combinations with a 'weight'
     the value can be either a fixed value or a part of the tree.
3. For all matchers where ALL required patterns were present the
   field/value/weight results are all combined. For fields where
   multiple values were present the value with the highest weight 'wins'

All matchers are with tests placed in yaml files in the UserAgents directory.
Because of the system with the weights the order of the matchers is not
checked or guaranteed in any way. So if the two matchers may set the same
field to a different value then better make sure they have different weights.

Useragent parse tree model
==========================
According to RFC-2616 a useragent consists of set of products. Each with an (optional) version and an (optional) comment block.

https://www.w3.org/Protocols/rfc2616/rfc2616-sec3.html#sec3.8
https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.43

After much analysis the model that this parser uses it as follows:



The whole string (the 'agent') is mostly cut into 'product' and 'text' parts.
Each child node in the tree is numbered (1,2,3,...) within the context of the parent.
A product has zero or more 'version' fields and zero or more 'comments' fields.
Each comments' has one or more 'entry':
In a comment block like this 'foo/1.0 (one; two three; four)' we will have 3 entries (they are ; separated).

From there the system recurses down within limitations.

Flattened form of the tree
==========================

If we take this useragent as an example the tree is flattened into a set of 'breadcrumb' type paths.

I put some spaces in vairous places so you can see what happens with those (in most places they are trimmed):

    foo/1.0 (one; two three; four) bar/2.0 (five; six seven)

will result in these paths (with their textual value)

    agent="foo/1.0 ( one  ; two three; four  ) bar/2.0 (five;six seven)"
    agent.(1)product="foo/1.0 ( one  ; two three; four  )"
    agent.(1)product.(1)name="foo"
    agent.(1)product.(1)name#1="foo"
    agent.(1)product.(1)name%1="foo"
    agent.(1)product.(1)version="1.0"
    agent.(1)product.(1)version#1="1"
    agent.(1)product.(1)version%1="1"
    agent.(1)product.(1)version#2="1.0"
    agent.(1)product.(1)version%2="0"
    agent.(1)product.(1)comments="( one  ; two three; four  )"
    agent.(1)product.(1)comments.(1)entry="one"
    agent.(1)product.(1)comments.(1)entry#1="one"
    agent.(1)product.(1)comments.(1)entry%1="one"
    agent.(1)product.(1)comments.(1)entry.(1)text="one"
    agent.(1)product.(1)comments.(1)entry.(1)text#1="one"
    agent.(1)product.(1)comments.(1)entry.(1)text%1="one"
    agent.(1)product.(1)comments.(2)entry="two three"
    agent.(1)product.(1)comments.(2)entry#1="two"
    agent.(1)product.(1)comments.(2)entry%1="two"
    agent.(1)product.(1)comments.(2)entry#2="two three"
    agent.(1)product.(1)comments.(2)entry%2="three"
    agent.(1)product.(1)comments.(2)entry.(1)text="two three"
    agent.(1)product.(1)comments.(2)entry.(1)text#1="two"
    agent.(1)product.(1)comments.(2)entry.(1)text%1="two"
    agent.(1)product.(1)comments.(2)entry.(1)text#2="two three"
    agent.(1)product.(1)comments.(2)entry.(1)text%2="three"
    agent.(1)product.(1)comments.(3)entry="four"
    agent.(1)product.(1)comments.(3)entry#1="four"
    agent.(1)product.(1)comments.(3)entry%1="four"
    agent.(1)product.(1)comments.(3)entry.(1)text="four"
    agent.(1)product.(1)comments.(3)entry.(1)text#1="four"
    agent.(1)product.(1)comments.(3)entry.(1)text%1="four"
    agent.(2)product="bar/2.0 (five;six seven)"
    agent.(2)product.(1)name="bar"
    agent.(2)product.(1)name#1="bar"
    agent.(2)product.(1)name%1="bar"
    agent.(2)product.(1)version="2.0"
    agent.(2)product.(1)version#1="2"
    agent.(2)product.(1)version%1="2"
    agent.(2)product.(1)version#2="2.0"
    agent.(2)product.(1)version%2="0"
    agent.(2)product.(1)comments="(five;six seven)"
    agent.(2)product.(1)comments.(1)entry="five"
    agent.(2)product.(1)comments.(1)entry#1="five"
    agent.(2)product.(1)comments.(1)entry%1="five"
    agent.(2)product.(1)comments.(1)entry.(1)text="five"
    agent.(2)product.(1)comments.(1)entry.(1)text#1="five"
    agent.(2)product.(1)comments.(1)entry.(1)text%1="five"
    agent.(2)product.(1)comments.(2)entry="six seven"
    agent.(2)product.(1)comments.(2)entry#1="six"
    agent.(2)product.(1)comments.(2)entry%1="six"
    agent.(2)product.(1)comments.(2)entry#2="six seven"
    agent.(2)product.(1)comments.(2)entry%2="seven"
    agent.(2)product.(1)comments.(2)entry.(1)text="six seven"
    agent.(2)product.(1)comments.(2)entry.(1)text#1="six"
    agent.(2)product.(1)comments.(2)entry.(1)text%1="six"
    agent.(2)product.(1)comments.(2)entry.(1)text#2="six seven"
    agent.(2)product.(1)comments.(2)entry.(1)text%2="seven"

As you can see ther are afew special operators '=', '#' and '%' that allow extracting specific words.

Walking around the tree
==========================
In many cases we want to get the version of the product with a specific name.
This means we should look to the product with that name and from there go 'up' to the product and from there select the 'version'.

For this purpose a special language has been created that allows walking through a tree from the point where a match was found and do additional
compare and string manipulation actions to obtain the value we are looking for.

This language also supports lookups (hashmaps) that do a case INsensitive lookup.

Note that for finding a matching pattern the system will recurse through the parse tree from left to right.
The FIRST matching pattern is used for the end result.

For demonstrating the operators I'm using this user agent to explain the effects:

    foo faa/1.0 2.3 (one; two three four) bar baz/2.0 3.0 (five; six seven)

Available operators:

Walking around the tree

Operation | Symbol | Example | Result value (if applicable)
:--- |:--- |:---|:---
Up to parent | ^ | agent.(1)product.name^ | agent.(1)product
Next Sibling | > | agent.(1)product> | agent.(2)product
Previous Sibling | < | agent.(2)product< | agent.(1)product
Down to child | .name | agent.(1)product.version |
Down to specific child | .(2)version | agent.(1)product.(2)version |
Down to specific child range | .([2-3])version | agent.(1)product.([2-3])version |

Comparing values in the tree

Operation | Symbol | Example | Result value (if applicable) | Explain
:--- |:--- |:--- |:---|:---
Equals | = | agent.(1)product.version="2.3" | agent.(1)product.(2)version | The second version is "2.3"
Not equals | != | agent.(1)product.version!="1.0" | agent.(1)product.(2)version | The second version is the first one when backtracking that is not "1.0"
Contains | ~ | agent.product.name~"ar" | agent.(2)product.(1)name="bar baz" | The first product name when backtracking that contains "ar"
Starts with | { | agent.product.name{"b" | agent.(2)product.(1)name="bar baz" | The first product name when backtracking that starts with "b"
Ends with | }| agent.product.name}"z" | agent.(2)product.(1)name="bar baz" | The first product name when backtracking that ends with "z"

Extracting substrings

Note this fact *agent.(1)product.(1)comments.(2)entry.(1)text="two three four"*

Operation | Symbol | Example | Value
:--- |:--- |:---|:---
First N Words | # | agent.(1)product.(1)comments.(2)entry.(1)text#2 | two three
Single Word at position N | % | agent.(1)product.(1)comments.(2)entry.(1)text%2 | three
Backto full value | @ | agent.(1)product.(1)comments.(2)entry.(1)text%2="three"@ | two three four

Special operations

Operation | Symbol | Example | Result value (if applicable)
:--- |:--- |:---|:---
Check if the expresssion resulted in a null 'no match' value. | IsNull[expression] | IsNull[agent.(1)product.(3)name] | true
Cleanup the version from an _ separated to a . separated string| CleanVersion[expression] | CleanVersion["1_2_3"] | 1.2.3
LookUp the value against a lookup table | LookUp[lookupname;expression] | LookUp[OSNames;agent.product.entry.text]
LookUp the value against a lookup table with fallback | LookUp[lookupname;expression;defaultvalue] | LookUp[OSNames;agent.product.entry.text;"Unknown"]

The anatomy of a matcher
========================
A matcher consists of 3 parts
require
extract
options

The "require" part holds the patterns that must all result in a non-null value. The IsNull operator is intended to check that a specific value actually IS null.
The "extract" part is to send either a fixed string or an extracted pattern into a field with a certain numerical confidence.

There are a few possible option flags

init    This means the init output for this matcher must be shown. This is the same effect when running it without any extract values
verbose This means that a LOT of debug output will be shown in the output.
only    This means that ONLY this test must be run. No others.

Only IFF all require AND all extract patterns yielded a non-null value then will all of the extracted values be added to the end result.
In general a field will receive a value from multiple matchers; the value with the highest confidence value will be in the output.

Creating a new rule
===================
Assume we got this useragent

    Mozilla/5.0 (compatible; Foo/3.1; Bar)

and let's just assume we want to set a field called MinorFooVersion if
the minor version of Foo is 1

To start we put the agent as a new test in one of the yaml files.
If we choose to create a new file make sure is starts with "config:"

    - test:
        input:
          user_agent_string: 'Mozilla/5.0 (compatible; Foo/3.1; Bar)'

Now we run the unit test "TestPredefinedBrowsers".
On a normal computer this will take only about 1-5 seconds.

The output contains all the field values for this useragent in the exact form of a unit test.

So once you are happy with the result you can simply copy and past this into the yaml file.

In this case is looks something like this:

    - test:
    #    options:
    #    - 'verbose'
    #    - 'init'
    #    - 'only'
        input:
          user_agent_string: 'Mozilla/5.0 (compatible; Foo/3.1; Bar)'
        expected:
          DeviceClass                          : 'Unknown'
          DeviceName                           : 'Unknown'
          OperatingSystemClass                 : 'Unknown'
          OperatingSystemName                  : 'Unknown'
          OperatingSystemVersion               : '??'
          LayoutEngineClass                    : 'Browser'
          LayoutEngineName                     : 'Mozilla'
          LayoutEngineVersion                  : '5.0'
          LayoutEngineVersionMajor             : '5'
          LayoutEngineNameVersion              : 'Mozilla 5.0'
          LayoutEngineNameVersionMajor         : 'Mozilla 5'
          AgentClass                           : 'Browser'
          AgentName                            : 'Foo'
          AgentVersion                         : '3.1'
          AgentVersionMajor                    : '3'
          AgentNameVersion                     : 'Foo 3.1'
          AgentNameVersionMajor                : 'Foo 3'

The output also contains all hard checked paths in the tree in thr rough form
of a matcher.

    - matcher:
    #    options:
    #    - 'verbose'
        require:
    #    - '__SyntaxError__="false"'
    #    - 'agent="Mozilla/5.0 (compatible; Foo/3.1; Bar)"'
    #    - 'agent.(1)product="Mozilla/5.0 (compatible; Foo/3.1; Bar)"'
    #    - 'agent.(1)product.(1)name="Mozilla"'
    #    - 'agent.(1)product.(1)name#1="Mozilla"'
    #    - 'agent.(1)product.(1)name%1="Mozilla"'
    #    - 'agent.(1)product.(1)version="5.0"'
    #    - 'agent.(1)product.(1)version#1="5"'
    #    - 'agent.(1)product.(1)version%1="5"'
    #    - 'agent.(1)product.(1)version#2="5.0"'
    #    - 'agent.(1)product.(1)version%2="0"'
    #    - 'agent.(1)product.(1)comments="(compatible; Foo/3.1; Bar)"'
    #    - 'agent.(1)product.(1)comments.(1)entry="compatible"'
    #    - 'agent.(1)product.(1)comments.(1)entry#1="compatible"'
    #    - 'agent.(1)product.(1)comments.(1)entry%1="compatible"'
    #    - 'agent.(1)product.(1)comments.(1)entry.(1)text="compatible"'
    #    - 'agent.(1)product.(1)comments.(1)entry.(1)text#1="compatible"'
    #    - 'agent.(1)product.(1)comments.(1)entry.(1)text%1="compatible"'
    #    - 'agent.(1)product.(1)comments.(2)entry="Foo/3.1"'
    #    - 'agent.(1)product.(1)comments.(2)entry#1="Foo"'
    #    - 'agent.(1)product.(1)comments.(2)entry%1="Foo"'
    #    - 'agent.(1)product.(1)comments.(2)entry#2="Foo/3"'
    #    - 'agent.(1)product.(1)comments.(2)entry%2="3"'
    #    - 'agent.(1)product.(1)comments.(2)entry#3="Foo/3.1"'
    #    - 'agent.(1)product.(1)comments.(2)entry%3="1"'
    #    - 'agent.(1)product.(1)comments.(2)entry.(1)product="Foo/3.1"'
    #    - 'agent.(1)product.(1)comments.(2)entry.(1)product.(1)name="Foo"'
    #    - 'agent.(1)product.(1)comments.(2)entry.(1)product.(1)name#1="Foo"'
    #    - 'agent.(1)product.(1)comments.(2)entry.(1)product.(1)name%1="Foo"'
    #    - 'agent.(1)product.(1)comments.(2)entry.(1)product.(1)version="3.1"'
    #    - 'agent.(1)product.(1)comments.(2)entry.(1)product.(1)version#1="3"'
    #    - 'agent.(1)product.(1)comments.(2)entry.(1)product.(1)version%1="3"'
    #    - 'agent.(1)product.(1)comments.(2)entry.(1)product.(1)version#2="3.1"'
    #    - 'agent.(1)product.(1)comments.(2)entry.(1)product.(1)version%2="1"'
    #    - 'agent.(1)product.(1)comments.(3)entry="Bar"'
    #    - 'agent.(1)product.(1)comments.(3)entry#1="Bar"'
    #    - 'agent.(1)product.(1)comments.(3)entry%1="Bar"'
    #    - 'agent.(1)product.(1)comments.(3)entry.(1)text="Bar"'
    #    - 'agent.(1)product.(1)comments.(3)entry.(1)text#1="Bar"'
    #    - 'agent.(1)product.(1)comments.(3)entry.(1)text%1="Bar"'
        extract:
    #    - 'DeviceClass           :   1:'
    #    - 'DeviceBrand           :   1:'
    #    - 'DeviceName            :   1:'
    #    - 'OperatingSystemClass  :   1:'
    #    - 'OperatingSystemName   :   1:'
    #    - 'OperatingSystemVersion:   1:'
    #    - 'LayoutEngineClass     :   1:'
    #    - 'LayoutEngineName      :   1:'
    #    - 'LayoutEngineVersion   :   1:'
    #    - 'AgentClass            :   1:'
    #    - 'AgentName             :   1:'
    #    - 'AgentVersion          :   1:'

In our case we are only interested in the second digit of the version of the product named "Foo"

So we copy this and make it like this:

    - matcher:
        extract:
        - 'MinorFooVersion :   1:agent.(1)product.(1)comments.entry.(1)product.(1)name="Foo"^.version%2'

What this does is that in the first product, in the first set of comments there is at any position an entry
that contains as the first element a product who's name is "Foo" that we go up in the tree and then down
to the version of that product and then take the second word.

Now if we run the unit test again we will see this:

    - test:
    #    options:
    #    - 'verbose'
    #    - 'init'
    #    - 'only'
        input:
          user_agent_string: 'Mozilla/5.0 (compatible; Foo/3.1; Bar)'
        expected:
          DeviceClass                          : 'Unknown'
          DeviceName                           : 'Unknown'
          OperatingSystemClass                 : 'Unknown'
          OperatingSystemName                  : 'Unknown'
          OperatingSystemVersion               : '??'
          LayoutEngineClass                    : 'Browser'
          LayoutEngineName                     : 'Mozilla'
          LayoutEngineVersion                  : '5.0'
          LayoutEngineVersionMajor             : '5'
          LayoutEngineNameVersion              : 'Mozilla 5.0'
          LayoutEngineNameVersionMajor         : 'Mozilla 5'
          AgentClass                           : 'Browser'
          AgentName                            : 'Foo'
          AgentVersion                         : '3.1'
          AgentVersionMajor                    : '3'
          AgentNameVersion                     : 'Foo 3.1'
          AgentNameVersionMajor                : 'Foo 3'
          MinorFooVersion                      : '1'

As you can see this clearly shows that a new field has been added with the value we wanted.

If you want to change or add more rules then simply iterate over the steps until you have the
end result you think is good.
Then simply copy the end 'test' into the yaml file and save.

If you are detecting a new field or a new value for an existing field
then be aware that this may also apply to tests that are already present
in the system. For all tests to pass these must be updated also.

You can choose to use this in a more "Test Driven Development" model also.
Simply run the unit test, copy the 'raw testcase' into the yaml file and
edit the values to what they should be and work from there.

If you run it this way your will get a 'failed test' which yields a bit
of extra output.
For the sake of demo if I simply put this testcase in the yaml file

    - test:
        input:
          user_agent_string: 'Mozilla/5.0 (compatible; Foo/3.1; Bar)'
        expected:
          AgentClass                           : 'Browser'
          AgentName                            : 'Foo'
          AgentVersion                         : '3.1'
          AgentVersionMajor                    : '3'
          AgentNameVersion                     : 'Foo 3.1'
          AgentNameVersionMajor                : 'Foo 3'
          MinorFooVersion                      : '1'

This would also show as test output:

    INFO  UserAgentAnalyzerTester:320 - +--------+------------------------------+-------------+------------+------------+
    INFO  UserAgentAnalyzerTester:337 - | Result | Field                        | Actual      | Confidence | Expected   |
    INFO  UserAgentAnalyzerTester:339 - +--------+------------------------------+-------------+------------+------------+
    ERROR UserAgentAnalyzerTester:381 - | -FAIL- | DeviceClass                  | Unknown     |         -1 | <<absent>> |
    ERROR UserAgentAnalyzerTester:381 - | -FAIL- | DeviceName                   | Unknown     |         -1 | <<absent>> |
    ERROR UserAgentAnalyzerTester:381 - | -FAIL- | OperatingSystemClass         | Unknown     |         -1 | <<absent>> |
    ERROR UserAgentAnalyzerTester:381 - | -FAIL- | OperatingSystemName          | Unknown     |         -1 | <<absent>> |
    ERROR UserAgentAnalyzerTester:381 - | -FAIL- | OperatingSystemVersion       | ??          |         -1 | <<absent>> |
    ERROR UserAgentAnalyzerTester:381 - | -FAIL- | LayoutEngineClass            | Browser     |          3 | <<absent>> |
    ERROR UserAgentAnalyzerTester:381 - | -FAIL- | LayoutEngineName             | Mozilla     |          3 | <<absent>> |
    ERROR UserAgentAnalyzerTester:381 - | -FAIL- | LayoutEngineVersion          | 5.0         |          3 | <<absent>> |
    ERROR UserAgentAnalyzerTester:381 - | -FAIL- | LayoutEngineVersionMajor     | 5           |          3 | <<absent>> |
    ERROR UserAgentAnalyzerTester:381 - | -FAIL- | LayoutEngineNameVersion      | Mozilla 5.0 |          3 | <<absent>> |
    ERROR UserAgentAnalyzerTester:381 - | -FAIL- | LayoutEngineNameVersionMajor | Mozilla 5   |          3 | <<absent>> |
    INFO  UserAgentAnalyzerTester:371 - |        | AgentClass                   | Browser     |         10 |            |
    INFO  UserAgentAnalyzerTester:371 - |        | AgentName                    | Foo         |         10 |            |
    INFO  UserAgentAnalyzerTester:371 - |        | AgentVersion                 | 3.1         |         10 |            |
    INFO  UserAgentAnalyzerTester:371 - |        | AgentVersionMajor            | 3           |         10 |            |
    INFO  UserAgentAnalyzerTester:371 - |        | AgentNameVersion             | Foo 3.1     |         10 |            |
    INFO  UserAgentAnalyzerTester:371 - |        | AgentNameVersionMajor        | Foo 3       |         10 |            |
    ERROR UserAgentAnalyzerTester:381 - | -FAIL- | MinorFooVersion              | <<<null>>>  |          0 | 1          |
    INFO  UserAgentAnalyzerTester:386 - +--------+------------------------------+-------------+------------+------------+

As you can see the system will fail on missing fields, unexpected fields and wrong values.

Performance considerations
==========================
**Todo**
Explain hashmap and = is fast, lookup and walking is slow.

Where the rules are located
===========================
Under the main resources there is a folder called *UserAgents* .
In the folder a collection of yaml files are located.
Each of those files contains matchers, lookups, tests or any combination of them
in any order. This means that in many cases you'll find a relevant test case close
to the rules.

The overall structure is this:

    config:
    - lookup:
      name: 'lookupname'
      map:
        "From1" : "To1"
        "From2" : "To2"
        "From3" : "To3"

    - matcher:
        options:
        - 'verbose'
        require:
        - 'Require pattern'
        - 'Require pattern'
        extract:
        - 'Extract pattern'
        - 'Extract pattern'

    - test:
        options:
        - 'verbose'
        - 'init'
        input:
          user_agent_string: 'Useragent'
          name: 'The name of the test'
        expected:
          FieldName     : 'ExpectedValue'
          FieldName     : 'ExpectedValue'
          FieldName     : 'ExpectedValue'



