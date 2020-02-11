### Build without Image

This section explains how to build the VPP and VPP Agent directly (without image). This option is recommended for `development or testing only`. 

Start with cloning the VPP Agent code:

 ```
 git clone git@github.com:ligato/vpp-agent.git
 ``` 

**Build VPP**

The VPP should be built first in order to provide required libraries for the host OS.

1. Clone the VPP code (described above) and checkout to the defined commit. Use the `vpp.env` as a source (in the Agent repo root). Building different VPP commit/version can cause the VPP Agent to fail at startup due to incompatibilities.

2. Use the following commands to build the VPP:

```bash
make install-dep 
make dpdk-install-dev
make build-release 
make pkg-deb
```

Then in the `build-root` directory unpack `*.deb` package files with `dpkg -i`

**Build the VPP Agent**

1. Go to the vpp-agent root directory, and use `make-build`. The binary file is available in `vpp-agent/cmd/vpp-agent`.



2. Start the VPP and verify the VPP Agent can connect to it.