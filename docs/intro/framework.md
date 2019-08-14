# Framework

---

## Context

Ligato is THE software platform for building and developing VPP-based cloud native network functions (CNFs). Because VPP is the preferred dataplane (linux kernel also supported), one way to view the architecture would be through the lens of a protocol stack. Now this is not necessarily your classic 7-layer OSI model but rather one that has evolved over the last few years. Applications pass on their intent to orchestrators such as Kubernetes. The orchestrators consume the intent and in cahoots with APIs and other software collaborators (e.g. kubectl, CNI, etc.) more or less convert it into low-level commands to configure network resources. 

With VPP a bit more precision and flexibility is required. This is because VPP supports a broad suite of L2/L3/L4 network functions and services. High performance L2 bridge domains interconnecting application pod via tunnels are supported. ACLs, NATs and IPsec tunnels can be provisioned and even wired up into K8s services and policies along with service functions. All with scale, performance and resiliency. Of course one size _does not fit_ all and it is a certainty that developers will build CNFs of different shapes, sizes and abilities exploiting the VPP dataplane in one way or another. 

However _one size can fit all_ when talking about a software platform, toolchains, programming languages, resource models, APIs, management and control. It really makes no sense to 'one-off' the CNF software ecosystem. It just leads to a slew of avoidable challenges including pidgeon-holed resources, code duplication, closed APIs, a high degree of partial function duplication and mis-matched orchestrator/resource interactions. Survivability and evolvability are called into question.   

A subtle point to note is that while all of these wonderful VPP functions are available to CNFs, the dynamics and velocity of cloud native enabled configuration starts/restarts/stops driven by available resource gravity fields requires a new configuration model. It needs to be fast, accurate, resilient, lightweight and all the while handle that delicate balancing act between application intent and run-time network configuration. Time is not our friend here. Guesswork is problematic. And an all-knowing centralized oracle is not the answer.   

## Ligato Architecture Stack

As suggested, we can view and describe the Ligato framework as a protocol Stack. See the `Ligato Architecture Stack` below. At the bottom we have the VPP and Linux kernel dataplanes. 

!!! Note 
    While much of the Ligato development effort and software is focused on VPP, there is and certainly can be new CNFs (or even non-CNFs) that employ the linux kernel dataplane. The `one size can fit all` discussion applies here with respect to dataplane realities in the CNF universe.

![ligato framework][ligato-framework]
<p style="text-align: center; font-weight: bold">Ligato Architecture Stack</p>

On the top, are the applications with the assumption that over time, they will be developed and deployed as microservices inside containers residing in pods deployed on VMs or bare metal servers all under the purview of Kubernetes.

Ligato resides in the middle. Explanations follow.


### Infra

Each management/control plane app built on top of the Infra framework (also referred to as cn-infra) is a set of modules called "plugins". Each plugin provides a very specific/focused functionality. Some plugins come with the cn-infra framework; some are written by the app developers. In other words, the cn-infra framework is made up of a set of plugins that together define the framework's functional potential. In the cloud native world of software development platforms, Ligato offers tablestakes: logging, health checks, messaging (e.g. Kafka), a common front-end API and back-end connectivity to various KV data stores (etcd, Cassandra, Redis, etc. ), and REST and gRPC APIs.

Infra provides plugin lifecycle management (initialization and graceful shutdown of plugins) and a set of framework plugins exposing APIs that in turn, app plugins can employ. App plugins themselves may provide their own APIs consumed by external clients. See the `CN-infra` figure below.

![cn-infra][infra]
<p style="text-align: center; font-weight: bold">CN-infra</p>

The framework is modular and extensible. Plugins supporting new functionality (e.g. another KV store or another message bus) can be easily swapped or spliced in to the existing set of Infra framework plugins. Moreover, Infra-based apps can be built in layers: a set of app plugins together with new Infra plugins can form a new framework providing APIs/services to higher layer apps. This approach was used in the building the vpp-agent discussed below.

Extending the code base does not mean that all plugins end up in all apps - app writers can pick and choose only those framework plugins that interest them. For example, if an app does not need a KV store, the Infra framework KV data store plugins need not be included in the app. Simple - use what you need.

### VPP Agent

The VPP Agent (referred to as the vpp-agent) is a set of VPP-specific plugins that leverage the Ligato Infra framework to interact with other services/microservices in a cloud native environment (e.g. a KV data store, messaging, telemetry, etc.). The vpp-agent exposes VPP dataplane functionality to client app components via a higher-level model-driven API. Clients that consume this API may be either external (connecting to the vpp-agent via REST, gRPC API, etcd or message bus transport), or local apps and/or extension plugins running on the same Ligato infra framework in the same Linux process. See the `ligato vpp-agent` figure below.

 
![vpp-agent][vpp-agent-new]
<p style="text-align: center; font-weight: bold">Ligato vpp-agent</p>
  

