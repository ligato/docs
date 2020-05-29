# VPP Plugins

This section describes the VPP agent plugins. Each plugin section provides:

- Short description
- Pointers to the `*.proto` containing configuration/NB protobuf API definitions, the 'models.go' file defining the model, and the conf file if one is available.  
- Example configuration interactions using an etcd data store, REST and gPRC.  

---

## GoVPPMux Plugin

The GoVPPMux plugin allows a VPP agent plugin to access VPP through its own dedicated communication channel. This is accomplished using connection multiplexing.

A specific plugin interacts with VPP by requesting the GoVPPMux plugin to obtain its own dedicated communication channel. Behind the scenes, all channels share the same connection between the GoVPP core and the VPP process. This channel-to-multiplexed connection is created during the plugin initialization using the GoVPP core function. 

The notion of channel multiplexing is illustrated from this figure extracted from the [fd.io/govpp repository][fdio-govpp-repo].  


```
                                       +--------------+
    +--------------+                   |              |
    |              |                   |    plugin    |
    |              |                   |              |
    |     App      |                   +--------------+
    |              |            +------+  GoVPP API   |
    |              |            |      +--------------+
    +--------------+   GoVPP    |
    |              |  channels  |      +--------------+
    |  GoVPP core  +------------+      |              |
    |              |            |      |    plugin    |
    +------+-------+            |      |              |
           |                    |      +--------------+
           |                    +------+  GoVPP API   |
           | binary API                +--------------+
           |
    +------+-------+
    |              |
    |  VPP process |
    |              |
    +--------------+
```

### Connection

Tho GoVPP multiplexer supports two connection types:

  - socket client
  - shared memory
    
By default, GoVPP connects to VPP using the `socket client`. This option can be changed using the environment variable of `GOVPPMUX_NOSOCK`, with a fallback to `shared memory`. In the shared memory case, the plugin connects to the VPP instance that uses the default shared memory segment prefix.
 
The default behaviour assumes that there is only a single VPP running in an environment together with the VPP agent. In the case where VPP runs with a customized SHM prefix, or there are several VPP instances running side-by-side, GoVPP must know the SHM prefix in order to connect to the desired VPP 
instance. The prefix must be included in the [GoVPPMux conf file][govppmux-conf-file], with the key `shm-prefix`, and the value matching the VPP shared memory prefix name.

### Multiplexing

The `NewAPIChannel` call returns a new API channel for communication with VPP using the GoVPP core. It uses default buffer sizes for the request and reply Go channels. By default, both are 100 messages long.

A user plugin could blast configuration requests in bulk mode which could overload VPP. To avoid these situations, it is recommended that one use `NewAPIChannelBuffered`, and increase the requests buffer size. This is useful since the buffer for responses is also used to carry VPP notifications and statistics, which can burst in size and frequency on occasion. By increasing the reply channel buffer size, the probability of dropping messages from VPP decreases at the cost of an increased memory footprint. 

**References:**

- [GoVPPMux metrics proto][govppmux-metrics-proto] 
- [GoVPPux metrics model][govppmux-metrics-models]
- [GoVPPMux conf file][govppmux-conf-file] 

---

### Health Checks
  
There are several health check mechanisms associated with the GoVPPMUX plugin. Health check timeouts, retry, intervals and reconnects options can be set in the [GoVPPMux conf file][govppmux-conf-file].

For convenience, here are those options: 

  - `health check probe interval`: time between health check probes
  - `health check reply timeout`: if this timer pops, probe is considered failed
  - `health check threshold`: number of consecutive failed health checks until an error is reported
  - `reply timeout`: if the reply from a channel does not arrive until the timeout elapses, the request fails
  - `shm-prefix`: used for the connection to VPP that is not using the default shared memory prefix
  - `resync-after-reconnect`: resync after the VPP reconnection

!!! Note
    `TraceEnabled` is obsolete and used only in older versions.

**References:**
    
- [GoVPPMux Conf File][govppmux-conf-file]

---  

## VPP Interface Plugin

The interface plugin is used to set up `VPP Interfaces`, manage interface status, and handle both interface and DHCP lease notifications. The interface plugin supports a number of [interface types](#interface-types), all of which correspond to a subset of all interfaces that can be configured in the VPP data plane.   

All of the interface types, except DPDK, can be created or removed directly in VPP, if all necessary conditions are met. For example, an AF_PACKET interface requires a virtual ethernet interface (VETH) on a host in order to attach to it. The PCI and physical DPDK interfaces can be only configured. They cannot be added or removed.
  
VPP uses multiple commands and/or binary APIs to create and configure shared binaries. The VPP agent simplifies this processes by providing a single model with specific extensions depending on the interface type. 
The NB configuration demands are translated to a sequence of binary API calls using the GoVPP library. The binary API calls are used to program a VPP interface.

**References:**

- [VPP interface proto][vpp-interface-proto] 
- [VPP interface models][vpp-interface-model]
- [VPP interface conf file][vpp-interface-conf-file] 

The interface proto file is divided into two parts:
 
 - data common to all interface types.
 - data for a specific interface type.
  
Note that there are exceptions in the common part, where not all fields can be defined on every interface type. For example, a physical address cannot be set to an AF_PACKET or DPDK interface despite the presence of the field in the definition. If set, the value is silently ignored.

Mandatory fields are `interface type` and `logical name`. The name is limited to 63 characters. The interface may define unique fields for the given type.

A key is defined for every interface type. Here is an example of a VPP interface key, with `<name>` being unique for a particular instance of the interface:

```
/vnf-agent/vpp1/config/vpp/v2/interfaces/<name>
```

See the [keys concepts][concept-keys] and the [keys reference][key-reference] to learn more about the interface keys. 

The interface's logical name is for use by the VPP agent. It serves as a reference for other models (e.g. bridge domains). VPP works with indexes, which are generated when the new interface instance is created. 

The index is a unique integer for identification and VPP internal references. In order to parse the name and index, the VPP agent uses a corresponding tag in the VPP binary API. This "name-to-index" mapping entry enables the VPP agent to locate the interface name and and the corresponding index. 

---

### Unnumbered Interfaces

An interface can be assigned with an IPv4 address, an IPv6 address, or both. A VPP 
interface can be set as unnumbered. In this scenario, the VPP interface "borrows" the IP address of from another interface, which is called the target interface. 

To set an interface as unnumbered, omit `ip_addresses` and set `interface_with_ip`. The target interface must exist. If the target interface does not exit, the unnumbered interface will be configured later when the required target interface appears. The target interface must be configured with at least one IP address. 

### Maximum Transmission Unit

The MTU is the size of the largest protocol data unit (PDU) that can be transmitted on the "wire". A custom MTU size for the interface can be set using the `mtu` value in the interface definition. If the field is left empty, VPP uses a default MTU size of 0.

The VPP agent provides an option to automatically set the MTU size for an interface using the [interface conf file][interface-config-file]. If the global MTU size is set to a non-zero value, but the interface configuration contains a different local value, the local value take precedence over the global value.

---

### Interface Types

!!! Note
    In the brief descriptions below, reference will be made to interface `links` or `structures` along with the contents defining various properties and functions specific to the respective interface type. Those details are contained in the VPP interface proto file.
 
| Type| Description |
| --- | ---|   
| UNDEFINED_TYPE | |
| SUB_INTERFACE | Derived from other interfaces by adding a VLAN ID. The sub-interface link of type `SubInterface` requires a name of the parent interface, and the VLAN ID.|
| SOFTWARE_LOOPBACK| internal software interface|
| DPDK | Physical interfaces cannot be added or removed by the VPP agent. The PCI interface must be connected to the VPP and should be configured for use by the Linux kernel. The VPP agent can configure the interface IP and MAC address information or set it as enabled. No special configuration fields are defined. |
| MEMIF | Shared memory packet interface types provide high-performance send and receive functions between VPP instances. Additional fields are defined in `MemifLink`. The most important is the socket filename. The default uses a default socket filename with a zero index which can be used to create memif interfaces. Note that the VPP agent needs to be resynced to register the new memif interface. |
| TAP |exists in two versions, TAPv1 and TAPv2, but only the latter can be configured. TAPv2 is based on [virtio][virtio] which means it runs in a virtual environment, and cooperates with the hypervisor. The `TapLink` provides several setup options available for TAPv2.|
| AF_PACKET |VPP "host" interface. Its primary function is to connect to the host via the OS VEth interface. The af-packet interface cannot exist without its host interface. If the VEth is missing, the configuration is postponed. If the host interface was removed, the VPP agent "un-configures" and caches related af-packet information. The `AfpacketLink` contains only one field with the name of the host interface as defined in the [Linux interface plugin][linux-interface-plugin-guide].
| VXLAN_TUNNEL |Tunneling protocol for encapsulating L2 frames in IP/UDP packets. |
| IPSEC_TUNNEL | Deprecated in VPP 20.01+. [Use IPIP_TUNNEL + ipsec.TunnelProtection] [ipsec-model] instead. See [PR #1638][ipip-tunn-prot-pr] for details.|
| VMXNET3_INTERFACE | High performance virtual network adapter used in VMware networks.  VmxNet3 requires VMware tools to be used. Setup options are described in `VmxNet3Link`.|
| BOND_INTERFACE | Support the Link Aggregation Control Protocol (LACP). Bonding is the process of combining or joining two or more network interfaces together into a single logical interface. |
| GRE_TUNNEL | IP encapsulation protocol defined in RFC2784|
| GTPU_TUNNEL |GPRS Tunneling  Protocol (GTP) for user data (U) employed in GPRS networks|
|IPIP_TUNNEL |IP encapsulation of IP packets defined in RFC1853. Used with ipsec.TunnelProtection as replacement for IPSEC_TUNNEL interface. See [PR #1638][ipip-tunn-prot-pr] for details.|   
 

---

### VPP Interface Configuration Examples

**KV Data Store**
 
Put the interface configuration data into an etcd data store using the correct [VPP interface key][vpp-key-reference].

Example data:
```json
{  
    "name":"loop1",
    "type":"SOFTWARE_LOOPBACK",
    "enabled":true,
    "phys_address":"7C:4E:E7:8A:63:68",
    "ip_addresses":[  
        "192.168.25.3/24",
        "172.125.45.1/24"
    ],
    "mtu":1478
}
```

Use `agentctl` to put the software_loopback interface key-value entry:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/v2/interfaces/loop1 '{"name":"loop1","type":"SOFTWARE_LOOPBACK","enabled":true,"phys_address":"7C:4E:E7:8A:63:68","ip_addresses":["192.168.25.3/24","172.125.45.1/24"],"mtu":1478}'
```

Another example with a memif type interface. Note the memif-specific fields in the `memif` section:
```json
{  
    "name":"memif1",
    "type":"MEMIF",
    "enabled":true,
    "phys_address":"4E:93:2A:38:A7:77",
    "ip_addresses":[  
        "172.125.40.1/24"
    ],
    "mtu":1478,
    "memif":{  
        "master":true,
        "id":1,
        "socket_filename":"/tmp/memif1.sock",
        "secret":"secret"
    }
}
```

Use `agentctl` to put the memif interface key-value entry:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/v2/interfaces/memif1 '{"name":"memif1","type":"MEMIF","enabled":true,"phys_address":"4E:93:2A:38:A7:77","ip_addresses":["172.125.40.1/24"],"mtu":1478,"memif":{"master":true,"id":1,"socket_filename":"/tmp/memif1.sock","secret":"secret"}}'
```

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data.

Use this cURL command to read all Linux interfaces:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces
```

Interface type-specific calls:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/loopback
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/ethernet
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/memif
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/tap
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/vxlan
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/afpacket
```

---

**gRPC**

Prepare the interface data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
	"github.com/ligato/vpp-agent/proto/ligato/vpp/interfaces"
)

memoryInterface := &vpp.Interface{
        Name:        "memif1",
        Type:        interfaces.Interface_MEMIF,
        Enabled:     true,
        IpAddresses: []string{"192.168.10.1/24"},
        Link: &interfaces.Interface_Memif{
            Memif: &interfaces.MemifLink{
                Id:             1,
                Master:         true,
                SocketFilename: "/tmp/memif1.sock",
            },
        },
    }
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Interfaces: []*interfaces.Interface{
				memoryInterface,
			},
			
		}
	}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update the data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

