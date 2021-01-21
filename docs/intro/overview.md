# Overview

_****_

---

## CNF

_**A cloud native network function (CNF) is a cloud native application that implements network functionality. A CNF consists of one or more microservices and has been developed using Cloud Native Principles including immutable infrastructure, declarative APIs, and a “repeatable deployment process”.**_<br></br>Source: [CNCF definition](https://github.com/cncf/telecom-user-group/blob/master/whitepaper/cloud_native_thinking_for_telecommunications.md#14-cloud-native-network-functions)

---

Cloud providers are shifting towards a cloud native approach for application development, operations and management. Applications are packaged up in containers. Kubernetes automates container startup, placement and teardown based on policies, services, scale and resiliency. The cloud native lifecycle simplifies the steps beginning with development all the way to deployment and operations in modern cloud environments. 

Cloud native networks (CNF) run inside contains and inherit all behaviors and benefits afforded to cloud native applications. Indeed, you can think of CNFs as just another application container. 

By applying the Kubernetes lifecycle to CNFs, you can spin up different network configurations to meet your respective needs. Several examples follow:

- Webscale services running on multiple application pods interconnected through a CNF virtual switch network overlay.
- Cloud service bundles composed of CNFs strung together in a service function chain.
- VPN Internet Gateway solutions employing IPsec crypto/tunnel CNFs. 

CNFs require two more very important attributes:

- High performance data plane.
- Programmability using cloud native APIs including k8s CRDs, gRPCs, etcd data stores

CNF solutions must keep pace with cloud native adoption and growth. The cloud native developer community requires solutions that simplify and expedite the development and implementation of customized, scalable and versatile CNFs. 

That's where Ligato comes into play.

To learn more about CNFs, check out the following:

- [What is a CNF?](https://ligato.io/cnf/cnf-def/)
- [X-Factor CNFs](https://x.cnf.dev/config/)
- [CNFs with a Dose of Ligato and FD.io/VPP](https://ligato.io/blog/cnf-ligato-fdio/)
- [Cloud Native Thinking for Telecommunications](https://github.com/cncf/telecom-user-group/blob/master/whitepaper/cloud_native_thinking_for_telecommunications.md)

---

## 100K foot View

_**Ligato is a Golang (Go) framework for developing software agents to control and manage CNFs.**_

CNF solutions with different personalities will exist in cloud native networks. Some replicate the functions of existing physical (PNF) and virtual NFs (VNF). Others will support new and emerging cloud-native functions.

![overview][docs-overview-100k]

With Ligato, you can develop CNFs that meet your individual network functionality requirements. 

- Features devoted to cloud native development and operational deployment.
- Focus on management and control plane agents for CNFs.
- Tailored for the high-performance FD.io/VPP data plane.
- Embraces integration with other open source projects.   
  

--- 

##10K foot View

_**Ligato provides the plugins and infrastructure to develop software agents.**

The next figure takes you down to the "10K foot" level. You have the following beginning at the top:

- **Applications** - External applications, rpc clients, telemetry apps, and data stores that typically run in their own separate containers. They communicate via declarative APIs to Ligato NB plugins.
<br></br>
- **KV Scheduler** - Core plugin that supports configuration item dependency resolution, and computes the proper programming sequence for multiple interdependent configuration items. The KV Scheduler receives configuration data from NB, determines configuration item dependencies, executure CRUD callbacks in the SB towards the VPP or Linux plugins.
<br></br>
- **VPP and Linux Plugins** - Set of plugins providing network functions, such as VPP routes or ACLs, you can use to assemble your Ligato agent. Note that you can "cherry pick" plugins as needed. And you can developer your own custom plugins.
<br></br>
- **Infra** - Provides plugin lifecycle management including initialization and graceful shutdown of plugins). Infra includes a set of framework or infrastructure plugins supporting health checking, NB data store communications, messaging, logging, and rpc APIs.     


![docs-overview-10k][docs-overview-10k]

You will most likely implement a mix of cn-infra plugins, VPP agent plugins, and/or custom plugins, to define CNF functionality.

---

## VPP Agent Functions

_** Ligato provides a VPP agent for programming a VPP data plane**_

You will likely hear the term, VPP agent, associated with Ligato. It loosely describes a Ligato agent that configures and monitors a VPP data plane.

This term also refers to the VPP agent repository containing plugin code, models and protos.

The following lists VPP agent functions: 

* Comes with VPP-specific plugins
* VPP agent and VPP data plane packaged up in single container
* Dependency handling between related configuration items
* Transaction-based configuration processing and tracking
* Failover NB/SB synchronization mechanisms
* Stateless configuration management based on a KV data store "watch" paradigm
* Direct access via REST or gRPC
* Component health checks
* Multi-VPP configuration support
* Multi-version configuration support

!!! Note
    The VPP agent provides a configuration and monitoring services for the VPP data plane. It does not provide packet processing functions. The VPP data plane handles that. 
 

---


## Developer Resources

_** Ligato provides a multitude of developer resources**_

- VPP agent and VPP dataplane in one container
- Golang programming language
- Model-driven protobuf APIs
- Common programming pattern for agents
- Out-of-the-box VPP and Linux data plane plugins
- Automatic plugin lookup and dependency injection
- KV scheduler core plugin for handling plugin dependencies and programming sequence
- Common KV descriptor API
- VPP and KV Scheduler APIs
- Agentctl CLI
- Developer guide
- Troubleshooting guide
- Tutorials and code examples
- Detailed logging
- Stateless configuration management based on KV data store "watch" paradigm
- Godocs

[docs-overview-100k]: ../img/intro/docs-overview-ligato.svg
[docs-overview-10k]: ../img/intro/docs-overview-10k.svg









  

