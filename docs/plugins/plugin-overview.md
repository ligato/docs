# Plugin Overview

---

This section provides a brief description of the Ligato plugins. For more information on each plugin, click details.

---

## KV Scheduler Plugin

[Details: KV Scheduler Plugin][kvscheduler]
 
Core plugin that works with SB VPP and Linux agents, and NB KV data stores and rpc clients. It manages configuration item dependencies, and computes the correct VPP configuration item programming sequence. 
 
 ---

## VPP plugins

VPP plugins allow you to manage and control the VPP data plane.   

Plugins:

- GoVPPMux
- Interface
- L2 
- L3
- Access Control List (ACL)
- ACL-based forwarding (ABF)
- IPFIX
- IPSec
- NAT
- Punt
- Segment Routing
- STN 
- Telemetry
- Wireguard

---

### GoVPPMux

[Details: GoVPPMux plugin][govppmux-plugin]

Use this plugin for communication between the VPP agent and the VPP data plane using [GoVPP][govpp-readme]. This plugin serves as the VPP agent's G0VPP wrapper, and provides access to VPP through an independent communication channel using a shared memory segment prefix, or socket client. It supports custom prefixes to connect to the correct VPP instance in a multi-VPP environment. 

---

### Interface

[Details: Interface plugin][interface-plugin]

Use this plugin to configure VPP interfaces. You can configure base fields including IP and MAC address, and more advanced features such as unnumbered interfaces or RX-mode. 

---

### L2 

[Details: L2 plugin][l2-plugin]

Use this plugin to program bridge domains, L2 forwarding tables (FIBs), and VPP cross connects. Dependent on the [interface plugin][interface-plugin-guide].


--- 

### L3 

[Details: L3 plugin][l3-plugin]

Use this plugin to program L3 ARP entries, proxy ARP, VPP routes, IP scan neighbor, VRFs, tunnel endpoint base entries, L3 cross connects, and VRRP. Dependent on the [interface plugin][interface-plugin-guide].

---

### ACL 

[Details: ACL plugin][acl-plugin]

Use this plugin to program VPP access control lists (ACL). If incoming traffic meets the ACL rules, VPP will perform the configured action on the packets.

---

### ACL-based forwarding (ABF)

[Details: ABF plugin][abf-plugin]

Use this plugin to program ACL-based forwarding rules. Performs policy-based routing (PBR) where ACL matches determine packet forwarding.

---

### IPFIX

[Details: IPFIX plugin][ipfix-plugin]

Use this plugin to implement VPP IPFIX monitoring and flow export.

---

### IPSec

[Details: IPSec plugin][ipsec-plugin]

Use this plugin to program VPP security policy databases (SPD), and security associations (SA). It handles relationships between the SPD and SA, or between SPD and an assigned interface. 

!!! Note
    The IPsec plugin does not configure **deprecated IPSec tunnel interfaces**. For more details, see [VPP interface plugin][interface-plugin]. 


---

### NAT

[Details: NAT plugin][nat-plugin]

Use this plugin to program VPP NAT. It provides a control plane function for the VPP NAT44 data plane. This plugin also supports DNAT44 with load balancing for optimized resource efficiency in K8s network clusters. Dependent on the [interface plugin][interface-plugin-guide].

---

### Punt

[Details: Punt plugin][punt-plugin]

Use this plugin to access the VPP punt feature. Incoming VPP traffic matching a set of pre-defined rules is punted, or redirected, to the host stack or socket.

---

### Segment routing

[Details: SR plugin][sr-plugin]

Use this plugin to program VPP segment routing IPv6 (SRv6).

---

### STN

[Details: STN plugin][stn-plugin]

Use this plugin to implement the VPP STN (Steal the NIC) control plane.

---

### Telemetry 

[Details: telemetry plugin][telemetry-plugin]

Use this plugin to collect VPP telemetry stats for export to external monitoring and management tools.

---

### Wireguard

