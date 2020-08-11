# Robot Framework Guide
This page contains a how-to guide for writing custom integration tests using the vpp-agent's libraries for the Robot framework.

---

!!! Attention
    Tests written with Robot Framework are no longer maintained!
    
    All our tests are now implemented in Go. Read [Integration Tests](https://docs.ligato.io/en/latest/testing/integration-tests/) for more info.

## Robot Test Setup

The following text describes how to write robot test suites (locally or in any other environment).

you need to create a test file in format:
```
../testname.robot
```
Example:
```
../tests/robot/suites/crud/acl_crud.robot
```
Resource setup in the next step is based on where you have created the file. 

### Setup Test

When you have the file, you need to define (libraries, resources, tags, variables).

- Base libraries used are "[OperatingSystem](http://robotframework.org/robotframework/latest/libraries/OperatingSystem.html)" and "[Collections](http://robotframework.org/robotframework/latest/libraries/Collections.html)". OperatingSystem - is for use with the files- create, delete, copy, check if exist file, ... . This is mostly used for configuration, make logs and other.
  Collections - Append To List, Combine Lists, Convert To Dictionary, Convert To List, Copy Dictionary, Copy List, Count Values In List, Dictionaries Should Be Equal, Should Contain Match, Should Not Contain Match, ... .
<br/>
 - Resources are defined in [our libraries](https://github.com/ligato/vpp-agent/tree/master/tests/robot/libraries). 
 - Tags are used to skip tests or mark them as non-critical. ["Robot Tags"](http://robotframework.org/robotframework/latest/RobotFrameworkUserGuide.html#tagging-test-cases)

Example:
```robot
Library      OperatingSystem
Library      Collections

Resource     ../../variables/${VARIABLES}_variables.robot
Resource     ../../libraries/all_libs.robot

Force Tags        crud     IPv4    #(ExpectedFailure, IPv6, traffic, misc, sfc)

Suite Setup       Testsuite Setup
Suite Teardown    Testsuite Teardown
Test Setup        TestSetup
Test Teardown     TestTeardown

*** Variables ***
...
```

If you need to add another library, then you add lib into -All libraries:
```
../vpp-agent/tests/robot/libraries/all_libs.robot
```

### Configuring the Test Environment

 - Base configuration:
```robot
Configure Environment
    [Tags]    setup
    Configure Environment 1
```
Setup nodes(docker) for the test. 

 - Configuration keyword is based on add and run VPP nodes:
```robot
Configure Environment 1
    Add Agent VPP Node    agent_vpp_1
    Add Agent VPP Node    agent_vpp_2
    Execute In Container    agent_vpp_1    echo $MICROSERVICE_LABEL
    Execute In Container    agent_vpp_2    echo $MICROSERVICE_LABEL
    Execute In Container    agent_vpp_1    ls -al
    Execute On Machine    docker    ${DOCKER_COMMAND} images
    Execute On Machine    docker    ${DOCKER_COMMAND} ps -as
```
 - Common Configurations:
```
https://github.com/ligato/vpp-agent/blob/master/tests/robot/libraries/configurations.robot
```

 - Specific configuration example:	
```robot
Configure Environment
    [Tags]    setup
    ${DATA_FOLDER}=       Catenate     SEPARATOR=/       ${CURDIR}         ${TEST_DATA_FOLDER}
    Set Suite Variable          ${DATA_FOLDER}
    Configure Environment 2        acl_basic.conf
```

### Test Setup and Teardown

 - Commonly used test is "make snapshots etcd". This is required for optimal test debugging. 

```robot
*** Keywords ***
TestSetup
    Make Datastore Snapshots    ${TEST_NAME}_test_setup

TestTeardown
    Make Datastore Snapshots    ${TEST_NAME}_test_teardown
```

### Test body

Test cases use keywords from our libraries. After configuration, setup and teardown, you can start with testing.
Before starting the test, it is good practice to check whether there is any configuration remaining from previous runs.
```robot
Show Something Before Setup
    ${interfaces}=    vpp_term: Show Interfaces    agent_vpp_1
```
This shows only the configuration. You can use keywords comparing output, or check if the output is empty.
The next step is to configure something (an example interface):
```robot
Add Something
    vpp_term: Interface Not Exists  node=agent_vpp_1    mac=${MAC_TAP1}
    Put TAP Interface With IP    node=agent_vpp_1    name=${NAME_TAP1}    mac=${MAC_TAP1}    ip=${IP_TAP1}    prefix=${PREFIX}    host_if_name=linux_${NAME_TAP1}
```

Every configuration must be checked. In some cases, a timeout is needed for that case, use the `Wait Until` keyword):
```robot
Check Something Is Created
    ${interfaces}=       vat_term: Interfaces Dump    node=agent_vpp_1
    Wait Until Keyword Succeeds   ${WAIT_TIMEOUT}   ${SYNC_SLEEP}    vpp_term: Interface Is Created    node=agent_vpp_1    mac=${MAC_TAP1}
    ${actual_state}=    vpp_term: Check TAP interface State    agent_vpp_1    ${NAME_TAP1}    mac=${MAC_TAP1}    ipv4=${IP_TAP1}/${PREFIX}    state=${UP_STATE}
```

It is alway useful to add a similar configuration and verify if it has been deleted. 
You can delete "No Operation" and use the same configuration from "Add Something" and "Check Something Is Created". You need only change the interface name, mac, ip and so on:
```robot
Add Something_Other
    No Operation

Check Something_Other Is Created
    No Operation
```

After this, you need to check if the first item (i.e. interface) is still configured and error free:
```robot
Check Something Is Still Configured
    ${actual_state}=    vpp_term: Check TAP interface State    agent_vpp_1    ${NAME_TAP1}    mac=${MAC_TAP1}    ipv4=${IP_TAP1}/${PREFIX}    state=${UP_STATE}
```
In the CRUD, the test should attempt to update and validate the configured data: 
```robot
Update Something
    Put TAP Interface With IP    node=agent_vpp_1    name=${NAME_TAP1}    mac=${MAC_TAP1_2}    ip=${IP_TAP1_2}    prefix=${PREFIX}    host_if_name=linux_${NAME_TAP1}

Check Something Is Created
    Wait Until Keyword Succeeds   ${WAIT_TIMEOUT}   ${SYNC_SLEEP}    vpp_term: Interface Is Created    node=agent_vpp_1    mac=${MAC_TAP1_2}
    ${actual_state}=    vpp_term: Check TAP interface State    agent_vpp_1    ${NAME_TAP1}    mac=${MAC_TAP1_2}    ipv4=${IP_TAP1_2}/${PREFIX}    state=${UP_STATE}
```

Also need:
```robot
Check Something_Other Has Not Changed
    No Operation
```
Try to delete and then check the configuration:
```robot
Delete Something
    Delete VPP Interface    agent_vpp_1    ${NAME_TAP1}

Check Something Has Been Deleted
    Wait Until Keyword Succeeds   ${WAIT_TIMEOUT}   ${SYNC_SLEEP}    vpp_term: Interface Not Exists  node=agent_vpp_1    mac=${MAC_TAP1_2}
```
After delete, check another configuration for changes (there should not be any):
```robot
Check Something_Other Is Still Configured
    No Operation
```

The last part can use your own image. An example: 
```robot
Show Interfaces And Other Objects After Setup
    vpp_term: Show Interfaces    agent_vpp_1
    Write To Machine    agent_vpp_1_term    show int addr
    Write To Machine    agent_vpp_1_term    show h
    Write To Machine    agent_vpp_1_term    show br
    Write To Machine    agent_vpp_1_term    show br 1 detail
    Write To Machine    agent_vpp_1_term    show vxlan tunnel
    Write To Machine    agent_vpp_1_term    show err
    vat_term: Interfaces Dump    agent_vpp_1
    Execute In Container    agent_vpp_1    ip a
```

## Reference Guide

In this section are examples of keywords, short descriptions and libraries which can be useful for tests.

### Docker container keywords
 - Add Agent Node, Add Agent VPP Node, Execute In Container, Write Command to Container, ...,
 are here:
```
https://github.com/ligato/vpp-agent/blob/master/tests/robot/libraries/docker.robot
```
### Read/Write ETCD
 - Get, Put, keywords for read, write into etcd:
```
https://github.com/ligato/vpp-agent/blob/master/tests/robot/libraries/etcdctl.robot
```
### Linux Interfaces and Commands 
 - If you will be using Linux interfaces and commands inside Linux for testing purposes you need this library:
```
https://github.com/ligato/vpp-agent/blob/master/tests/robot/libraries/linux.robot
```
Inside are keywords for create, delete or read interfaces and interface status. Also, there the keywords for ping tests(traffic tests).

### Create, Delete, Update
 - Most keywords for create, delete, update, show interfaces and routes in vpp-gent are here:
```
https://github.com/ligato/vpp-agent/blob/master/tests/robot/libraries/pretty_keywords.robot
```
### SSH Connection, Test Setup/Teardown
 - Keyword used for Open SSH Connection, Testsuite Setup, Test Setup, Testsuite Teardown, Test Teardown, Logs, Datastore, Dump, Snapshots:
```
https://github.com/ligato/vpp-agent/blob/master/tests/robot/libraries/setup-teardown.robot
```
### SSH usage
 - SSH - Execute On Machine, Write To Machine:
```
https://github.com/ligato/vpp-agent/blob/master/tests/robot/libraries/ssh.robot
```
### Read/Write VAT terminal
 - VAT terminal keywords are used to check status for interfaces, bridge domains, ... in docker. You read ("Write To Machine") from docker.
```
https://github.com/ligato/vpp-agent/blob/master/tests/robot/libraries/vat_term.robot
```
### Start/Stop VPP
 - Start, stop keywords for VPP (in docker) and write into VPP:
```
https://github.com/ligato/vpp-agent/blob/master/tests/robot/libraries/vpp.robot
```
### Read/Write VPP terminal
 - Direct keywords to show (interfaces, acl, BD, route, DNAT,...), ping and other commands in VPP (not docker, inside VPP):
```
https://github.com/ligato/vpp-agent/blob/master/tests/robot/libraries/vpp_term.robot
```
### VXLAN keywords
```
https://github.com/ligato/vpp-agent/blob/master/tests/robot/libraries/vxlan.robot
```
### Wait until Keyword
 - used when some amount of time is needed for the test:
```robot
Wait Until Keyword Succeeds   ${WAIT_TIMEOUT}   ${SYNC_SLEEP}    ...
```

## Test Examples

Tests example directory is `vpp-agent/tests/robot/examples/`.

 - CRUD test example on github:
```
https://github.com/ligato/vpp-agent/blob/dev/tests/robot/examples/example_crud_test.robot
```
  - complete example:
```robot
*** Settings ***
Library      OperatingSystem
Library      Collections

Resource     ../../variables/${VARIABLES}_variables.robot

Resource     ../../libraries/all_libs.robot

Force Tags        crud     IPv4
Suite Setup       Testsuite Setup
Suite Teardown    Testsuite Teardown
Test Setup        TestSetup
Test Teardown     TestTeardown

*** Variables ***
${VARIABLES}=        common
${ENV}=              common
${WAIT_TIMEOUT}=     20s
${SYNC_SLEEP}=       3s

${NAME_TAP1}=        vpp1_tap1
${NAME_TAP2}=        vpp1_tap2
${MAC_TAP1}=         12:21:21:11:11:11
${MAC_TAP1_2}=       22:21:21:11:11:11
${MAC_TAP2}=         22:21:21:22:22:22
${IP_TAP1}=          20.20.1.1
${IP_TAP1_2}=        21.20.1.2
${IP_TAP2}=          20.20.2.1
${PREFIX}=           24
${MTU}=              4800
${UP_STATE}=         up


*** Test Cases ***
Configure Environment
    [Tags]    setup
    Configure Environment 1

Show Something Before Setup
    ${interfaces}=    vpp_term: Show Interfaces    agent_vpp_1

Add Something
    vpp_term: Interface Not Exists  node=agent_vpp_1    mac=${MAC_TAP1}
    Put TAP Interface With IP    node=agent_vpp_1    name=${NAME_TAP1}    mac=${MAC_TAP1}    ip=${IP_TAP1}    prefix=${PREFIX}    host_if_name=linux_${NAME_TAP1}

Check Something Is Created
    ${interfaces}=       vat_term: Interfaces Dump    node=agent_vpp_1
    Wait Until Keyword Succeeds   ${WAIT_TIMEOUT}   ${SYNC_SLEEP}    vpp_term: Interface Is Created    node=agent_vpp_1    mac=${MAC_TAP1}
    ${actual_state}=    vpp_term: Check TAP interface State    agent_vpp_1    ${NAME_TAP1}    mac=${MAC_TAP1}    ipv4=${IP_TAP1}/${PREFIX}    state=${UP_STATE}

Add Something_Other
    No Operation

Check Something_Other Is Created
    No Operation

Check Something Is Still Configured
    ${actual_state}=    vpp_term: Check TAP interface State    agent_vpp_1    ${NAME_TAP1}    mac=${MAC_TAP1}    ipv4=${IP_TAP1}/${PREFIX}    state=${UP_STATE}

Update Something
    Put TAP Interface With IP    node=agent_vpp_1    name=${NAME_TAP1}    mac=${MAC_TAP1_2}    ip=${IP_TAP1_2}    prefix=${PREFIX}    host_if_name=linux_${NAME_TAP1}

Check Something Is Created
    Wait Until Keyword Succeeds   ${WAIT_TIMEOUT}   ${SYNC_SLEEP}    vpp_term: Interface Is Created    node=agent_vpp_1    mac=${MAC_TAP1_2}
    ${actual_state}=    vpp_term: Check TAP interface State    agent_vpp_1    ${NAME_TAP1}    mac=${MAC_TAP1_2}    ipv4=${IP_TAP1_2}/${PREFIX}    state=${UP_STATE}

Check Something_Other Has Not Changed
    No Operation

Delete Something
    Delete VPP Interface    agent_vpp_1    ${NAME_TAP1}

Check Something Has Been Deleted
    Wait Until Keyword Succeeds   ${WAIT_TIMEOUT}   ${SYNC_SLEEP}    vpp_term: Interface Not Exists  node=agent_vpp_1    mac=${MAC_TAP1_2}

Check Something_Other Is Still Configured
    No Operation

Show Interfaces And Other Objects After Setup
    vpp_term: Show Interfaces    agent_vpp_1
    Write To Machine    agent_vpp_1_term    show int addr
    Write To Machine    agent_vpp_1_term    show h
    Write To Machine    agent_vpp_1_term    show br
    Write To Machine    agent_vpp_1_term    show br 1 detail
    Write To Machine    agent_vpp_1_term    show vxlan tunnel
    Write To Machine    agent_vpp_1_term    show err
    vat_term: Interfaces Dump    agent_vpp_1
    Execute In Container    agent_vpp_1    ip a

*** Keywords ***

TestSetup
    Make Datastore Snapshots    ${TEST_NAME}_test_setup

TestTeardown
    Make Datastore Snapshots    ${TEST_NAME}_test_teardown

```
