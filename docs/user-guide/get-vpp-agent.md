# VPP Agent Setup

This section shows you how to prepare and run an environment supporting the VPP agent.   

!!! Terminology
    `VPP Agent` refers to a Ligato agent tailored for a VPP data plane. Both run in a single container. `vpp-agent` refers to the actual code, syntax, or container name. `VPP` refers to the VPP data plane.  


---

## Supported VPP versions

The VPP agent supports different VPP versions. You can find out about the compatible VPP versions by referring to the [VPP agent release notes](https://github.com/ligato/vpp-agent/releases) and the [vpp.env environment variable.](https://github.com/ligato/vpp-agent/blob/master/vpp.env) 

---

## Set up your environment

You have two options to set up your environment:

* Pre-built docker image pull from [dockerhub][dockerhub].
* Build a Local Image.

Pulling a pre-built docker image is the simplest option. The local build option includes a couple of extra steps.
  
!!! note
    The instructions below use Docker 19.03.5. Use the `docker --version` command to determine the version you are running.

---

### Docker image pull

The docker image exists in two versions: 

- **Pre-built development** - Used for development purposes and supports a debug mode.
<br></br>
- **Pre-built production** -  Lightweight version of the development image.

---

You can run the VPP agent in the following supported architectures:

* AMD64 (x86_64)
* ARM64 (AArch64)

!!! Note
    For more details on working with ARM64 (AArch64), see [ARM64 considerations](#arm64-considerations).

---

#### Pre-built production image

The pre-built production image comes with the following:

- VPP agent binaries with default config files.
- Compatible VPP.

---

**Installation**

- Open a new terminal session.
<br></br>
- Pull the pre-built production image from [DockerHub][dockerhub]:

```
docker pull ligato/vpp-agent
```

---

#### Pre-built development image

The pre-built development image comes with the following:

- VPP agent code with default config files and source code.
- Compatible VPP installation including source code and build artifacts.
- Development environment to build the VPP agent and VPP.
- Tools to generate code that include models, protos, and binary APIs.

---

**Installation**

- Open a new terminal session.
<br></br>
- Pull the pre-built development image from [DockerHub][dockerhub]:

```
docker pull ligato/dev-vpp-agent
```

---

### Local image build

!!! Note
    Local image build steps described below use the Linux OS.

---

Clone the [VPP agent repository][ligato-vpp-agent-repo]:

```
git clone git@github.com:ligato/vpp-agent.git
```

---

Navigate to the [docker][docker] folder. You can choose either the production image or development image to work with.

Build the production image:
```
make prod-image
```    

---

Build the development image:

```
make dev-image 
```

---

Build both images: 

```
make images 
```

---

The build scripts perform the following:

- If you chose the production image, builds the production image with files taken from the `vpp-agent`.
<br></br>
- If you chose the development image, builds the development image with files taken from the `dev-vpp-agent`. 
<br></br>
- Recognizes the AMD64 or ARM64 architecture.
<br></br>
- Uses the [vpp.env value][vpp-env] to determine the correct VPP source code URL and commit ID.

---

**Development image in debug mode**

Build the development image in `DEBUG` mode:

```
$ VPP_DEBUG_DEB=y ./build.sh
```

Start the development image in debug mode using the `RUN_VPP_DEBUG` environment variable:
```
sudo docker run -it --name dev-vpp-agent --privileged -e RUN_VPP_DEBUG=y --rm ligato/dev-vpp-agent
``` 

---


### Start the image

!!! Note
    You cannot start VPP agent pre-built images unless you have fist installed and started an etcd or Kafka data store. See [Connect to a Key-Value Data Store](#connect-to-a-key-value-data-store) for more details.

---

Start the VPP agent:

```
sudo docker run -it --name vpp-agent --privileged --rm ligato/vpp-agent
```

The VPP agent and VPP start automatically by default in a single container.

---

Note that the VPP agent executes in `privileged` mode. Several VPP agent operations, such as Linux namespace handling, require permissions on the target host instance. Running in non-privileged mode may cause the VPP agent to fail to start.

!!! Note
    `vpp-agent` is the name of the container. `ligato/vpp-agent` is the name of the image.

---

You can assign the following environment variables with `docker -e` at image start:

- `OMIT_AGENT` - do not start the VPP agent together with the image:
```
sudo docker run -it --name vpp-agent --privileged -e OMIT_AGENT --rm ligato/vpp-agent
```
- `RETAIN_SUPERVISOR` - prevents the situation where an unexpected VPP agent, or VPP, causes the supervisor to quit:
```
sudo docker run -it --name vpp-agent --privileged -e RETAIN_SUPERVISOR --rm ligato/vpp-agent
```

---

## Start VPP and the VPP agent

You can separately start VPP and the VPP agent.

---

**Start VPP**

Open a terminal session:
```
sudo docker exec -it vpp-agent bash
```

---

You two options for starting VPP:

Start VPP with DPDK:
```
vpp unix { interactive } dpdk { no-pci }
```

---

Start VPP without DPDK.
```
vpp unix { interactive } plugins { plugin dpdk_plugin.so { disable } }
```

!!! Note
    The **VPP with DPDK** option could result in errors and VPP fails to start. If that is the case,  use the **VPP without DPDK** option.

---

**Start the VPP Agent**

Open a terminal session:
```
sudo docker exec -it vpp-agent bash
```

Start the VPP agent:

```
vpp-agent
```

---

Check the status using [agentctl][agentctl]:
```
agentctl status
```

---

Access the VPP CLI
```
vppctl -s localhost:5002
```


You can implement additional functions and features using VPP agent plugins. 

To learn more about VPP agent plugins, including how to build your own custom plugins, see [plugins](../plugins/plugin-overview.md) and [custom plugin](../plugins/custom-plugin.md).

---


### Connect to a key-value data store

Ligato stores VPP agent configuration information in a [KV data store](concepts.md#key-value-data-store) as key-value pairs. The VPP agent uses the [etcd plugin](../plugins/db-plugins.md#etcd) to communicate with the etcd server. The VPP agent listens for configuration updates published by the KV data store. 

The instructions below use etcd and Kafka as two examples of KV data stores. 

To learn more about data stores supported by Ligato, see the following:

- [Supported KV data stores](../user-guide/concepts.md#supported-kv-data-stores). 
- [Database plugins](../plugins/db-plugins.md).

---

#### Start etcd


Open a terminal session. Start the etcd server in a docker container. If you don't have etcd on your localhost, docker downloads it for you.
```
docker run --rm --name etcd -p 2379:2379 -e ETCDCTL_API=3 quay.io/coreos/etcd /usr/local/bin/etcd -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379
```

---

In the default docker environment, the VPP agent communicates with the etcd server at `172.17.0.1:2379`. You can change some etcd parameters with command flags. 
 
To review supported etcd command flags, see the [etcd configuration guide](https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/configuration.md).

---

The VPP agent defines a flag for the etcd plugin called `--etcd-config`. This flag points to the location of the `etcd.conf` file that contains information used by the etcd plugin at startup.

Example `etcd.conf` file:

```
insecure-transport: true
dial-timeout: 1s
endpoints: 
 - "172.17.0.1:2379"
```

You can start the VPP agent and point to the `/opt/vpp-agent/dev` directory containing the `etcd.conf` file:
```
vpp-agent --etcd-config=/opt/vpp-agent/dev/etcd.conf
```

To review etcd plugin configuration options, see [etcd conf files](config-files.md#etcd).

---

#### Start kafka 

Open a terminal session. Start Kafka in a docker container. If you don't have Kafka on your localhost, docker downloads it for you.  

```
docker run -p 2181:2181 -p 9092:9092 --name kafka --rm --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 spotify/kafka
```

In the default docker environment, the VPP agent communicates with the kafka server at `172.17.0.1:9092`.
You can define a different Kafka IP address:port number in the `kafka.conf` file.

Example `kafka.conf` file:

```
addrs:
 - "172.17.0.1:9092"
```

You can start the VPP agent and point to the `/opt/vpp-agent/dev` directory containing the `kafka.conf` file:
```
vpp-agent --kafka-config=/opt/vpp-agent/dev/kafka.conf
```

For more information on Kafka plugin configuration options, see [kafka conf file](config-files.md#kafka).

---

## Executables


The [`cmd` package](https://pkg.go.dev/github.com/ligato/vpp-agent@v1.8.1/cmd) groups executables built from sources in the VPP agent repository. 

The [cmd/ folder](https://github.com/ligato/vpp-agent/tree/master/cmd) contains the executables.

```
├── agentctl
├── doc.go
├── vpp-agent
└── vpp-agent-init
```

- **agentctl** - [agentctl](../user-guide/agentctl.md).
<br></br>
- **vpp-agent** - default off-the-shelf VPP agent executable without app or extenstion plugins. Bundled with off-the-shelf VPP.
<br></br>
- **vpp-agent-init** - VPP agent init container structured as a generic agent. 


---

## Using multiple VPP agents

!!! Note
    Multiple VPP agents means multiple VPP agents talking to a single KV data store instance. [Multi-Version](../user-guide/concepts.md#multi-version) refers to the case where your VPP agent communicates with a specific version of VPP.

---

**Microservice Label**

Ligato supports the ability to run multiple VPP agents in your deployment. A single instance of a VPP agent serves a single instance of VPP. However, you might have multiple VPP agents talking to a single KV data store. 

You distinguish VPP agents by using a `microservice-label`. You assign a unique label to each of your VPP agents. Ligato uses the microservice-label to form KV data store keys. Each key identifies configuration information belonging to a specific VPP agent. 

The default microservice label is `vpp1`. You can modify this value using one of the following options:


- Set the command line flag `microservice-label`.
<br></br>
- Set the `MICROSERVICE_LABEL` environment variable.
<br></br>
- Use the [service label conf file](config-files.md#service-label).

You could use the same label to "broadcast" shared configuration data to multiple VPP agents. However, this approach is generally not recommended.

For more details on microservice labels, see [keys and microservice labels](../user-guide/concepts.md#keys-and-microservice-label).

---

**Shared Memory Prefix**

You can run multiple VPP instances on the same host. This requires a different shared memory prefix (SHM) to distinguish communication sockets for given VPP instances. In order to connect the VPP agent to VPP with a custom socket, the [GoVPP mux plugin][govppmux-plugin] must define a valid SHM-prefix.

To learn more about shared memory prefixes, see [GoVPP mux plugin][govppmux-plugin].  
 
---


## Create your first configuration

**Put configuration to the KV data store:**

Use [agentctl][agentctl] to perform the following:

- Put a route to the etcd server:

```
agentctl kvdb put /vnf-agent/vpp1/config/vpp/v2/route/vrf/1/dst/10.1.1.3/32/gw/192.168.1.13 '{
    "dst_network": "10.1.1.3/32",
	"next_hop_addr": "192.168.1.13"
}'
```

---

- Delete the route from the etcd server:
```
agentctl kvdb del /vnf-agent/vpp1/config/vpp/v2/route/vrf/1/dst/10.1.1.3/32/gw/192.168.1.13
```

To look over other configuration examples using the KV data store, see the following:

- [Quickstart](../user-guide/quickstart.md#5-managing-the-vpp-agent).
<br></br>
- Plugin Configuration Examples in [Plugins](../plugins/plugin-overview.md).

---

**Use Client v2**

The [clientv2][clientv2] package contains API definitions for the supported configuration items. You can use client v2 to pass configuration data without the use of an external KV data store.

To look over client v2 examples, see [examples](examples.md).

---

## ARM64 considerations

You can run the VPP agent on an ARM64 architecture. The procedures for VPP agent ARM64 mirror those outlined above for the AMD64 (x86_64) architecture. However, there are two exceptions: etcd and Kafka installation and startup. 

!!! Note
    The ARM64 docker images have not been updated since early 2020. Check with the Ligato development team on slack or github issues to find out more about ARM64 support and image status.

---
    
### ARM64 docker image pull

Pull the pre-built ARM64 production image:
```json
docker pull ligato/vpp-agent-arm64
```

Pull the pre-built ARM64 development image:
```json
docker pull ligato/dev-vpp-agent-arm64
```

---

### Start VPP agent ARM64

!!! Note
    You cannot start VPP agent ARM64 pre-built images unless you have fist installed and started an etcd or Kafka data store. See [etcd ARM64](#etcd-arm64) and [Kafka ARM64](#kafka-arm64) for more details.

---

Start Production Image
```json
docker run -it --rm --name vpp-agent-arm64 -p 5002:5002 -p 9191:9191 --privileged ligato/vpp-agent-arm64
```
Start Development Image
```json
docker run -it --rm --name dev-vpp-agent-64 -p 5002:5002 -p 9191:9191 --privileged ligato/dev-vpp-agent-arm64
```

---

Check the status using [agentctl][agentctl]:
```json
docker exec -it vpp-agent-arm64 agentctl status
```
---

### etcd ARM64

!!! note
    Check for the proper etcd ARM64 docker image in the [official repository][etcd]. You must use the parameter `-e ETCD_UNSUPPORTED_ARCH=arm64` when installing etcd.
    
Open a terminal session. Start the etcd server in a docker container. If you don't have etcd on your localhost, docker downloads it for you.
```
docker run -p 2379:2379 --name etcd -e ETCDCTL_API=3 -e ETCD_UNSUPPORTED_ARCH=arm64 \
    quay.io/coreos/etcd:v3.3.8-arm64 /usr/local/bin/etcd \
    -advertise-client-urls http://0.0.0.0:2379 \
    -listen-client-urls http://0.0.0.0:2379
```

---

### Kafka ARM64

An official spotify/kafka image for the ARM64 platform is not supported. However, you can build a kafka-arm64 image following the steps outlined in the [Kafka repository][kafka].

---

Modify the Kafka/Dockerfile before building the image:
```
//FROM java:openjdk-8-jre
FROM openjdk:8-jre
...
// ENV KAFKA_VERSION 0.10.1.0
ENV KAFKA_VERSION 0.10.2.1
...
```

---

Prepare the kafka-arm64 image:

```
git clone https://github.com/spotify/docker-kafka.git
cd docker-kafka
sed -i -e "s/FROM java:openjdk-8-jre/FROM openjdk:8-jre/g" kafka/Dockerfile
sed -i -e "s/ENV KAFKA_VERSION 0.10.1.0/ENV KAFKA_VERSION 0.10.2.1/g" kafka/Dockerfile
docker build -t kafka-arm64 kafka/
```

---

Start Kafka in a separate container:
```
sudo docker run -p 2181:2181 -p 9092:9092 --name kafka --rm \
 --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 kafka-arm64
```





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
[kafka]: https://github.com/spotify/docker-kafka#build-from-source
[etcd]: https://quay.io/repository/coreos/etcd?tag=latest&tab=tags
*[DPDK]: Data Plane Development Kit
*[KVDB]: Key-Value Database
*[NAT]: Network Address Translation










