# Overview

This page provides high-level overview of Ligato projects.

---

## Agent

The Agent (also referred to as VPP-Agent) is a Go implementation of the control/management plane for the VPP based cloud-native virtual network functions (VNFs). The Agent is developed with cloud-native infra framework to provide extensibility and allow users to build customized agents.

### What can it do?

The most notable features provided by the Agent:

* VPP configuration management
* Linux networking configuration management
* Metrics collection from the VPP & Linux networking

and features provided by the cloud-native infra framework:

* Dependency handling between related configuration items
* Transaction-based configuration processing
* Failover synchronization mechanism (a.k.a. resync)
* Various northbound accessors available for clients
* All northbound API using Protobuf
* Plugin-based architecture (develop custom Agent using only components you need)
* Support for different types of key-value data stores
* Direct access via REST or gRPC
* Health-checks for any component

#### VPP config management

Configuring VPP via CLI is not very convenient. The VPP CLI commands mostly reflect binary API calls which have several disadvantages, notably the fact that they have to be called in the specific order, because some configuration items might depend on others, or that the single command often sets only a part of the configuration item (e.g. interfaces have to be created with one call, then set to UP state with another and IP address has to be assigned with separate call too, etc..). 

The Agent solves this by providing advanced dependency-solving mechanism for its entire configuration accessible from northbound API defined with Protobuf. Configuration items which are often referenced use logical names instead of software indexes to ensure consistent configuration setup, even across process runtimes. For example, interfaces can be referenced before they are even created because the user defines its own logical name for them. 

The Agent actually configures VPP only with parts that are possible to configure at the moment (those that have all dependencies satisfied or have no dependency) for everything else it will simply cache the incomplete parts of configuration or configure it partially if impossible. At some later point in time, when the missing dependency item appears, all pending configuration is re-validated and configured (if all other requirements are met). All configuration including those with dependencies between VPP and Linux is managed the same way.

Another significant feature is ability to retrieve existing VPP configuration. The retrieved data is used not only for status reporting, but also for resynchronization during which it is compared to desired configuration to resolve required changes and minimize the impact during restarts.

We also often stumbled upon the case where configured VPP worked as expected, and after restart/reconfiguration the VPP state looked exactly the same - but did not work as before. The reason for this behavior is that the binary API calls followed incorrect order during restart, configuring the VPP only ostensibly correct. This can be also solved by the Agent - plugins ensure all API calls are called in the correct sequence in order to make it work right.

#### Resync mechanism

The resync is one of the major Agent features - it ensures consistency between configuration provided from an external source, internal Agent state and the actual VPP state. The automatic resync fetches all the data from any connected persistent data store and reflects changes to the VPP. The synchronization is also performed against the Linux host. 
The resync is by default started on the Agent startup, but can also automatically be launched on certain events (VPP restart, reconnection to the data base). 

#### Plugin concept

The Agent is plugin-based. It allows building simple agents performing only elementary tasks (basic configuration), or universal solutions with a wide variety of plugins. Many plugin's behaviors can be set up or modified on startup with a particular configuration file. The plugin definition is standardized in the agent, so it can be easily extended with a custom user-defined plugin and started together with other custom or any pre-defined plugins in order to fit user's needs. 
  
### What it cannot do?

The VPP-Agent is management plane - it provides configuring and monitoring services. It does not decide what to do with packets arriving at any of the VPP interfaces and does not change any of the configuration parameters (routes, FIBs) based on the actual VPP state.

### What to do next?

Wnat to learn more about Ligato?

* Read about the [concepts][concepts] used in Ligato.
* Learn how to use Ligato in **User Guide** section.

Want to start developing with Ligato?

* Go through tutorials in **Tutorial** section for guided development.
* Try to make your own plugin with the help of the **Developer Guide**

[concepts]: ../user-guide/concepts.md

