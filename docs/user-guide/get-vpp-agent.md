# VPP Agent Setup

This section explains how to prepare and run an environment supporting the VPP Agent and VPP in a single container.  

!!! Note
    `VPP Agent` is used when discussing the Ligato VPP Agent. `VPP` is used to describe the VPP dataplane.


---

## Set Up Your Environment

There are two options here: 

* Pre-built docker image pull from [dockerhub](https://hub.docker.com/u/ligato)
 * Build a Local Image

The simplest one is to pull a pre-built docker image. The local build option has a couple of extra steps but is straightforward.
  
!!! note
    Docker ce 17.05 or higher is required.

### Docker Image Pull

The docker image exists in two versions: production and development (which can be also used for debugging). The production image is a lightweight version of the development image.

Supported architectures:

* AMD64 (a.k.a. x86_64)
* ARM64 (a.k.a. aarch64) - [see documentation for ARM][arm-doc]

** What is included in the pre-built production image:**

- Binaries for the VPP Agent with default config files
- Installation of the compatible VPP

**What is included in the pre-built development image:**

- VPP Agent with default config files including source code
- Compatible VPP installation including source code and build artifacts
- Development environment with all requirements to build the VPP Agent and the VPP
- Tools to generate code (proto models, binary APIs)

Command to pull the docker image:
```
docker pull ligato/dev-vpp-agent
```


---



### Local Image Build

First clone the VPP Agent repository:

```
git clone git@github.com:ligato/vpp-agent.git
```

Next, navigate to the [docker][docker] folder. Inside, choose either the production or development image to work with. 

    
Command to build the `production image`:
```
make prod-image
```    
<br/>
Command to build the `development image`:

```
make dev-image 
```
<br/>
Command to build both the `production and development images`:

```
make images 
```

The development option constructs an image using the `dev_vpp-agent` image; the production option is built with files taken from the `vpp-agent` image.

The scripts also recognizes the architecture. The correct VPP code source and commit ID is taken from the [vpp.env][vpp-env] file (inside the vpp-agent repository).


**Image in debug mode**
The development image can be built in `DEBUG` mode. 

```
$ VPP_DEBUG_DEB=y ./build.sh
```

This only builds the image with debug mode support. Use `RUN_VPP_DEBUG=y` to start the image in the debug mode.

---


### Start the Image

Command to start the agent:

```
sudo docker run -it --name vpp_agent --privileged --rm prod_vpp_agent
```

Note that the VPP Agent is executed in `privileged` mode. Several VPP Agent operations (e.g. Linux namespace handling) require permissions on the target host instance. Running in non-privileged mode may cause the VPP Agent to fail to start ([more information here][interface-plugin]).

Open another terminal and run the following command:
```
sudo docker exec -it vpp_agent bash
```

The VPP Agent and VPP are started automatically by default. 

Image-specific environment variables available (assign with `docker -e` on image start):
- `OMIT_AGENT` - do not start the Agent together with the Image
- `RETAIN_SUPERVISOR` - unexpected Agent or VPP shutdown causes the supervisor to quit. This setting prevents that.

---

## Start VPP and the VPP Agent

!!! note
    The VPP Agent will terminate if unable to connect to the VPP, a database or the Kafka message server (if required by the config file). 

**Start VPP**:

Command to start VPP:  
```
vpp unix { interactive } dpdk { no-pci }
```

Command to start VPP without DPDK:
```
vpp unix { interactive } plugins { plugin dpdk_plugin.so { disable } }
```

It is possible to apply additional VPP Agent features (e.g. plugins) with the specific startup flags. Examples include:



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

<br/>


**Start the VPP Agent**

Command to start the VPP Agent

```
vpp-agent
```

To enable certain features (database connection, messaging), the VPP Agent requires static configuration files referred to as `.conf` files. The majority of plugins support .conf files. Some plugins cannot be loaded without a `.conf` file even if defined as part of the plugin feature set(e.g. Kafka). Other plugins load `.conf` files as the default value. This makes it easy to use `.conf` files to modify certain plugin behavior.

---


### Connect VPP Agent to the Key-Value Datastore

Configuration information is stored in a key-value datastore (or KV datastore for short). The VPP Agent needs to know how to connect to a particular instance of a KV datastore so that it can receive the config data. The connection information (IP address, port) is provided via a .conf file. Every KV datastore plugin defines its own config file. 

Click [KV datastore][kv-overview] for more information.

**Start ETCD:**

The ETCD server is started in its own container on a local host. The command is as follows:

```
docker run -p 2379:2379 --name etcd --rm quay.io/coreos/etcd:v3.1.0 /usr/local/bin/etcd -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379
```

The ETCD server docker image is downloaded (if it does not exist already) and started, ready to serve requests. In the default docker environment, the ETCD server will be available on the IP `172.17.0.1` on port `2379`. The command above can be edited to run a version other than `3.1.0`, or change the listen/advertise address. 

The VPP defines flag for the ETCD plugin `--etcd-config` which can be used to provide static config (most importantly the IP address of an ETCD enpodint(s)). 

Here is an example configuration:

```
insecure-transport: true
dial-timeout: 1000000000
endpoints: 
 - "172.17.0.1:2379"
```

Start the VPP Agent with the config flag (default path to .conf file is `/opt/vpp-agent/dev`):
```
vpp-agent --etcd-config=/opt/vpp-agent/dev/etcd.conf
```
<br/>

**Example with Kafka (optional):**

The Kafka server is started in a separate container. The command fetches the Kafka container (if missing) and starts the Kafka server:

```
docker run -p 2181:2181 -p 9092:9092 --name kafka --rm --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 spotify/kafka
```

The VPP Agent needs to know the IP address and port of the Kafka server. This information is set via the config file, which is provided with flag `--kafka-conf`. The example config:

```
addrs:
 - "172.17.0.1:9092"
```

Start the VPP Agent with the config flag (default path to .conf file is `/opt/vpp-agent/dev`): 
```
vpp-agent --kafka-config=/opt/vpp-agent/dev/kafka.conf
```

---

### Executables


The `/cmd` package groups executables that can be built from sources in the vpp-agent repository:

- **vpp-agent** - the default off-the-shelf VPP Agent 
  executable (i.e. no app or extension plugins) that can be bundled with
  an off-the-shelf VPP to form a simple cloud-native VNF,
  such as a vswitch.
- **[agentctl][agentctl]** - CLI tool that allows to show
  the state and to configure VPP Agents connected to etcd


---

### How to Use Multiple VPP Agents

**1. Microservice label**

Currently, a single instance of a `VPP Agent` can serve a single `VPP` instance, while multiple VPP Agents can use (or listen to) the same KV datatstore. VPP Agent instances (there is an analogy to a microservice) are distinguished with the microservice label. The label is a part of the KV datastore key to make clear which part of configuration belongs to which VPP Agent. Note that every VPP Agent only watches keys associated with its own microservice label. 

The default microservice label is `vpp1`. It can be changed with the environment variable `MICROSERVICE_LABEL`, or via the `-microservice-label` flag.

It is possible to use the same label to "broadcast" shared configuration data to multiple VPP Agents. However, this approach is generally not recommended.

**2. Shared Memory Prefix**

Running multiple `VPPs` on the same host requires a different shared memory prefix (SHM) to distinguish communication sockets for given VPP instances. In order to connect the VPP Agent to the VPP with a custom socket, a valid SHM needs to be provided to the GoVPP mux plugin (see [plugin's readme][govppmux-plugin])
 
---


## Make Your First Configuration

**Put the configuration into the KV datastore:**

Store the values following the given model with the proper key to the KV datastore. We recommend the use of ETCD as the KV datastore and the 'etcdctl` tool. 

Information about keys and data structures can be found [here][references].

**Use clientv2**

Package [clientv2][clientv2] contains API definition for every supported configuration item and can be used to pass data without a external database.

[arm-doc]: https://github.com/ligato/vpp-agent/blob/master/docs/arm64/README.md
[clientv2]: ../user-guide/concepts.md#client-v2
[docker]: https://github.com/ligato/vpp-agent/tree/master/docker
[govppmux-plugin]: ../plugins/vpp-plugins.md#govppmux-plugin
[interface-plugin]: ../plugins/linux-plugins.md#interface-plugin
[kv-overview]: ../user-guide/concepts.md#key-value-store-overview
[references]: ../user-guide/reference.md
[vpp-env]: https://github.com/ligato/vpp-agent/blob/master/vpp.env
[agentctl]: ../tools/agentctl.md#AgentCTL   

*[DPDK]: Data Plane Development Kit
*[KVDB]: Key-Value Database
*[NAT]: Network Address Translation