### Interface Status

Interface status is a **collection of interface data** that can be stored in an external database, or sent as a notification.

The VPP agent waits on several types of VPP events such as object creation, deletion or counter updates. The event is processed and all data extracted. Using the collected information, the VPP agent builds a notification which is sent to all registered publishers or databases. Any errors that occurred are also stored. The statistics readout is performed every second.

!!! Note
    The publishing of interface status is disabled by default, since no default publishers are defined. Publishing interface status can be set by adding one or more `StatusPublishers` to the [VPP interface conf file][vpp-interface-conf-file].  

**References:**

- [VPP interface status proto][vpp-interface-state-proto]
- [VPP interface status models][vpp-interface-model]
- [VPP interface status conf file][vpp-interface-conf-file]

The interface status proto defines two objects:

 - InterfaceState - containing status data
 - InterfaceNotification -  a wrapper around the state with additional information required to send a status notification.

Keys used for interface status and errors:

```
// Interface status
/vnf-agent/<label>/vpp/status/v2/interface/<name>

// Errors
/vnf-agent/<label>/vpp/status/v2/interface/errors/<name>
```

The interface status is read-only and published only by the VPP agent.

Data provided by interface status:

  * identification - logical and internal name, software interface index, interface type
  * administrative and operational status
  * physical address, MTU value
  * used duplex 
  * statistics (received, transmitted or dropped packets, bytes, error count, etc.)

---

**Interface Status Usage**

To read status data, use any tool with access to a given database (**etcdctl** or **agentctl** for etcd, **redis-cli** for Redis, **boltbrowser** for BoltDB). 

Use this `agentctl` command to read interface status for all interfaces:
```
agentctl kvdb list /vnf-agent/vpp1/vpp/status/v2/interface/
```

Specific interfaces can be identified by appending their name to the end of the key. Use these `agentctl` commands to read the status of the loop1 and memif1 interfaces respectively:
```
agentctl kvdb list /vnf-agent/vpp1/vpp/status/v2/interface/loop1
agentctl kvdb list /vnf-agent/vpp1/vpp/status/v2/interface/memif1
```
!!! note
    Do not forget to enable status in the conf file. Otherwise, status will not be published to the KV data store or database.


---

## L2 Plugin

The L2 plugin is used to configure VPP link-layer configuration items, notably bridge domains, forwarding tables (FIBs) and VPP cross connects.  

### Bridge Domains

The L2 bridge domain (BD) is a set of interfaces that belong to the same flooding or broadcast domain. Every BD contains several attributes including MAC learning, unicast forwarding, flooding, and an ARP termination table, which all can be enabled or disabled. The BD is identified by a unique ID, which is managed by the plugin and cannot be externally configured. 

**References:**

- [BD proto][bd-proto]
- [BD model][bd-model]  

The BD proto consists of the standard bridge domain configuration parameters including forwarding, learning, assigned interfaces and an ARP termination table. BD interfaces are only referenced. 

The configuration of the interface is handled by the interface plugin. Every referenced interface also contains the bridge domain-specific fields consisting of a bridged virtual interface (BVI) and a split horizon group. 

!!! Note
    Only one BVI is allowed per bridge domain.

The mandatory field for the BD is a logical name. This logical name can be used as a reference for other configuration types, that are configured as part of the specific BD. 

The BD index can be obtained using the standard retrieval methods such as etcdctl get, REST, and agentctl and VPP CLI. Note this value is informative only, since it cannot be changed, set or referenced.

---

**BD Configuration Examples**

**KV Data Store** 
 
Put the BD configuration data into an etcd data store using the [bridge domain key][vpp-key-reference].

Example Data:

```json
{
    "name": "bd1",
    "learn": true,
    "flood": true,
    "forward": true,
    "unknown_unicast_flood": true,
    "arp_termination": true,
    "interfaces": [
        {
            "name": "if1",
            "split_horizon_group": 0,
            "bridged_virtual_interface": true
        },
        {
            "name": "if2",
            "split_horizon_group": 0
        },
        {
            "name": "if2",
            "split_horizon_group": 0
        }
    ],
    "arp_termination_table": [
        {
            "ip_address": "192.168.10.10",
            "phys_address": "a7:65:f1:b5:dc:f6"
        },
        {
            "ip_address": "10.10.0.1",
            "phys_address": "59:6C:45:59:8E:BC"
        }
    ] 
}
```

Use this `etcdctl` command to put the key-value entry:

```bash
etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1 '{"name":"bd1","learn":true,"flood":true,"forward":true,"unknown_unicast_flood":true,"arp_termination":true,"interfaces":[{"name":"if1","split_horizon_group":0,"bridged_virtual_interface":true},{"name":"if2","split_horizon_group":0},{"name":"if2","split_horizon_group":0}],"arp_termination_table":[{"ip_address":"192.168.10.10","phys_address":"a7:65:f1:b5:dc:f6"},{"ip_address":"10.10.0.1","phys_address":"59:6C:45:59:8E:BC"}]}'
```

Use this command to remove the configuration:

```bash
etcdctl del /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1
```

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data.

