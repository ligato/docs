# Overview

_****_

---

## CNF

_**A cloud native network function (CNF) is a cloud native application that implements network functionality. A CNF consists of one or more microservices and has been developed using Cloud Native Principles including immutable infrastructure, declarative APIs, and a “repeatable deployment process”.**_<br></br>Source: [CNCF definition](https://github.com/cncf/telecom-user-group/blob/master/whitepaper/cloud_native_thinking_for_telecommunications.md#14-cloud-native-network-functions)

---

Cloud providers are shifting towards a cloud native approach for application development, operations and management. Applications are packaged up in containers. Kubernetes automates container startup, placement and teardown based on policies, services, scale and resiliency. The cloud native lifecycle simplifies the steps beginning with development all the way to deployment and operations in modern cloud environments. 

Cloud native network functions (CNF) run inside contains and inherit all behaviors and benefits afforded to cloud native applications. Indeed, you can think of CNFs as just another application container. 

By applying the Kubernetes lifecycle to CNFs, you can spin up different network configurations to meet your respective needs. Several examples follow:

- Webscale services running on multiple application pods interconnected through a CNF virtual switch network overlay.
- Cloud service bundles composed of CNFs strung together in a service function chain.
- VPN internet gateway solutions employing IPsec crypto/tunnel CNFs. 

CNFs require two more important attributes:

- High performance data plane.
- Programmability and observability using cloud native APIs including k8s CRDs, gRPCs, etcd data stores and REST APIs.

CNF solutions must keep pace with cloud native adoption and growth. The cloud native developer community requires solutions that simplify and expedite the development and implementation of customized, scalable and versatile CNFs. 

That's where Ligato comes into play.

---

## 100K foot View

_**Ligato is a Golang (Go) framework for developing software agents to control and manage CNFs.**_

CNF solutions with different personalities exist in cloud native networks. Some replicate the functions of existing physical (PNF) and virtual NFs (VNF). Others will support new and emerging cloud native functions.

![overview][docs-overview-100k]
<p style="text-align: center; font-weight: bold">Ligato at a 100K foot view</p>

With Ligato, you can develop CNFs that meet your individual network functionality requirements. Ligato provides the following: 

- Features devoted to cloud native development and operational deployment.
- Focus on management and control plane agents for CNFs.
- Tailored for the high-performance FD.io/VPP data plane.
- Embraces integration with other open source projects.   
  

--- 

##10K foot View

_**Ligato provides the plugins and infrastructure to develop software agents.**_

The next figure takes you down to the "10K foot" level. You have the following beginning at the top:

![docs-overview-10k][docs-overview-10k]
<p style="text-align: center; font-weight: bold">Ligato at the 10K foot view</p>

- **Applications** - External applications, rpc clients, telemetry apps, and data stores that typically run in their own containers. They communicate via declarative APIs to Ligato NB plugins.
<br></br>
- **KV Scheduler** - Core plugin that supports configuration item dependency resolution, and computes the proper programming sequence for multiple interdependent configuration items. The KV Scheduler receives configuration data from NB, determines the configuration item dependencies, executes CRUD callbacks in the SB towards the VPP or Linux plugins.
<br></br>
- **VPP and Linux Plugins** - Set of plugins providing network functions, such as VPP routes or ACLs, you can use to assemble your Ligato agent. Note that you can "cherry pick" plugins as needed. And you can develop your own custom plugins.
<br></br>
- **Infra** - Provides plugin lifecycle management including initialization and graceful shutdown of plugins. Infra includes a set of framework or infrastructure plugins supporting health checking, NB data store communications, messaging, logging, and rpc APIs. To look more into the infra code and plugins, see the [cn-infra repository](https://github.com/ligato/cn-infra). 
       

You will most likely implement a mix of cn-infra plugins, VPP agent plugins, and/or custom plugins, to define your CNF functionality.

---

## VPP Agent Functions

_** Ligato provides the pieces to construct a VPP agent for programming a VPP data plane**_

You will likely hear the term, VPP agent, associated with Ligato. It loosely describes a Ligato agent that configures and monitors a VPP data plane.

This term also refers to the [VPP agent repository](https://github.com/ligato/vpp-agent) containing plugin code, models and protos.

The following lists VPP agent functions: 

* Comes with VPP-specific plugins.
* VPP agent and VPP data plane packaged in a single container.
* Dependency handling between related configuration items.
* Transaction-based configuration processing and tracking.
* Failover NB/SB synchronization mechanisms.
* Stateless configuration management based on a KV data store "watch" paradigm.
* Direct access via REST or gRPC.
* Component health checks.
* Multi-VPP configuration support.
* Multi-version configuration support.

!!! Note
    The VPP agent provides a configuration and monitoring services for the VPP data plane. It does not provide packet processing functions. The VPP data plane handles that. 
 

---


## Developer Resources

_** Ligato supports a multitude of developer resources**_

- Quickstart Guide
- VPP agent setup instructions 
- VPP agent and VPP dataplane in one container.
- Golang programming language.
- Model-driven protobuf APIs.
- Common programming pattern for agents.
- Multiple out-of-the-box VPP plugins.
- Automatic plugin lookup and dependency injection.
- KV scheduler core plugin for handling plugin dependencies and programming sequence
- Common KV descriptor API
- VPP and KV Scheduler APIs
- Agentctl CLI
- Developer guide
- KV Scheduler troubleshooting guide
- Tutorials and code examples.
- Detailed logging.
- Stateless configuration management based on KV data store "watch" paradigm.
- Godocs

[docs-overview-100k]: ../img/intro/docs-overview-ligato.svg
[docs-overview-10k]: ../img/intro/docs-overview-10k.svg









  

