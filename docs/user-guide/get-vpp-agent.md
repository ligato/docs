# VPP Agent Setup

This section explains how to prepare and run an environment supporting the vpp-agent and VPP in a single container.  

!!! Note
    `VPP Agent` is used when discussing the Ligato VPP Agent in a higher level context with examples being an architecture description. `vpp-agent` is a term used to define the actual code itself and/or in describing direct interactions with other associated software components such as docker. You will see this term appear in commands, APIs and code. `VPP` is used to describe the VPP dataplane. 


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

- Binaries for the vpp-agent with default config files
- Installation of the compatible VPP

**What is included in the pre-built development image:**

- vpp-agent with default config files including source code
- Compatible VPP installation including source code and build artifacts
- Development environment with all requirements to build the vpp-agent and the VPP
- Tools to generate code (proto models, binary APIs)

Command to pull the docker image:
```
docker pull ligato/dev-vpp-agent
```


---



### Local Image Build

First clone the vpp-agent repository:

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

The scripts also recognizes the architecture. The correct VPP code source URL and commit ID is taken from the [vpp.env][vpp-env] file located inside the vpp-agent repository.


**Image in debug mode**
The development image can be built in `DEBUG` mode. 

```
$ VPP_DEBUG_DEB=y ./build.sh
```

This only builds the image with debug mode support. Use `RUN_VPP_DEBUG=y` to start the image in debug mode.

---


### Start the Image

Command to start the vpp-agent:

```
sudo docker run -it --name vpp_agent --privileged --rm prod_vpp_agent
```

Note that the vpp-agent is executed in `privileged` mode. Several vpp-agent operations (e.g. Linux namespace handling) require permissions on the target host instance. Running in non-privileged mode may cause the vpp-agent to fail to start. [More information here][interface-plugin].

Open another terminal and run the following command:
```
sudo docker exec -it vpp_agent bash
```

The vpp-agent and VPP are started automatically by default. 

Image-specific environment variables available that can be assigned with `docker -e` on image start:

- `OMIT_AGENT` - do not start the vpp-agent together with the image
- `RETAIN_SUPERVISOR` - unexpected vpp-agent or VPP shutdown causes the supervisor to quit. This setting prevents that.

---

## Start VPP and the vpp-agent

!!! note
    The vpp-agent will terminate if unable to connect to the VPP, a database or the Kafka message server. 

**Start VPP**:

Command to start VPP:  
```
vpp unix { interactive } dpdk { no-pci }
```

Command to start VPP without DPDK:
```
vpp unix { interactive } plugins { plugin dpdk_plugin.so { disable } }
```

---

**Start the vpp-agent**

Command to start the vpp-agent

```
vpp-agent
```

To enable certain features (database connection, messaging), the vpp-agent requires static [configuration files][config-files] referred to as `.conf` files. Quick notes on conf files:

- majority of plugins support .conf files. 
- Some plugins cannot be loaded without a `.conf` file even if defined as part of the plugin feature set (e.g. Kafka). 
- Other plugins load `.conf` files as the default value. This makes it easy to use `.conf` files to modify certain plugin behavior.

---


### Connect vpp-agent to the Key-Value Data Store

!!! Note
    There are a number of similar terms in this documentation and elsewhere used to define what is essentially a data store or database of key-value (KV) pairs or more generally structured data objects. These terms include but are not limited to KV database, KVDB, KV store and so on. Unless otherwise noted, we will use the term `KV data store` in the documentation text in this section and elsewhere in the documentation. The term `KVDB` might appear in the code examples and this will remain as is.


Configuration information is stored in a KV data store. The vpp-agent needs to know how to connect to a particular instance of a KV data store so that it can listen for and receive config data. The connection information (IP address, port) is provided via a .conf file. Every KV data store plugin defines its own config file. 

Click [KV data store][kv-overview] for more information.

**Start etcd:**

The etcd server is started in its own container on a local host. The command is as follows:

```
docker run -p 2379:2379 --name etcd --rm quay.io/coreos/etcd:v3.1.0 /usr/local/bin/etcd -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379
```

The etcd server docker image is downloaded (if it does not exist already) and started, ready to serve requests. In the default docker environment, the etcd server will be available on the IP `172.17.0.1` on port `2379`. The command above can be edited to run a version other than `3.1.0`, or change the listen or advertise addresses. 

