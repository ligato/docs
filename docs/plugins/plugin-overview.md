# Plugin Overview

---

## KV Scheduler Plugin

More in: [KV Scheduler Plugin][kvscheduler]

The KV Scheduler is the first step in any VPP or Linux related data processing. It validates the existence of the configuration item dependencies, handles local caching, and performs retries if possible or allowed by the underlying plugin. The KV Scheduler does not operate with data directly nor does it call any VPP binary API); It only determines what operations are needed to achieve the desired configuration result. 

## VPP Plugins

VPP agent core VPP plugins (e.g. plugins which are always required when working with VPP). Plugin list:

- GoVPPMux
- Interface plugin
- L2 plugin
- L3 plugin
- Access Control List (ACL)
- ACL-based forwarding (ABF)
- IPSec plugin
- NAT plugin
- Punt plugin
- Segment routing plugin
- STN plugin 
- Telemetry plugin

### GoVPPMux Plugin

More at: [GoVPPMux plugin][govppmux-plugin]

The GoVPPMux plugin is the VPP agent's wrapper around [GoVPP][govpp-readme] and provides access to the VPP. Every plugin can interact with the VPP using the GoVPPMux. It provides an independent communication channel to the VPP instance. The communication is done via shared memory segment prefix or through it socket client. The plugin supports custom prefixes to connect to the correct VPP instance in a multi-VPP environment.


### Interface Plugin

More at: [Interface plugin][interface-plugin]

Create various types of interfaces (e.g. DPDK, MEMIF, TAP ...) in VPP. It can also configure base fields (IP address, MAC address, etc.) as well as more advanced features such as unnumbered interfaces or RX-mode. 

### L2 Plugin

More at: [L2 plugin][l2-plugin]

Setup link-layer configuration items such as **bridge domains**, **forwarding tables** (FIBs) and **VPP cross connects**. Dependent on [interface plugin][interface-plugin-guide]. 

### L3 Plugin

More at: [L3 plugin][l3-plugin]

Configure **ARP** entries (including **proxy ARP**), **VPP routes** and the **IP scan neighbor** feature. The L3 plugin is dependent on the [interface plugin][interface-plugin-guide].

### ACL Plugin

More at: [ACL plugin][acl-plugin]

Handles VPP access control lists. If rules defined in the access list are met by the incoming traffic, the configured action is applied to the packets.

### ACL-based forwarding (ABF) Plugin

Implementation of the ACL-based forwarding feature. Performs policy-based routing (PBR) where forwarding is performed based on ACL matches rather than destination address prefix.

More at: [ABF plugin][abf-plugin]

### IPSec plugin

More at: [IPSec plugin][ipsec-plugin]

Allows one to configure **security policy databases** (SPD) and **security associations** (SA) in VPP. It also handles relationships between the SPD and SA or between SPD and an assigned interface. `IPSec tunnel interfaces are not part of the IPSec plugin.` Their configuration is handled by the [VPP interface plugin][interface-plugin-guide]).

### NAT Plugin

More at: [NAT-plugin][nat-plugins]

Network address translation, or NAT is a method for translating one IP address into another by modifying (rewriting) packet headers. The VPP agent NAT plugin is a `control plane` plugin for the VPP NAT dataplane implementation of NAT44. Can also DNAT44 (with load balancing useful in K8s network clusters. The NAT plugin is dependent on the [interface plugin][interface-plugin-guide].

### Punt Plugin

More at: [punt-plugin][punt-plugin]

Provides access to the VPP punt feature, where incoming VPP traffic matching a set pre-defined rules is 'punted' or redirected to the host stack or socket.

### Segment Routing

More at: [SR plugin][sr-plugin]

Configure segment routing IPv6 (SRv6)

### STN Plugin

Implementation of the `control plane` for the VPP STN (Steal the NIC) feature

### Telemetry

More at: [telemetry plugin][telemetry-plugin]

Collects telemetry statistics from VPP for export to external monitoring and management tools.

## Linux Plugins

This section describes the VPP agent's Linux plugins used to configure the host OS. These plugins can be used as they are, or together with VPP plugins.

- Linux interface plugin
- Linux L3 plugin
- IP Tables plugin
- Namespace plugin

### Linux Interface Plugin

More at: [Linux Interface plugin][linux-interface-pluign]

Processes Linux interfaces related to the adjacent VPP configuration (VEth, TAPv2). Virtual Ethernet interfaces can be created by the VPP agent.

### Linux L3 Plugin

More at: [Linux L3 plugin][linux-l3-plugin]

Configure Linux routes  and ARPs. 

### IP Tables Plugin

More at: [Linux iptables plugin][linux-iptables-plugin]

Implementation of the [Linux IPtables feature][linux-iptables].

### Namespace Plugin

More at: [Namespace plugin][linux-namespace-pluign]

Helper plugin tied in with the Linux interface/l3 plugins. It manages namespaces in terms of Linux (named namespace) or as a microservice in container-based environment. 

## Connection plugins

VPP agent connection plugins enable external data read/write without the use of a data store.

- REST plugin
- GRPC plugin


### REST Plugin

More at: [REST plugin][rest-plugin]

REST API support including security.

### GRPC Plugin

More at: [GRPC plugin][grpc-plugin]

Provides base GRPC support for the VPP agent for handling GRPC requests.

## Database Plugins

This section describes the VPP agent database plugins.

- Datasync abstraction plugin
- Data Broker plugin
- etcd
- Redis
- Consul
- Bolt
- Cassandra
- FileDB

### Datasync Plugin

More at: [Datasync plugin][datasync-plugin]

defines the interfaces for the abstraction of data synchronization between app plugins and different backend data sources.

### Data Broker Plugin

More at: [Data Broker plugin][data-broker-plugin]

data broker abstraction.

### etcd

More at: [etcd plugin][etcd-plugin]

Provides access to an etcd KV data store.

### Redis

More at: [Redis plugin][redis-plugin]

Provides access to an Redis KV data store.

### Consul

More at: [Consul plugin][consul-plugin]

Provides access to a consul KV data store.

### Bolt

More at: [Bolt plugin][bolt-plugin]

Provides access to a Bolt data store.

### Cassandra

More at: [Cassandra][cassandra-plugin]

Provides access to a Cassandra MYSQL data store.

### FileDB 

More at: [FileDB plugin][file-db-plugin]

Uses the OS file system as a KV data store.

## Infra Plugins

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

In a scenario where a single VPP agent instance receives configuration data from multiple sources (KV data store, GRPC, etc), the orchestrator plugin synchronizes retrieved data and resolve conflicts from individual sources. Data-processing plugins then see the data as coming from a single source.

### Status Check 

More at: [Status Check plugin][status-check]

Monitors the status of a Ligato infra app by collecting and aggregating partial status of VPP agent plugins.

### Index Map Plugin

More at: [Index Map plugin][index-map]

Provides an enhanced mapping structure.

### Log Manager

More at: [Log Manager plugin][log-manager]

View and modify log levels of loggers using a REST API.

### Messaging/Kafka

More at: [Messaging/Kafka plugin][messaging-kafka]

Provides single purpose clients for publishing synchronous/asynchronous messages and for consuming selected topics.

### Process Manager

More at: [Process Manager plugin][process-manager]

Set of methods to create a process instance to manage and monitor plugins.

### Service Label Plugin

More at: [Service Label plugin][service-label]

Other plugins can use this plugin to obtain the microservice label, or more specifically the string used to identify a particular VPP instance. 

[abf-plugin]: vpp-plugins.md#abf-plugin
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
[sr-plugin]: vpp-plugins.md#sr-plugin
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
