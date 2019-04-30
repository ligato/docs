# Plugin Overview

---

## KV Scheduler plugin

More in: [KVScheduler plugin][kvscheduler]

The KV Scheduler is the first step in any VPP or Linux related data processing. It validates the existence of the configuration item dependencies, handles local caching and performs retries if possible or allowed by the underlying plugin. The KV Scheduler does not operate with data directly (does not call any VPP binary API), only determines what operations are needed to achieve the desired result. Data are processed into low-level objects in adjacent VPP/Linux plugins.

## VPP plugins

VPP-Agent core VPP plugins (e.g. plugins which are always required when working with the VPP). Plugin list:

- GoVPP Multiplexer
- Interface plugin
- L2 plugin
- L3 plugin
- Access Control List (ACL)
- ACL-based forwarding (ABD)
- IPSec plugin
- NAT plugin
- Punt plugin
- STN plugin 
- Telemetry plugin

### GoVPPMux plugin

More in: [GoVPPMux plugin][govppmux-plugin]

The GoVPP mux plugin is Agent's wrapper around the [GoVPP][govpp-readme]and provides access to the VPP. Every plugin can interact with the VPP using the GoVPP mux, which provides an independent communication channel to the VPP instance. The communication is done via shared memory segment prefix, and the plugin also supports custom prefixes to connect to 
the correct VPP instance in the multi-VPP environment.


### Interface plugin

More in: [Interface plugin][interface-plugin]

The interface plugin creates various types of interfaces (e.g. DPDK, MEMIF, TAP ...) in the VPP, and also configure base fields (IP address, MAC address) or more advanced features like unnumbered interfaces or RX-mode. 

### L2 plugin

More in: [l2-plugin][l2-plugin]

The VPP L2 plugin is a base plugin which can be used to configure link-layer related configuration items to the VPP, notably **bridge domains**, **forwarding tables** (FIBs) and **VPP cross connects**. The L2 plugin is strongly dependent on [interface plugin][interface-plugin-guide] since every configuration item has some kind of dependency on it. 

### L3 plugin

More in: [l3-plugin][l3-plugin]

The VPP L3 plugin is capable of configuring **ARP** entries (including **proxy ARP**), **VPP routes** and **IP neighbor** feature. The L3 plugin is dependent on the [interface plugin][interface-plugin-guide] in many aspects since several configuration items require the interface to be already present.

### ACL plugin

More in: [ACL plugin][acl-plugin]

The ACL plugin handles the configuration of the VPP access control lists. The ACL stands out as the permission attached to a given interface. If rules defined in the access list are met by the incoming traffic, packets are allowed to perform defined action.

### ACL-based forwarding (ABF) plugin

Implementation of the ACL-based forwarding feature.

### IPSec plugin

More in: [IPSec plugin][ipsec-plugin]

The IPSec plugin allows to configure **security policy databases** and **security associations** to the VPP, and also handles relations between the SPD and SA or between SPD and an assigned interface. Note that the IPSec tunnel interfaces are not a part of IPSec plugin (their configuration is handled in the [VPP interface plugin][interface-plugin-guide]).

### NAT plugin

More in: [NAT-plugin][nat-plugins]

Network address translation, or NAT is a method of remapping IP address space into another IP address space modifying address information in the packet header. The VPP-Agent Network address translation is control plane plugin for the VPP NAT implementation of NAT44. The NAT plugin is dependent on [interface plugin][interface-plugin-guide].

### Punt plugin

More in: [punt-plugin][punt-plugin]

The punt plugin provides access to the VPP punt feature, where incoming VPP traffic matching pre-defined rules is 'punted' or redirected.

### Telemetry

More in: [telemetry plugin][telemetry-plugin]

The telemetry plugin collects telemetry statistics from the VPP which are then exported for monitoring.

### STN plugin

The implementation of the control plane for the VPP STN (Steal the NIC)

## Linux plugins

