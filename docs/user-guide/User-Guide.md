Welcome to the user guide for the vpp-agent.

# Table of contents
- [Getting started with vpp-agent](#getting-started)
- [Concepts in vpp-agent](#concepts)
- [Reference guide for vpp-agent](#reference)
- [Components of vpp-agent](#components)
- [Miscellaneous](#miscellaneous)

<a name="getting-started"></a>
## Getting started
- Get the general [Overview](../articles/Overview) of the vpp-agent, what it is, what it can do and what components is it made of.
- Begin with the [Quickstart](../articles/Quickstart) guide how to download, install and run the vpp-agent. 
- [Learn how-to](Learn-how-to) use the vpp-agent for configuration.

<a name="concepts"></a>
## Concepts
- [Models](Models) - describes model concept used in the northbound API.
- [Key-value Datastore](KV-Store) - describes key-value datastore usage in vpp-agent.
- Learn more about the [VPP configuration ordering](VPP-conf-order) 
- [VPP multi-version support](VPP-multiversion) - describes what is the VPP multi-version support and how it works

<a name="reference"></a>
## Reference
- Find a list of [all model keys](KeyOverview) currently supported.

<a name="components"></a>
## Components of vpp-agent
- VPP related components:
  * VPP configuration management:
    * [Access Lists](../articles/ACL-plugin)
    * [Interfaces](../articles/VPP-Interface-plugin)
    * [IPSec](../articles/IPSec-plugin)
    * [L2 plugin](../articles/L2-plugin)
    * [L3 plugin](../articles/L3-plugin)
    * [NAT plugin](../articles/NAT-plugin)
    * [Punt](../articles/Punt-plugin)
    * STN plugin
  * Collecting and exporting the VPP statistics:
    - [Telemetry](../articles/Telemetry)
- Linux configuration management plugins:
  * [Linux Interfaces](../articles/Linux-Interface-plugin)
  * [Linux L3 plugin](../articles/Linux-L3)
  * [Namespaces](../articles/Namespace-plugin)
* Northbound access:
  - [Clientv2](../articles/Clientv2)
  - [REST API](../articles/REST)
  - [GRPC](../articles/GRPC)
* Data processing & synchronization:
  - Configurator  
  - Orchestrator
  - [KV Scheduler](../articles/KVScheduler)

<a name="miscellaneous"></a>
## Miscellaneous
- The [VPP-Agent-ctl](https://github.com/ligato/vpp-agent/blob/master/cmd/vpp-agent-ctl/README.md) to test the vpp-agent with pre-prepared configuration data.
- [Examples](https://github.com/ligato/vpp-agent/blob/master/examples/README.md) for various vpp-agent features.        