Each (northbound) VPP API is implemented by a vpp-agent plugin, which translates northbound (NB) API calls/operations into southbound (SB) low level VPP Binary API calls. Northbound APIs are defined using protobufs, which allow for the same functionality to be accessible over multiple transport protocols (HTTP, gRPC, etcd, ...). The  GoVPP library is used to interact with VPP.


### KVScheduler

Applications are packaged up into containers. The nature of cloud native networks means these containers can be started, restarted, stopped, added, removed at any time. A VPP-based CNF is no exception and is, like any pod or containerized microservice, subject to the whims of the K8s orchestrator and its own configuration idiosyncrasies.

This introduces a bevy of challenges:

- Both data plane and control plane can be implemented as a set of microservices independent of each other. Therefore, the overall configuration of a system may be incomplete at times, since one object may refer to another object owned by a service that has not been instantiated yet. 
- VPP dataplane is pretty strict on the correct sequence of the configuration VPP binary API calls.
- Removal of a dependency configuration item, unless performed in the correct sequence could appear successful on the outside, but under the covers, inconsistent VPP behavior lurks.
- Asking each vpp-agent plugin to resolve the configuration dependency problem creates too much complexity. Plugins needed to talk to other plugins.  Not feasible as system scales.
- Non-standard methods for tracking and logging the configuration item transactions, caches, errors etc.
- Synchronizing the northbound (NB) intent with the southbound (SB) config state is difficult.

Thanks to the KVScheduler, these challenges are addressed. This is a core Ligato plugin that works with vpp-agents and linux agents on the SB side, orchestrators/external data sources (i.e. KV data stores, RPC clients) on the NB side. In a nutshell, it keeps track of the correct configuration order along with any dependencies. It will cache any configuration items until all dependencies are resolved.    

![kvs-arch-pict][ligato-kvs-arch] 
<p style="text-align: center; font-weight: bold">Ligato KVScheduler</p>
 
KVScheduler builds a graph of vertices and edges. The vertices represent configuration items. The edges represent relationships (i.e. dependency, derived from) between the configuration items. Each configuration item has its own set of KVDescriptors which are in essence pointers to CRUD callback operations. Thus the KVSchedulers does not need to know the low-level details of configuration items. 

The KVScheduler walks the tree to sequence the correct configuration actions. It builds a transaction plan that drives CRUD operations to the vpp-agent in the SB direction. Configuration items can be cached until outstanding dependencies are resolved. 

NB partial, SB partial or full state reconciliation (synchronization) is supported. And transaction plans, cached values, errors and so are exposed via REST APIs.

Note that any plugin that requires an object that is dependent on another object can leverage the KVSscheduler.   

### Ligato Plugins

The ethos of a component-based architecture ("plugginability") in software platforms has emerged as the defacto standard with React and Go being two popular examples. With its diverse assortment of plugins, the Ligato framework has adopted this very same notion. The benefits bequethed to CNF developeprs are many:

- pick and choose only those plugins needed. 
- flexibility to swap in or out different plugins performing similar functions. 
- create app plugins compatible with Ligato plugins.

Ligato plugins are defined using common pattern making it straightforward to understand existing plugins as well as create new ones. In addition config files are used to define plugin functionality at startup.


[infra]: ../img/intro/ligato-framework-arch-infra.svg
[context]: ../img/intro/context.png "VPP Agent & Plugins on top of CN-infra"
[contiv]: http://contiv.github.io/
[deployment-embeded]: ../img/intro/deployment_embeded.png
[deployment-with-ds]: ../img/intro/deployment_with_data_store.png
[grpc-nb]: ../img/intro/deployment_nb_grpc.png
[k8s-integ]: ../img/intro/k8s_deployment.png "VPP Agent - K8s integration"
[kubernetes]: https://kubernetes.io/
[ligato-framework]: ../img/intro/ligato-framework-arch2.svg
[ligato-kvs-arch]: ../img/intro/ligato-framework-arch-KVS2.svg
[protobuf]: https://developers.google.com/protocol-buffers/
[vpp-agent]: ../img/intro/vpp_agent.png "VPP Agent & plugins on top of CN-infra"
[vpp-agent-new]: ../img/intro/ligato-framework-vpp-agent2-picture.svg
[vpp-agent-10k]: ../img/intro/vpp_agent_10K_feet.png

*[ACL]: Access Control List
*[DPDK]: Data Plane Development Kit
*[HTTP]: Hypertext Transfer Protocol
*[REST]: Representational State Transfer
*[VNF]: Virtual Network Function
*[VPP]: Vector Packet Processing