This page contains the user guide for VPP-Agent Linux plugins which configure host OS. These plugins can be used as they are, or together with the VPP plugins.

- Linux Interface plugin
- Linux L3 plugin
- IP-tables plugin
- Namespace plugin

### Linux Interface plugin

More in: [Linux Interface plugin][linux-interface-pluign]

The Linux interface plugin processes Linux interfaces related to the adjacent VPP configuration (VEth, TAPv2). Virtual Ethernet interfaces can be directly created by the Agent.

### Linux L3 plugin

More in: [Linux L3 plugin][linux-l3-plugin]:

The Linux L3 plugin can be used to configure Linux routes or ARPs. 

### IP-tables plugin

Implementation of the Linux IP-tables feature.

### Namespace plugin

More in: [Namespace plugin][linux-namespace-pluign]

The namespace plugin is a helper plugin tied with the Linux interface/l3 plugins. It manages namespaces in terms of Linux (named namespace) or as a microservice in the container-based environment. 

## Connection plugins

Connection plugins are VPP-Agent plugins allowing data read or write from outside without data store.

- REST plugin
- GRPC plugin

### REST plugin

More in: [REST plugin][rest-plugin]

VPP-Agent REST API support and REST plugin including security.

### GRPC plugin

More in: [GRPC plugin][grpc-plugin]

The base of the GRPC support in the VPP-Agent is a GRPC plugin, which is an infrastructure plugin allowing to handle GRPC requests.

## Database plugins

User guide for VPP-Agent database plugins.

- Datasync abstraction plugin
- Data Broker plugin
- ETCD
- Redis
- Consul
- Bolt
- Cassandra
- FileDB

### Datasync plugin

More in: [Datasync plugin][datasync-plugin]

Package datasync defines the interfaces for the abstraction of a data synchronization between app plugins and different backend data sources.

### Data Broker plugin

More in: [Data Broker plugin][data-broker-plugin]

The VPP-Agent data broker abstraction.

### ETCD

More in: [ETCD plugin][etcd-plugin]

The ETCD plugin provides access to an ETCD key-value data store.

### Redis

More in: [Redis plugin][redis-plugin]

The Redis plugin provides access to an Redis key-value data store.

### Consul

More in: [Consul plugin][consul-plugin]

The Consul plugin provides access to a consul key-value data store.

### Bolt

The Bolt plugin provides access to a Bolt key-value data store.

### Cassandra

The Cassandra plugin provides access to a Cassandra MYSQL data store.

### FileDB 

More in: [FileDB plugin][file-db-plugin]

The fileDB plugin allows to use the file system of a operating system as a key-value data store.

## Infra plugins

User guide for VPP-Agent infra plugins.

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

In a scenario where a single Agent instance receives configuration data from multiple sources (KV data store, GRPC, etc), orchestrator is used to synchronizing retrieved data and resolve conflicts from individual sources. Data-processing plugins then see the data as from the single source.

### Status Check 

More in: [Status Check plugin][status-check]

An infrastructure plugin monitors the overall status of a CN-Infra based app by collecting and aggregating partial statuses of agents plugins.

### Index Map plugin

More in: [Index Map plugin][index-map]

The idxmap package provides an enhanced mapping structure.

### Log Manager

More in: [Log Manager plugin][log-manager]

Log manager plugin allows to view and modify log levels of loggers using REST API.

### Messaging/Kafka

More in: [Messaging/Kafka plugin][messaging-kafka]

The client package provides single purpose clients for publishing synchronous/asynchronous messages and for consuming selected topics.

### Process Manager

More in: [Process Manager plugin][process-manager]

The process manager plugin provides a set of methods to create a plugin-defined processes instance implementing a set of methods to manage and monitor them.

### Service Label plugin

More in: [Service Label plugin][service-label]

The service label is a small Core Agent Plugin, which other plugins can use to obtain the microservice label, i.e. the string used to identify the particular VNF. 

[acl-plugin]: vpp-plugins.md#access-control-lists-plugin
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