Use this cURL command to read all BD entries:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/bd
```

---

**gRPC**

Prepare the BD data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
	"github.com/ligato/vpp-agent/proto/ligato/vpp/l2"
)

bd := &l2.BridgeDomains_BridgeDomain{
		Name:                "bd1",
		Flood:               false,
		UnknownUnicastFlood: false,
		Forward:             true,
		Learn:               true,
		ArpTermination:      false,
		Interfaces: []*l2.BridgeDomains_BridgeDomain_Interfaces{
			{
				Name: "if1",
				BridgedVirtualInterface: true,
			}
		},
	}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			BridgeDomains: []*l2.BridgeDomain{
				bd,
			},
			
		}
	}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

### L2 Forwarding Tables  

An L2 forwarding information base (FIB) can be used in network bridging or routing. Inbound packets are forwarded to an outbound interface defined in the FIB table. The VPP agent enables the configuration of static FIB entries with a combination of interface and bridge domain values.

**References:**

- [FIB proto][bd-proto]
- [FIB model][bd-model] 

To configure a FIB entry, a MAC address, interface and bridge domain must be provided, with the condition that the interface is part of the BD. The interface and the BD are referenced with logical names. Note that the FIB entry will not appear in VPP until all conditions are met.  

---

**FIB Configuration Examples**

**KV Data Store**

Put the FIB configuration data into an etcd data store using the [FIB key][vpp-key-reference]
.

Example Data:
```json
{
    "bridge_domain": "bd1",
    "outgoing_interface": "tap1",
    "phys_address": "62:89:C6:A3:6D:5C",
    "action":"FORWARD",
    "static_config": true,
    "bridged_virtual_interface": false
}
```

Use this `etcdctl` command to put the key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/fib/bd1/mac/62:89:C6:A3:6D:5C '{"phys_address":"62:89:C6:A3:6D:5C","bridge_domain":"bd1","outgoing_interface":"tap1","action":"FORWARD","static_config":true,"bridged_virtual_interface":false}'
```

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data.

Use this cURL command to read all FIB entries:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/fib
```

---

**gRPC**

Prepare the L2 FIB data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
	"github.com/ligato/vpp-agent/proto/ligato/vpp/l2"
)

fib := &l2.FIBEntry{
		PhysAddress: "a7:35:45:55:65:75",
		BridgeDomain: "bd1",
		Action: l2.FIBEntry_FORWARD,
		OutgoingInterface: "if1",
	}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Fibs: []*l2.FIBEntry {
				bd,
			},	
		}
	}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:

```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

### L2 Cross Connects

The VPP agent supports the L2 cross connect (xconnect) feature which sets an interface pair to cross connect (xconnect) mode. All packets received on the first interface are transmitted out the second interface. This mode is not bidirectional by default. If required both interfaces in each direction must be set.

**References:**

- [xconnect proto][xc-proto]
- [xconnect model][bd-model]

The xconnect proto references the transmit and receive interfaces. Both interfaces are mandatory fields and must exist in order to set the mode successfully. If one or both are missing, the configuration is postponed. 

---

**Xconnect Configuration Examples**

**KV Data Store**

Put the cross connect data into an etcd data store using the [cross connect key][vpp-key-reference].

Example Data:
```json
{
    "receive_interface": "if1",
    "transmit_interface": "if2"
}
```

Use this `etcdctl` command to put the key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/xconnect/if1 '{"receive_interface":"if1","transmit_interface":"if2"}'
```

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data.

Use this cURL command to read all L2 xconnects: 

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/xc
```

---

**gRPC**

Prepare the cross-connect data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
	"github.com/ligato/vpp-agent/proto/ligato/vpp/l2"
)

xc := &l2.XConnectPair{
		ReceiveInterface: "if1",
		TransmitInterface: "if2",
	}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			XconnectPairs: []*l2.XConnectPair {
				xc,
			},	
		}
	}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

## L3 Plugin

The L3 plugin is capable of configuring ARP entries (including proxy ARP), VPP routes, IP neighbor functions, L3 cross connects, and VRFs.

### ARP  

The address resolution protocol (ARP) is a communication protocol for discovering the MAC address associated with a given IP address. The L3 plugin works with the VPP implementation of ARP. The ARP entry is uniquely identified by the interface and IP address.

The structure of an ARP entry is defined in the [VPP ARP proto][arp-proto]. Every ARP entry includes a MAC address, IP address and an interface. The two latter values are a part of the [VPP ARP entries key][key reference].

**References:**

- [VPP ARP proto][arp-proto]
- [VPP ARP model][L3-models]

---

**ARP Configuration Examples**

**KV Data Store**

Put the ARP data into an etcd data store using the [VPP ARP entry key][vpp-key-reference].

Example data:
```json
{  
    "interface":"if1",
    "ip_address":"192.168.10.21",
    "phys_address":"59:6C:45:59:8E:BD",
    "static":true
}
```
Use this `etcdctl` command to put the key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/arp/tap1/192.168.10.21 '{"interface":"tap1","ip_address":"192.168.10.21","phys_address":"59:6C:45:59:8E:BD","static":true}'
```

Use this command to remove the configuration:

```bash
etcdctl del /vnf-agent/vpp1/config/vpp/v2/arp/tap1/192.168.10.21
```

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data.

Use this cURL command to read all ARP entries:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/arps
```

---

**gRPC**

Prepare the ARP data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
	"github.com/ligato/vpp-agent/proto/ligato/vpp/l3"
)

arp := &l3.ARPEntry{
		Interface: "if1",
		IpAddress: "10.20.30.40/24",
		PhysAddress: "55:65:75:a7:35:45",
	}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Arps: []*l3.ARPEntry {    
			    arp,
			},	
		}
	}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

### Proxy ARP

Proxy ARP is a technique whereby a device on a subnet responds to ARP queries for the IP address of a host on a different network. Proxy ARP IP address ranges are defined via the respective binary API call. In addition, the desired interfaces must be enabled for the proxy ARP feature.  

**References:**

- [VPP Proxy ARP proto][L3-proto]
- [VPP Proxy ARP model][L3-models]  

The Proxy ARP proto lays out a list of IP address ranges, and another for interfaces. The interface must exist in order for it to be enabled for proxy ARP. 

---

**Proxy ARP Configuration Examples**  

**KV Data Store**

Put the ARP data into an etcd data store using the [VPP ARP entry key][vpp-key-reference].

Example Data:
```json
{  
    "interfaces":[  
        {  
            "name":"tap1"
        },
        {  
            "name":"tap2"
        }
    ],
    "ranges":[  
        {  
            "first_ip_addr":"10.0.0.1",
            "last_ip_addr":"10.0.0.3"
        }
    ]
}
```

Use this `etcdctl` command to put the key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/proxyarp-global/settings '{"interfaces":[{"name":"tap1"},{"name":"tap2"}],"ranges":[{"first_ip_addr":"10.0.0.1","last_ip_addr":"10.0.0.3"}]}'
```

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data.

Use these two cURL commands to read the entries for the interfaces and address ranges supported by proxy ARP:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/proxyarp/interfaces
curl -X GET http://localhost:9191/dump/vpp/v2/proxyarp/ranges

```

---

**gRPC**

Prepare the proxy ARP data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
	"github.com/ligato/vpp-agent/proto/ligato/vpp/l3"
)

xc := &l2.XConnectPair{
		ReceiveInterface: "if1",
		TransmitInterface: "if2",
	}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

proxyArp := &l3.ProxyARP{
		Interfaces: []*l3.ProxyARP_Interface{
			{
				Name: "if1",
			},
		},
		Ranges: []*l3.ProxyARP_Range{
			{
				FirstIpAddr: "10.0.0.1",
				LastIpAddr: "10.0.0.255",
			},
		},
	}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```


---

### Routes 

The VPP routing table contains destination network prefixes, a corresponding next_hop IP address and the outbound interface. Routes are grouped in virtual routing and forwarding (VRF) tables, with a default table of index 0.

**References:**

- [VPP routes proto][route-proto]
- [VPP routes model][L3-models]

The VPP routes proto explains the standard routing table fields such a destination network, next_hop IP address. Weight and preference fields are included to assist in path selection.  

The proto defines several route types:

- Intra-VRF route: forwarding is done by a lookup in the local VRF table.

- Inter-VRF route: forwarding is done by a lookup into an external VRF table. 
 
- Drop route: drops network communication destined for specified IP address.

---
 
**Route Configuration Examples**

**KV Data Store**

Put the route data into an etcd data store using the [VPP route key][vpp-key-reference].

Example data for an `intra-VRF route`:
```json
{  
    "type": "INTRA_VRF",
    "dst_network":"10.1.1.3/32",
    "next_hop_addr":"192.168.1.13",
    "outgoing_interface":"tap1",
    "weight":6
}
```

Use the `etcdctl` command to put an `intra-VRF route`:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/route/vrf/0/dst/10.1.1.3/32/gw/192.168.1.13 '{"type":"INTRA_VRF","dst_network":"10.1.1.3/32","next_hop_addr":"192.168.1.13","outgoing_interface":"tap1","weight":6}'
```