[Details: Wireguard plugin](vpp-plugins.md#wireguard-plugin)

Use this plugin to program VPP [wireguard VPN tunnels](https://www.wireguard.com/).

---

## Linux plugins

This section describes the VPP agent's Linux plugins. You can use these plugins independently, or together with VPP plugins.

Plugins:

- Linux interface
- Linux L3
- IP Tables
- Namespace
- Punt

---

### Linux Interface

[Details: Linux Interface plugin][linux-interface-pluign]

Use this plugin to program Linux interfaces. Interface types include VETH, TAP, loopback, existing, VRF, and dummy. 

--- 

### Linux L3 

[Details: Linux L3 plugin][linux-l3-plugin]

Use this plugin to program Linux routes and ARP entries. Dependent on the Linux interface plugin.

---

### IP Tables 

[Details: Linux iptables plugin][linux-iptables-plugin]

Use this plugin to program [Linux IPtables][linux-iptables].

---

### Namespace 

[Details: Namespace plugin][linux-namespace-pluign]

Use this plugin to provide a helper function tied in with the Linux interface and l3 plugins. It manages Linux namespaces, or as a microservice in container-based environment. 

---

### Punt 

[Details: Punt plugin](https://github.com/ligato/vpp-agent/tree/master/proto/ligato/linux/punt)

Use this plugin to implement the Linux punt-to-host function.

---

## Connection plugins 

VPP agent connection plugins enable external data read/write without the use of a data store.

Plugins:

- REST
- gRPC

---

### REST 

[Details: REST plugin][rest-plugin]

Use this plugin to implement REST API support for a plugin or agent. 

---

### gRPC 

[Details: GRPC plugin][grpc-plugin]

Use this plugin to enable VPP agent gRPC communications. 

---

## Database plugins

Ligato provides plugins for external data store connectivity and integration.

Plugins:

- Datasync
- Data Broker
- etcd
- Redis
- Consul
- Bolt
- Cassandra
- FileDB

### Datasync

[Details: Datasync plugin][datasync-plugin]

Use this plugin to define data synchronization abstractions between your app plugins and different backend data sources such as data stores, message buses, or rpc-connected clients. 

---

### Data broker

[Details: Data Broker plugin][data-broker-plugin]

Use this plugin as a common API abstraction for client access to KV data stores and sql databases.  

---

### etcd

[Details: etcd plugin][etcd-plugin]

Use this plugin for access to an etcd data store.

---

### Redis

[Details: Redis plugin][redis-plugin]

Use this plugin for access to a Redis KV data store.

---

### Consul

[Details: Consul plugin][consul-plugin]

Use this plugin for access to a consul KV data store.

---

### Bolt

[Details: Bolt plugin][bolt-plugin]

Use this plugin for access to a Bolt data store.

---

### Cassandra

[Details: Cassandra][cassandra-plugin]

Use this plugin for access to a Cassandra MYSQL data store.

---

### FileDB 

[Details: FileDB plugin][file-db-plugin]

Use this plugin to access the OS file system serving as an KV data store.

---

## Infra Plugins

Ligato provides infra plugins for logging, messaging, process management, status checking and service label. 

Plugins:

- Configurator
- Orchestrator
- Status Check 
- Index Map
- Log Manager
- Messaging/Kafka
- Process Manager
- Service Label

---

### Configurator

[Details: Configurator plugin][configurator-plugin]

Use this plugin to perform operations on the VPP agent configuration. 

---

### Orchestrator

[Details: Orchestrator plugin][orchestrator-plugin]

Use this plugin to synchronize retrieved data from multiple sources, resolve conflicts, and convey configuration data to the KV Scheduler. The orchestrator reads/watches for config updates from NB clients, and communicates config status, running state, metadata, and metrics for NB client consumption.

---

### Status Check 

Details: [Status Check plugin][status-check]

Use this plugin to monitor agent status by collecting and aggregating partial status of VPP agent plugins.

---

### Index Map plugin

Details: [Index Map plugin][index-map]

Use this plugin to employ a mapping structure that supports configuration item change notifications and retrieval by fields in the value structure.

---

### Log Manager

Details: [Log Manager plugin][log-manager]

Use this plugin to manage global and per-logger log levels.

---

### Messaging/Kafka

Details: [Messaging/Kafka plugin][messaging-kafka]

Use this plugin so agents can publish synchronous/asynchronous messages, and consume selected topics. 

---

### Process Manager

Details: [Process Manager plugin][process-manager]

Use this plugin to create a process instance that manages and monitors plugins.

---

### Service Label 

Details: [Service Label plugin][service-label]

Use this plugin with other plugins to obtain the microservice label string that identifies a particular VPP instance. 

[abf-plugin]: vpp-plugins.md#abf-plugin
[acl-plugin]: vpp-plugins.md#access-control-lists-plugin
[bolt-plugin]: https://github.com/ligato/cn-infra/tree/master/db/keyval/bolt
[cassandra-plugin]: https://github.com/ligato/cn-infra/tree/master/db/sql/cassandra
[configurator-plugin]: https://github.com/ligato/vpp-agent/tree/master/plugins/configurator 
[consul-plugin]: db-plugins.md#consul-plugin
[data-broker-plugin]: db-plugins.md#data-broker
[datasync-plugin]: db-plugins.md#datasync-plugin
[etcd-plugin]: db-plugins.md#etcd-plugin
[file-db-plugin]: db-plugins.md#filedb
[govpp-readme]: https://github.com/FDio/govpp/blob/master/README.md
[govppmux-plugin]: vpp-plugins.md#govppmux-plugin
[grpc-plugin]: connection-plugins.md#grpc-plugin
[index-map]: infra-plugins.md#index-map
[interface-plugin]: vpp-plugins.md#vpp-interface-plugin
[interface-plugin-guide]: vpp-plugins.md#interface-plugin
[ipfix-plugin]: vpp-plugins.md#ipfix-plugin
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
[orchestrator-plugin]: https://github.com/ligato/vpp-agent/tree/master/plugins/orchestrator
[process-manager]: infra-plugins.md#process-manager
[punt-plugin]: vpp-plugins.md#punt-plugin
[redis-plugin]: db-plugins.md#redis
[rest-plugin]: connection-plugins.md#rest-plugin
[service-label]: infra-plugins.md#status-check
[sr-plugin]: vpp-plugins.md#sr-plugin
[status-check]: infra-plugins.md#status-check
[stn-plugin]: vpp-plugins.md#stn-plugin
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
