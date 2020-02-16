# Framework

---


## Ligato Architecture Stack

The Ligato framework can be viewed as a protocol Stack. At the bottom we have the VPP and Linux kernel data planes.




![ligato framework][ligato-framework]
<p style="text-align: center; font-weight: bold">Ligato Architecture Stack</p>

On the top, are applications that make use of the functions enabled by Ligato. From an architecture perspective, this is a _placeholder_ for any number of different application possibilities.

Examples are:

- Simple ACL configuration tool
- IKE control plane for an IPsec VPN gateway
- Order fulfillment process that constructs a service function chain delivered to a customer as a communication service.

Ligato is agnostic to the composition and functions implemented in an application. Simply put, Ligato is composed of building blocks used by developers to assemble a solution.

Returning to the stack diagram, Ligato is slotted in the middle. Explanations of its components follow.


### cn-infra

cn-infra and the vpp-agent are the two constituent frameworks that together, form the basis of the Ligato framework. With Ligato, each management or control plane applicaton (app) can utilize one or modules called plugins. Each plugin provides a specific function or functions. Some plugins come with the cn-infra framework; Others come with the vpp-agent; Yet others can be created by app developers performing a custom task.

cn-infra can be decomposed into a suite of plugins supporting functions present in modern cloud-native CNFs and apps. cn-infra plugins offer logging, health checks, messaging, KV data store connectivity, and REST and gRPC APIs. Developers can implement a mix of cn-infra plugins, that together with other vpp-agent and/or custom plugins, define the CNF functionality. cn-infra provides plugin lifecycle management such as initialization and graceful shutdown.

![cn-infra][infra]
<p style="text-align: center; font-weight: bold">CN-infra</p>

The framework is modular and extensible. Plugins supporting new functionality (e.g. another KV store or another message bus) can be swapped in or out as needed. Moreover, cn-ifra-based apps can be built in layers; app plugins together with new cn-infra plugins can form a new foundation providing APIs or services to higher layer apps. This approach was used in constructing the vpp-agent discussed below.

Extending the code base does not mean that all plugins are present in all apps. Again, developers can cherry-pick the plugins best suited for their management or control plane app. For example, if an app does not require a KV store, cn-infra's  KV data store plugins need not be included.

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