Example data for `inter-VRF route`:
```json
{  
    "type":"INTER_VRF",
    "dst_network":"1.2.3.4/32",
    "via_vrf_id":1
}
```

Use this `etcdctl` command to put an inter-VRF route:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/route/vrf/0/dst/1.2.3.4/32/gw '{"type":"INTER_VRF","dst_network":"1.2.3.4/32","via_vrf_id":1}'
```

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data.

Use this cURL command to read VPP routing table entries:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/routes
```

---

**gRPC**

Prepare the VPP route data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
	"github.com/ligato/vpp-agent/proto/ligato/vpp/l3"
)

route := &l3.Route{
		VrfId:             0,
		DstNetwork:        "10.1.1.3/32",
		NextHopAddr:       "192.168.1.13",
		Weight:            60,
		OutgoingInterface: "if1",
	}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Routes: []*l3.Route {}
				route,
			},	
		}
	}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```
 
---
 
### IP Scan-Neighbor

The VPP IP scan-neighbor feature is used to enable or disable periodic IP neighbor scans. The IP scan-neighbor proto defines scan intervals, timers, delays, IPv4 mode, IPv6 mode or dual-stack.

**References:**

- [VPP IP scan-neighbor proto][L3-proto]
- [VPP IP scan-neighbor][L3-models] 

---

**IP Scan-Neighbor Configuration Examples**  

**KV Data Store**

Put the IP scan-neighbor data into an etcd data store using the [VPP IP Scan Neighbor key][vpp-key-reference].

Example data:
```json
{  
    "mode":"BOTH",
    "scan_interval":11,
    "max_proc_time":36,
    "max_update":5,
    "scan_int_delay":16,
    "stale_threshold":26
}
```

Use this `etcdctl` command to put IP scan-neighbor data:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/ipscanneigh-global/settings '{"mode":"BOTH","scan_interval":11,"max_proc_time":36,"max_update":5,"scan_int_delay":16,"stale_threshold":26}'
```

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data.

Use this cURL command to read IP scan-neighbor configuration data:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/ipscanneigh
```

---

**gRPC**

Prepare the IP scan neighbor data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
	"github.com/ligato/vpp-agent/proto/ligato/vpp/l3"
)

ipScanNeigh := &l3.IPScanNeighbor{
		Mode:           l3.IPScanNeighbor_BOTH,
		ScanInterval:   11,
		MaxProcTime:    36,
		MaxUpdate:      5,
		ScanIntDelay:   16,
		StaleThreshold: 26,
	}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			IpscanNeighbor: *l3.IPScanNeighbor {  
				ipScanNeigh,
			},	
		}
	}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

### L3 Cross Connects

The VPP agent supports the L3 cross connect (L3xc) feature. It applies the xconnect paradigm for IPv4 or IPv6 packets by cross connecting all ingress traffic received on an L3-configured interface to an outbound L3 FIB path.

**References:**

- [VPP L3xc proto][l3xc-proto]
- [VPP l3xc model][l3-models]  

The L3xc proto references an ingress IP interface, and outbound path composed of an outgoing interface, next_hop IP address, weight and preference. 

Note that the same function can be achieved using a dedicated VRF table with a default route. The L3xc is more efficient in both memory and CPU consumption.

---

### VRF

The L3 plugin supports VRF tables in scenarios where multiple discrete routing tables are required.    

**References:**

- [VPP VRF table proto][vrf-proto]
- [VPP VRF table model][L3-models]

The VRF table proto defines an ID, protocol setting for IPv4 or IPv6, optional label description, and flow hash settings.

---

## IPSec Plugin

The IPSec plugin handles the configuration of **Security Policy Databases** (SPD) and **Security Associations** (SA) in VPP. 

!!! Note
 The IPsec plugin does not handle IPSec tunnel programming. This is supported by the IPIP_TUNNEL with ipsec.TunnelProtection. The IPSEC_TUNNEL interface has been deprecated.
 
--- 

### Security Policy Database

The SPD specifies the policies that determine the disposition of all inbound or outbound IP traffic from either the host, or the security gateway. The SPD is bound to an SPD interface and contains a table policy entries. Every policy entry points to an IPsec security association (SA). 

**References:**

- [VPP SPD proto][ipsec-proto]
- [VPP SPD model][ipsec-model]

The SPD defines its own unique index within VPP. The user has an option to set their own index in the `uint32` range. The index is a mandatory field in the model because it serves as a unique identifier for the VPP agent as it is a part of the SPD key. 

!!! Note
    Pay special attention when defining an index in the model. Despite the fact the field uses a `string` format, it only accepts plain numbers. Attempts to set non-numeric characters will result in an error.     

Every policy entry field includes `sa_index` used in the SPD for reference to the SA. The field is mandatory. A missing value will generate a configuration error.

The SPD defines two bindings: SA and interface. The interface binding is important since VPP needs to create an empty SPD first. This requires an interface binding. After that, all policy entries are configured and associated with a specific SA. 

---
 
**SPD Configuration Examples**

**KV Data Store**

Put the SPD data into an etcd data store using the [VPP SPD key][vpp-key-reference].

Example Data:
```json
{
  "index":"1",
  "interfaces": [
    { "name": "tap1" }
  ],
  "policy_entries": [
    {
      "priority": 10,
      "is_outbound": false,
      "remote_addr_start": "10.0.0.1",
      "remote_addr_stop": "10.0.0.1",
      "local_addr_start": "10.0.0.2",
      "local_addr_stop": "10.0.0.2",
      "action": 3,
      "sa_index": "20"
    },
    {
      "priority": 10,
      "is_outbound": true,
      "remote_addr_start": "10.0.0.1",
      "remote_addr_stop": "10.0.0.1",
      "local_addr_start": "10.0.0.2",
      "local_addr_stop": "10.0.0.2",
      "action": 3,
      "sa_index": "10"
    }
  ]
}
```

Use this `etcdctl` command to put the key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/ipsec/v2/spd/1 '{"index":"1","interfaces":[{"name":"tap1"}],"policy_entries":[{"priority":10,"is_outbound":false,"remote_addr_start":"10.0.0.1","remote_addr_stop":"10.0.0.1","local_addr_start":"10.0.0.2","local_addr_stop":"10.0.0.2","action":3,"sa_index":"20"},{"priority":10,"is_outbound":true,"remote_addr_start":"10.0.0.1","remote_addr_stop":"10.0.0.1","local_addr_start":"10.0.0.2","local_addr_stop":"10.0.0.2","action":3,"sa_index":"10"}]}'
```

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data.

Use this cURL command to read SPD configuration data:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/ipsec/spds
```

---

**gRPC**

Prepare the IPSec SPD data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
	"github.com/ligato/vpp-agent/proto/ligato/vpp/ipsec"
)

spd := ipsec.SecurityPolicyDatabase{
		Index: "1",
		Interfaces: []*ipsec.SecurityPolicyDatabase_Interface{
			{
				Name: "if1",
			}
		},
		PolicyEntries: []*ipsec.SecurityPolicyDatabase_PolicyEntry{
			{
				Priority:        10,
				IsOutbound:      false,
				RemoteAddrStart: "10.0.0.1",
				RemoteAddrStop:  "10.0.0.1",
				LocalAddrStart:  "10.0.0.2",
				LocalAddrStop:   "10.0.0.2",
				Action:          3,
				SaIndex:         "1",
			},
		},
	}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			IpsecSpds: []*ipsec.SecurityPolicyDatabase {
				spd,
			},	
		}
	}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

### Security Associations

An IPsec security association (SA) is a set of security behaviors negotiated between two or more devices for the purpose of communicating in a secure fashion. The SA includes the agreed upon crypto methods for authentication, encryption and extended sequence number. The IPsec tunnel encapsulation type of authentication header (AH), or encapsulating security protocol (ESP) is established as well. 

**References:**

- [VPP SA proto][ipsec-proto]
- [VPP SA model][ipsec-model] 

!!! Note
 The SPD and SA are applied to an IPIP_TUNNEL interface, and NOT the deprecated IPSEC_TUNNEL interface. 

The SA uses the same indexing system as SPD. The index is a user-defined unique identifier in the `uint32` range. Like the SPD, the SA index is defined as `string` type field in the proto, but can be set only to numerical values. Attempts to use values other than numerical will cause errors. The SA has no dependencies on other configuration types.

---

**SA Configuration Examples**

**KV Data Store**

Put the SA configuration data into an etcd data store using the [VPP SA key][vpp-key-reference].

Example data:
```json
{
  "index":"1",
  "spi": 1001,
  "protocol": 1,
  "crypto_alg": 1,
  "crypto_key": "4a506a794f574265564551694d653768",
  "integ_alg": 2,
  "integ_key": "4339314b55523947594d6d3547666b45764e6a58"
}
```

Use this `etcdctl` command to put the key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/ipsec/v2/sa/1 '{"index":"1","spi":1001,"protocol":1,"crypto_alg":1,"crypto_key":"4a506a794f574265564551694d653768","integ_alg":2,"integ_key":"4339314b55523947594d6d3547666b45764e6a58"}'
```

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data. 

