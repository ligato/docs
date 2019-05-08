# Framework

---

## Architecture

The VPP Agent is basically a set of VPP-specific plugins that use the CN-Infra framework to interact with other services/microservices in the cloud (e.g. a KV data store, messaging, log warehouse, etc.). The VPP Agent
exposes VPP functionality to client apps via a higher-level model-driven API. Clients that consume this API may be either external (connecting to the VPP Agent via REST, gRPC API, ETCD or message bus transport), or local Apps and/or Extension plugins running on the same CN-Infra framework in the same Linux process. 

The VNF Agent architecture is shown in the following figure: 

![vpp agent][vpp-agent]

Each (northbound) VPP API - L2, L3, ACL, ... - is implemented by a specific VNF Agent plugin, which translates northbound API calls/operations into (southbound) low level VPP Binary API calls. Northbound APIs are defined using protobufs, which allow for the same functionality to be accessible over multiple transport protocols (HTTP, gRPC, ETCD, ...). Plugins use the  GoVPP library to interact with the VPP.

The following figure shows the VPP Agent in context of a cloud-native VNF, where the VNF's data plane is implemented using VPP/DPDK and its management/control planes are implemented using the VNF agent:

![context][context]

## Design

![VPP agent 10.000 feet][vpp-agent-10k]

Brief description:
* **SFC Controller** - renders a logical network topology (which defines how VNF are logically interconnected) onto a given physical network and VNF placement; the rendering process creates configuration for VNFs and network devices.
  
* **Control Plane Apps** - renders specific VPP configuration for multiple agents to the Data Store

* **Clientv1** - VPP Agent clients (control plane apps & tools) can use the clientv1 library to interact with one or more VPP Agents. The clientv1 library is based on generated GO structures from protobuf messages. The library contains helper methods for working with KV Data Stores, used for generation of keys and storage/retrieval of configuration data to/from Data Store under a given key.

* **Data Store** (ETCD, Redis, etc.) is used to:
  * store VPP configuration; this data is put into the Data Store by one or more orchestrator apps and retrieved by one or more VPP Agents.
  * store operational state, such as network counters /statistics or errors; this data is put into the Data Store by one or more VPP Agents and retrieved by one or more monitoring/analytics applications.

* **VPP vSwitch** - privileged container that cross-connects multiple VNFs

* **VPP VNF** - a container with a Virtual Network Function implementation 
  based on VPP

* **Non VPP VNF** - container with a non-VPP based Virtual Network Function; that can nevertheless interact with VPP containers (see below MEMIFs, VETH)

* **Messaging** - AD-HOC events (e.g. link UP/Down)

### Requirements

The VPP Agent was designed to meet the following requirements:

* Modular design with API contracts
* Cloud native
* Fault tolerant
* Rapid deployment
* High performance & minimal footprint

### Modular design with API contract

The VPP Agent codebase is basically a collection of plugins. Each plugin provides a specific service defined by the service's API. VPP Agent's functionality can be easily extended by introducing new plugins. Well-defined API contracts facilitate seamless integration of new plugins into the VPP Agent or integration of multiple plugins into a new application.

### Cloud native

*Assumption*: both data plane and control plane can be implemented as a set of microservices independent of each other. Therefore, the overall configuration of a system may be incomplete at times, since one object may refer to another object owned by a service that has not been instantiated yet. The VPP agent can handle this case - at first it skips
the incomplete parts of the overall configuration, and later, when the configuration is updated, it tries again to configure what has been previously skipped.

The VPP Agent is usually deployed in a container together with VPP. There can be many of these containers in a given cloud or in a more complex data plane implementation. Containers can be used in many different infrastructures (on-premise, hybrid, or public cloud). The VPP + VPP Agent containers have been tested with [Kubernetes][kubernetes].

Control Plane microservices do not really depend on the current lifecycle phase of the VPP Agents. Control Plane can render config data for one or more VPP Agents and store it in the KV Data Store even if (some of) the VPP Agents have not been started yet. This is possible because:

- The Control Plane does not access the VPP Agents directly, instead it accesses the KV Data Store; the VPP Agents are responsible for downloading their data from the Data Store.
- Data structures in configuration files use logical object names, not internal identifiers of the VPP). See the 
  [protobuf][protobuf] definitions in the `model` sub folders in various VPP Agent plugins. 

