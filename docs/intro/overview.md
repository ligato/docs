# Overview

## Context

_**Ligato is a Golang (Go) framework for developing applications to control and manage cloud native network functions (CNF).**_

Cloud providers are shifting towards a cloud-native approach to application development, operations and management. Applications are implemented as microservices, packaged up in containers, and dynamically deployed in pods over a network of physical and virtual hosts. Kubernetes schedules and orchestrates container startup, placement and teardown based on policies, services, scale and resiliency. This automated lifecycle simplifies the tasks of operating and managing application and services deployments.


Networks play a key role in cloud-native environments. Network functions such as L2/L3 switching and routing, IPsec VPN gateways, NAT and load balancers operate in containers, and are referred to as cloud-native network functions (CNF). Applying the Kubernetes lifecycle to CNFs allows a provider to spin up different network configurations to meet their respective needs. Some examples:

- Webscale services deployed on multiple application pods interconnected through a virtual overlay composed of CNF virtual switches
- Customized service "bundles" composed of individual router, load balancer, firewall and NAT CNFs strung together in a service function chain (SFC)
- VPN Interconnect employing IPsec crypto/tunnel CNFs

CNF solutions must keep pace with cloud-native adoption growth. Therefore the cloud-native community requires solutions that simplify and expedite the development and implementation of customized, scalable and versatile CNFs.

Ligato is the solution.


---

## Infrastructure

_**Ligato provides the infrastructure and agents developers can use to build customized CNF solutions**_

* Supplies an assortment of reusable plugins allowing developers to "cherry pick" and implement as needed
* Plugins for VPP, Linux, etcd, Kafka, redis, Cassandra, gRPC, REST and more
* Model-driven architecture for configuration object (i.e. interface) abstraction
* Northbound APIs based on protobufs
* Connectors for interfacing with external databases
* Examples and tutorials for building customized plugins

---

## vpp-agent

_** Ligato provides a vpp-agent for programming a VPP data plane**_

* Supplies VPP-specific plugins
* vpp-agent and vpp packaged up in single container
* Dependency handling between related configuration items
* Transaction-based configuration processing
* Failover synchronization mechanisms (aka resync)
* Stateless configuration management based on KV data store "watch" paradigm
* Direct access via REST or gRPC
* Component health checks

!!! Note
    The vpp-agent is a control/management plane. It provides a configuration and monitoring services for the VPP dataplane. It does not decide what to do with packets arriving on any of the VPP interfaces and does not change any of the configuration parameters (routes, FIBs) based on the actual VPP state.


---

## VPP Configuration Management

Configuring VPP via CLI is challenging. The CLI commands mostly reflect low-level binary API calls and must be executed in a specific order or sequence for the configuration to succeed. In some cases, a configuration item (e.g. interface) could depend on another separate configuration item. In other cases, a specific sequence of commands (API calls), each handling an individual configuration item, must be completed before the system is brought up to the desired state. As networks scale and the number of configuration items grows, it becomes vastly more difficult to ensure correct network configuration deployment and operation even under normal conditions. 

The vpp-agent addresses this challenge by providing a mechanism to sort out dependencies for the entire configuration, all accessible from a northbound (NB) protobuf API. Configuration items use logical names instead of software indexes to ensure consistent configuration setup, even across process runtimes. For example, interfaces can be referenced before they are even created because the user defines each with a logical name.  

The vpp-agent configuration behavior is as follows:

* Configuration items with all dependencies satisfied or no dependencies are programmed into VPP

* Incomplete configuration items (those with unresolved dependencies) are cached or if possible partially completed

*  At some later point in time, when the missing dependency items appear, all pending configuration items are re-validated and executed. All configuration items including those with dependencies between VPP and Linux is managed the same way.

Another significant feature is the ability to retrieve existing VPP configuration. In addition to status reporting, this data is used to perform resynchronization. This is the process by which the active VPP configuration is compared to the desired VPP configuration to resolve any required changes and to minimize the impact during restarts.

!!! note
    The need to address the configuration dependency problem arose when a configured VPP worked as expected. Following a restart/reconfiguration, the VPP state looked exactly the same - but did not work as before. The reason for this behavior is that the binary API calls followed an incorrect order during restart, resulting in a partially complete configuration and VPP dataplane errors.

## Resync

Resync is one of the major features available with the vpp-agent. It ensures consistency between configuration provided from an external source, internal vpp-agent state and the actual VPP state. The automatic resync fetches all the data from any connected persistent data store (e.g. etcd) and reflects any changes to the VPP. Synchronization is performed against the Linux host as well. 

The resync is by default initiated on vpp-agent startup. It can also be automatically launched on certain events (i.e. VPP restart, reconnection to the data base). 

## Plugin Concept

The vpp-agent is based on the Ligato framework plugin architecture. In general, a plugin is a small chunk of code that performs a specific function (or functions). Plugins can be assembled in any combination to build solutions ranging from simple elementary tasks (e.g. basic configuration) to larger more complex applications such as managing configuration state across multiple nodes in a network. Plugins are designed to be setup and/or modified at startup based on a configuration file. The plugin definition is standardized in the Ligato framework, so it can be easily extended with customized plugins to create new solutions and applications.
  