Use this cURL command to read SA configuration data:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/ipsec/sas
```

---

**gRPC**

Prepare the IPSec SA data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
	"github.com/ligato/vpp-agent/proto/ligato/vpp/ipsec"
)

sa := ipsec.SecurityAssociation{
		Index:          "1",
		Spi:            1001,
		Protocol:       1,
		CryptoAlg:      1,
		CryptoKey:      "4a506a794f574265564551694d653768",
		IntegAlg:       2,
		IntegKey:       "4339314b55523947594d6d3547666b45764e6a58",
		EnableUdpEncap: true,
	}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			IpsecSas: []*ipsec.SecurityAssociation {
				sa,
			},	
		}
	}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

## Punt Plugin

The punt plugin is used to configure VPP to punt (re-direct) packets to the TCP/IP host stack. The plugin supports `punt to the host` either directly, or via a Unix domain socket.

**References:**

- [VPP Punt proto][punt-proto]
- [VPP Punt model][punt-model]

---

### Punt to Host Stack

All incoming traffic that match the following criteria will be punted to the host:
 
- matching one of the VPP interface addresses
- matching a defined L3 protocol, L4 protocol, and port
- and would otherwise be dropped

If an optional Unix domain socket path is defined, traffic will be punted using that socket. All fields which serve as a traffic filter are mandatory.

The punt plugin supports two configuration items defined by different keys: L3/L4 protocol and IP redirect. The former in the key is defined as a `string` value. However, that value is transformed to a numeric representation in the VPP binary API call. 

The usage of the L3 protocol, `ALL`, is exclusive for IP punt to host (without socket registration) in the VPP binary VPP API. If used for the IP punt with socket registration, the VPP agent calls the VPP binary API twice with the same parameters for both, IPv4 and IPv6.

!!! danger "Important"
    In order to configure a punt to host via a Unix domain socket, a specific VPP startup-config is required. The attempt to set punt without this results in errors in VPP. This is an example startup-config `punt { socket /tmp/socket/punt }`. The path must match with the one in the northbound data. 

---

**Punt to Host Configuration Examples**

**KV Data Store**

Put the punt configuration data into an etcd data store using the [VPP punt key][vpp-key-reference]. 

Example data:
```json
{  
    "l3_protocol":"IPv4",
    "l4_protocol":"UDP",
    "port":9000,
    "socket_path":"/tmp/socket/path"
}
```

Use this `etcdctl` command to put the key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/tohost/l3/IPv4/l4/UDP/port/9000 {"l3_protocol":"IPv4","l4_protocol":"UDP","port":9000,"socket_path":"/tmp/socket/path"}
```

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data.

Use this cURL command to read punt configuration data:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/punt/sockets
```

---

**gRPC**

Prepare the Punt data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
	"github.com/ligato/vpp-agent/proto/ligato/vpp/punt"
)

punt := &punt.ToHost{
    L3Protocol: punt.L3Protocol_IPv4,
    L4Protocol: punt.L4Protocol_UDP,
    Port:       9000,
    SocketPath: "/tmp/socket/path",
}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
    VppConfig: &vpp.ConfigData{
        PuntTohosts: []*punt.ToHost {
            punt,
        },	
    }
}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

### IP Redirect

IP redirect enables traffic that arrives on an interface, and matching a given IP protocol, to be punted (redirect) to a transmit (TX) interface and next_hop IP address. The IP protocol can be IPv4 or IPv6. The IP protocol, TX, and next_hop fields are defined in the ip redirect proto. Optionally, the received interface (RX) interface can be used to redirect traffic only received on that interface. 

**References:**

- [VPP IP redirect proto][punt-proto]
- [VPP IP redirect model][punt-model] 

IP redirect is defined as the `IpRedirect` object in the proto file. The L3 protocol is defined as a `string` value that is transformed to numeric in the VPP binary API call. 

If L3 protocol is set to `ALL`, the respective API is called for IPv4 and IPv6 separately.

---

**IP Redirect Configuration Example**

**KV Data Store**

Put the IP redirct configuration data into an etcd data store using the [VPP ip redirect key][vpp-key-reference]. 

Example value:
```json
{  
    "l3_protocol":"IPv4",
    "tx_interface":"tap1",
    "next_hop":"192.168.0.1"
}
```

Use this `etcdctl` command to put the key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/ipredirect/l3/IPv4/tx/tap1 '{"l3_protocol":"IPv4","tx_interface":"tap1","next_hop":"192.168.0.1"}'
```

---

**REST** 

There is no Punt IP redirect REST API. 

---

**gRPC**

Prepare the IP redirect data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
	"github.com/ligato/vpp-agent/proto/ligato/vpp/punt"
)

punt := &punt.IPRedirect{
    L3Protocol:  punt.L3Protocol_IPv4,
    TxInterface: "if1",
    NextHop:     "192.168.0.1",
}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
    VppConfig: &vpp.ConfigData{
        PuntTohosts: []*punt.ToHost {
            punt,
        },	
    }
}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

### Limitations

Current limitations for punt to host:

- UDP configuration cannot be shown or configured using the VPP CLI.
- VPP does not provide an API to dump configuration. Thus the VPP agent does not have the opportunity to read existing entries and this may cause certain issues with resync.
* Although the VPP agent supports the L4 TCP  protocol to filter incoming traffic, the VPP data plane does not.
* Configured punt to host entries cannot be removed because VPP does not support this option. Any attempt to do so exits with an error.

Current limitations for a punt to host via unix domain socket:

- Configuration cannot be shown or configured using in the VPP CLI.
- VPP agent cannot read registered entries since the VPP does not provide an API to do so.
- The VPP startup configuration punt section requires a defined unix domain socket path. The VPP limitation is that only one path can be defined at any one time.

Current limitations for IP redirect:

- VPP does not provide API calls to dump existing IP redirect entries. This may cause resync problems.

---

### Known Issues

- If the Unix domain socket path is defined in the startup config, the path `must exist`. Otherwise, VPP will fail to start. 

---

## ACL Plugin

Access Control Lists (ACL) filter network traffic by controlling whether packets are forwarded (permitted) or blocked (deny) at the router’s interfaces based on the criteria specified in the access list.

**References:**

- [VPP ACL proto][acl-proto]
- [VPP ACL model][acl-model]

The VPP agent ACL plugin uses the binary API of the VPP data plane ACL plugin. The version of the VPP ACL plugin is displayed at VPP agent startup. Every ACL consists of  `match` rules that classify the packets to be acted upon, and  `action` rules (actions) to be applied to those packets.

The VPP agent defines an access list with a unique name. VPP generates an index, but the association is under the purview of the VPP agent. Every ACL must contain match rules, and action rules. 

The IP match rule can be specified for a variety of protocols, each with their own parameters. For example, the IP rule for the IP protocol can define the source and destination network addresses the packet must match in order to execute the defined action. Other supported protocols are TCP, UDP and ICMP. 

A single ACL rule can cover multiple protocol-based rules. The MAC-IP match (MACIP rule) defines IP address + mask and MAC address + mask as filtering rules. Note that the IP rules and MACIP rules cannot be combined in the same ACL.

---

**ACL Configuration Examples**

**KV Data Store**

Put the ACL configuration data into an etcd data store using the [VPP ACL key][vpp-key-reference].

Example configuration:
```json
{
    "name": "acl1",
    "interfaces": {
        "egress": [
            "tap1",
            "tap2"
        ],
        "ingress": [
            "tap3",
            "tap4"
        ]
    },
    "rules": [
        {
            "action": 1,
            "ip_rule": {
                "ip": {
                    "destination_network": "10.20.1.0/24",
                    "source_network": "192.168.1.2/32"
                },
                "tcp": {
                    "destination_port_range": {
                        "lower_port": 1150,
                        "upper_port": 1250
                    },
                    "source_port_range": {
                        "lower_port": 150,
                        "upper_port": 250
                    },
                    "tcp_flags_mask": 20,
                    "tcp_flags_value": 10
                }
            }
        }
    ]
}
```

Use this `etcdctl` command to put the key-value entry:
```
etcdctl put /vnf-agent/vpp1/config/vpp/acls/v2/acl/acl1 '{"name":"acl1","interfaces":{"egress":["tap1","tap2"],"ingress":["tap3","tap4"]},"rules":[{"action":1,"ip_rule":{"ip":{"destination_network":"10.20.1.0/24","source_network":"192.168.1.2/32"},"tcp":{"destination_port_range":{"lower_port":1150,"upper_port":1250},"source_port_range":{"lower_port":150,"upper_port":250},"tcp_flags_mask":20,"tcp_flags_value":10}}}]}'
```

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data.