### Fault tolerance

Each microservice has its own lifecycle, therefore the VPP Agent is designed to recover from HA events even when some subset of microservices (e.g. db, message bus...) are temporary unavailable.

The same principle can be applied also for the VPP Agent process and the VPP process inside one container. The VPP Agent checks the VPP actual configuration and does data synchronization by polling latest configuration from the KV Data Store.

VPP Agent also reports status of the VPP in probes & Status Check Plugin.  

In general, VPP Agents:

 * propagate errors to upper layers & report to the Status Check Plugin 
 * fault recovery is performed with two different strategies:
   * easily recoverable errors: retry data synchronization (Data Store configuration -> VPP Binary API calls)
   * otherwise: report error to control plane which can failover or recreate the microservice

### Rapid deployment

Containers allow to reduce deployment time to seconds. This is due to the fact that containers are created at process level and there is no need to boot an OS. Moreover, K8s helps with the (un)deployment of possibly different versions of multiple microservice instances.

### High performance & minimal footprint

Performance optimization is currently a work-in-progress.
Several bottlenecks that can be optimized have been identified:
- GOVPP
- minimize context switching
- replace blocking calls to non-blocking (asynchronous) calls

## Deployment

VPP Agent can run on any server where VPP is installed. It can run on a bare metal, in a VM, or in a container.
 
Benefits of putting a VPP-based VNF (basically just a combination of VPP and the VPP Agent) into a container are:
 * simplified upgrades and startup/shutdown, which results in improved scalability
 * Container-based VNFs are in essence lightweight and reusable data-plane microservices which can be used to build larger systems and applications
 * supports container healing 
 
### K8s integration

The following diagram shows VPP deployement in:
- Data Plane vSwitch
- Control Plane vSwitch ([Contiv][contiv] integration)
- VPP VNF Container
- Non-VPP Container

![K8s integration][k8s-integ]

K8s:
- starts/stops the containers running on multiple hosts
- checks health of the individual containers (using probes - HTTP calls)

### NB (North-bound) configuration vs. deployment

VPP + Agent can be deployed in different environments. Several deployment alternatives are briefly described in the following sub-chapters. Regardless of the deployment, the VPP Agent can be configured using the same Client v1 interface. There are three different implementations of the interface:
 - local client
 - remote client using Data Broker
 - remote client using GRPC

### Key Value Data Store for NB (north-bound)

The Control Plane using remote client writes (north-bound) configuration to the Data Store (tested with ETCD, Redis). VPP Agent watches dedicated key prefixes in Data Store using dbsync package.

![deployment with data store][deployment-with-ds]

### GRPC for NB (north-bound)

The Control Plane using remote client sends configuration (north-bound) via Remote Procedure Call (in this case GRPC). VPP Agent exposes GRPC service.

![grpc northbound][grpc-nb]

### Embedded deployment

VPP Agent can be directly embedded into a different project. For integration with [Contiv][contiv] we decided to
embed the VPP Agent directly into the netplugin process as a new driver for VPP-based networking. Using the Local client v1 the driver is able to propagate the configuration into the Agent via Go channels.

![embeded deployment][deployment-embeded]
TBD links to the code

[context]: ../img/intro/context.png "VPP Agent & Plugins on top of CN-infra"
[contiv]: http://contiv.github.io/
[deployment-embeded]: ../img/intro/deployment_embeded.png
[deployment-with-ds]: ../img/intro/deployment_with_data_store.png
[grpc-nb]: ../img/intro/deployment_nb_grpc.png
[k8s-integ]: ../img/intro/k8s_deployment.png "VPP Agent - K8s integration"
[kubernetes]: https://kubernetes.io/
[protobuf]: https://developers.google.com/protocol-buffers/
[vpp-agent]: ../img/intro/vpp_agent.png "VPP Agent & plugins on top of CN-infra"
[vpp-agent-10k]: ../img/intro/vpp_agent_10K_feet.png

*[ACL]: Access Control List
*[DPDK]: Data Plane Development Kit
*[HTTP]: Hypertext Transfer Protocol
*[REST]: Representational State Transfer
*[VNF]: Virtual Network Function
*[VPP]: Vector Packet Processing