# Overview

---

This section defines a CNF, Ligato CNF development functions and features, and includes a list of developer resources.

---

## CNF

_**"A cloud native network function (CNF) is a cloud native application that implements network functionality. A CNF
consists of one or more microservices developed using Cloud Native principles including immutable infrastructure,
declarative APIs, and a “repeatable deployment process”.**_<br></br>
Source: [CNCF definition](https://github.com/cncf/telecom-user-group/blob/master/whitepaper/cloud_native_thinking_for_telecommunications.md#14-cloud-native-network-functions)

---

Cloud providers are shifting towards a cloud native approach for application development, operations and management.
Applications are packaged up in containers. Kubernetes (K8s) automates container startup, placement and teardown based
on policies, services, scale and resiliency. The cloud native lifecycle simplifies these steps beginning with
development all the way to deployment automation and operations in modern cloud environments.

Cloud native network functions (CNF) run inside containers and inherit all behaviors and benefits afforded to cloud
native applications. Indeed, you can think of CNFs as just another application container.

By applying the Kubernetes lifecycle to CNFs, you can spin up different network configurations to meet your respective
needs. Several examples follow.

- Webscale services running on multiple application pods interconnected through a CNF virtual switch network overlay.
- Cloud service bundles composed of CNFs strung together in a service function chain.
- VPN internet gateway solutions employing IPsec crypto/tunnel CNFs.

CNFs require two more important attributes.

- High performance data plane.
- Programmability and observability using cloud native APIs including K8s CRDs, gRPCs, etcd data stores and REST APIs.

CNF solutions must keep pace with cloud native adoption and growth. The cloud native developer community requires
solutions that simplify and expedite the development and implementation of customized, scalable and versatile CNFs.

That's where Ligato comes into play.

---

## 100K foot view

_**Ligato is a Golang (Go) framework for developing software agents to control and manage CNFs.**_

CNF solutions with different personalities exist in cloud native networks. Some replicate the functions of existing
physical (PNF) and virtual NFs (VNF). Others support new and emerging cloud native functions.

The following figure shows a high-level conceptual view of Ligato CNF solutions, features and a sampling of compatible
open source components.

![overview][docs-overview-100k]
<p style="text-align: center; font-weight: bold">Ligato at a 100K foot view</p>

---

You can develop CNFs that address your individual cloud native network requirements. The following lists the Ligato "
take-aways" to help you achieve that goal.

- Features devoted to CNF development and operational deployment.
- Focus on CNF configuration and monitoring.
- Tailored for the high-performance FD.io/VPP data plane.
- Embraces integration with other open source projects.

--- 

## 10K foot view

_**Ligato supplies you with the plugins, infrastructure, code and documentation to develop and implement software agents
to control and manage CNFs.**_

The next figure takes you down to a "10K foot" view beginning at the top with applications.

![docs-overview-10k][docs-overview-10k]
<p style="text-align: center; font-weight: bold">Ligato at a 10K foot view</p>

---

- **Applications** - External applications, rpc clients, telemetry apps and data stores. They communicate via
  declarative APIs with Ligato northbound (NB) plugins.
  <br></br>
- **Northbound Plugins** - Set of reusable plugins enabling Ligato software agents to communicate with external
  applications.
  <br></br>
- **KV Scheduler** - Core plugin that calculates configuration item dependency resolution and the proper
  programming sequence for multiple interdependent configuration items. The KV Scheduler receives configuration data
  from NB, determines dependencies and executes CRUD callbacks in the southbound (SB) towards the VPP and Linux
  plugins.
  <br></br>
- **VPP and Linux Plugins** - Set of plugins providing network functions you can use in your Ligato software
  agent. You can "cherry pick" plugins as needed and even develop your own custom plugins.
  <br></br>
- **Infra** - Provides plugin lifecycle management including initialization and graceful shutdown. Infra includes a set
  of plugins for health checking, NB data store communications, messaging, logging and rpc APIs.

You can employ any mix of cn-infra plugins, VPP agent plugins and custom plugins to define your CNF.

!!! Note 
    Two repositories contain the Ligato code: [VPP agent](https://github.com/ligato/vpp-agent)
    and [cn-infra](https://github.com/ligato/cn-infra). You can import dependencies from each as needed.

---

## VPP agent functions

_** Ligato makes it easy to develop a VPP agent for programming a VPP data plane.**_

You will hear the term, VPP agent, associated with Ligato. It describes a software agent that configures and
monitors a VPP data plane. This software agent can include infrastructure and custom plugins as well. 

VPP agent features include:

* Implemented as VPP and Linux plugins.
* VPP agent + VPP data plane in one container.
* Multi-VPP configuration support.
* Multi-version configuration support.

---

* Dependency handling between related configuration items.
* Transaction-based configuration processing and tracking.
* Failover NB/SB synchronization mechanisms.
* Stateless configuration management based on a KV data store "watch" paradigm.

---

* Direct access via REST or gRPC.
* Component health checks.
* Agentctl CLI.

!!! Note The VPP agent does not perform packet processing when forwarding packets. The VPP data plane handles that for
you.

---

## Developer resources

_** Ligato comes with a comprehensive suite developer resources.**_

- [Quickstart Guide](../user-guide/quickstart.md).
- [VPP agent setup instructions](../user-guide/get-vpp-agent.md).
- [Agentctl](../user-guide/agentctl.md).
- [Tutorials](../tutorials/00_tutorial-setup.md).
- [Code Examples](../user-guide/examples.md).

---

- Golang programming language.
- [Model-driven protobuf APIs](../user-guide/concepts.md).
- Common programming pattern for agents.

---

- [Multiple out-of-the-box plugins](../plugins/plugin-overview.md).
- [Automatic plugin lookup and dependency injection](../developer-guide/plugin-lookup.md).
- Plugin KV data store, REST, and gRPC examples.
- KV scheduler core plugin for handling plugin dependencies and programming sequence.
- [Custom agent coding example](https://github.com/ligato/vpp-agent/tree/master/examples/customize/custom_vpp_plugin).

---

- [KV descriptor API](../developer-guide/kvdescriptor.md).
- [VPP Agent REST APIs](../api/api-vpp-agent.md).
- [KV Scheduler REST APIs](../api/api-kvs.md).

---

- Developer guide
- [KV Scheduler troubleshooting guide](../developer-guide/kvs-troubleshooting.md).
- [Startup troubleshooting](../troubleshooting/startup-ts.md).
- [Configuration troubleshooting](../troubleshooting/config-ts.md).
- [Detailed logging](../developer-guide/kvs-troubleshooting.md#how-to-set-up-logging).

---

- [Godocs](../developer-guide/godocs.md).
- [Integration tests](../testing/integration-tests.md).
- [End-to-end tests](../testing/end-to-end-tests.md).

[docs-overview-100k]: ../img/intro/docs-overview-ligato.svg

[docs-overview-10k]: ../img/intro/docs-overview-10k.svg









  

