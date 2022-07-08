---
title: "Robot Framework 2"
date: 2022-06-10T19:02:18+05:30
categories:
- robotframework
#- subcategory
tags:
- robotframework
# - tag2
keywords:
- tech
thumbnailImage: /images/rf_2.png
coverImage: /images/rf_2.png
---



It's been a while since my first [blog post](https://sohanrai09.github.io/blog/robot-framework/) as I mentioned there, I'm at a new job (well it's been 2 months, so 'newish') and I've been busy getting to know the job(as with any job, there is a LOT to learn!).

Learning [Robot Framework](https://robotframework.org/) (RF) has been one of my objectives in these first few months as that is entirely new for me. As I continue to learn and explore RF, I wanted to take some time out to share few things I have learnt. When it comes to network testing, it's not always about testing features, protocols, it also involves testing the hardware components. In this blog post, I want to discuss one such test scenario. This particular Test scenario would be to restart a FPC numerous times and to check and ensure it comes back up to Online state. This is what we call a `Negative Trigger Event` Testing. Now imagine having to do this manually, needless to say it's a time consuming task, this is where RF comes into play.

In this post, I will only explain anything new I'm using in RF as compared to my first [blog post](https://sohanrai09.github.io/blog/robot-framework/).

### varibales.yaml
In this post, I will be using a yaml file to input the variables rather than encoding them directly in the Test Suite file. We define the variables in a yaml file, like shown below, and access them as dictionaries in the Test Suite file.

```
VARS:
  host: 192.168.1.1  #Device under testing
  fpc: 3  #FPC to reboot
```

With this variable file, we can access the `host` value as `${VARS.host}` and similarly `fpc` value as `${VARS.fpc}`. 

### utilities.py
As mentioned in my previous post, any `Keywords` needed for Test execution will be defined in a Python script as `functions`. For this Test Case, I have defined two functions, one to check and return the state of the FPC and another one to trigger the FPC restart event, see below.

```
from jnpr.junos import Device

def fpc_restart(host,fpc):
    '''
    Function to restart the FPC
    '''
    dev = Device(host=host, user='user', password='password')
    dev.open()
    dev.rpc.request_chassis_fpc(slot=str(fpc),restart=True)

def fpc_state(host,fpc):
    '''
    Function to check the FPC status
    '''
    dev = Device(host=host, user='user', password='password')
    dev.open()
    fpc_state_cli = dev.rpc.get_fpc_information().findtext(f"./fpc[slot='{fpc}']/state")
    return fpc_state_cli
```

I'm using Juniper's PyEz library here as the RPC based options it provides are quite handy than CLI scraping. If you are new to PyEz, you can check out my [Github repo](https://github.com/sohanrai09/my_PyEz) where I've explained how to get started with PyEz and you can also find few basic scripts.

### Test Suite
Now let's look at the Test Suite itself, bit by bit. First, the `Settings` section. Here we define the Library file, which is where we have our Keywords defined and also Variable file, which is where we have the Variables in yaml.

```
*** Settings ***
Library    utilities.py
Variables    variables.yaml

```

Next we look at the Test Case

```
*** Test Cases ***
TC - 1 Restart FPC
    [Documentation]    Test Case to reboot the given FPC
    FOR  ${i}    IN RANGE    2  # i = 0, 1
        Log To Console    \n~~~ Checking FPC State ~~~
        ${fpc_state} =    FPC State   ${VARS.host}    ${VARS.fpc}
        
        Run Keyword If    '${fpc_state}' == 'Online'    Log To Console    \n~~~ FPC ${VARS.fpc} is Online\nProceeding with restart ~~~
        ...    ELSE    Fail    \n~~~ FPC ${VARS.fpc} is ${fpc_state},please recheck ~~~
        Run Keyword If    '${fpc_state}' == 'Online'    FPC Restart    ${VARS.host}     ${VARS.fpc}
        Sleep    300s

        Log To Console    \n~~~ Checking FPC State after restart ~~~
        ${fpc_state} =    FPC State    ${VARS.host}     ${VARS.fpc}
        Run Keyword If    '${fpc_state}' == 'Online'    Log To Console    \n~~~ FPC is back Online! ~~~
        Log To Console    \n~~~ Iteration ${i} completed ~~~
    END
```

There seems to be a LOT going on here but if you look at it closely, there actually isn't. Going past the `Documentation` field, we see a FOR LOOP. To anyone familiar with programming, this is nothing new, all you need to worry about is the syntax. I always refer to the official [user guide](https://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html#) to understand the syntax and usage. In our Test Case, number of iterations are 2, which means we will be 'restarting' the given FPC 2 times.

`Log To Console` is a builtin Keyword RF offers, which as it says is to print something to the console. Again, I refer to the official [documentation](https://robotframework.org/robotframework/latest/libraries/BuiltIn.html#) for this, here you can explore various builtin Keywords available for you to work with.

Next, we see our first Keyword (user defined) coming into play.
```
${fpc_state} =    FPC State   ${VARS.host}    ${VARS.fpc}

Run Keyword If    '${fpc_state}' == 'Online'    Log To Console    \n~~~ FPC ${VARS.fpc} is Online\nProceeding with restart ~~~
...    ELSE    Fail    \n~~~ FPC ${VARS.fpc} is ${fpc_state},please recheck ~~~
```
Here we are invoking a Keyword `FPC State`, which is nothing but the `fpc_state(host,fpc)` function we defined in the utilities.py file, and passing in the variables `${VARS.host}` and `${VARS.fpc}`. 'return' value from the function/Keyword is then stored in a variable `${fpc_state}`.

Then we see another inbuilt Keyword, `Run Keyword If`, which as it says, would run a Keyword if a particular condition is met. Condition in our case is if the output returned from the FPC State function is Online or not. If our condition is met (or in other words 'True') then we execute a Keyword `Log To Console`, which as explained before would log a message that the FPC we are checking is Online and that we are proceeding with restarting it. But if our condition FAILs, we have a ELSE clause to take care of that. If the ELSE clause is hit, we will first and foremast FAIL our Test Case, that is why we have used another builtin Keyword `FAIL` and we can also print a message to Console as to why we are failing the Test Case, neat isn't it? Note, if you are wondering what the `...` signifies, it is used when you want to continue the code on next line, so as to not over extend the same line.

Continuing further, we are now invoking our next function/Keyword but only when our condition is met! Condition is still the same one we looked at above!
```
Run Keyword If    '${fpc_state}' == 'Online'    FPC Restart    ${VARS.host}     ${VARS.fpc}
Sleep    300s
```

Same 2 variables are passed to `FPC Restart` Keyword as well. `Sleep` as you would have guessed it, is a builtin Keyword to, there is no better way to put it, sleep :)

Next section is failry simple as we now check the FPC State again (after 300s remember) to ensure it has come back Online.
```
Log To Console    \n~~~ Checking FPC State after restart ~~~
${fpc_state} =    FPC State    ${VARS.host}     ${VARS.fpc}
Run Keyword If    '${fpc_state}' == 'Online'    Log To Console    \n~~~ FPC is back Online! ~~~
Log To Console    \n~~~ Iteration ${i} completed ~~~
```

The whole process will be repeated as a part of our `FOR LOOP`, in this case, twice.

### Execution
```
(personal_env) sohanr@sohanr-mbp:~/envs$ robot fpc_reboot.robot 
==============================================================================
Fpc Reboot                                                                    
==============================================================================
TC - 1 Restart FPC :: Test Case to reboot the given FPC               
~~~ Checking FPC State ~~~

~~~ FPC 3 is Online
Proceeding with restart ~~~

~~~ Checking FPC State after restart ~~~

~~~ FPC is back Online! ~~~

~~~ Iteration 0 completed ~~~

~~~ Checking FPC State ~~~

~~~ FPC 3 is Online
Proceeding with restart ~~~

~~~ Checking FPC State after restart ~~~

~~~ FPC is back Online! ~~~

~~~ Iteration 1 completed ~~~
TC - 1 Restart FPC :: Test Case to reboot the given FPC               | PASS |
------------------------------------------------------------------------------
Fpc Reboot                                                            | PASS |
1 test, 1 passed, 0 failed
==============================================================================
Output:  /homes/sohanr/envs/output.xml
Log:     /homes/sohanr/envs/log.html
Report:  /homes/sohanr/envs/report.html
(personal_env) sohanr@sohanr-mbp:~/envs$ 
```

I'm hoping the output is self explaintory, as always with RF, we get the Log and Report in html format which makes viewing it really easy.

Also to just show a `FAIL` scenario, I will provide a FPC slot which is not in use and let's see the output for it.
```
(personal_env) sohanr@sohanr-mbp:~/envs$ robot fpc_reboot.robot 
==============================================================================
Fpc Reboot                                                                    
==============================================================================
TC - 1 Restart FPC :: Test Case to reboot the given FPC               
~~~ Checking FPC State ~~~
TC - 1 Restart FPC :: Test Case to reboot the given FPC               | FAIL |
~~~ FPC 2 is Empty,please recheck ~~~
------------------------------------------------------------------------------
Fpc Reboot                                                            | FAIL |
1 test, 0 passed, 1 failed
==============================================================================
Output:  /homes/sohanr/envs/output.xml
Log:     /homes/sohanr/envs/log.html
Report:  /homes/sohanr/envs/report.html
(personal_env) sohanr@sohanr-mbp:~/envs$ 
```

### Conclusion
As you might have seen with this post and the previous one, working with Robot Framework is fairly easy as the Keywords are in plain English and they do exactly what they say! I hope this blog post has been informative, as always please feel to reach out to me with any feedback/comments.