Use these cURL commands to obtain IP or MACIP based Access lists:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/acl/ip
curl -X GET http://localhost:9191/dump/vpp/v2/acl/macip

```

---

**gRPC**

Prepare the ACL data:
```go
import acl "github.com/ligato/vpp-agent/proto/ligato/vpp/acl"

acl := &acl.ACL{
		Name: "aclip1",
		Rules: []*acl.ACL_Rule{
			// ACL IP rule
			{
				Action: acl.ACL_Rule_PERMIT,
				IpRule: &acl.ACL_Rule_IpRule{
					Ip: &acl.ACL_Rule_IpRule_Ip{
						SourceNetwork:      "192.168.1.1/32",
						DestinationNetwork: "10.20.0.1/24",
					},
				},
			},
			// ACL ICMP rule
			{
				Action: acl.ACL_Rule_PERMIT,
				IpRule: &acl.ACL_Rule_IpRule{
					Icmp: &acl.ACL_Rule_IpRule_Icmp{
						Icmpv6: false,
						IcmpCodeRange: &acl.ACL_Rule_IpRule_Icmp_Range{
							First: 150,
							Last:  250,
						},
						IcmpTypeRange: &acl.ACL_Rule_IpRule_Icmp_Range{
							First: 1150,
							Last:  1250,
						},
					},
				},
			},
			// ACL TCP rule
			{
				Action: acl.ACL_Rule_PERMIT,
				IpRule: &acl.ACL_Rule_IpRule{
					Tcp: &acl.ACL_Rule_IpRule_Tcp{
						TcpFlagsMask:  20,
						TcpFlagsValue: 10,
						SourcePortRange: &acl.ACL_Rule_IpRule_PortRange{
							LowerPort: 150,
							UpperPort: 250,
						},
						DestinationPortRange: &acl.ACL_Rule_IpRule_PortRange{
							LowerPort: 1150,
							UpperPort: 1250,
						},
					},
				},
			},
			// ACL UDP rule
			{
				Action: acl.ACL_Rule_PERMIT,
				IpRule: &acl.ACL_Rule_IpRule{
					Udp: &acl.ACL_Rule_IpRule_Udp{
						SourcePortRange: &acl.ACL_Rule_IpRule_PortRange{
							LowerPort: 150,
							UpperPort: 250,
						},
						DestinationPortRange: &acl.ACL_Rule_IpRule_PortRange{
							LowerPort: 1150,
							UpperPort: 1250,
						},
					},
				},
			},
		},
		Interfaces: &acl.ACL_Interfaces{
			Ingress: []string{"tap1", "tap2"},
			Egress:  []string{"tap1", "tap2"},
		},
	}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Acls: []*acl.ACL {
				acl,
			},	
		}
	}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update the data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

## ABF Plugin

The ACL-based forwarding plugin is an implementation of policy-based routing (PBR). With ABF, packets are forwarded based on user-defined fields expressed by ACLs, rather than by a longest match lookup in a routing table.  

**References:**

- [VPP ABF proto][abf-proto]
- [VPP ABF model][abf-model]

The ABF entry is identifed by a numeric index. ABF data consists of a list of interfaces the ABF is attached to, the forwarding paths, and the name of the associated ACL. The ACL represents a dependency for the given ABF; If the ACL is not present, the ABF configuration will be cached until the ACL is created. The same applies for ABF interfaces.   

---

**ABF Configuration Examples**

**KV Data Store**

Put the ABF configuration data into an etcd data store using the [VPP ABF key][vpp-key-reference].

Example value:
```json
{  
   "index":1,
   "acl_name":"aclip1",
   "attached_interfaces":[  
      {  
         "input_interface":"tap1",
         "priority":40
      },
      {  
         "input_interface":"memif1",
         "priority":60
      }
   ],
   "forwarding_paths":[  
      {  
         "next_hop_ip":"10.0.0.10",
         "interface_name":"loop1",
         "weight":20,
         "preference":25
      }
   ]
}
```

Use this `etcdctl` command to put the key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/abfs/v2/abf/1 '{"index":1,"acl_name":"aclip1","attached_interfaces":[{"input_interface":"tap1","priority":40},{"input_interface":"memif1","priority":60}],"forwarding_paths":[{"next_hop_ip":"10.0.0.10","interface_name":"loop1","weight":20,"preference":25}]}'
```

To remove the configuration:
```bash
etcdctl del /vnf-agent/vpp1/config/vpp/abfs/v2/abf/1
``` 

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data. 

Use this cURL command to read ABF configuration data:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/abf
```

---

**gRPC**
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
	vpp_abf "github.com/ligato/vpp-agent/proto/ligato/vpp/abf"
)

abfData := vpp.ABF{
		Index: 1,
		AclName: "acl1",
		AttachedInterfaces: []*vpp_abf.ABF_AttachedInterface{
			{
				InputInterface: "if1",
				Priority: 40,
			},
			{
				InputInterface: "if2",
				Priority: 60,
			},
		},
		ForwardingPaths: []*vpp_abf.ABF_ForwardingPath{
			{
				NextHopIp: "10.0.0.1",
				InterfaceName: "if1",
				Weight: 1,
				Preference: 40,
			},
			{
				NextHopIp: "20.0.0.1",
				InterfaceName: "if2",
				Weight: 1,
				Preference: 60,
			},
		},
	}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
		Abfs: &vpp.ConfigData{
			Abfs: []*vpp.ABF{ {
				natGlobal,
			},
		}
	}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update the data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

## NAT Plugin

Network address translation (NAT) is a method for translating IP addresses belonging to different address domains by modifying the address information in the packet header. The NAT plugin provides control plane functionality for the VPP data plane implementation of NAT44. 

---

### NAT Global Configuration

The global NAT configuration is a special case of data grouped under single key. This means there is no unique character as a part of the key, so there is only one global NAT configuration. Interfaces marked as NAT-enabled should be present in VPP but if not, the KV scheduler caches the configuration for later use when the interface becomes available. 

**References:**

- [VPP NAT global proto][nat-proto]
- [VPP NAT global model ][nat-model]


The NAT global configuration is divided into several independent parts pertaining to specific VPP NAT features:

  - **Forwarding** is a boolean field to enable or disable forwarding.
  - **NAT interfaces** represents a list of interfaces enabled for NAT. If the interface does not exist in VPP, it is cached and potentially configured later. Every interface is defined by its logical name and whether it is an `inside` or an `outside` interface. The enable or disable output feature is also defined here.
  - **Address pools** is a list of "NAT-able" IP addresses for a given VRF table. Despite the name, only one address is defined in the single "pool" entry.
  - **Virtual reassembly** support for datagram fragmentation handling to allow the correct recalculation of higher-level checksums.

---

**NAT Global Configuration Examples**

**KV Data Store**

Put the NAT global configuration data into an etcd data store using the [VPP NAT key][vpp-key-reference].

Example value:
```json
{  
    "nat_interfaces":[  
        {  
            "name":"if1"
        },
        {  
            "name":"if2",
            "output_feature":true
        },
        {  
            "name":"if3",
            "is_inside":true
        }
    ],
    "address_pool":[  
        {  
            "address":"192.168.0.1"
        },
        {  
            "address":"175.124.0.1"
        },
        {  
            "address":"10.10.0.1"
        }
    ],
    "virtual_reassembly":{  
        "timeout":10,
        "max_reassemblies":20,
        "max_fragments":10,
        "drop_fragments":true
    }
}
```

Use this `etcdctl` command to put the key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/nat/v2/nat44-global/settings {"nat_interfaces":[{"name":"tap1"},{"name":"tap2","output_feature":true},{"name":"tap3","is_inside":true}],"address_pool":[{"address":"192.168.0.1"},{"address":"175.124.0.1"},{"address":"10.10.0.1"}],"virtual_reassembly":{"timeout":10,"max_reassemblies":20,"max_fragments":10,"drop_fragments":true}}
```

To remove the configuration:
```bash
etcdctl del /vnf-agent/vpp1/config/vpp/nat/v2/nat44-global/settings
```

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data.

Use this cURL command to read NAT global configuration data:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/nat/global
```

---

**gRPC**

Prepare the global NAT data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
	"github.com/ligato/vpp-agent/proto/ligato/vpp/nat"
)

natGlobal := &nat.Nat44Global{
		Forwarding: false,
		NatInterfaces: []*nat.Nat44Global_Interface{
			{
				Name:          "if1",
				IsInside:      false,
				OutputFeature: false,
			},
			{
				Name:          "if2",
				IsInside:      false,
				OutputFeature: true,
			}
		},
		AddressPool: []*nat.Nat44Global_Address{
			{
				VrfId:    0,
				Address:  "192.168.0.1",
				TwiceNat: false,
			},
			{
				VrfId:    0,
				Address:  "175.124.0.1",
				TwiceNat: false,
			}
		},
		VirtualReassembly: &nat.VirtualReassembly{
			Timeout:         10,
			MaxReassemblies: 20,
			MaxFragments:    10,
			DropFragments:   true,
		},
	}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Nat44Global: &nat.Nat44Global {
				natGlobal,
			},
			
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

