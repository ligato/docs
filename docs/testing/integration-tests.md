# Integration Tests

This page contains information about integration tests for the vpp-agent.

---

The integration tests are written using Go and test mainly the _vppcalls layer_ that provides VPP-version agnostic API interface. The integration test suite can be executed for any of the VPP versions defined in `vpp.env`.

## Quick Start - execute entire integration test suite

The easiest way to run entire integration test suite is to use make target `integration-tests`.

Run tests with default VPP version (which is defined by `VPP_DEFAULT`):

```sh
# run tests with default VPP version 
make integration-tests
```

Run tests with specific VPP version:

```sh
# run tests with VPP 20.05
make integration-tests VPP_VERSION=2005

# run tests with VPP 20.01
make integration-tests VPP_VERSION=2001

# run tests with VPP 19.08
make integration-tests VPP_VERSION=1908
```

## Customize Test Run

The integration test suite can be executed manually by running script `./tests/integration/vpp_integration.sh`.
This script supports adding extra arguments for the test run.

Before running the tests using the script you have to set `VPP_IMG` variable to the image that should be used (make target does this for you). 

The simplest way is to export it like this:

```sh
export VPP_IMG=ligato/vpp-base:20.01
```

Then you can simply execute this script:

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

Run tests with any additional flags supported by `go test` tool:

```sh
# run with coverage report
bash ./tests/integration/vpp_integration.sh -test.cover

# run with CPU profiling
bash ./tests/integration/vpp_integration.sh -test.cpuprofile cpu.out
```

---

##### References

- README: https://github.com/ligato/vpp-agent/blob/master/tests/integration/README.md
