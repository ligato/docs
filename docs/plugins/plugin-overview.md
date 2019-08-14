# Plugin Overview

---

## KV Scheduler plugin

More in: [KVScheduler plugin][kvscheduler]

The KV Scheduler is the first step in any VPP or Linux related data processing. It validates the existence of the configuration item dependencies, handles local caching and performs retries if possible or allowed by the underlying plugin. The KV Scheduler does not operate with data directly (i.e. does not call any VPP binary API); it only determines what operations are needed to achieve the desired result. 

## VPP plugins

vpp-gent core VPP plugins (e.g. plugins which are always required when working with VPP). Plugin list:

- GoVPP Multiplexer
- Interface plugin
- L2 plugin
- L3 plugin
- Access Control List (ACL)
- ACL-based forwarding (ABF)
- IPSec plugin
- NAT plugin
- Punt plugin
- STN plugin 
- Telemetry plugin

### GoVPPMux Plugin

More in: [GoVPPMux plugin][govppmux-plugin]

The GoVPP mux plugin is the vpp-agent's wrapper around the [GoVPP][govpp-readme] and provides access to the VPP. Every plugin can interact with the VPP using the GoVPP mux. It provides an independent communication channel to the VPP instance. The communication is done via shared memory segment prefix, and the plugin also supports custom prefixes to connect to 
the correct VPP instance in multi-VPP environment.


### Interface plugin

More in: [Interface plugin][interface-plugin]

Create various types of interfaces (e.g. DPDK, MEMIF, TAP ...) in VPP. It can also configure base fields (IP address, MAC address, etc.) as well as more advanced features such as unnumbered interfaces or RX-mode. 

### L2 plugin

More in: [l2-plugin][l2-plugin]

Setup link-layer configuration items such as **bridge domains**, **forwarding tables** (FIBs) and **VPP cross connects**. Dependent on [interface plugin][interface-plugin-guide]. 

### L3 plugin

More in: [l3-plugin][l3-plugin]

Configure **ARP** entries (including **proxy ARP**), **VPP routes** and the **IP scan neighbor** feature. The L3 plugin is dependent on the [interface plugin][interface-plugin-guide].

### ACL plugin

More in: [ACL plugin][acl-plugin]

Handles VPP access control lists. If rules defined in the access list are met by the incoming traffic, the configured action is applied to the packets.

### ACL-based forwarding (ABF) plugin

Implementation of the ACL-based forwarding feature. Performs policy-based routing (PBR) where forwarding is done based on ACL matches rather than destination address prefix.

### IPSec plugin

More in: [IPSec plugin][ipsec-plugin]

Allows one to configure **security policy databases** (SPD) and **security associations** (SA) in VPP. It also handles relationships between the SPD and SA or between SPD and an assigned interface. `IPSec tunnel interfaces are not part of the IPSec plugin.` Their configuration is handled by the [VPP interface plugin][interface-plugin-guide]).

### NAT plugin

More in: [NAT-plugin][nat-plugins]

