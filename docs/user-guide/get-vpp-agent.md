# VPP Agent Setup

This section explains how to prepare and run an environment supporting the vpp-agent and VPP in a single container.  

!!! Note
    `VPP Agent` is used when discussing the Ligato VPP Agent in a higher level context with examples being an architecture description. `vpp-agent` is a term used to define the actual code itself and/or in describing direct interactions with other associated software components such as docker. You will see this term appear in commands, APIs and code. `VPP` is used to describe the VPP dataplane. 


---

## Set Up Your Environment

There are two options:

* Pre-built docker image pull from [dockerhub][dockerhub]
* Build a Local Image

The simplest option is to pull a pre-built docker image. The local build option has a couple of extra steps but is straightforward.
  
!!! note
    Docker 19.03.5 was used in preparing these instructions. Use the `docker --version` command to determine which version is running.

---

### Docker Image Pull

The docker image exists in two versions: production and development. The development version supports a debug mode. The production image is a lightweight version of the development image.

Supported architectures:

* AMD64 (x86_64)
* ARM64 (AArch64)

!!! Note
    Details and procedures for working with ARM64 images are found [here](arm64.md). The steps covered in this section are based on the AMD64 x86_64 images.

** Included in the pre-built production image:**

- Binaries for the VPP agent with default config files
- Installation of the compatible VPP

Pull the `pre-built production image` from [DockerHub][dockerhub].

```
docker pull ligato/vpp-agent
```

