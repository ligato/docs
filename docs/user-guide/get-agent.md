# Get Agent

This page describes in more detail how to prepare and work with the vpp-agent.

---

## Set up your environment

Options are:

 * get pre-prepared docker image from the dockerhub
 * build the image locally 
 * build the VPP-Agent with the VPP locally without the image

### Get the Docker image

The image exists in two versions, for production and for development (which can be also used for debugging). Production image is a lightweight version of the development image.

Supported architectures:

* AMD64 (a.k.a. x86_64)
* ARM64 (a.k.a. aarch64) - [see documentation for ARM][arm-doc]

**What is included in the production image:**

- binaries for the VPP-Agent with default config files and the VPP-Agent-ctl
- installation of the compatible VPP

Get from docker hub: `docker pull ligato/vpp-agent`

**What is included in the development image:**

- VPP-Agent with default config files including source code
- compatible VPP installation including source code and build artifacts
- development environment with all requirements to build the VPP-Agent and the VPP
- tools needed to generate code (proto models, binary APIs)
- the VPP-Agent-ctl

Get from docker hub: `docker pull ligato/dev-vpp-agent`

### Build local image

1. Clone the Agent repository:

```
git clone https://github.com/ligato/vpp-agent.git
```

2. Navigate to the [docker][docker] directory. Inside, choose whether the production or the development image should be built. Both directories contain `build.sh` script with the automatic build process. The development image builds an image using `dev_vpp-agent` image, production image is built with files taken from the `vpp-agent` image.

The script also recognizes architecture. The correct VPP code source and commit ID is taken from the [vpp.env][vpp-env] file (inside the Agent repository).

**Image in debug mode**

The development image can be built in the `DEBUG` mode, instead of the release mode. 

```
$ VPP_DEBUG_DEB=y ./build.sh
```

Although, this only builds the image with debug mode support. Use `RUN_VPP_DEBUG=y` to start the image in the debug mode.

The image can be shrunk with script `./shrink.sh`, which creates a new image with removed installation files (which decreases its size). To execute this procedure successfully, docker version 1.13 or newer is required.

### Start the image

Execution command to start the agent:

```
sudo docker run -it --name vpp_agent --privileged --rm prod_vpp_agent
```

Note that the Agent is executed in `privileged` mode. Several Agent operations (like Linux namespace handling) require permissions on target host instance. Running in non-privileged mode may cause Agent to fail to start ([more information here][interface-plugin]).

Open another terminal:
```
sudo docker exec -it vpp_agent bash
```

The Agent and the VPP are started automatically by default. 

Image-specific environment variables available (assign with `docker -e` on image start):
- `OMIT_AGENT` - do not start the Agent together with the Image
- `RETAIN_SUPERVISOR` - unexpected Agent or VPP shutdown causes the supervisor to quit. This setting prevents that.

### Build the Agent and VPP without image

Another alternative is to build the VPP and the Agent directly (without image). This option is not recommended for anything else but development or testing. Start with getting the Agent code with `git clone https://github.com/ligato/vpp-agent.git` 

**Build VPP**

The VPP should be built first in order to provide required libraries to the host OS.

1. Clone the VPP code and checkout to the defined commit. Use the `vpp.env` as a source (in the Agent repo root). Building different VPP commit/version can cause the Agent to fail at start because of the incompatibility.

2. Use the following commands to build the VPP:

```bash
make install-dep 
make dpdk-install-dev
make build-release 
make pkg-deb
```

Then in the `build-root` directory unpack `*.deb` package files with `dpkg -i`

**Build the Agent**

1. Go to the vpp-agent root directory, and use `make-build`. The binary file is available in `vpp-agent/cmd/vpp-agent`.

2. Start the VPP and verify the Agent can connect to it.

## Start the VPP and the Agent

!!! note
    The agent will terminate if unable to connect to the VPP, a database or the Kafka message server (if required by the config file). 

**Start the VPP**:  
```
vpp unix { interactive } dpdk { no-pci }
```

Start the VPP lite (no DPDK)
```
vpp unix { interactive } plugins { plugin dpdk_plugin.so { disable } }
```

Another startup flags are required using certain Agent features:

For the full functionality of the NAT plugin: 
```
nat {
  endpoint-dependent
}
```

VPP socket definition for the Punt plugin:
```
punt {
  socket /tmp/socket/punt
}
```

Support for VPP stats (reports by the interface plugin, etc.)
```
statseg {
  default
}
```