Network address translation, or NAT is a method for translating one IP address into another by modifying (rewriting) packet headers. The vpp-agent NAT plugin is a `control plane` plugin for the VPP NAT dataplane implementation of NAT44. Can also DNAT44 (with load balancing useful in K8s network clusters. The NAT plugin is dependent on the [interface plugin][interface-plugin-guide].

### Punt plugin

More in: [punt-plugin][punt-plugin]

Provides access to the VPP punt feature, where incoming VPP traffic matching a set pre-defined rules is 'punted' or redirected to the host stack or socket.

### Telemetry

More in: [telemetry plugin][telemetry-plugin]

Collects telemetry statistics from VPP for export to external monitoring and management tools.

### STN plugin

Implementation of the `control plane` for the VPP STN (Steal the NIC) feature

## Linux plugins

This section describes the vpp-agent's Linux plugins used to configure the host OS. These plugins can be used as they are, or together with VPP plugins.

- Linux interface plugin
- Linux L3 plugin
- IP-tables plugin
- Namespace plugin

### Linux Interface plugin

More in: [Linux Interface plugin][linux-interface-pluign]

Processes Linux interfaces related to the adjacent VPP configuration (VEth, TAPv2). Virtual Ethernet interfaces can be created by the vpp-agent.

### Linux L3 plugin

More in: [Linux L3 plugin][linux-l3-plugin]

Configure Linux routes  and ARPs. 

### IPtables plugin

More in: [Linux iptables plugin][linux-iptables-plugin]

Implementation of the [Linux IPtables feature][linux-iptables].

### Namespace plugin

More in: [Namespace plugin][linux-namespace-pluign]

Helper plugin tied in with the Linux interface/l3 plugins. It manages namespaces in terms of Linux (named namespace) or as a microservice in container-based environment. 

## Connection plugins

vpp-agent connection plugins enable external data read or write without the use of a data store.

- REST plugin
- GRPC plugin


### REST plugin

More in: [REST plugin][rest-plugin]

REST API support including security.

### GRPC plugin

More in: [GRPC plugin][grpc-plugin]

Provides base GRPC support for the vpp-agent for handling GRPC requests.

## Database plugins

This section describes the vpp-agent database plugins.

- Datasync abstraction plugin
- Data Broker plugin
- etcd
- Redis
- Consul
- Bolt
- Cassandra
- FileDB

### Datasync plugin

More in: [Datasync plugin][datasync-plugin]

defines the interfaces for the abstraction of data synchronization between app plugins and different backend data sources.

### Data Broker plugin

More in: [Data Broker plugin][data-broker-plugin]

data broker abstraction.

### ETCD

More in: [etcd plugin][etcd-plugin]

Provides access to an etcd key-value data store.

### Redis

More in: [Redis plugin][redis-plugin]

Provides access to an Redis key-value data store.

### Consul

More in: [Consul plugin][consul-plugin]

Provides access to a consul key-value data store.

### Bolt

More in: [Bolt plugin][bolt-plugin]

Provides access to a Bolt key-value data store.

### Cassandra

More in: [Cassandra][cassandra-plugin]

Provides access to a Cassandra MYSQL data store.

### FileDB 

More in: [FileDB plugin][file-db-plugin]

Uses the OS file system as a key-value data store.

## Infra plugins

Discusses the Ligato infra plugins.

- Configurator
- Orchestrator
- Status Check 
- Index Map plugin
- Log Manager
- Messaging/Kafka
- Process Manager
- Service Label plugin

### Configurator

Configurator plugin.

### Orchestrator

In a scenario where a single vpp-agent instance receives configuration data from multiple sources (KV data store, GRPC, etc), the orchestrator plugin synchronizes retrieved data and resolve conflicts from individual sources. Data-processing plugins then see the data as coming from a single source.

### Status Check 

More in: [Status Check plugin][status-check]

Monitors the status of a Ligato infra app by collecting and aggregating partial status of vpp-agent plugins.

### Index Map plugin

More in: [Index Map plugin][index-map]

Provides an enhanced mapping structure.

### Log Manager

More in: [Log Manager plugin][log-manager]

View and modify log levels of loggers using a REST API.

### Messaging/Kafka

More in: [Messaging/Kafka plugin][messaging-kafka]

Provides single purpose clients for publishing synchronous/asynchronous messages and for consuming selected topics.

### Process Manager

More in: [Process Manager plugin][process-manager]

Set of methods to create a process instance to manage and monitor plugins.

### Service Label plugin

More in: [Service Label plugin][service-label]

Other plugins can use this to obtain the microservice label, more specifically the string used to identify a particular VPP instance. 

[acl-plugin]: vpp-plugins.md#access-control-lists-plugin
[bolt-plugin]: https://github.com/ligato/cn-infra/tree/master/db/keyval/bolt
[cassandra-plugin]: https://github.com/ligato/cn-infra/tree/master/db/sql/cassandra
[consul-plugin]: db-plugins.md#consul-plugin
[data-broker-plugin]: db-plugins.md#data-broker
[datasync-plugin]: db-plugins.md#datasync-plugin
[etcd-plugin]: db-plugins.md#etcd-plugin
[file-db-plugin]: db-plugins.md#filedb
[govpp-readme]: https://github.com/FDio/govpp/blob/master/README.md
[govppmux-plugin]: vpp-plugins.md#govppmux-plugin
[grpc-plugin]: connection-plugins.md#grpc-plugin
[index-map]: infra-plugins.md#index-map
[interface-plugin]: vpp-plugins.md#interface-plugin
[interface-plugin-guide]: vpp-plugins.md#interface-plugin
[ipsec-plugin]: vpp-plugins.md#ipsec-plugin
[kvscheduler]: kvs-plugin.md
[l2-plugin]: vpp-plugins.md#l2-plugin
[l3-plugin]: vpp-plugins.md#l3-plugin
[linux-interface-pluign]: linux-plugins.md#interface-plugin
[linux-l3-plugin]: linux-plugins.md#l3-plugin
[linux-namespace-pluign]: linux-plugins.md#namespace-plugin
[linux-iptables-plugin]: https://github.com/ligato/vpp-agent/tree/master/plugins/linux/iptablesplugin
[linux-iptables]: https://linux.die.net/man/8/iptables
[log-manager]: infra-plugins.md#log-manager
[messaging-kafka]: infra-plugins.md#messagingkafka
[nat-plugin]: vpp-plugins.md#nat-plugin
[process-manager]: infra-plugins.md#process-manager
[punt-plugin]: vpp-plugins.md#punt-plugin
[redis-plugin]: db-plugins.md#redis
[rest-plugin]: connection-plugins.md#rest-plugin
[service-label]: infra-plugins.md#status-check
[status-check]: infra-plugins.md#status-check
[telemetry-plugin]: vpp-plugins.md#telemetry 

*[ACL]: Access Control List
*[ARP]: Address Resolution Protocol
*[DPDK]: Data Plane Development Kit
*[NAT]: Network Address Translation
*[SA]: Security Association
*[SPD]: Security Policy Database
*[REST]: Representational State Transfer
*[VNF]: Virtual Network Function
*[VPP]: Vector Packet Processing
