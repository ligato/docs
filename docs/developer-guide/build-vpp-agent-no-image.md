### Build without Image

---

This section explains how to build VPP and the VPP agent without an image.

!!! Note
    Use this option for **development or testing purposes only**.

---

Start by cloning the VPP Agent code:

```
git clone git@github.com:ligato/vpp-agent.git
```

---

**Build VPP**

You should first build VPP in order to provide the required libraries for the host OS.


1. Clone the VPP code as described above, and then checkout the defined commit. Use the `vpp.env` contained in the ligato/vpp-agent folder as a source. Building different VPP commit/versions may cause the VPP agent to fail at startup due to incompatibilities.
</br>
</br>

2. Use the following commands to build VPP:

```bash
make install-dep 
make dpdk-install-dev
make build-release 
make pkg-deb
```

Next, in the `build-root` directory, unpack `*.deb` package files with `dpkg -i`.

**Build the VPP Agent**

1. Go to the VPP agent root directory, and use `make-build`. The `vpp-agent/cmd/vpp-agent` folder contains the binary file.



2. Start VPP and verify the VPP agent can connect to it.