The vpp-agent defines a flag for the etcd plugin called `--etcd-config`. It points to the location of the `etcd.conf` file that holds static config information used by the etcd plugin upon startup. The most important is the IP address(es) of etcd enpodint(s)). 

Here is an example configuration:

```
insecure-transport: true
dial-timeout: 1000000000
endpoints: 
 - "172.17.0.1:2379"
```

Start the vpp-agent with the config flag (default path to the etcd.conf file is `/opt/vpp-agent/dev`):
```
vpp-agent --etcd-config=/opt/vpp-agent/dev/etcd.conf
```
<br/>

**Example with Kafka (optional):**

The Kafka server is started in a separate container. The command fetches the Kafka container (if missing) and starts the Kafka server:

```
docker run -p 2181:2181 -p 9092:9092 --name kafka --rm --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 spotify/kafka
```

The vpp-agent needs to know the IP address and port of the Kafka server. This information is defined in the `kafka.conf` file, which is provided using the flag `--kafka-conf`. 

An example config:

```
addrs:
 - "172.17.0.1:9092"
```

Start the vpp-agent with the config flag using the default path to the `kafka.conf` file which is `/opt/vpp-agent/dev`): 
```
vpp-agent --kafka-config=/opt/vpp-agent/dev/kafka.conf
```

---

### Executables


The `/cmd` package groups executables that can be built from sources in the vpp-agent repository:

- **vpp-agent** - the default off-the-shelf vpp-agent 
  executable (i.e. no app or extension plugins) that can be bundled with
  an off-the-shelf VPP to form a simple cloud-native VNF,
  such as a vswitch.
- **[agentctl][agentctl]** - CLI tool that allows to show
  the state and to configure vpp-agents connected to etcd


---

### How to Use Multiple vpp-agents

**1. Microservice Label**

Currently, a single instance of a `vpp-agent` can serve a single `VPP` instance, while multiple vpp-agents can use (or listen to) the same KV data store. vpp-agent instances (there is an analogy to a microservice) are distinguished with the microservice label. The label is a part of the KV data store key to make clear which part of configuration belongs to which vpp-agent. Note that every vpp-agent only watches keys associated with its own microservice label. 

The default microservice label is `vpp1`. It can be changed with the environment variable `MICROSERVICE_LABEL`, or using the `-microservice-label` flag.

It is possible to use the same label to "broadcast" shared configuration data to multiple vpp-agents. However, this approach is generally not recommended.

The [concepts][concepts] section of the user-guide provides more details on keys, microservice labels and KV data stores.

**2. Shared Memory Prefix**

Running multiple `VPPs` on the same host requires a different shared memory prefix (SHM) to distinguish communication sockets for given VPP instances. In order to connect the vpp-agent to the VPP with a custom socket, a valid SHM needs to be provided to the [GoVPP mux plugin][govppmux-plugin].
 
---


## Create Your First Configuration

**Put the configuration into the KV data store:**

Store the values following the given model with the proper key to the KV data store. We recommend the use of etcd as the KV data store and the 'etcdctl` tool as the CLI.

Information about keys and data structures can be found [here][references].

An example of how to perform this task is shown in [section 5.1][quickstart-guide-51-keys] of the Quickstart Guide.

**Use clientv2**

The [clientv2][clientv2] package contains API definitions for every supported configuration item and can be used to pass data without an external database (e.g. etcd)

[arm-doc]: https://github.com/ligato/vpp-agent/blob/master/docs/arm64/README.md
[clientv2]: ../user-guide/concepts.md#client-v2
[docker]: https://github.com/ligato/vpp-agent/tree/master/docker
[govppmux-plugin]: ../plugins/vpp-plugins.md#govppmux-plugin
[interface-plugin]: ../plugins/linux-plugins.md#interface-plugin
[kv-overview]: ../user-guide/concepts.md#key-value-data-store-overview
[references]: ../user-guide/reference.md
[vpp-env]: https://github.com/ligato/vpp-agent/blob/master/vpp.env
[agentctl]: ../user-guide/agentctl.md#AgentCTL
[config-files]: ../user-guide/config-files.md
[concepts]: ../user-guide/concepts.md
[quickstart-guide-51-keys]: ../user-guide/quickstart.md#51-configure-the-vpp-dataplane-using-the-vpp-agent   

*[DPDK]: Data Plane Development Kit
*[KVDB]: Key-Value Database
*[NAT]: Network Address Translation










