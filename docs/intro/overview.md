# Overview

This section provides an overview of the Ligato VPP agent.

---

## Ligato VPP Agent

Ligato is a Golang (Go) framework for developing applications to control and manage cloud native network functions (CNF). The Ligato VPP Agent (VPP agent for short) is a Go implementation of control and management plane for VPP-based CNFs. The VPP agent comes with a set of VPP-specific plugins for programming the VPP dataplane. The Ligato framework's infra system and plugin-based architecture allows developers to build customized control plane applications and solutiuons that extend and/or compliment the functions provided by the VPP agent.  

### What Can It Do?

The features supported by the VPP agent are:

* VPP configuration management
* Linux networking configuration management
* Metrics collection for the VPP and Linux networking systems.

The features provided by the Ligato framework in support of the VPP agent include: 

* Dependency handling between related configuration items
* Transaction-based configuration processing
* Failover synchronization mechanisms (aka resync)
* Various northbound accessors available to clients
* Use of Protobufs for northbound APIs
* Plugin-based architecture
* Key-value (KV) data store support
* Direct access via REST or gRPC
* Component health checks

#### VPP Configuration Management

Configuring VPP via CLI is challenging. The CLI commands mostly reflect low-level binary API calls and must be executed in a specific order or sequence for the configuration to succeed. In some cases, a configuration item (e.g. interface) could depend on another separate configuration item. In other cases, a specific sequence of commands (API calls), each handling an individual configuration item, must be completed before the system is brought up to the desired state. As networks scale and the number of configuration items grows, it becomes vastly more difficult to ensure correct network configuration deployment and operation even under normal conditions. 

The VPP agent addresses this challenge by providing a mechanism to sort out dependencies for the entire configuration, all accessible from a northbound protobuf API. Configuration items use logical names instead of software indexes to ensure consistent configuration setup, even across process runtimes. For example, interfaces can be referenced before they are even created because the user defines each with a logical name.  

The VPP agent configuration behavior is as follows:

* Configuration items with all dependencies satisfied or no dependencies are programmed into VPP

* Incomplete configuration items (those with unresolved dependencies) are cached or if possible partially completed

*  At some later point in time, when the missing dependency items appear, all pending configuration items are re-validated and executed. All configuration items including those with dependencies between VPP and Linux is managed the same way.

Another significant feature is the ability to retrieve existing VPP configuration. In addition to status reporting, this data is used to perform resynchronization. This is the process by which the active VPP configuration is compared to the desired VPP configuration to resolve any required changes and to minimize the impact during restarts.

```
The need to address the configuration dependency problem arose when a configured VPP worked as expected. Following a restart/reconfiguration, the VPP state looked exactly the same - but did not work as before. The reason for this behavior is that the binary API calls followed an incorrect order during restart, resulting in a partially complete configuration and VPP dataplane errors.
```

#### Resync

Resync is one of the major features available with the VPP agent. It ensures consistency between configuration provided from an external source, internal VPP agent state and the actual VPP state. The automatic resync fetches all the data from any connected persistent data store (e.g. etcd) and reflects any changes to the VPP. Synchronization is also performed against the Linux host as well. 

The resync is by default initiated on VPP agent startup. It can also be automatically launched on certain events (i.e. VPP restart, reconnection to the data base). 

#### Plugin Concept

The VPP agent is based on the Ligato framework plugin architecture. In general, a plugin is a small chunk of code that performs a specific function (or functions). Plugins can be assembled in any combination to build solutions ranging from simple elementary tasks (e.g. basic configuration) to larger more complex applications such as managing configuration state across multiple nodes in a network. Plugins are designed to be setup and/or modified at startup based on a configuration file. The plugin definition is standardized in the Ligato framework, so it can be easily extended with customized plugins to create new solutions and applications.
  
### What It Cannot Do?

The VPP agent is a control/management plane. It provides a configuration and monitoring services for the VPP dataplane. It does not decide what to do with packets arriving on any of the VPP interfaces and does not change any of the configuration parameters (routes, FIBs) based on the actual VPP state.

### What's Next?

Want to learn more about Ligato?

* Read about the [concepts][concepts] used in Ligato.
* Learn how to use Ligato in **User Guide** section.

Want to start developing with Ligato?

* Go through tutorials in **Tutorial** section for guided development.
* Try to make your own plugin with the help of the **Developer Guide**

[concepts]: ../user-guide/concepts.md

*[CLI]: Command-Line Interface
*[FIB]: Forwarding Information Base
