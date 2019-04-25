# Overview

This page is an overview for the VPP-Agent

---

The VPP Agent (or just Agent) is a Go implementation of the control/management plane for the VPP based cloud-native virtual network functions (VNFs). The Agent is built on the cloud-native infrastructure platform.

### What can it do?

The most notable list of the Agent features:

* Manage the VPP platform

* Manage related Linux configuration 

* Dependency handling between configuration items

* Failover synchronization mechanism (resync)

* Gather statistics from the VPP

* Hi-level northbound API

* Transaction-based logging

Related features provided by the CN-Infra platform:

* Plugin-based (build an Agent with only components you need)

* Connection to various types of external databases

* Accessibility via the REST or the GRPC

* Health check for plugins and other components (database, VPP, ...)

* Messaging (Kafka)

#### VPP configuration and management

Setup of the VPP via CLI or using the VAT console (which is expected to be deprecated soon) is not always convenient. VPP CLI commands mostly reflect binary API calls which have several disadvantages, notably the fact that they have to be called in the specific order (because some configuration items might depend on others), or that the single command often sets only a part of the configuration item (an example can be the interface, which has to be created in one API call, set as 'up' in another, the IP address has to be assigned separately, etc.). 

The VPP-Agent solves many of such problems defining easy-to-read proto-defined API on one side and advanced dependency-solving mechanism on the other. Items, which are often referenced use logical names (instead of generated indexes) to ensure better readability and more convenient setup. For example, interfaces can be referenced before they are even created because the user defines its own name for the instance. The Agent configures what is possible (like empty bridge domains, ACLs which are not attached to any interface) or simply caches given configuration if partial is impossible. When the desired interface or any other required item appears, all pending data are re-validated and set (if all other requirements are met). Dependencies between VPP-related config and the VPP Linux-based host OS config is managed the same way. 

Another significant feature is ability to retrieve existing VPP configuration. Data read is used not only for status reporting, but more importantly for resynchronization. 

We also often stumbled upon the case where configured VPP worked as expected, and after restart/reconfiguration the VPP state looked exactly the same - but did not work as before. The reason for this behavior is that the binary API calls followed incorrect order during restart, configuring the VPP only ostensibly correct. This can be also solved by the VPP-Agent - plugins ensure all API calls are called in the correct sequence in order to make it work right.

#### Resync

The resync is one of the major Agent features - it ensures consistency between configuration provided from an external source, internal Agent state and the actual VPP state. The automatic resync fetches all the data from any connected persistent data store and reflects changes to the VPP. The synchronization is also performed against the Linux host. 
The resync is by default started on the Agent startup, but can also automatically be launched on certain events (VPP restart, reconnection to the data base). 

#### Plugin concept

The Agent is plugin-based. It allows building simple agents performing only elementary tasks (basic configuration), or universal solutions with a wide variety of plugins. Many plugin's behaviors can be set up or modified on startup with a particular configuration file. The plugin definition is standardized in the agent, so it can be easily extended with a custom user-defined plugin and started together with other custom or any pre-defined plugins in order to fit user's needs. 
  
### What it cannot do

* The VPP-Agent is management plane - it provides configuring and monitoring services. It does not decide what to do with packets arriving at any of the VPP interfaces and does not change any of the configuration parameters (routes, FIBs) based on the actual VPP state.

## Tools

The repository also contains tools for building, testing, and troubleshooting of the VPP-Agent.

### VPP-Agent-ctl

More in: [vpp-agent-ctl][vpp-agent-ctl]

The VPP-Agent-ctl is a utility tool whose primary purpose is to test and troubleshoot the VPP-Agent. The tool allows to put pre-defined configuration of any type currently supported in the agent to the ETCD, use custom data specifying key and value (as JSON), or read current ETCD config.

### Docker

Container-based development environment for the VPP-Agent and for app/extension plugins. Docker image is available for [development][development] and [production][production] version of the VPP-Agent with compatible VPP.

## What to do next?

Already familiar with the VPP-Agent basics? [Learn more about the concepts][concepts], try some of our [examples][examples], read more in other user-guide articles or try to make your own plugin with the help of the developer guide. 

[concepts]: ../features/concepts.md
[development]: https://hub.docker.com/r/ligato/dev-vpp-agent
[examples]: https://github.com/ligato/vpp-agent/blob/master/examples/README.md
[production]: https://hub.docker.com/r/ligato/vpp-agent
[vpp-agent-ctl]: ../tools/vpp-agent-ctl.md