### DNAT44

Destination network address translation (DNAT) translates the destination IP address of a packet in one direction and performs the inverse NAT function for packets returning in the opposite direction. In the VPP agent, the DNAT configuration is composed of a list of static and/or identity mappings labelled under a single key. DNAT44 configuration is defined in the [NAT model][nat-model].

**References:**

- [VPP DNAT44 proto][nat-proto]
- [VPP DNAT44 model][nat-model]

DNAT44 consists from two main parts: static mappings and identity mappings.

- static mapping is the case where a single external interface, IP address and port number is mapped to one or more local IP address and port numbers in a static configuration. When packets arrive on the external interface, DNAT44 can load balance packets across two or more local IP address and port number entries present in the static mapping. In other words, if more than one local IP address is defined for single static mapping, the [VPP NAT load balancer][vpp-nat-lb] is automatically invoked.
- Identity mapping defines the interface and IP address from which packets originated from.   

DNAT44 contains a unique label serving as an identifier. However, the DNAT44 configuration is not limited. An arbitrary number of static and identity mappings can be listed under a single label. 

---

**DNAT44 Configuration Examples**

**KV Data Store**

Put the DNAT44 configuration data into an etcd data store using the [VPP DNAT44 key][vpp-key-reference].

Example value:
```json
{  
    "label":"dnat1",
    "st_mappings":[  
        {  
            "external_interface":"if1",
            "external_ip":"192.168.0.1",
            "external_port":8989,
            "local_ips":[  
                {  
                    "local_ip":"172.124.0.2",
                    "local_port":6500,
                    "probability":40
                },
                {  
                    "local_ip":"172.125.10.5",
                    "local_port":2300,
                    "probability":40
                }
            ],
            "protocol":"UDP",
            "twice_nat":"ENABLED"
        }
    ],
    "id_mappings":[  
        {  
            "ip_address":"10.10.0.1",
            "port":2525
        }
    ]
}
```

Use this `etcdctl` command to put the key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/nat/v2/dnat44/dnat1 {"label":"dnat1","st_mappings":[{"external_interface":"tap1","external_ip":"192.168.0.1","external_port":8989,"local_ips":[{"local_ip":"172.124.0.2","local_port":6500,"probability":40},{"local_ip":"172.125.10.5","local_port":2300,"probability":40}],"protocol":"UDP","twice_nat":"ENABLED"}],"id_mappings":[{"ip_address":"10.10.0.1","port":2525}]}
```

To remove the configuration:

```bash
etcdctl del /vnf-agent/vpp1/config/vpp/nat/v2/dnat44/dnat1
```

---

**REST**

REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data. 

Use this cURL command to read IP DNAT44 configuration data:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/nat/dnat
```

---

**gRPC**

Prepare the DNAT data:
```go
dNat := &nat.DNat44{
    Label: "dnat1",
    StMappings: []*nat.DNat44_StaticMapping{
        {
            ExternalInterface: "if1",
            ExternalIp:        "192.168.0.1",
            ExternalPort:      8989,
            LocalIps: []*nat.DNat44_StaticMapping_LocalIP{
                {
                    VrfId:       0,
                    LocalIp:     "172.124.0.2",
                    LocalPort:   6500,
                    Probability: 40,
                },
                {
                    VrfId:       0,
                    LocalIp:     "172.125.10.5",
                    LocalPort:   2300,
                    Probability: 40,
                },
            },
            Protocol: 1,
            TwiceNat: nat.DNat44_StaticMapping_ENABLED,
        },
    },
    IdMappings: []*nat.DNat44_IdentityMapping{
        {
            VrfId:     0,
            IpAddress: "10.10.0.1",
            Port:      2525,
            Protocol:  0,
        },
    },
}
```

Prepare the gRPC configuration data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/vpp"
)

config := &configurator.Config{
    VppConfig: &vpp.ConfigData{
        Dnat44S: []*nat.DNat44 {
            natGlobal,
        },
        
    }
}
```

The configuration data can be combined with any other VPP or Linux configuration.

Update the data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

## SR Plugin

The `srplugin` is designed to configure Segment Routing for IPv6 (SRv6) in VPP.

**References:**

- [VPP SRv6 proto][srv6-proto]
- [VPP SRv6 model][srv6-model]

SRv6 configuration data must be stored in etcd using the srv6 key:
 
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/
```

---

### Configuring Local SIDs
The local SID can be configured using this key:
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/localsid/<SID>
```
where ```<SID>``` (Segment ID) is a unique ID of a local sid and it must be a valid IPv6 address. 

---

### Configuring Policy
The segment routing policy can be configured using this key:
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/policy/<bsid>
```
where ```<bsid>``` is the unique binding SID of the policy. As with any other SRv6 SIDs, it must be a valid IPv6 address. 

The policy can be defined inside multiple segment lists. The VPP implementation does not allow a policy without at least one segment list. Therefore inserting(updating with) policy that has not defined at least one segment list will fail. Note that the value can be written to etcd, but its application to VPP will result in a validation error).

---

### Configuring Steering
The VPP method for steering traffic into an SR policy can be configured using this key:
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/steering/<name>
```
where ```<name>``` is a unique name of steering.

---

## STN Plugin

In some deployments, a host is outfitted with two NICs: one controlled by VPP and the other controlled by the host stack for K8s control plane communications. 

The steal-the-NIC (STN) plugin is used when only a single NIC is supported on the host. In this [single-NIC setup][stn-contiv-vpp], the interface will be “stolen” from the host network stack just before starting the VPP and configured with the same IP address on VPP. The same VPP interface IP address will also be used on the host-to-VPP interconnect TAP interface.

**References:**

- [VPP STN proto][stn-proto]
- [VPP STNmodel][stn-model] 

---

## Telemetry Plugin

The Telemetry plugin is used for exporting telemetry statistics from VPP to [Prometheus][prometheus]. Statistics are published via the registry path `/vpp` on port `9191` and updated every 30 seconds.

**Exported Data**

- VPP runtime (`show runtime`)

```bash
                 Name                 State         Calls          Vectors        Suspends         Clocks       Vectors/Call
    acl-plugin-fa-cleaner-process  event wait                0               0               1          4.24e4            0.00
    api-rx-from-ring                any wait                 0               0              18          2.92e5            0.00
    avf-process                    event wait                0               0               1          1.18e4            0.00
    bfd-process                    event wait                0               0               1          1.21e4            0.00
    cdp-process                     any wait                 0               0               1          1.34e5            0.00
    dhcp-client-process             any wait                 0               0               1          4.88e3            0.00
    dns-resolver-process            any wait                 0               0               1          5.88e3            0.00
    fib-walk                        any wait                 0               0               4          1.67e4            0.00
    flow-report-process             any wait                 0               0               1          3.19e3            0.00
    flowprobe-timer-process         any wait                 0               0               1          1.40e4            0.00
    igmp-timer-process             event wait                0               0               1          1.29e4            0.00
    ikev2-manager-process           any wait                 0               0               7          4.58e3            0.00
    ioam-export-process             any wait                 0               0               1          3.49e3            0.00
    ip-route-resolver-process       any wait                 0               0               1          7.07e3            0.00
    ip4-reassembly-expire-walk      any wait                 0               0               1          3.92e3            0.00
    ip6-icmp-neighbor-discovery-ev  any wait                 0               0               7          4.78e3            0.00
    ip6-reassembly-expire-walk      any wait                 0               0               1          5.16e3            0.00
    l2fib-mac-age-scanner-process  event wait                0               0               1          4.57e3            0.00
    lacp-process                   event wait                0               0               1          2.46e5            0.00
    lisp-retry-service              any wait                 0               0               4          1.05e4            0.00
    lldp-process                   event wait                0               0               1          6.79e4            0.00
    memif-process                  event wait                0               0               1          1.94e4            0.00
    nat-det-expire-walk               done                   1               0               0          5.68e3            0.00
    nat64-expire-walk              event wait                0               0               1          5.01e7            0.00
    rd-cp-process                   any wait                 0               0          174857          4.04e2            0.00
    send-rs-process                 any wait                 0               0               1          3.22e3            0.00
    startup-config-process            done                   1               0               1          4.99e3            0.00
    udp-ping-process                any wait                 0               0               1          2.65e4            0.00
    unix-cli-127.0.0.1:38288         active                  0               0               9          4.62e8            0.00
    unix-epoll-input                 polling          12735239               0               0          1.14e3            0.00
    vhost-user-process              any wait                 0               0               1          5.66e3            0.00
    vhost-user-send-interrupt-proc  any wait                 0               0               1          1.95e3            0.00
    vpe-link-state-process         event wait                0               0               1          2.27e3            0.00
    vpe-oam-process                 any wait                 0               0               4          1.11e4            0.00
    vxlan-gpe-ioam-export-process   any wait                 0               0               1          4.04e3            0.00
    wildcard-ip4-arp-publisher-pro event wait                0               0               1          5.49e4            0.00

