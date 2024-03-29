---
layout: post
title: "Robot Framework Syntax Cheat Sheet"
---

This post serves as a quick-reference guide to various Robot Framework syntax elements. It also includes outcome-based examples of how to accomplish common tasks in modern Robot Framework syntax.

The overwhelming majority of this cheat sheet was originally created by [Robocorp](https://robocorp.com/), who develop tooling for customers that help automate IT and business processes. The original cheat sheet was located [here](https://robocorp.com/docs/languages-and-frameworks/robot-framework/cheat-sheet), but has since been removed as Robocorp is refocusing their efforts on Python-based automation instead of Robot Framework. Nonetheless, the cheat sheet was an invaluable resource for Robot Framework users, so I wanted to preserve it here for posterity.

```robot
*** Settings ***
Documentation       Robot Framework 5 syntax recipes cheat sheet robot.
...                 Demonstrates Robot Framework syntax in a concise format.

Library             MyLibrary
Library             MyLibrary    WITH NAME    HelloLibrary
Library             MyLibrary    greeting=Howdy!    WITH NAME    HowdyLibrary
Resource            keywords.robot
Variables           variables.py

Suite Setup         Log    Suite Setup!
Suite Teardown      Log    Suite Teardown!
Task Setup          Log    Task Setup!
Task Teardown       Log    Task Teardown!
Task Timeout        2 minutes


*** Variables ***
${STRING}=                  cat
${NUMBER}=                  ${1}
@{LIST}=                    one    two    three
&{DICTIONARY}=              string=${STRING}    number=${NUMBER}    list=@{LIST}
${ENVIRONMENT_VARIABLE}=    %{PATH=Default value}


*** Tasks ***
Call keywords with a varying number of arguments
    A keyword without arguments
    A keyword with a required argument    Argument
    A keyword with a required argument    argument=Argument
    A keyword with an optional argument
    A keyword with an optional argument    Argument
    A keyword with an optional argument    argument=Argument
    A keyword with any number of arguments
    A keyword with any number of arguments    arg1    arg2    arg3    arg4    arg5
    A keyword with one or more arguments    arg1
    A keyword with one or more arguments    arg1    arg2    arg3

Call a keyword that returns a value
    ${value}=    A keyword that returns a value
    Log    ${value}    # Return value

Do conditional IF - ELSE IF - ELSE execution
    IF    ${NUMBER} > 1
        Log    Greater than one.
    ELSE IF    "${STRING}" == "dog"
        Log    It's a dog!
    ELSE
        Log    Probably a cat.
    END

Loop a list
    Log    ${LIST}    # ['one', 'two', 'three']
    FOR    ${item}    IN    @{LIST}
        Log    ${item}    # one, two, three
    END
    FOR    ${item}    IN    one    two    three
        Log    ${item}    # one, two, three
    END

Loop a dictionary
    Log    ${DICTIONARY}
    # {'string': 'cat', 'number': 1, 'list': ['one', 'two', 'three']}
    FOR    ${key_value_tuple}    IN    &{DICTIONARY}
        Log    ${key_value_tuple}
        # ('string', 'cat'), ('number', 1), ('list', ['one', 'two', 'three'])
    END
    FOR    ${key}    IN    @{DICTIONARY}
        Log    ${key}=${DICTIONARY}[${key}]
        # string=cat, number=1, list=['one', 'two', 'three']
    END

Loop a range from 0 to end index
    FOR    ${index}    IN RANGE    10
        Log    ${index}    # 0-9
    END

Loop a range from start to end index
    FOR    ${index}    IN RANGE    1    10
        Log    ${index}    # 1-9
    END

Loop a range from start to end index with steps
    FOR    ${index}    IN RANGE    0    10    2
        Log    ${index}    # 0, 2, 4, 6, 8
    END

Nest loops
    @{alphabets}=    Create List    a    b    c
    Log    ${alphabets}    # ['a', 'b', 'c']
    @{numbers}=    Create List    ${1}    ${2}    ${3}
    Log    ${numbers}    # [1, 2, 3]
    FOR    ${alphabet}    IN    @{alphabets}
        FOR    ${number}    IN    @{numbers}
            Log    ${alphabet}${number}
            # a1, a2, a3, b1, b2, b3, c1, c2, c3
        END
    END

Exit a loop on condition
    FOR    ${i}    IN RANGE    5
        IF    ${i} == 2    BREAK
        Log    ${i}    # 0, 1
    END

Continue a loop from the next iteration on condition
    FOR    ${i}    IN RANGE    3
        IF    ${i} == 1    CONTINUE
        Log    ${i}    # 0, 2
    END

Create a scalar variable
    ${animal}=    Set Variable    dog
    Log    ${animal}    # dog
    Log    ${animal}[0]    # d
    Log    ${animal}[-1]    # g

Create a number variable
    ${π}=    Set Variable    ${3.14}
    Log    ${π}    # 3.14

Create a list variable
    @{animals}=    Create List    dog    cat    bear
    Log    ${animals}    # ['dog', 'cat', 'bear']
    Log    ${animals}[0]    # dog
    Log    ${animals}[-1]    # bear

Create a dictionary variable
    &{dictionary}=    Create Dictionary    key1=value1    key2=value2
    Log    ${dictionary}    # {'key1': 'value1', 'key2': 'value2'}
    Log    ${dictionary}[key1]    # value1
    Log    ${dictionary.key2}    # value2

Access the items in a sequence (list, string)
    ${string}=    Set Variable    Hello world!
    Log    ${string}[0]    # H
    Log    ${string}[:5]    # Hello
    Log    ${string}[6:]    # world!
    Log    ${string}[-1]    # !
    @{list}=    Create List    one    two    three    four    five
    Log    ${list}    # ['one', 'two', 'three', 'four', 'five']
    Log    ${list}[0:6:2]    # ['one', 'three', 'five']

Call a custom Python library
    ${greeting}=    MyLibrary.Get Greeting
    Log    ${greeting}    # Hello!
    ${greeting}=    HelloLibrary.Get Greeting
    Log    ${greeting}    # Hello!
    ${greeting}=    HowdyLibrary.Get Greeting
    Log    ${greeting}    # Howdy!

Call a keyword from a separate resource file
    My keyword in a separate resource file

Access a variable in a separate variable file
    Log    ${MY_VARIABLE_FROM_A_SEPARATE_VARIABLE_FILE}

Split arguments to multiple lines
    A keyword with any number of arguments
    ...    arg1
    ...    arg2
    ...    arg3

Log available variables
    Log Variables
    # ${/} = /
    # &{DICTIONARY} = { string=cat | number=1 | list=['one', 'two', 'three'] }
    # ${OUTPUT_DIR} = /Users/<username>/...
    # ...

Evaluate Python expressions
    ${path}=    Evaluate    os.environ.get("PATH")
    ${path}=    Set Variable    ${{os.environ.get("PATH")}}

Use special variables
    Log    ${EMPTY}    # Like the ${SPACE}, but without the space.
    Log    ${False}    # Boolean False.
    Log    ${None}    # Python None
    Log    ${null}    # Java null.
    Log    ${SPACE}    # ASCII space (\x20).
    Log    ${SPACE * 4}    # Four spaces.
    Log    "${SPACE}"    # Quoted space (" ").
    Log    ${True}    # Boolean True.

TRY / EXCEPT: Catch any exception
    TRY
        Fail
    EXCEPT
        Log    EXCEPT with no arguments catches any exception.
    END

TRY / EXCEPT: Catch an exception by exact message
    TRY
        Fail    Error message
    EXCEPT    Error message
        Log    Catches only "Error message" exceptions.
        Log    Enables error-specific exception handling.
    END

TRY / EXCEPT: Multiple EXCEPT statements
    TRY
        Fail    Error message
    EXCEPT    Another error message
        Log    Catches only "Another error message" exceptions.
    EXCEPT    Error message
        Log    Catches the "Error message" exception.
    END

TRY / EXCEPT: Multiple messages in EXCEPT statement
    TRY
        Fail    CCC
    EXCEPT    AAA    BBB    CCC
        Log    Catches any "AAA", "BBB", or "CCC" exception.
    END

TRY / EXCEPT: Catch a specific exception, or an unexpected exception
    TRY
        Fail    Error message
    EXCEPT    Another message
        Log    Catches only "Another message" exceptions.
    EXCEPT
        Log    Catches any exception.
        Log    Useful for handling unexpected exceptions.
    END

TRY / EXCEPT: Catch exceptions where the message starts with
    TRY
        Fail    A long error message with lots of details
    EXCEPT    A long error message    type=start
        Log    Matches the start of an error message.
    END

TRY / EXCEPT: Capture the error message
    TRY
        Fail    Goodbye, world!
    EXCEPT    AS    ${error_message}
        Log    ${error_message}    # Goodbye, world!
    END

TRY / EXCEPT: Using ELSE when no exceptions occured
    TRY
        Log    All good!
    EXCEPT    Error message
        Log    An error occured.
    ELSE
        Log    No error occured.
    END

TRY / EXCEPT / FINALLY: Always execute code no matter if exceptions or not
    TRY
        Log    All good!
    FINALLY
        Log    FINALLY is always executed.
    END
    TRY
        Fail    Catastrophic failure!
    EXCEPT
        Log    Catches any exception.
    FINALLY
        Log    FINALLY is always executed.
    END

TRY / EXCEPT / ELSE / FINALLY: All together!
    TRY
        Fail    Error message
    EXCEPT
        Log    Executed if any exception occurs.
    ELSE
        Log    Executed if no exceptions occur.
    FINALLY
        Log    FINALLY is always executed.
    END

TRY / EXCEPT: Glob pattern matching
    TRY
        Fail    My error: 99 occured
    EXCEPT    My error: *    type=glob
        Log    Catches by glob pattern matching.
    END

TRY / EXCEPT: Regular expression matching
    TRY
        Fail    error 99 occured
    EXCEPT    [Ee]rror \\d+ occured    type=regexp
        Log    Catches by regular expression pattern matching.
    END

WHILE: Loop while the default limit (10000) is hit
    TRY
        WHILE    True
            Log    Executed until the default loop limit (10000) is hit.
        END
    EXCEPT    WHILE loop was aborted    type=start
        Log    The loop did not finish within the limit.
    END

WHILE: Loop while the given limit is hit
    TRY
        WHILE    True    limit=10
            Log    Executed until the given loop limit (10) is hit.
        END
    EXCEPT    WHILE loop was aborted    type=start
        Log    The loop did not finish within the limit.
    END

WHILE: Loop while condition evaluates to True
    ${x}=    Set Variable    ${0}
    WHILE    ${x} < 3
        Log    Executed as long as the condition is True.
        ${x}=    Evaluate    ${x} + 1
    END

WHILE: Skip a loop iteration with CONTINUE
    ${x}=    Set Variable    ${0}
    WHILE    ${x} < 3
        ${x}=    Evaluate    ${x} + 1
        #    Skip this iteration.
        IF    ${x} == 2    CONTINUE
        Log    x = ${x}    # x = 1, x = 3
    END

WHILE: Exit loop with BREAK
    WHILE    True
        BREAK
        Log    This will not be logged.
    END


*** Keywords ***
A keyword without arguments
    Log    No arguments.

A keyword with a required argument
    [Arguments]    ${argument}
    Log    Required argument: ${argument}

A keyword with an optional argument
    [Arguments]    ${argument}=Default value
    Log    Optional argument: ${argument}

A keyword with any number of arguments
    [Arguments]    @{varargs}
    Log    Any number of arguments: @{varargs}

A keyword with one or more arguments
    [Arguments]    ${argument}    @{varargs}
    Log    One or more arguments: ${argument} @{varargs}

A keyword that returns a value
    RETURN    Return value

RETURN: Return a value from a keyword
    IF    True    RETURN    It is true!    ELSE    RETURN    It is not true!

RETURN: Return without a value
    IF    True    RETURN
    Log    This will not be logged.

A keyword with documentation
    [Documentation]    This is keyword documentation.
    No Operation
```

## How To Add Robot Documentation

```robot
*** Settings ***
Documentation       Robot Framework 5 syntax recipes cheat sheet robot.
...                 Demonstrates Robot Framework syntax in a concise format.
```

## How To Import Libraries

```robot
*** Settings ***
Library             MyLibrary
Library             MyLibrary    WITH NAME    HelloLibrary
Library             MyLibrary    greeting=Howdy!    WITH NAME    HowdyLibrary
```

## How To Import Resource Files, Such As Keywords

```robot
*** Settings ***
Resource            keywords.robot
```

## How To Run Suite And Task Setup And Teardown

```robot
*** Settings ***
Suite Setup         Log    Suite Setup!
Suite Teardown      Log    Suite Teardown!
Task Setup          Log    Task Setup!
Task Teardown       Log    Task Teardown!
```

## How To Set Task Timeout

```robot
*** Settings ***
Task Timeout        2 minutes
```

## How To Import Variable Files

```robot
*** Settings ***
Variables           variables.py
```

## How To Define Suite Variables

```robot
*** Variables ***
${STRING}=                  cat
${NUMBER}=                  ${1}
@{LIST}=                    one    two    three
&{DICTIONARY}=              string=${STRING}    number=${NUMBER}    list=@{LIST}
${ENVIRONMENT_VARIABLE}=    %{PATH=Default value}
```

## How To Implement A Keyword Without Arguments

```robot
*** Keywords ***
A keyword without arguments
    Log    No arguments.
```

## How To Implement A Keyword With A Required Argument

```robot
*** Keywords ***
A keyword with a required argument
    [Arguments]    ${argument}
    Log    Required argument: ${argument}
```

## How To Implement A Keyword With An Optional Argument

```robot
*** Keywords ***
A keyword with an optional argument
    [Arguments]    ${argument}=Default value
    Log    Optional argument: ${argument}
```

## How To Implement A Keyword With Any Number Of Arguments

```robot
*** Keywords ***
A keyword with any number of arguments
    [Arguments]    @{varargs}
    Log    Any number of arguments: @{varargs}
```

## How To Implement A Keyword With One Or More Arguments

```robot
*** Keywords ***
A keyword with one or more arguments
    [Arguments]    ${argument}    @{varargs}
    Log    One or more arguments: ${argument} @{varargs}
```

## How To Implement A Keyword That Returns A Value

```robot
*** Keywords ***
A keyword that returns a value
    RETURN    Return value
```

## How To Implement A Keyword That Returns A Value On Condition

```robot
*** Keywords ***
RETURN: Return a value from a keyword
    IF    True    RETURN    It is true!    ELSE    RETURN    It is not true!
```

## How To Implement A Keyword That Returns Without A Value

```robot
*** Keywords ***
RETURN: Return without a value
    IF    True    RETURN
    Log    This will not be logged.
```

## How To Add Keyword Documentation

```robot
*** Keywords ***
A keyword with documentation
    [Documentation]    This is keyword documentation.
    No Operation
```

## How To Call Keywords With A Varying Number Of Arguments

```robot
*** Tasks ***
Call keywords with a varying number of arguments
    A keyword without arguments
    A keyword with a required argument    Argument
    A keyword with a required argument    argument=Argument
    A keyword with an optional argument
    A keyword with an optional argument    Argument
    A keyword with an optional argument    argument=Argument
    A keyword with any number of arguments
    A keyword with any number of arguments    arg1    arg2    arg3    arg4    arg5
    A keyword with one or more arguments    arg1
    A keyword with one or more arguments    arg1    arg2    arg3
```

## How To Call A Keyword That Returns A Value

```robot
*** Tasks ***
Call a keyword that returns a value
    ${value}=    A keyword that returns a value
    Log    ${value}    # Return value
```

## How To Do Conditional IF - ELSE IF - ELSE Execution

```robot
*** Tasks ***
Do conditional IF - ELSE IF - ELSE execution
    IF    ${NUMBER} > 1
        Log    Greater than one.
    ELSE IF    "${STRING}" == "dog"
        Log    It's a dog!
    ELSE
        Log    Probably a cat. 🤔
    END
```

## How To Loop A List

```robot
*** Tasks ***
Loop a list
    Log    ${LIST}    # ['one', 'two', 'three']
    FOR    ${item}    IN    @{LIST}
        Log    ${item}    # one, two, three
    END
    FOR    ${item}    IN    one    two    three
        Log    ${item}    # one, two, three
    END
```

## How To Loop A Dictionary

```robot
*** Tasks ***
Loop a dictionary
    Log    ${DICTIONARY}
    # {'string': 'cat', 'number': 1, 'list': ['one', 'two', 'three']}
    FOR    ${key_value_tuple}    IN    &{DICTIONARY}
        Log    ${key_value_tuple}
        # ('string', 'cat'), ('number', 1), ('list', ['one', 'two', 'three'])
    END
    FOR    ${key}    IN    @{DICTIONARY}
        Log    ${key}=${DICTIONARY}[${key}]
        # string=cat, number=1, list=['one', 'two', 'three']
    END
```

## How To Loop A Range From 0 To End Index

```robot
*** Tasks ***
Loop a range from 0 to end index
    FOR    ${index}    IN RANGE    10
        Log    ${index}    # 0-9
    END
```

## How To Loop A Range From Start To End Index

```robot
*** Tasks ***
Loop a range from start to end index
    FOR    ${index}    IN RANGE    1    10
        Log    ${index}    # 1-9
    END
```

## How To Loop A Range From Start To End Index With Steps

```robot
*** Tasks ***
Loop a range from start to end index with steps
    FOR    ${index}    IN RANGE    0    10    2
        Log    ${index}    # 0, 2, 4, 6, 8
    END
```

## How To Nest Loops

```robot
*** Tasks ***
Nest loops
    @{alphabets}=    Create List    a    b    c
    Log    ${alphabets}    # ['a', 'b', 'c']
    @{numbers}=    Create List    ${1}    ${2}    ${3}
    Log    ${numbers}    # [1, 2, 3]
    FOR    ${alphabet}    IN    @{alphabets}
        FOR    ${number}    IN    @{numbers}
            Log    ${alphabet}${number}
            # a1, a2, a3, b1, b2, b3, c1, c2, c3
        END
    END
```

## How To Exit A Loop On Condition

```robot
*** Tasks ***
Exit a loop on condition
    FOR    ${i}    IN RANGE    5
        Exit For Loop If    ${i} == 2
        Log    ${i}    # 0, 1
    END
```

## How To Continue A Loop From The Next Iteration On Condition

```robot
*** Tasks ***
Continue a loop from the next iteration on condition
    FOR    ${i}    IN RANGE    3
        Continue For Loop If    ${i} == 1
        Log    ${i}    # 0, 2
    END
```

## How To Create A Scalar Variable

```robot
*** Tasks ***
Create a scalar variable
    ${animal}=    Set Variable    dog
    Log    ${animal}    # dog
    Log    ${animal}[0]    # d
    Log    ${animal}[-1]    # g
```

## How To Create A Number Variable

```robot
*** Tasks ***
Create a number variable
    ${π}=    Set Variable    ${3.14}
    Log    ${π}    # 3.14
```

## How To Create A List Variable

```robot
*** Tasks ***
Create a list variable
    @{animals}=    Create List    dog    cat    bear
    Log    ${animals}    # ['dog', 'cat', 'bear']
    Log    ${animals}[0]    # dog
    Log    ${animals}[-1]    # bear
```

## How To Create A Dictionary Variable

```robot
*** Tasks ***
Create a dictionary variable
    &{dictionary}=    Create Dictionary    key1=value1    key2=value2
    Log    ${dictionary}    # {'key1': 'value1', 'key2': 'value2'}
    Log    ${dictionary}[key1]    # value1
    Log    ${dictionary.key2}    # value2
```

## How To Access The Items In A Sequence (list, String)

```robot
*** Tasks ***
Access the items in a sequence (list, string)
    ${string}=    Set Variable    Hello world!
    Log    ${string}[0]    # H
    Log    ${string}[:5]    # Hello
    Log    ${string}[6:]    # world!
    Log    ${string}[-1]    # !
    @{list}=    Create List    one    two    three    four    five
    Log    ${list}    # ['one', 'two', 'three', 'four', 'five']
    Log    ${list}[0:6:2]    # ['one', 'three', 'five']
```

## How To Use TRY / EXCEPT / FINALLY

```robot
*** Tasks ***
TRY / EXCEPT: Catch any exception
    TRY
        Fail
    EXCEPT
        Log    EXCEPT with no arguments catches any exception.
    END

TRY / EXCEPT: Catch an exception by exact message
    TRY
        Fail    Error message
    EXCEPT    Error message
        Log    Catches only "Error message" exceptions.
        Log    Enables error-specific exception handling.
    END

TRY / EXCEPT: Multiple EXCEPT statements
    TRY
        Fail    Error message
    EXCEPT    Another error message
        Log    Catches only "Another error message" exceptions.
    EXCEPT    Error message
        Log    Catches the "Error message" exception.
    END

TRY / EXCEPT: Multiple messages in EXCEPT statement
    TRY
        Fail    CCC
    EXCEPT    AAA    BBB    CCC
        Log    Catches any "AAA", "BBB", or "CCC" exception.
    END

TRY / EXCEPT: Catch a specific exception, or an unexpected exception
    TRY
        Fail    Error message
    EXCEPT    Another message
        Log    Catches only "Another message" exceptions.
    EXCEPT
        Log    Catches any exception.
        Log    Useful for handling unexpected exceptions.
    END

TRY / EXCEPT: Catch exceptions where the message starts with
    TRY
        Fail    A long error message with lots of details
    EXCEPT    A long error message    type=start
        Log    Matches the start of an error message.
    END

TRY / EXCEPT: Capture the error message
    TRY
        Fail    Goodbye, world!
    EXCEPT    AS    ${error_message}
        Log    ${error_message}    # Goodbye, world!
    END

TRY / EXCEPT: Using ELSE when no exceptions occured
    TRY
        Log    All good!
    EXCEPT    Error message
        Log    An error occured.
    ELSE
        Log    No error occured.
    END

TRY / EXCEPT / FINALLY: Always execute code no matter if exceptions or not
    TRY
        Log    All good!
    FINALLY
        Log    FINALLY is always executed.
    END
    TRY
        Fail    Catastrophic failure!
    EXCEPT
        Log    Catches any exception.
    FINALLY
        Log    FINALLY is always executed.
    END

TRY / EXCEPT / ELSE / FINALLY: All together!
    TRY
        Fail    Error message
    EXCEPT
        Log    Executed if any exception occurs.
    ELSE
        Log    Executed if no exceptions occur.
    FINALLY
        Log    FINALLY is always executed.
    END

TRY / EXCEPT: Glob pattern matching
    TRY
        Fail    My error: 99 occured
    EXCEPT    My error: *    type=glob
        Log    Catches by glob pattern matching.
    END

TRY / EXCEPT: Regular expression matching
    TRY
        Fail    error 99 occured
    EXCEPT    [Ee]rror \\d+ occured    type=regexp
        Log    Catches by regular expression pattern matching.
    END
```

## How To Use WHILE Loops

```robot
*** Tasks ***
WHILE: Loop while the default limit (10000) is hit
    TRY
        WHILE    True
            Log    Executed until the default loop limit (10000) is hit.
        END
    EXCEPT    WHILE loop was aborted    type=start
        Log    The loop did not finish within the limit.
    END

WHILE: Loop while the given limit is hit
    TRY
        WHILE    True    limit=10
            Log    Executed until the given loop limit (10) is hit.
        END
    EXCEPT    WHILE loop was aborted    type=start
        Log    The loop did not finish within the limit.
    END

WHILE: Loop while condition evaluates to True
    ${x}=    Set Variable    ${0}
    WHILE    ${x} < 3
        Log    Executed as long as the condition is True.
        ${x}=    Evaluate    ${x} + 1
    END

WHILE: Skip a loop iteration with CONTINUE
    ${x}=    Set Variable    ${0}
    WHILE    ${x} < 3
        ${x}=    Evaluate    ${x} + 1
        #    Skip this iteration.
        IF    ${x} == 2    CONTINUE
        Log    x = ${x}    # x = 1, x = 3
    END

WHILE: Exit loop with BREAK
    WHILE    True
        BREAK
        Log    This will not be logged.
    END
```

## How To Call A Custom Python Library

```robot
MyLibrary.py file
class MyLibrary:

    def __init__(self, greeting="Hello!"):
        self._greeting = greeting

    def get_greeting(self):
        return self._greeting
Importing and calling the library
*** Settings ***
Library           MyLibrary
Library           MyLibrary    WITH NAME    HelloLibrary
Library           MyLibrary    greeting=Howdy!    WITH NAME    HowdyLibrary

*** Tasks ***
Call a custom Python library
    ${greeting}=    MyLibrary.Get Greeting
    Log    ${greeting}    # Hello!
    ${greeting}=    HelloLibrary.Get Greeting
    Log    ${greeting}    # Hello!
    ${greeting}=    HowdyLibrary.Get Greeting
    Log    ${greeting}    # Howdy!
```

## How To Call A Keyword From A Separate Resource File

```robot
keywords.robot
*** Settings ***
Documentation     Keywords in a separate resource file.

*** Keywords ***
My keyword in a separate resource file
    Log To Console    My keyword in a separate resource file!
Importing and calling the keyword
*** Settings ***
Resource          keywords.robot

*** Tasks ***
Call a keyword from a separate resource file
    My keyword in a separate resource file
```

## How To Access A Variable In A Separate Variable File

```robot
variables.py
MY_VARIABLE_FROM_A_SEPARATE_VARIABLE_FILE = "My variable from a separate variable file"
Importing and using the variable
*** Settings ***
Variables         variables.py

*** Tasks ***
Access a variable in a separate variable file
    Log    ${MY_VARIABLE_FROM_A_SEPARATE_VARIABLE_FILE}
```

## How To Split Arguments To Multiple Lines

```robot
*** Tasks ***
Split arguments to multiple lines
    A keyword with any number of arguments
    ...    arg1
    ...    arg2
    ...    arg3
```

## How To Log Available Variables

```robot
*** Tasks ***
Log available variables
    Log Variables
    # ${/} = /
    # &{DICTIONARY} = { string=cat | number=1 | list=['one', 'two', 'three'] }
    # ${OUTPUT_DIR} = /Users/<username>/...
    # ...
```

## How To Evaluate Python Expressions

```robot
*** Tasks ***
Evaluate Python expressions
    ${path}=    Evaluate    os.environ.get("PATH")
    ${path}=    Set Variable    ${{os.environ.get("PATH")}}
Learn more
```
