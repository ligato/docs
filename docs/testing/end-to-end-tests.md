# End-to-End Tests

This section contains information about end-to-end (e2e) tests for the VPP agent.

---

The e2e tests are written using Go. The e2e test suite can be executed for any of the VPP versions 
defined in [vpp.env](https://github.com/ligato/vpp-agent/blob/master/vpp.env).

!!!Note
    Running the tests on a mac using the make command could result in errors such as missing syscalls. 


---

## Run E2E Test Suite

The easiest way to run the entire e2e test suite is to use make target `e2e-tests`.

Run tests using the default VPP version defined by `VPP_DEFAULT`:

```sh
# run tests with default VPP version
make e2e-tests
```

Run tests for a specific `VPP_VERSION`:

```sh
# run tests with VPP 20.05
make e2e-tests VPP_VERSION=2005
# run tests with VPP 20.01
make e2e-tests VPP_VERSION=2001
# run tests with VPP 19.08
make e2e-tests VPP_VERSION=1908
```

---

## Customize E2E Test Run

The e2e test suite can be executed manually by running the following script: 

```
./tests/e2e/run_e2e.sh
```
This script supports the addition of extra arguments for the test run.

Before running the tests, you must set the `VPP_IMG` variable to the desired image. 
 
The simplest way to set the `VPP_IMG` variable uses the export command:

```sh
export VPP_IMG=ligato/vpp-base:20.01
```

Then you can execute the script like so:

```sh
bash ./tests/e2e/run_e2e.sh
```

Run tests in verbose mode:

```sh
bash ./tests/e2e/run_e2e.sh -test.v 
```

Run tests cases for specific functionality:

```sh
# test cases for ACLs
bash ./tests/e2e/run_e2e.sh -test.run=ACL

# test cases for L3 routes
bash ./tests/e2e/run_e2e.sh -test.run=Route
```

Run tests with any additional flags supported by the `go test` tool:

```sh
# run with coverage report
bash ./tests/e2e/run_e2e.sh -test.cover

# run with CPU profiling
bash ./tests/e2e/run_e2e.sh -test.cpuprofile cpu.out
```

---

Reference: [VPP Agent tests repo folder](https://github.com/ligato/vpp-agent/tree/master/tests)
