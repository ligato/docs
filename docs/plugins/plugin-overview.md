# Plugin Overview

---

This section provides a brief description of the Ligato plugins. For more information, click details.

---

## KV Scheduler Plugin

[Details: KV Scheduler Plugin][kvscheduler]
 
 Core plugin that works with SB VPP and Linux agents, and NB KV data stores and rpc clients. It resolves configuration item dependencies, and computes the correct VPP configuration item programming sequence. 
 
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

### GoVPPMux plugin

Details: [GoVPPMux plugin][govppmux-plugin]

Use this plugin for VPP agent - VPP data plane communication using [GoVPP][govpp-readme]. This plugin serves as the VPP agent's G0VPP wrapper, and provides access to VPP through an independent communication channel using a shared memory segment prefix, or socket client. It supports custom prefixes to connect to the correct VPP instance in a multi-VPP environment. 

---

### Interface plugin

Details: [Interface plugin][interface-plugin]

Use this plugin to configure VPP interfaces. You can configure base fields including IP and MAC address, and more advanced features such as unnumbered interfaces or RX-mode. 

---

### L2 plugin

Details: [L2 plugin][l2-plugin]

Use this plugin to program link-layer configuration items: bridge domains,  L2 forwarding tables (FIBs), and VPP cross connects. Dependent on the [interface plugin][interface-plugin-guide].


--- 

### L3 plugin

Details: [L3 plugin][l3-plugin]

Use this plugin to program L3 configuration items: ARP entries, proxy ARP, VPP routes, IP scan neighbor, VRFs, tunnel endpoint base entries, L3 cross connects, and VRRP. Dependent on the [interface plugin][interface-plugin-guide].

---

### ACL plugin

Details: [ACL plugin][acl-plugin]

Use this plugin to program VPP access control lists (ACL). If incoming traffic meets the ACL rules, VPP will perform the configured action on the packets.

---

### ACL-based forwarding (ABF) plugin

Details: [ABF plugin][abf-plugin]

Use this plugin to program ACL-based forwarding rules. Performs policy-based routing (PBR) where ACL matches determine packet forwarding.

---

### IPFIX plugin

Details: [IPFIX plugin][ipfix-plugin]

Use this plugin to implement VPP IPFIX monitoring and flow information export.

---

### IPSec plugin

Details: [IPSec plugin][ipsec-plugin]

Use this plugin to program VPP security policy databases (SPD), and security associations (SA). It handles relationships between the SPD and SA, or between SPD and an assigned interface. 

!!! Note
    The IPsec plugin does not configure **deprecated IPSec tunnel interfaces**. For more details, see [VPP interface plugin][interface-plugin]. 


---

### NAT plugin

Details: [NAT-plugin][nat-plugins]

Use this plugin to program VPP NAT. It provides a control plane function for the VPP NAT44 data plane. This plugin also supports DNAT44 with load balancing for optimized resource efficiency in K8s network clusters. Dependent on the [interface plugin][interface-plugin-guide].

---

### Punt plugin

Details: [punt-plugin][punt-plugin]

Use this plugin to access the VPP punt feature. Incoming VPP traffic matching a set of pre-defined rules is punted, or redirected, to the host stack or socket.

---

### Segment routing plugin

Details: [SR plugin][sr-plugin]

Use this plugin to program VPP segment routing IPv6 (SRv6).

---

### STN plugin

Details: [STN plugin][stn-plugin]

Use this plugin to implement the VPP STN (Steal the NIC) control plane.

---

### Telemetry plugin

Details: [telemetry plugin][telemetry-plugin]

Use this plugin to collect VPP telemetry stats for export to external monitoring and management tools.

---

### Wireguard plugin

Details: [wireguard plugin](vpp-plugins.md#wireguard-plugin)

Use this plugin to program [wireguard VPN tunnels](https://www.wireguard.com/) in VPP.

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

### Linux Interface plugin

Details: [Linux Interface plugin][linux-interface-pluign]

Use this plugin to program Linux interfaces. Interface types include VETH, TAP, loopback, existing, VRF, and dummy. 

--- 

### Linux L3 plugin

Details: [Linux L3 plugin][linux-l3-plugin]

Use this plugin to program Linux routes and ARP entries. Dependent on the Linux interface plugin.

---

### IP Tables plugin

Details: [Linux iptables plugin][linux-iptables-plugin]

Use this plugin to program [Linux IPtables][linux-iptables].

---

### Namespace plugin

Details: [Namespace plugin][linux-namespace-pluign]

Use this plugin to provide a helper function tied in with the Linux interface/l3 plugins. It manages Linux namespaces, or as a microservice in container-based environment. 

---

### Punt plugin

Details: [Punt plugin](https://github.com/ligato/vpp-agent/tree/master/proto/ligato/linux/punt)

Use this plugin to implement the Linux punt-to-host function.

---

## Connection plugins

VPP agent connection plugins enable external data read/write without the use of a data store.

Plugins:

- REST
- gRPC

---

### REST plugin

Details: [REST plugin][rest-plugin]

Use this plugin to implement REST API support for a plugin or agent access. 

---

### gRPC plugin

Details: [GRPC plugin][grpc-plugin]

Use this plugin to enable VPP agent gRPC communications. 

---

## Database plugins



- Datasync abstraction plugin
- Data Broker plugin
- etcd
- Redis
- Consul
- Bolt
- Cassandra
- FileDB

### Datasync plugin

Details: [Datasync plugin][datasync-plugin]

defines the interfaces for the abstraction of data synchronization between app plugins and different backend data sources.

### Data Broker plugin

Details: [Data Broker plugin][data-broker-plugin]

data broker abstraction.

### etcd

Details: [etcd plugin][etcd-plugin]

Provides access to an etcd KV data store.

### Redis

Details: [Redis plugin][redis-plugin]

Provides access to an Redis KV data store.

### Consul

Details: [Consul plugin][consul-plugin]

Provides access to a consul KV data store.

### Bolt

Details: [Bolt plugin][bolt-plugin]

Provides access to a Bolt data store.

### Cassandra

Details: [Cassandra][cassandra-plugin]

Provides access to a Cassandra MYSQL data store.

### FileDB 

Details: [FileDB plugin][file-db-plugin]

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

Details: [Status Check plugin][status-check]

Monitors the status of a Ligato infra app by collecting and aggregating partial status of VPP agent plugins.

### Index Map plugin

Details: [Index Map plugin][index-map]

Provides an enhanced mapping structure.

### Log Manager

Details: [Log Manager plugin][log-manager]

View and modify log levels of loggers using a REST API.

### Messaging/Kafka

Details: [Messaging/Kafka plugin][messaging-kafka]

Provides single purpose clients for publishing synchronous/asynchronous messages and for consuming selected topics.

### Process Manager

Details: [Process Manager plugin][process-manager]

Set of methods to create a process instance to manage and monitor plugins.

### Service Label plugin

Details: [Service Label plugin][service-label]

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
