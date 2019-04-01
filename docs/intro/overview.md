This page is an overview for the VPP-Agent
 
# VPP-Agent overview

The VPP Agent (or just Agent) is a Go implementation of the control/management plane for the VPP based cloud-native virtual network functions (VNFs). The Agent is built on the cloud-native infrastructure platform ([CN-Infra](https://github.com/ligato/cn-infra/wiki)).

**What can it do?**

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

**VPP configuration and management**

Setup of the VPP via CLI or using the VAT console (which is expected to be deprecated soon) is not always convenient. VPP CLI commands mostly reflect binary API calls which have several disadvantages, notably the fact that they have to be called in the specific order (because some configuration items might depend on others), or that the single command often sets only a part of the configuration item (an example can be the interface, which has to be created in one API call, set as 'up' in another, the IP address has to be assigned separately, etc.). 

The VPP-Agent solves many of such problems defining easy-to-read proto-defined API on one side and advanced dependency-solving mechanism on the other. Items, which are often referenced use logical names (instead of generated indexes) to ensure better readability and more convenient setup. For example, interfaces can be referenced before they are even created because the user defines its own name for the instance. The Agent configures what is possible (like empty bridge domains, ACLs which are not attached to any interface) or simply caches given configuration if partial is impossible. When the desired interface or any other required item appears, all pending data are re-validated and set (if all other requirements are met). Dependencies between VPP-related config and the VPP Linux-based host OS config is managed the same way. 

Another significant feature is ability to retrieve existing VPP configuration. Data read is used not only for status reporting, but more importantly for resynchronization. 

We also often stumbled upon the case where configured VPP worked as expected, and after restart/reconfiguration the VPP state looked exactly the same - but did not work as before. The reason for this behavior is that the binary API calls followed incorrect order during restart, configuring the VPP only ostensibly correct. This can be also solved by the VPP-Agent - plugins ensure all API calls are called in the correct sequence in order to make it work right.

**Resync**

The resync is one of the major Agent features - it ensures consistency between configuration provided from an external source, internal Agent state and the actual VPP state. The automatic resync fetches all the data from any connected persistent data store and reflects changes to the VPP. The synchronization is also performed against the Linux host. 
The resync is by default started on the Agent startup, but can also automatically be launched on certain events (VPP restart, reconnection to the data base). 

**Plugin concept**

The Agent is plugin-based. It allows building simple agents performing only elementary tasks (basic configuration), or universal solutions with a wide variety of plugins. Many plugin's behaviors can be set up or modified on startup with a particular configuration file. The plugin definition is standardized in the agent, so it can be easily extended with a custom user-defined plugin and started together with other custom or any pre-defined plugins in order to fit user's needs. 
// TODO link to list of pre-defined plugins
  
**What it cannot do**

* The VPP-Agent is management plane - it provides configuring and monitoring services. It does not decide what to do with packets arriving at any of the VPP interfaces and does not change any of the configuration parameters (routes, FIBs) based on the actual VPP state.

# Framework plugins

Overview of the Agent plugins

**Configurator**

More in: configurator plugin // TODO 

// TBD

**GoVPPMux**

More in: [govppmux plugin](../user-guide/framework-plugins.md#govpp-mux)

The GoVPP mux plugin is Agent's wrapper around the [GoVPP](https://github.com/FDio/govpp/blob/master/README.md) and provides access to the VPP. Every plugin can interact with the VPP using the GoVPP mux, which provides an independent communication channel to the VPP instance. The communication is done via shared memory segment prefix, and the plugin also supports custom prefixes to connect to the correct VPP instance in the multi-VPP environment.

**Orchestrator**

More in: orchestrator plugin // TODO  

In a scenario where a single Agent instance receives configuration data from multiple sources (KV data store, GRPC, etc), orchestrator is used to synchronizing retrieved data and resolve conflicts from individual sources. Data-processing plugins then see the data as from the single source.

**Rest API**

More in: [restapi plugin](../user-guide/framework-plugins.md#vpp-agent-rest)

A core agent plugin that exposes REST API, which can be used to pass VPP CLI commands, retrieve existing VPP configuration or Agent northbound objects. Note: In the future, the plugin will support also VPP configuration via REST.

**Telemetry**

More in: [telemetry plugin](../user-guide/framework-plugins.md#telemetry-plugin)

The telemetry plugin collects telemetry statistics from the VPP which are then exported for monitoring.

**KVScheduler**

More in: [KVScheduler plugin](../user-guide/framework-plugins.md#kv-scheduler)

The KV Scheduler is the first step in any VPP or Linux related data processing. It validates the existence of the configuration item dependencies, handles local caching and performs retries if possible or allowed by the underlying plugin. The KV Scheduler does not operate with data directly (does not call any VPP binary API), only determines what operations are needed to achieve the desired result. Data are processed into low-level objects in adjacent VPP/Linux plugins.

# VPP plugins

Overview of the core plugins provided functionality to the default VPP functionality

**ACL plugin**

More in: [ACL plugin](../user-guide/other-vpp-plugins.md#access-control-lists-plugin)

The ACL plugin handles the configuration of the VPP access control lists. The ACL stands out as the permission attached to a given interface. If rules defined in the access list are met by the incoming traffic, packets are allowed to perform defined action.

**Interface plugin**

More in: [if plugin](../user-guide/default-vpp-plugins.md#interface-plugin) 

The interface plugin creates various types of interfaces (e.g. DPDK, MEMIF, TAP ...) in the VPP, and also configure base fields (IP address, MAC address) or more advanced features like unnumbered interfaces or RX-mode. 

**IPSec plugin**

More in: [IPSec plugin](../user-guide/other-vpp-plugins.md#ipsec-plugin)

The plugin configures VPP Security policy databases (SPD) and security associations (SA) which set security attributes (such algorithms) between communicating parties.

**L2 plugin**

More in: [l2-plugin](../user-guide/default-vpp-plugins.md#l2-plugin)

The plugin handles data link layer configuration items - VPP bridge domains, forwarding information base (FIB) or VPP cross-connects.

**L3 plugin**

More in: [l3-plugin](../user-guide/default-vpp-plugins.md#l3-plugin)

The L3 plugin configures L3-related features like ARPs (including proxy ARPs) or Routes.

**NAT plugin**

More in: [NAT-plugin](../user-guide/other-vpp-plugins.md#network-address-translation-plugin)

Handles data related to the network address translation. 

**Punt plugin**

More in: [punt-plugin](../user-guide/other-vpp-plugins.md#punt-plugin)

The punt plugin provides access to the VPP punt feature, where incoming VPP traffic matching pre-defined rules is 'punted' or redirected.

**STN plugin**

More in: stn-plugin //TODO

The implementation of the control plane for the VPP STN (Steal the NIC)

### Linux plugins

**Linux Interface plugin**

More in: [linux-if-plugin](../user-guide/linux-plugins.md#linux-interface-plugin)

The Linux interface plugin processes Linux interfaces related to the adjacent VPP configuration (VEth, TAPv2). Virtual Ethernet interfaces
can be directly created by the Agent.

**Linux L3 plugin**

More in: [linux-l3-plugin](../user-guide/linux-plugins.md#linux-l3-plugin)

The Linux L3 plugin can be used to configure Linux routes or ARPs. 

**Namespace plugin**

More in: [ns-plugin](../user-guide/linux-plugins.md#namespace-plugin)

The namespace plugin is a helper plugin tied with the Linux interface/l3 plugins. It manages namespaces in terms of Linux (named namespace) or as a microservice in the container-based environment.  

# CN-Infra plugins

The list of plugins provided by the CN-Infra:

//TODO link to cn-infra wiki

# Tools

The repository also contains tools for building, testing, and troubleshooting of the VPP-Agent.

**VPP-Agent-ctl**

More in: [vpp-agent-ctl](../tools/vpp-agent-ctl.md)

The VPP-Agent-ctl is a utility tool whose primary purpose is to test and troubleshoot the VPP-Agent. The tool allows to put pre-defined configuration of any type currently supported in the agent to the ETCD, use custom data specifying key and value (as JSON), or read current ETCD config.

**Docker**

Container-based development environment for the VPP-Agent and for app/extension plugins. Docker image is available for [development](https://hub.docker.com/r/ligato/dev-vpp-agent) and [production](https://hub.docker.com/r/ligato/vpp-agent) version of the VPP-Agent with compatible VPP.

# What to do next?

Already familiar with the VPP-Agent basics? [Learn more about the concepts](../user-guide/concepts.md), try some of our [examples](https://github.com/ligato/vpp-agent/blob/master/examples/README.md), read more in other user-guide articles or try to make your own plugin with the help of the developer guide. 


   