```

---

Example:

```bash
# HELP vpp_runtime_calls Number of calls
# TYPE vpp_runtime_calls gauge
...
vpp_runtime_calls{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 7.65806939e+08
...
# HELP vpp_runtime_clocks Number of clocks
# TYPE vpp_runtime_clocks gauge
...
vpp_runtime_clocks{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 1150
...
# HELP vpp_runtime_suspends Number of suspends
# TYPE vpp_runtime_suspends gauge
...
vpp_runtime_suspends{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 0
...
# HELP vpp_runtime_vectors Number of vectors
# TYPE vpp_runtime_vectors gauge
...
vpp_runtime_vectors{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 0
...
# HELP vpp_runtime_vectors_per_call Number of vectors per call
# TYPE vpp_runtime_vectors_per_call gauge
...
vpp_runtime_vectors_per_call{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 0
...
```

- VPP buffers (`show buffers`)

```bash
 Thread             Name                 Index       Size        Alloc       Free       #Alloc       #Free
      0                       default           0        2048      0           0           0           0
      0                 lacp-ethernet           1         256      0           0           0           0
      0               marker-ethernet           2         256      0           0           0           0
      0                       ip4 arp           3         256      0           0           0           0
      0        ip6 neighbor discovery           4         256      0           0           0           0
      0                  cdp-ethernet           5         256      0           0           0           0
      0                 lldp-ethernet           6         256      0           0           0           0
      0           replication-recycle           7        1024      0           0           0           0
      0                       default           8        2048      0           0           0           0
```

---

Example:

```bash
# HELP vpp_buffers_alloc Allocated
# TYPE vpp_buffers_alloc gauge
vpp_buffers_alloc{agent="agent1",index="0",item="default",threadID="0"} 0
vpp_buffers_alloc{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 0
...
# HELP vpp_buffers_free Free
# TYPE vpp_buffers_free gauge
vpp_buffers_free{agent="agent1",index="0",item="default",threadID="0"} 0
vpp_buffers_free{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 0
...
# HELP vpp_buffers_num_alloc Number of allocated
# TYPE vpp_buffers_num_alloc gauge
vpp_buffers_num_alloc{agent="agent1",index="0",item="default",threadID="0"} 0
vpp_buffers_num_alloc{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 0
...
# HELP vpp_buffers_num_free Number of free
# TYPE vpp_buffers_num_free gauge
vpp_buffers_num_free{agent="agent1",index="0",item="default",threadID="0"} 0
vpp_buffers_num_free{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 0
...
# HELP vpp_buffers_size Size of buffer
# TYPE vpp_buffers_size gauge
vpp_buffers_size{agent="agent1",index="0",item="default",threadID="0"} 2048
vpp_buffers_size{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 256
...
...
```

---

- VPP Memory (`show memory`)

```bash
Thread 0 vpp_main
20071 objects, 14276k of 14771k used, 21k free, 12k reclaimed, 315k overhead, 1048572k capacity
```

Example:

```bash
# HELP vpp_memory_capacity Capacity
# TYPE vpp_memory_capacity gauge
vpp_memory_capacity{agent="agent1",thread="vpp_main",threadID="0"} 1.048572e+09
# HELP vpp_memory_free Free memory
# TYPE vpp_memory_free gauge
vpp_memory_free{agent="agent1",thread="vpp_main",threadID="0"} 4000
# HELP vpp_memory_objects Number of objects
# TYPE vpp_memory_objects gauge
vpp_memory_objects{agent="agent1",thread="vpp_main",threadID="0"} 20331
# HELP vpp_memory_overhead Overhead
# TYPE vpp_memory_overhead gauge
vpp_memory_overhead{agent="agent1",thread="vpp_main",threadID="0"} 319000
# HELP vpp_memory_reclaimed Reclaimed memory
# TYPE vpp_memory_reclaimed gauge
vpp_memory_reclaimed{agent="agent1",thread="vpp_main",threadID="0"} 0
# HELP vpp_memory_total Total memory
# TYPE vpp_memory_total gauge
vpp_memory_total{agent="agent1",thread="vpp_main",threadID="0"} 1.471e+07
# HELP vpp_memory_used Used memory
# TYPE vpp_memory_used gauge
vpp_memory_used{agent="agent1",thread="vpp_main",threadID="0"} 1.4227e+07
```

---

- VPP node counters (`show node counters`)

```bash
Count                    Node                  Reason
120406            ipsec-output-ip4            IPSec policy protect
120406               esp-encrypt              ESP pkts received
123692             ipsec-input-ip4            IPSEC pkts received
  3286             ip4-icmp-input             unknown type
120406             ip4-icmp-input             echo replies sent
    14             ethernet-input             l3 mac mismatch
   102                arp-input               ARP replies sent
```

Example:

```bash
# HELP vpp_node_counter_count Count
# TYPE vpp_node_counter_count gauge
vpp_node_counter_count{agent="agent1",item="arp-input",reason="ARP replies sent"} 103
vpp_node_counter_count{agent="agent1",item="esp-encrypt",reason="ESP pkts received"} 124669
vpp_node_counter_count{agent="agent1",item="ethernet-input",reason="l3 mac mismatch"} 14
vpp_node_counter_count{agent="agent1",item="ip4-icmp-input",reason="echo replies sent"} 124669
vpp_node_counter_count{agent="agent1",item="ip4-icmp-input",reason="unknown type"} 3358
vpp_node_counter_count{agent="agent1",item="ipsec-input-ip4",reason="IPSEC pkts received"} 128027
vpp_node_counter_count{agent="agent1",item="ipsec-output-ip4",reason="IPSec policy protect"} 124669
```

---
    
**Configuration File**

The telemetry plugin conf file allows one to change the polling interval, or turn polling off. The `polling-interval` is the time in nanoseconds between reads from the VPP. The parameter `disabled` can be set to `true` in order to disable the telemetry plugin.

[abf-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/abf/models.go
[abf-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/abf/abf.proto
[acl-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/acl/models.go
[acl-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/acl/acl.proto
[arp-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/arp.proto
[bd-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l2/models.go
[bd-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l2/bridge_domain.proto
[concept-keys]: ../user-guide/concepts.md#keys
[fdio-govpp-repo]: https://github.com/FDio/govpp
[fib-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l2/fib.proto
[fdio-vpp-features]: https://fd.io/vppproject/vppfeatures/
[govppmux-conf-file]: ../user-guide/config-files.md#govppmux
[govppmux-metrics-models]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/govppmux/models.go
[govppmux-metrics-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/govppmux/metrics.proto 
[grpc-tutorial]: ../tutorials/08_grpc-tutorial.md
[vpp-interface-conf-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/ifplugin/vpp-ifplugin.conf
[vpp-interface-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/interfaces/models.go
[vpp-interface-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/interfaces/interface.proto
[vpp-interface-state-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/interfaces/state.proto
[ipip-tunn-prot-pr]: https://github.com/ligato/vpp-agent/pull/1638
[ipsec-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/ipsec/models.go
[ipsec-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/ipsec/ipsec.proto
[key-reference]: ../user-guide/reference.md
[L3-models]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/models.go
[L3-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/l3.proto
[l3xc-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/l3xc.proto
[linux-key-reference]: ../user-guide/reference.md#linux-keys
[linux-interface-plugin-guide]: linux-plugins.md#interface-plugin
[nat-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/nat/models.go
[nat-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/nat/nat.proto
[punt-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/punt/models.go
[punt-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/punt/punt.proto
[route-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/route.proto
[srv6-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/srv6/models.go
[srv6-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/srv6/srv6.proto
[xc-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l2/xconnect.proto
[links]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/interface.proto
[vpp-key-reference]: ../user-guide/reference.md#vpp-keys
[vpp-nat-lb]: https://jira.fd.io/browse/VPP-954
[prometheus]: https://prometheus.io/
[stn-contiv-vpp]: https://github.com/contiv/vpp/blob/master/docs/setup/SINGLE_NIC_SETUP.md
[stn-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/stn/models.go
[stn-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/stn/stn.proto
[virtio]: https://wiki.libvirt.org/page/Virtio
[vrf-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/vrf.proto  

*[ABF]: ACL-Based Forwarding
*[ACL]: Access Control List
*[ARP]: Address Resolution Protocol
*[DHCP]: Dynamic Host Configuration Protocol
*[DPDK]: Data Plane Development Kit
*[FIB]: Forwarding Information Base
*[NAT]: Network Address Translation
*[REST]: Representational State Transfer
*[SA]: Security Association
*[SPD]: Security Policy Database
*[VPP]: Vector Packet Processing
