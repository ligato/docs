### Build without Image

---

This section explains how to build VPP and the VPP agent without an image.

!!! Note
    This option is recommended for **development or testing purposes only**.

---

Start by cloning the VPP Agent code:

```
git clone git@github.com:ligato/vpp-agent.git
```

---

**Build VPP**

You should build VPP first in order to provide required libraries for the host OS.


1. Clone the VPP code as described above, and then checkout the defined commit. Use the `vpp.env` as a source. It can be found in the ligato/vpp-agent folder. Building different VPP commit/versions can cause the VPP agent to fail at startup due to incompatibilities.
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

1. Go to the VPP agent root directory, and use `make-build`. The binary file is available in `vpp-agent/cmd/vpp-agent`.



2. Start VPP and verify the VPP agent can connect to it.