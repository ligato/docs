# Framework

---


## Ligato Architecture Stack

The Ligato framework can be framed as a protocol stack. At the bottom we have the VPP and Linux kernel data planes.




![ligato framework][ligato-framework]
<p style="text-align: center; font-weight: bold">Ligato Architecture Stack</p>

On top, are applications that make use of the functions enabled by Ligato. From an architecture perspective, this is a _placeholder_ for any number of different application possibilities.

Examples are:

- Simple ACL configuration tool
- IKE control plane for an IPsec VPN gateway
- Order fulfillment process that constructs a service function chain delivered to a customer as a communication service

Ligato is agnostic to the composition and functions implemented in an application. Simply put, Ligato is made up of software building blocks used by developers to assemble a solution.

Returning to the stack diagram, Ligato is slotted in the middle. Explanations of its components follow.


### cn-infra

cn-infra and the vpp-agent are the two constituent frameworks that together, form the basis of the Ligato framework. With Ligato, each management or control plane applicaton (app)utilize one or modules called plugins. Each plugin provides a specific function or functions. Some plugins come with the cn-infra framework; Others come with the vpp-agent; Yet others created by app developers perform custom tasks.

cn-infra can be decomposed into a suite of plugins supporting capabilities present in modern cloud-native CNFs and apps. Plugins offering logging, health checks, messaging, KV data store connectivity, REST and gRPC APIs are available. Developers can implement a mix of cn-infra plugins, that together with other vpp-agent and/or custom plugins, define CNF functionality. cn-infra provides plugin lifecycle management such as initialization and graceful shutdown.

![cn-infra][infra]
<p style="text-align: center; font-weight: bold">CN-infra</p>

The framework is modular and extensible. Plugins supporting new functionality (e.g. another KV store or another message bus) can be swapped in or out as needed. It is possible to build cn-infra-based apps in layers; App plugins together with new cn-infra plugins can form a new foundation providing APIs or services to higher layer apps. This approach was used in constructing the vpp-agent discussed below.

Extending the code base does not mean that all plugins are present in all apps. Again, developers can cherry-pick the plugins best suited for their management or control plane app. For example, if an app does not require a KV store, cn-infra's  KV data store plugins need not be included.

### VPP Agent

The VPP Agent (vpp-agent) is composed of a set of VPP-specific plugins. Used in combination with cn-infra, CNFs and apps can be developed to interact with other services/microservices in a cloud-native environment. For example, a CNF might need to interact with an external KV data store to receive configuration updates.

The vpp-agent exposes VPP functionality to client app components via a higher-level model-driven API construct. Clients that consume this API are:

 - external such as connecting to the vpp-agent through a variety of methods such as REST/gRPC APIs, etcd or message bus transport
 - local including apps and/or extension plugins operating in the same Linux process as one example

 
![vpp-agent][vpp-agent-new]
<p style="text-align: center; font-weight: bold">vpp-agent</p>
  

Each (northbound) VPP API is implemented by a vpp-agent plugin. Northbound (NB) API calls and operations are translated into southbound (SB) low level VPP Binary API calls. Northbound APIs are defined using protobufs. GoVPP is used to interact with VPP in the southbound direction.


### KVScheduler

The runtime configuration state of a CNF must accurately reflect the desired network function. In just about all cases, this involves programming multiple configuration items such as an interface and a route. Moreover, one configuration item could be dependent on the successful configuration of another item. Example: a route cannot be configured without an interface, therefore we say the interface is a dependency configuration item of the route.

CNF configuration dependency constraints can pose unique challenges:

* Both data plane and control plane can be implemented as microservices independent of one another. The system configuration may be incomplete at times, since one object may refer to another object owned by a service that has not been instantiated.
* Inconsistent VPP behavior if the strict sequence of configuration actions is not adhered to involving VPP binary API calls, and dependency configuration item removal.
* Not feasible or scalable for vpp-agent plugins to communicate with each other to resolve dependencies.
* No standard methods for tracking and logging configuration item activity such as transactions, caches and errors.
* Synchronizing northbound (NB) intent with southbound (SB) configuration state is difficult.

The KVScheduler was implemented to address these challenges. It is a plugin that works with vpp-agents and Linux agents on the SB side, and orchestrators/external data sources such as KV data stores and RPC clients on the NB side. In a nutshell, it keeps track of the correct configuration order and associated dependencies.

![kvs-arch-pict][ligato-kvs-arch] 
<p style="text-align: center; font-weight: bold">KVScheduler</p>
 
Internally, the KVScheduler builds a graph. The vertices represent configuration items. The edges represent relationships (i.e. dependency, derived from) between the configuration items. Each configuration item has its own set of KVDescriptors which are, in essence, pointers to CRUD callback operations. With this level of abstraction, the KVScheduler does not need to know the low-level details of configuration items.

The KVScheduler walks the tree to sequence the correct configuration actions. It builds a transaction plan that drives CRUD operations to the vpp-agent in the SB direction. Configuration items can be cached until outstanding dependencies are resolved.

NB partial, SB partial or full state reconciliation (synchronization) is supported.  Information on transaction plans, cached values and errors are exposed via REST APIs.

Note that any plugin requiring an object, that is dependent on another object, can leverage the KVScheduler.

### Ligato Plugins

The ethos of a component-based architecture ("plugginability") in software platforms has emerged as the defacto standard with React and Go being two popular examples. Ligato has adopted the same approach for CNF developers.

- pick and choose only those plugins needed
- flexibility to swap in or out different plugins for development, testing and implementation
- create custom app plugins

Ligato plugins are defined using common pattern making it straightforward to understand existing plugins as well as create new ones. In addition, config files are used to define plugin functionality at startup.


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