# Integration Tests

This section contains information about integration tests for the VPP agent.

---

The integration tests are written using Go and test mainly the _vppcalls layer_ that provides 
VPP-version agnostic API interface. The integration test suite can be executed for any of the 
VPP versions defined in [vpp.env](https://github.com/ligato/vpp-agent/blob/master/vpp.env).

!!!Note
    Running the tests on a mac using the make command could result in errors such as missing syscalls.

---

## Run Integration Test Suite

The easiest way to run the entire integration test suite is to use make target `integration-tests`.

Run tests using the default VPP version defined by `VPP_DEFAULT`:

```
# run tests with default VPP version 
make integration-tests
```

Run tests for a specific `VPP_VERSION`:

```
# run tests with VPP 20.05
make integration-tests VPP_VERSION=2005

# run tests with VPP 20.01
make integration-tests VPP_VERSION=2001

# run tests with VPP 19.08
make integration-tests VPP_VERSION=1908
```

---

## Customize Integration Test Run

The integration test suite can be executed manually by running the following script:
 
```
./tests/integration/vpp_integration.sh
```
This script supports the addition of extra arguments for the test run.

Before running the tests, you must set the `VPP_IMG` variable to the desired image. 

The simplest way to set the `VPP_IMG` variable uses the export command:

```sh
export VPP_IMG=ligato/vpp-base:20.01
```

Then you can execute the script like so:

```sh
bash ./tests/integration/vpp_integration.sh
```

Run tests in verbose mode:

```sh
bash ./tests/integration/vpp_integration.sh -test.v 
```

Run tests cases for specific functionality:

```sh
# test cases for ACLs
bash ./tests/integration/vpp_integration.sh -test.run=ACL

# test cases for L3 routes
bash ./tests/integration/vpp_integration.sh -test.run=Route
```

Run tests with any additional flags supported by the `go test` tool:

```sh
# run with coverage report
bash ./tests/integration/vpp_integration.sh -test.cover

# run with CPU profiling
bash ./tests/integration/vpp_integration.sh -test.cpuprofile cpu.out
```

---

Reference: [VPP Agent tests repo folder](https://github.com/ligato/vpp-agent/tree/master/tests)