**Start the agent**

Run:

```
vpp-agent
```

To enable certain features (database connection, messaging), Agent requires static configuration files, referred to as `.conf`. Majority of plugins support .conf files. While some plugins cannot be loaded without it even if defined as part of the agent plugin set(for example Kafka), some of them use the default value, thus .conf files can be used to modify certain plugin behavior.

### Connect Agent to the KVDB

In order to provide configuration from any KVDB (key-value database), the agent needs to know how to connect to the desired instance. The connection information (IP address, port) is provided via the particular .conf file. Every KVDB plugin defines its own config file. 

More information about the KVDB: [KV-Store overview][kv-overview]

**Start the ETCD:**

The ETCD server is started in its own container on a local host. The command is as follows:

```
docker run -p 2379:2379 --name etcd --rm quay.io/coreos/etcd:v3.1.0 /usr/local/bin/etcd -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379
```

The ETCD docker image is downloaded (if does not exist already) and started, ready to serve requests. In the default docker environment, the ETCD server will be available on the IP `172.17.0.1` on port `2379`. The command above can be edited to get different version that `3.1.0`, or change the listen/advertise address. 

The vpp-agent defines flag for the ETCD plugin `--etcd-config` which can be used to provide static config (most importantly what is the IP address of an ETCD enpodint(s)). The example configuration (and the default one):

```
insecure-transport: true
dial-timeout: 1000000000
endpoints: 
 - "172.17.0.1:2379"
```

Start the agent with the config flag (default path to .conf file is `/opt/vpp-agent/dev`):
```
vpp-agent --etcd-config=/opt/vpp-agent/dev/etcd.conf
```

**Example with Kafka (optional):**

Kafka server is started in the separate contained. The command fetches the Kafka container (if missing) and starts the Kafka server:

```
docker run -p 2181:2181 -p 9092:9092 --name kafka --rm --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 spotify/kafka
```

The vpp-agent needs to know the IP address and port of the Kafka server. This information is set via the config file, which is provided with flag `--kafka-conf`. The example config:

```
addrs:
 - "172.17.0.1:9092"
```

Start the agent with the config flag (default path to .conf file is `/opt/vpp-agent/dev`): 
```
vpp-agent --kafka-config=/opt/vpp-agent/dev/kafka.conf
```

### How to use multiple Agents

**1. Microservice label**

Currently, a single Agent instance can serve single VPP instance, while multiple agents can use the same KVDB store. Agent instances (analogy to microservices) are distinguished with the microservice label. The label is a part of the KVDB key to make clear which part of configuration belongs to which Agent since every Agent watches only to keys with its own microservice label. 

The default microservice label is `vpp1`, it can be changed with environment variable `MICROSERVICE_LABEL`, or via `-microservice-label` flag.

It is possible to use the same label for multiple agents to "broadcast" identical configuration, however, this approach is in general not recommended.

**2. Shared memory prefix**

Running multiple VPPs on the same host requires different shared memory prefix (SHM) to distinguish communication sockets for given VPP instances. In order to connect the Agent to the VPP with a custom socket, correct SHM has to be provided to the GoVPP mux plugin (see [plugin's readme][govppmux-plugin])
 
## Make your first configuration

**Put the configuration to the KVDB:**

Store value following the given model with the proper key to the KVDB. Depending on the KVDB type, we recommend to use the appropriate tool to put key-value data (e.g. `etcdctl` for the ETCD, or `redis-cli` for the Redis).
. Information about keys and data structures can be found [here][references].

**Use clientv2**

Package [clientv2][clientv2] contains API definition for every supported configuration item and can be used to pass data without a external database.

[arm-doc]: https://github.com/ligato/vpp-agent/blob/master/docs/arm64/README.md
[clientv2]: ../user-guide/concepts.md#client-v2
[docker]: https://github.com/ligato/vpp-agent/tree/master/docker
[govppmux-plugin]: ../plugins/core-vpp-plugins.md#govppmux-plugin
[interface-plugin]: ../plugins/linux-plugins.md#interface-plugin
[kv-overview]: ../user-guide/concepts.md#key-value-store-overview
[references]: ../user-guide/reference.md
[vpp-env]: https://github.com/ligato/vpp-agent/blob/master/vpp.env   

*[DPDK]: Data Plane Development Kit
*[KVDB]: Key-Value Database
*[NAT]: Network Address Translation