After the pull, follow the [steps](quickstart.md#2-download-image) in the quickstart guide to install, configure and run the VPP agent.


**Included in the pre-built development image:**

- VPP agent code with default config files including source code
- Compatible VPP installation including source code and build artifacts
- Development environment with all requirements to build the VPP agent and VPP
- Tools to generate code (proto models, binary APIs)

Pull the `pre-built development image` from [DockerHub][dockerhub].

```
docker pull ligato/dev-vpp-agent
```

After the pull, follow the [steps](quickstart.md#2-download-image) in the quickstart guide to install, configure and run the VPP agent.

---


### Local Image Build

!!! Note
    Local image builds in this section were performed under the Linux OS.

Clone the [VPP agent repository][ligato-vpp-agent-repo]:

```
git clone git@github.com:ligato/vpp-agent.git
```

Navigate to the [docker][docker] folder. Inside, choose either the production or development image to work with.

Build the `production image`:
```
make prod-image
```    

Build the `development image`:

```
make dev-image 
```
Build both the `production and development images`:

```
make images 
```

The production option is built with files taken from the `vpp-agent` image. The development option is built with files taken from the `dev_vpp-agent` image.

The scripts recognize the architecture. The correct VPP code source URL and commit ID is taken from the [vpp.env][vpp-env] file located inside the VPP agent repository.


**Image in debug mode**

The development image can be built in `DEBUG` mode:

```
$ VPP_DEBUG_DEB=y ./build.sh
```

This builds an image with debug mode support. Use `RUN_VPP_DEBUG=y` to start the image in debug mode.

---


### Start the Image

Start the VPP agent:

```
sudo docker run -it --name vpp_agent --privileged --rm prod_vpp_agent
```

Note that the VPP agent is executed in `privileged` mode. Several VPP agent operations,  Linux namespace handling is one, require permissions on the target host instance. Running in non-privileged mode may cause the VPP agent to fail to start.

!!! Note
    `vpp_agent` is the name of the container. `vpp-agent` is the actual vpp-agent. `prod_vpp_agent` is the name of the image.

The VPP agent and VPP are started automatically by default.

The following are ENV variables that can be assigned with `docker -e` on image start:

- `OMIT_AGENT` - do not start the VPP agent together with the image
- `RETAIN_SUPERVISOR` - prevents the situation where an unexpected VPP agent or VPP causes the supervisor to quit.

---

## Start VPP and the VPP Agent


**Start VPP**:

Open a terminal and run the following command:
```
sudo docker exec -it vpp_agent bash
```

There are two options for starting VPP.

Start VPP with DPDK:
```
vpp unix { interactive } dpdk { no-pci }
```

Start VPP without DPDK.
```
vpp unix { interactive } plugins { plugin dpdk_plugin.so { disable } }
```

The first option could result in errors and VPP fails to start. Use the second option, `without DPDK`, if that is the case.

---

**Start the VPP Agent**

Open a terminal and run the following command:
```
sudo docker exec -it vpp_agent bash
```

Start the VPP agent:

```
vpp-agent
```

Check the status using [agentctl][agentctl]:
```
agentctl status
```

Access the VPP CLI
```
vppctl -s localhost:5002
```


The VPP agent can be augmented with additional functions and features through the use of [plugins](../plugins/plugin-overview.md). To enable certain plugin features such as database connectivity or messaging, the VPP agent requires plugin-specific static configuration files referred to as `.conf` files.

List the plugin .conf files and optional ENV variable configuration options:
```
vpp-agent -h
```

Details on .conf files can be found [here][config-files].

---


### Connect to a Key-Value Data Store

Configuration information is stored in a [KV data store](concepts.md#key-value-data-store) as key-value pairs. The VPP agent connects to a particular instance of a KV data store, such as etcd, so that it can listen for and receive configuration updates.

**Start etcd:**

The following command starts the etcd server in a docker container. If the image is not present on your localhost, docker will download it first.
```
docker run --rm --name etcd -p 2379:2379 -e ETCDCTL_API=3 quay.io/coreos/etcd /usr/local/bin/etcd -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2380
```

In the default docker environment, the etcd server will be available at `172.17.0.1:2379`. It is possible to change some of the command flags such as the advertise or listen client URLs. More on the etcd command flags can be found in the [etcd configuration guide](https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/configuration.md).

The VPP agent defines a flag for the etcd plugin called `--etcd-config`. It points to the location of the `etcd.conf` file that contains static config information used by the etcd plugin upon startup. The most important item contained is the etcd.conf is the endpoints IP address:port number.

An example configuration contained in the etcd.conf file:

```
insecure-transport: true
dial-timeout: 1000000000
endpoints: 
 - "172.17.0.1:2379"
```

Start the VPP agent with the config flag. This uses the default path to the etcd.conf file which is `/opt/vpp-agent/dev`:
```
vpp-agent --etcd-config=/opt/vpp-agent/dev/etcd.conf
```

More information on the etcd plugin configuration options can be found [here](config-files.md#etcd).

**Example with Kafka (optional):**

Start the Kafka server in a separate container on a localhost:

```
docker run -p 2181:2181 -p 9092:9092 --name kafka --rm --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 spotify/kafka
```

The Kafka server docker image is downloaded, if it does not exist already, started, and ready to consume data. The VPP agent needs the IP address and port number of the Kafka server. This information is defined in the `kafka.conf` file.

An example configuration contained in the kafka.conf file:

```
addrs:
 - "172.17.0.1:9092"
```

Start the VPP agent with the `--kafka-config` flag using the default path to the `kafka.conf` file, which is `/opt/vpp-agent/dev`:
```
vpp-agent --kafka-config=/opt/vpp-agent/dev/kafka.conf
```

More information on the Kafka plugin configuration options can be found [here](config-files.md#kafka).

---

### Executables


The `/cmd` package groups executables that can be built from sources in the VPP agent repository:

**VPP agent** - the default off-the-shelf VPP agent
  executable with no app or extension plugins. It is bundled with
  an off-the-shelf VPP to form the core of a programmable VPP vSwitch.
  
**[agentctl][agentctl]** - CLI tool to manage VPP agents.

Both have been used in the steps above.

---

### Using Multiple VPP Agents

**1. Microservice Label**

Currently, a single instance of a `VPP agent` can serve a single `VPP` instance, while multiple VPP agents can interact with the same KV data store. VPP agent instances are distinguished with the `microservice label`. The label is a part of the KV data store key used to identify configuration data belonging to a specific VPP agent. Note that every VPP agent watches keys containing its microservice label.

The default microservice label is `vpp1`. It can be changed with the ENV variable `MICROSERVICE_LABEL`, or using the [`-microservice-label`](config-files.md#service-label) flag.

It is possible to use the same label to "broadcast" shared configuration data to multiple VPP agents. However, this approach is generally not recommended.

The [concepts][concepts] section of the user guide provides more details on keys, key prefixes, microservice labels and KV data stores.

**2. Shared Memory Prefix**

Running multiple `VPPs` on the same host requires a different shared memory prefix (SHM) to distinguish communication sockets for given VPP instances. In order to connect the VPP agent to the VPP with a custom socket, a valid SHM needs to be provided to the [GoVPP mux plugin][govppmux-plugin].
 
---


## Create Your First Configuration

**Put the configuration into the KV data store:**

Store the values following the given model with the proper key to the KV data store. We recommend the use of etcd as the KV data store.

An example using the etcdctl tool to perform the `put` configuration task can be found in the [quickstart guide.](quickstart.md#5-managing-the-vpp-agent)

Here is an example using [agentctl][agentctl]:

- put a route into the etcd server
- delete the route from the etcd server
```
agentctl kvdb put /vnf-agent/vpp1/config/vpp/v2/route/vrf/1/dst/10.1.1.3/32/gw/192.168.1.13 '{
    "dst_network": "10.1.1.3/32",
	"next_hop_addr": "192.168.1.13"
}'

agentctl kvdb del /vnf-agent/vpp1/config/vpp/v2/route/vrf/1/dst/10.1.1.3/32/gw/192.168.1.13
```

**Use clientv2**

The [clientv2][clientv2] package contains API definitions for the supported configuration items. It can be used to pass data without an external database such as etcd.

Examples of localclient functions in action can be found in the [examples](examples.md) section.



[arm-doc]: https://github.com/ligato/vpp-agent/blob/master/docs/arm64/README.md
[clientv2]: ../user-guide/concepts.md#client-v2
[docker]: https://github.com/ligato/vpp-agent/tree/master/docker
[dockerhub]: https://hub.docker.com/u/ligato
[etcd configuration guide]: https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/configuration.md
[govppmux-plugin]: ../plugins/vpp-plugins.md#govppmux-plugin
[interface-plugin]: ../plugins/linux-plugins.md#interface-plugin
[kv-overview]: ../user-guide/concepts.md#key-value-data-store-overview
[references]: ../user-guide/reference.md
[vpp-env]: https://github.com/ligato/vpp-agent/blob/master/vpp.env
[agentctl]: ../user-guide/agentctl.md#AgentCTL
[config-files]: ../user-guide/concepts.md#plugin-config-files
[concepts]: ../user-guide/concepts.md
[quickstart-guide-51-keys]: ../user-guide/quickstart.md#51-configure-the-vpp-dataplane-using-the-vpp-agent 
[ligato-vpp-agent-repo]: https://github.com/ligato/vpp-agent  

*[DPDK]: Data Plane Development Kit
*[KVDB]: Key-Value Database
*[NAT]: Network Address Translation










