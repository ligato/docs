# VPP Plugins

This section describes the VPP agent plugins. The information provided for each plugin includes:

- Short description.
- Pointers to the `.proto` files containing configuration/NB protobuf API definitions, the 'models.go' file defining the model, and conf file if available.  
- Configuration programming examples, if available, using an etcd data store, REST and gPRC. 

---

**Agentctl** 

You can manage VPP configurations using [agentctl][agentctl].
```
Usage:	agentctl config COMMAND

COMMANDS
  delete      Delete config in agent
  get         Get config from agent
  history     Show config history
  resync      Run config resync
  retrieve    Retrieve currently running config
  update      Update config in agent
  watch       Watch events
```

---
  
## GoVPPMux Plugin

The GoVPPMux plugin allows a VPP plugin to talk to VPP through its own dedicated communication channel using connection multiplexing.

A VPP plugin interacts with VPP by asking the GoVPPMux plugin to obtain a communication channel. Behind the scenes, all channels share the same connection between the GoVPP core, and the VPP process. Plugin initialization creates the channel-to-multiplexed connection using the GoVPP core function. 

Channel multiplexing figure from the [fd.io/govpp repository][fdio-govpp-repo].  


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

---

### Connection

Tho GoVPP multiplexer supports two connection types:

  - socket client
  - shared memory
    
By default, GoVPP connects to VPP using the `socket client`. You can change this function using the `GOVPPMUX_NOSOCK` environment variable, with a fallback to `shared memory`. In the shared memory case, the plugin connects to the VPP instance that uses the default shared memory segment prefix.
 
The default behaviour assumes only a single VPP running with the VPP agent. In the case where VPP runs with a customized SHM prefix, or you have several VPP instances running side-by-side, GoVPP needs to know the SHM prefix to connect to the correct VPP instance. 

You must include the SHM prefix in the [GoVPPMux conf file][govppmux-conf-file], with the key `shm-prefix`, and the value matching the VPP shared memory prefix name.

---

### Multiplexing

The `NewAPIChannel()` call returns a new API channel for communication with VPP using the GoVPP core. It uses a default buffer size of 100 messages for the request and reply Go channels. 

You can customize the request and reply Go channel buffer sizes using the `NewAPIChannelBuffered`call:

- Increase `reqChanBufSize()` to avoid VPP overload if your plugin sends configuration requests in bulk mode. 
<br>
</br>
- Increase `replyChanBufSize` so you don't drop important VPP notifications and stats, noting they can burst in size and frequency. By increasing the reply buffer size, you decrease VPP message drops at the expense of a larger memory footprint.    

**References:**

- [GoVPPMux metrics proto][govppmux-metrics-proto] 
- [GoVPPux metrics model][govppmux-metrics-models]
- [GoVPPMux conf file][govppmux-conf-file] 

---

### Health Checks
  
The GoVPPMUX plugin supports VPP connection health checking. You can set health check timeouts, retries, intervals, and reconnects options in the [GoVPPMux conf file][govppmux-conf-file].

For your convenience, the VPP connection health check options consist of: 

  - `health-check-probe-interval`: time between health check probes.
<br>
</br>
  - `health-check-reply-timeout`: if this timer pops, probe has failed.
<br>
</br>
  - `health-check-threshold`: number of consecutive failed health checks before reporting an error.

**References:**
    
- [GoVPPMux Conf File][govppmux-conf-file]
- [GoVPPMux config.go](https://github.com/ligato/vpp-agent/blob/master/plugins/govppmux/config.go)

---  

## VPP Interface Plugin

The interface plugin creates VPP interfaces, manages status, and handles notifications. It supports multiple [interface types](#interface-types).    

Some particulars of plugin interface types:

- You can create or remove all interface types in VPP, except PCI and physical DPDK-supported interface types.
<br>
</br>
- You can only _configure_ a PCI and physical DPDK interface; you cannot create or delete these interface types.
<br>
</br>
- You must meet any necessary pre-conditions before you create or remove an interface. For example, before creating an AF_PACKET interface, you need a host VETH interface in place to attach it to.        
</br>


VPP uses multiple commands and/or binary APIs to create and configure interfaces. The VPP agent simplifies this process by providing a single model with specific extensions depending on the interface type. 

The VPP agent translates NB configuration statements into a sequence of binary API calls using the GoVPP library. It uses these binary API calls to program a VPP interface.

**References:**

- [VPP interface proto][vpp-interface-proto] 
- [VPP interface models][vpp-interface-model]
- [VPP interface conf file][vpp-interface-conf-file]

---
 
The VPP agent divides interface proto files into two parts:
 
 - Configuration items common to all interface types. Excepts exist. You cannot define common fields on some interface types. For example, you cannot set a physical address for an AF_PACKET or DPDK interface despite the presence of the field in the definition. If set, the value is silently ignored.
<br>
</br> 
 - Configuration items specific to a given interface type.
  
The `type` and `name` fields are mandatory and limited to 63 characters. The interface proto file may contains fields unique for a given type.

!!! Note
    You must use a unique interface `name` across all VPP interfaces of the same type. For example, two memif interfaces cannot share the same `name`. 
---

The VPP agent defines every interface type with a key.

Example VPP interface key with a unique `<name>` for a specific instance of the interface:

```
/vnf-agent/vpp1/config/vpp/v2/interfaces/<name>
```

To learn more about keys, see [keys concepts][concept-keys] and [keys reference][key-reference]. 

---

The VPP agent uses the interface's logical name as a reference for other models, such as bridge domains, that use the interface. VPP works with indexes that are generated when you create a new instance of an interface. The index is a unique integer for VPP internal referencing and identification. 

In order to parse the name and index, the VPP agent uses a corresponding tag in the VPP binary API. With this "name-to-index" mapping entry, the VPP agent can locate the interface name and corresponding index. 

---

### Unnumbered Interfaces

You typically assign an IPv4 address, an IPv6 address, or both, to an interface. You can also define an interface as _unnumbered_. In this scenario, the VPP interface "borrows" the IP address from another interface, which is called the target interface. 

To configure an unnumbered interface, omit `ip_addresses` and set `interface_with_ip`. The target interface must exist and configured with at least one IP address. If the target interface does not exist, the VPP agent will not configure the unnumbered interface until the required target interface appears. 

### Maximum Transmission Unit

The MTU is the size of the largest protocol data unit (PDU) VPP can transmit on the "wire". You can customize the interface MTU size using the `mtu` value in the interface definition. If you leave the field empty, VPP uses a default MTU size of 0.

The VPP agent provides an option to automatically set the MTU size for an interface using the [interface conf file][interface-config-file]. If you set the global MTU size to a non-zero value, but define a different local value in the interface configuration, the _local value takes precedence_ over the global value.

---

### Interface Types

This table contains the interface type name and notes. Any words labeled as `code` correspond to message names or field names contained in the interface proto file.

 
| Type| Description |
| --- | ---|   
| UNDEFINED_TYPE | |
| SUB_INTERFACE | Derived from other interfaces by adding a VLAN ID. `SubInterface` requires parent interface name, and the VLAN ID.|
| SOFTWARE_LOOPBACK| Internal software interface|
| DPDK | DPDK provides user space drivers to talk directly to physical NICs. You cannot add or remove physical interfaces with the VPP agent. You must connect the PCI to VPP, and configure for Linux kernel user. <br></br>You can configure the interface IP and MAC addresses, or set enable/disable. No special configuration fields are defined. |
| MEMIF | Provides high-performance send/receive functions between VPP instances. `MemifLink` contains additional fields. <br></br>`socket_filename` is the most important. Default is default socket filename with a zero index, and you must use to create memif interfaces. You must perform resync to register a new memif interface. |
| TAP |Exists in two versions, TAPv1 and TAPv2, but you can only configure TAPv2.<br></br>TAPv2 is based on [virtio][virtio]. It runs in a virtual environment, and cooperates with the hypervisor. `TapLink` provide more setup options.|
| AF_PACKET |VPP "host" interface to connects to host via a OS VETH interface. The af-packet interface cannot exist without its host interface. <br></br>If you do not configure the VETH, KV Scheduler postpones the configuration. If you remove the host interface, the VPP agent "un-configures" and caches related af-packet information. <br></br>The `AfpacketLink` contains only one field with the host interface defined as in the [Linux interface plugin][linux-interface-plugin-guide].<br></br>To look over config example, see [afpacket control flow][afpacket-control-flow].
| VXLAN_TUNNEL |Tunneling protocol for encapsulating L2 frames in IP/UDP packets. |
| IPSEC_TUNNEL | Deprecated in VPP 20.01+. [Use IPIP_TUNNEL + ipsec.TunnelProtection] [ipsec-model] instead. See [PR #1638][ipip-tunn-prot-pr] for details.|
| VMXNET3_INTERFACE | VMware virtual network adapter. `VmxNet3Link` contains setup options.|
| BOND_INTERFACE | Supports the Link Aggregation Control Protocol (LACP). Bonding combines two or more network interfaces into a single logical interface. |
| GRE_TUNNEL | IP encapsulation protocol defined in RFC2784|
| GTPU_TUNNEL |GPRS Tunneling  Protocol (GTP) for user data (U) employed in GPRS networks|
|IPIP_TUNNEL |IP packet encapsulation defined in RFC1853. This interface + ipsec.TunnelProtection _replaces_ IPSEC_TUNNEL interface. <br></br>See [PR #1638][ipip-tunn-prot-pr] for details.| 
|WIREGUARD_TUNNEL | VPP wireguard interface. See [VPP wireguard plugin](https://github.com/FDio/vpp/tree/master/src/plugins/wireguard) for details.
|RDMA | RDMA Ethernet Driver for accessing Mellanox NICs in VPP. See [VPP rdma readme](https://github.com/FDio/vpp/blob/master/src/plugins/rdma/rdma_doc.md) for details.| 
 

---

### VPP Interface Configuration Examples

**KV Data Store**
 
Example interface data:
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

Put interface:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/v2/interfaces/loop1 '{"name":"loop1","type":"SOFTWARE_LOOPBACK","enabled":true,"phys_address":"7C:4E:E7:8A:63:68","ip_addresses":["192.168.25.3/24","172.125.45.1/24"],"mtu":1478}'
```

Example memif interface data. Note the memif-specific fields in the `memif` section:
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

Put `MEMIF` interface:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/v2/interfaces/memif1 '{"name":"memif1","type":"MEMIF","enabled":true,"phys_address":"4E:93:2A:38:A7:77","ip_addresses":["172.125.40.1/24"],"mtu":1478,"memif":{"master":true,"id":1,"socket_filename":"/tmp/memif1.sock","secret":"secret"}}'
```

---

**REST**

API Reference: [VPP Interfaces][vpp-interface-rest-api]

GET all VPP interfaces:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces
```

GET VPP interface by type:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/loopback
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/ethernet
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/memif
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/tap
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/vxlan
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/afpacket
```

!!! Note
    Some interface types do not support REST at this time.

---

**gRPC**

Prepare data:
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

---

Prepare configuration:
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

---

Update the VPP configuration using the client of the `ConfigurationClient` type:   
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see:

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)
 
---

### Interface Status

The interface plugin collects interface status information composed of counters, state, and stats. You can store this information in an external database, or generate and transmit a notification.

The VPP agent waits on several types of VPP events such as object creation, object deletion, or counter updates. It processes the event, and extracts all data. 

Using the collected information, the VPP agent builds a notification, and then sends this information, and any errors, to all registered publishers or databases. Stats readout occurs every second. Interface status is read-only, and published only by the VPP agent.

!!! Note
    Publishing interface status is disabled by default. The VPP interface conf file does not contain any default status publishers. Configure interface status publishing by adding one or more `status-publishers` to the [VPP interface conf file][vpp-interface-conf-file].  

**References:**

- [VPP interface status proto][vpp-interface-state-proto]
- [VPP interface status models][vpp-interface-model]
- [VPP interface status conf file][vpp-interface-conf-file]

The interface status proto defines three objects:

 - InterfaceStats - contains counters.
 <br>
 </br>
 - InterfaceState - contains status data
<br></br>
 - InterfaceNotification -  state wrapper with additional information required to send a status notification.

Key for interface status:
```
/vnf-agent/<label>/vpp/status/v2/interface/<name>
```
Key for interface errors:
```
// Errors
/vnf-agent/<label>/vpp/status/v2/interface/errors/<name>
```

---

**Interface Status Usage**

To read status data, use any tool with access to a given database. etcdctl and agentctl to access etcd; redis-cli to access Redis; boltbrowser to access BoltDB, consul cli for consul, and so on. 

Read interface status for all interfaces:
```
agentctl kvdb list /vnf-agent/vpp1/vpp/status/v2/interface/
```

You can specify a particular interface by appending its name to the end of the key. 

Read interface status for `loop1`, and `memif1` interfaces respectively:
```
agentctl kvdb list /vnf-agent/vpp1/vpp/status/v2/interface/loop1
agentctl kvdb list /vnf-agent/vpp1/vpp/status/v2/interface/memif1
```
!!! note
    Do not forget to enable status publishing in the VPP interface conf file. Otherwise, the VPP agent will not publish interface status to the KV data store or database.


---

## L2 Plugin

The L2 plugin supports programming of VPP L2 bridge domains, L2 forwarding tables (FIBs), and VPP L2 cross connects. The [interface plugin](#vpp-interface-plugin) is a dependency.
 

### Bridge Domains

An L2 bridge domain (BD) consists of interfaces that belong to the same flooding or broadcast domain. Every BD contains attributes including MAC learning, unicast forwarding, flooding, and an ARP termination table. You can enable or disable one or more of these attributes.

The L2 plugin identifies a BD by a unique internal ID that you cannot configure. 

**References:**

- [BD proto][bd-proto]
- [BD model][bd-model]
 
The BD proto defines standard bridge domain configuration parameters that include forwarding, mac learning, assigned interfaces, and an ARP termination table.  

The interface plugin handles interface configuration. Every referenced interface contains BD-specific fields consisting of a bridged virtual interface (BVI), and a split horizon group. 

!!! Note
    A bridge domain can only have one BVI.

A BD configuration requires a `name` field. Other configuration types, such as L2 FIBs, use the `name` to reference the BD.   

You can obtain the BD index using etcdctl, REST, agentctl, and VPP CLI. This value is informative only. You cannot change, set or reference it.

---

**BD Configuration Examples**

**KV Data Store** 
 
Example `bd1` BD data:

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

Put `bd1` BD:

```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1 '{"name":"bd1","learn":true,"flood":true,"forward":true,"unknown_unicast_flood":true,"arp_termination":true,"interfaces":[{"name":"if1","split_horizon_group":0,"bridged_virtual_interface":true},{"name":"if2","split_horizon_group":0},{"name":"if2","split_horizon_group":0}],"arp_termination_table":[{"ip_address":"192.168.10.10","phys_address":"a7:65:f1:b5:dc:f6"},{"ip_address":"10.10.0.1","phys_address":"59:6C:45:59:8E:BC"}]}'
```

---

**REST**

API Reference: [L2 Bridge Domain][vpp-bd-rest-api]

GET all bridge domains:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/bd
```

---

**gRPC**

Prepare data:
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

Prepare configuration:
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


Update the VPP configuration using the client of the `ConfigurationClient` type:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see:

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)

---

### L2 Forwarding Tables  

L2 forwarding switches incoming packets to an outbound interface defined in a FIB table. The VPP agent enables the configuration of static FIB entries with a combination of interface and bridge domain values.

**References:**

- [FIB proto][bd-proto]
- [FIB model][bd-model] 

To configure a FIB entry, you need a mac address, interface, and bridge domain (BD). You must configure the interface in the bridge domain. The FIB entries reference the interface and BD with logical names. Note that the FIB entry will not appear in VPP until you meet all conditions. 

---

**FIB Configuration Examples**

**KV Data Store**

Example FIB data:
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

Put FIB:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/l2/v2/fib/bd1/mac/62:89:C6:A3:6D:5C '{"phys_address":"62:89:C6:A3:6D:5C","bridge_domain":"bd1","outgoing_interface":"tap1","action":"FORWARD","static_config":true,"bridged_virtual_interface":false}'
```

---

**REST**

API Reference: [L2 FIB][vpp-l2fib-rest-api]

GET all FIB entries:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/fib
```

---

**gRPC**

Prepare data:
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

---

Prepare configuration:
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

---

Update the VPP configuration using the client of the `ConfigurationClient` type:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see: 

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)


---

### L2 Cross Connects

L2 cross connect (xconnect) defines a pair of interfaces configured as a cross connect. An xconnect switches packets from an incoming interface to an outgoing interface in one direction. If you need a bidirectional  xconnect between the same interface pair, you must configure both interfaces in each direction.

**References:**

- [xconnect proto][xc-proto]
- [xconnect model][bd-model]

The xconnect proto defines receive and transmit interfaces. You must configure both interfaces to set up an xconnect. The KV Scheduler postpones the xconnect configuration until you configure both interfaces.  

---

**Xconnect Configuration Examples**

**KV Data Store**

Example xconnect data:
```json
{
    "receive_interface": "if1",
    "transmit_interface": "if2"
}
```

Put xconnect:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/l2/v2/xconnect/if1 '{"receive_interface":"if1","transmit_interface":"if2"}'
```

---

**REST**

API Reference: [L2 X-Connect][vpp-l2xc-rest-api]

GET all L2 xconnects: 

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/xc
```

---

**gRPC**

Prepare data:
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

---

Prepare configuration:
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

---

Update the VPP configuration using the client of the `ConfigurationClient` type:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see: 

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)

---

## L3 Plugin

The L3 plugin supports the configuration of ARP entries, proxy ARP, VPP routes, IP neighbor functions, L3 cross connects, VRFs, and VRRP.

### ARP  

The address resolution protocol (ARP) discovers the MAC address associated with a given IP address. An interface and IP address define a unique ARP entry in the ARP table. 

The [VPP ARP proto][arp-proto] defines the structure of an ARP entry. Every ARP entry includes a mac address, IP address, and an interface. 

**References:**

- [VPP ARP proto][arp-proto]
- [VPP ARP model][L3-models]

---

**ARP Configuration Examples**

**KV Data Store**

Example ARP data:
```json
{  
    "interface":"if1",
    "ip_address":"192.168.10.21",
    "phys_address":"59:6C:45:59:8E:BD",
    "static":true
}
```
Put ARP entry:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/v2/arp/tap1/192.168.10.21 '{"interface":"tap1","ip_address":"192.168.10.21","phys_address":"59:6C:45:59:8E:BD","static":true}'
```

---

**REST**

API Reference: [L3 ARPs][vpp-l3-arps]

GET all ARP entries:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/arps
```

---

**gRPC**

Prepare data:
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

---

Prepare configuration:
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

---

Update the VPP configuration using the client of the `ConfigurationClient` type::
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see: 

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)


---

### Proxy ARP

Proxy ARP is a technique whereby a device on a subnet responds to ARP queries for the IP address of a host on a different network. You must enable the desired interfaces to support proxy ARP.   

**References:**

- [VPP Proxy ARP proto][L3-proto]
- [VPP Proxy ARP model][L3-models]  

The Proxy ARP proto lays out a list of IP address ranges, and another list for interfaces. You must configure the interface before enabling it for proxy ARP. 

---

**Proxy ARP Configuration Examples**  

**KV Data Store**

Example proxy ARP data:
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

Put proxy ARP:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/v2/proxyarp-global/settings '{"interfaces":[{"name":"tap1"},{"name":"tap2"}],"ranges":[{"first_ip_addr":"10.0.0.1","last_ip_addr":"10.0.0.3"}]}'
```

---

**REST**

API References:

- [VPP Proxy ARP Ranges][vpp-proxy-arp-ranges]
- [VPP Proxy ARP Interfaces][vpp-proxy-arp-interfaces]

GET proxy ARP interfaces:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/proxyarp/interfaces
```
GET proxy ARP ranges:
```
curl -X GET http://localhost:9191/dump/vpp/v2/proxyarp/ranges
```

---

**gRPC**

Prepare data:
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

---

Prepare configuration:
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

---

Update the VPP configuration using the client of the `ConfigurationClient` type:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```
You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see: 

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)




---

### Routes 

The VPP routing table contains destination network prefixes, a corresponding next_hop IP address and the outbound interface. You can deploy one or more virtual routing and forwarding (VRF) tables, with a default table of index 0.

**References:**

- [VPP routes proto][route-proto]
- [VPP routes model][L3-models]

The VPP routes proto defines the standard routing table fields such a destination network, next_hop IP address. The proto includes weight and preference fields to assist in path selection.  

The proto defines several route types:

- Intra-VRF route: prefix lookup done in the local VRF table.

- Inter-VRF route: prefix lookup an done in an external VRF table. 
 
- Drop route: drops packets destined for specified IP address.

---
 
**Route Configuration Examples**

**KV Data Store**

Example `intra-VRF route` data:
```json
{  
    "type": "INTRA_VRF",
    "dst_network":"10.1.1.3/32",
    "next_hop_addr":"192.168.1.13",
    "outgoing_interface":"tap1",
    "weight":6
}
```

Put `intra-VRF route`:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/v2/route/vrf/0/dst/10.1.1.3/32/gw/192.168.1.13 '{"type":"INTRA_VRF","dst_network":"10.1.1.3/32","next_hop_addr":"192.168.1.13","outgoing_interface":"tap1","weight":6}'
```

Example `inter-VRF route`:
```json
{  
    "type":"INTER_VRF",
    "dst_network":"1.2.3.4/32",
    "via_vrf_id":1
}
```

Put `inter-VRF route`:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/v2/route/vrf/0/dst/1.2.3.4/32/gw '{"type":"INTER_VRF","dst_network":"1.2.3.4/32","via_vrf_id":1}'
```

---

**REST**

API Reference: [VPP L3 routes][vpp-l3-routes]

GET VPP routing table entries:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/routes
```

---

**gRPC**

Prepare data:
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

---

Prepare configuration:
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

---

Update the VPP configuration using the client of the `ConfigurationClient` type:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```
You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see: 

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)

 
---
 
### IP Scan-Neighbor

The VPP IP scan-neighbor feature supports periodic IP neighbor scans. You can enable or disable this feature. The IP scan-neighbor proto defines scan intervals, timers, delays, IPv4 mode, IPv6 mode or dual-stack.

**References:**

- [VPP IP scan-neighbor proto][L3-proto]
- [VPP IP scan-neighbor][L3-models] 

---

**IP Scan-Neighbor Configuration Examples**  

**KV Data Store**

Example IP scan neighbor data:
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

Put IP scan-neighbor data:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/v2/ipscanneigh-global/settings '{"mode":"BOTH","scan_interval":11,"max_proc_time":36,"max_update":5,"scan_int_delay":16,"stale_threshold":26}'
```

---

**REST**

API Reference: [VPP IP Scan Neighbor][vpp-l3-ip-scan-neighbor]

GET IP scan-neighbor configuration data:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/ipscanneigh
```

---

**gRPC**

Prepare data:
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

Prepare configuration:
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

Update the VPP configuration using the client of the `ConfigurationClient` type:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```
You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see: 

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)

---

### L3 Cross Connects

The L3 cross connect (L3xc) feature cross connects ingress IPv4 or IPv6 traffic received on an L3-configured interface to an outbound L3 FIB path.

**References:**

- [VPP L3xc proto][l3xc-proto]
- [VPP l3xc model][l3-models]  

The L3xc proto defines an ingress IP interface, and outbound path composed of an outgoing interface, next_hop IP address, weight and preference. 

!!! Note
    You can define the same function using a dedicated VRF table with a default route.

---

### VRF

The L3 plugin supports VRF tables for scenarios where you require multiple discrete routing tables.    

**References:**

- [VPP VRF table proto][vrf-proto]
- [VPP VRF table model][L3-models]

The VRF table proto defines an ID, protocol setting for IPv4 or IPv6, optional label description, and flow hash settings.

---

### VRRP

The L3 plugin supports the [virtual router redundancy protocol](https://tools.ietf.org/html/rfc5798) (VRRP). This protocol enables hosts on a LAN access to one of several gateway routers on the same LAN. You configure a group of VRRP routers as a single virtual router supporting a single gateway address. If the primary router in the group fails, a secondary router with the same gateway address takes over.   

**References:**

- [VRRP proto][vrrp-proto]
- [VRRP model][L3-models]
- [fd.io VRRP API](https://git.fd.io/vpp/tree/src/plugins/vrrp/vrrp.api) 

Example VRRP data:

```
{
    "interface": "if1"
    "vrID": 4
    "priority": 100
    "interval": 100
    "preempt": false
    "accept": false
    "unicast": unicast
    "ip_addresses": [
        "192.168.10.1",
        ]
    "enabled": true
}
            
```
**REST**

API Reference: [VPP VRRP](../api/api-vpp-agent.md#vpp-l3-vrrp)

GET VRRP configuration data:
```json
curl -X GET http://localhost:9191/dump/vpp/v2/vrrps
```
---

## IPFIX Plugin

The IPFIX plugin is used to manage [IPFIX][ipfix-rfc] configuration in VPP. It allows one to:

- configure export of flowprobe information
- configure flowprobe parameters
- enable/disable the flowprobe feature for a specific interface

**References**

- [VPP IPFIX proto][vpp-ipfix-proto]
- [VPP Flowprobe proto][vpp-flowprobe-proto]
- [VPP IPFIX model][vpp-ipfix-model]

Items to keep in mind when deploying flowprobe functionality on an interface:

- flowprobe `cannot` be configured for any interface if flowprobe parameters were not set.
- flowprobe parameters `cannot` be changed if the flowprobe Feature was enabled for at least one interface.

---

## IPSec Plugin

The IPSec plugin handles the configuration of **Security Policies** (SP), a **Security Policy Database (SPD)**, and **Security Associations** (SA) in VPP. 

!!! Note
    The IPsec plugin does not handle IPSec tunnel programming. This is supported by the IPIP_TUNNEL with ipsec.TunnelProtection. The IPSEC_TUNNEL interface has been deprecated.
    

---


### Security Policy Database

!!! Warning
    The previous model of configuring IPsec security policies inside an IPsec security policy database (SPD) is now **deprecated**. In practice, the content of SPDs changes frequently and 
    using a single SPD model for defining all security policies is not convenient or efficient. A new model, SecurityPolicy, was added that allows one to configure each security policy
    as a separate proto message instance. Although the new SecurityPolicy model is backward-compatible with the old one, 
    a request to configure a security policy through the SPD model will return: `error: it is deprecated and no longer supported to define SPs inside SPD model. 
    (use SecurityPolicy model instead)`. See [PR #1679](https://github.com/ligato/vpp-agent/pull/1679) for more details.  

**References:**

- [VPP SPD proto][ipsec-proto]
- [VPP SPD model][ipsec-model]

The SPD defines its own unique index within VPP. The user has an option to set their own index in the `uint32` range. The index is a mandatory field in the model because it serves as a unique identifier for the VPP agent, as it is a part of the [SPD key][vpp-key-reference]. 

In addition, the SPD includes one or more interfaces where configured SA and SP entries are applied. SP entries described below contain an `spd_index` referencing the SPD.
    

---

**SPD Configuration Examples**

**KV Data Store**

Example SPD data:
```json
{
  "index":"1",
  "interfaces": [
    { "name": "tap1" }
  ],
}
```

Put SPD:
```bash
kvdb put /vnf-agent/vpp1/config/vpp/ipsec/v2/spd/1 '{"index":1, "interfaces": [{ "name": "tap1" }]}'
```

---

**REST**

API Reference: [VPP IPsec SPD][vpp-ipsec-spd]

GET SPD configuration data:
```json
curl -X GET http://localhost:9191/dump/vpp/v2/ipsec/spds
```

---

### Security Associations

An IPsec security association (SA) is a set of security behaviors negotiated between two or more devices for the purpose of communicating in a secure fashion. The SA includes the agreed upon crypto methods for authentication, encryption and extended sequence number. The IPsec tunnel encapsulation type of authentication header (AH), or encapsulating security protocol (ESP) is established as well. 

**References:**

- [VPP SA proto][ipsec-proto]
- [VPP SA model][ipsec-model] 

!!! Note
    The SPD, SA and associated security policies are applied to an IPIP_TUNNEL interface, and NOT the deprecated IPSEC_TUNNEL interface. 


Security policy entries described below contain an `sa_index` referencing the SA by its `index` value.

---

**SA Configuration Examples**

**KV Data Store**

Example SA data:
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

Put SA:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/ipsec/v2/sa/1 '{"index":"1","spi":1001,"protocol":1,"crypto_alg":1,"crypto_key":"4a506a794f574265564551694d653768","integ_alg":2,"integ_key":"4339314b55523947594d6d3547666b45764e6a58"}'
```

---

**REST**

API Reference: [VPP IPsec SA][vpp-ipsec-sa] 

GET SA configuration data:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/ipsec/sas
```

---

**gRPC**

Prepare data:
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

Prepare configuration:
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

---

Update the VPP configuration using the client of the `ConfigurationClient` type:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```
You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see: 

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)


---

### Security Policy

IPsec security policies (SP) determine the disposition of all inbound or outbound IP traffic from either the host, or the security gateway. Each SP entry contains indicies pointing to an SPD and SA respectively.

**References:**

- [VPP SP proto][ipsec-proto]
- [VPP SP model][ipsec-model]

---

**Security Policy Configuration Examples**

**KV Data Store**

Example security policy data:
```json
  {
    "spd_index": 1,
    "sa_index": 1,
    "priority": 5,
    "is_outbound": true,
    "remote_addr_start": "0.0.0.0",
    "remote_addr_stop": "255.255.255.255",
    "local_addr_start": "0.0.0.0",
    "local_addr_stop": "255.255.255.255",
    "protocol": 4,
    "remote_port_start": 65535,
    "local_port_start": 65535,
    "action": 3
  }
```

Put SP:
```json
agentctl kvdb put /vnf-agent/vpp1/config/vpp/ipsec/v2/sp/spd/1/sa/1/outbound/local-addresses/0.0.0.0-255.255.255.255/remote-addresses/0.0.0.0-255.255.255.255 '{"spd_index": 1, "sa_index": 1, "priority": 5, "is_outbound": true, "remote_addr_start": "0.0.0.0", "remote_addr_stop": "255.255.255.255", "local_addr_start": "0.0.0.0", "local_addr_stop": "255.255.255.255", "protocol": 4, "remote_port_start": 65535, "remote_port_stop": 65535, "local_port_start": 65535, "local_port_stop": 65535, "action": 3}'
```

---

**REST**

API Reference: [VPP IPsec SP][vpp-ipsec-sp]

GET SP configuration data:
```json
 curl -X GET http://localhost:9191/dump/vpp/v2/ipsec/sps
```

---

## Wireguard Plugin

Use the wireguard plugin to configure [wireguard][wireguard] VPN tunnels in VPP.

**References**

- [Wireguard proto][wireguard-proto]
- [Wireguard model][wireguard-model]

**Wireguard Configuration Examples**

**KV Data Store**

Example wireguard data:
```json
{
  "public_key": "xxxYYYzzz"
  "port": 12314,
  "persistent_keepalive": 10,
  "endpoint": "10.10.2.1",
  "wg_if_name": "wg1",
  "flags": 0,
  "allowed_ips": ["10.10.0.0/24"
  ]
}
```


Put wireguard data:
```
agentctl kvdb put /vnf-agent/vpp1/config/vpp/wg/v1/peer/wg1/endpoint/12314 '{
  "public_key": "xxxYYYzzz"
  "port": 12314,
  "persistent_keepalive": 10,
  "endpoint": "10.10.2.1",
  "wg_if_name": "wg1",
  "flags": 0,
  "allowed_ips": ["10.10.0.0/24"
  ]
}'
```

---

## Punt Plugin

Use the punt plugin to configure VPP to punt (re-direct) packets to the TCP/IP host stack. The plugin supports `punt to the host` either directly, or via a Unix domain socket.

**References:**

- [VPP Punt proto][punt-proto]
- [VPP Punt model][punt-model]

---

### Punt to Host Stack

All incoming traffic that match the following criteria will be punted to the host:
 
- Matching one of the VPP interface addresses.
- Matching a defined L3 protocol, L4 protocol, and port.

Otherwise, drop the packets.

You can define an optional Unix domain socket path to punt traffic. You must include all mandatory traffic filter fields.

The punt plugin supports two configuration items defined by different keys: L3/L4 protocol and IP redirect. The former defines the key as a `string` value. However, that value is transformed to a numeric representation in the VPP binary API call. 

The usage of the L3 protocol, `ALL`, is exclusive for IP punt to host (without socket registration) in the VPP binary VPP API. If used for the IP punt with socket registration, the VPP agent calls the VPP binary API twice with the same parameters for both, IPv4 and IPv6.

!!! danger "Important"
    To configure a punt-to-host via a Unix domain socket, you must include a VPP startup config. Attempts to set punt without this file result in VPP errors. An example startup-config is `punt { socket /tmp/socket/punt }`. The path must match the one in the NB data. 

---

**Punt to Host Configuration Examples**

**KV Data Store**


Example put-to-host data:
```json
{  
    "l3_protocol":"IPv4",
    "l4_protocol":"UDP",
    "port":9000,
    "socket_path":"/tmp/socket/path"
}
```

Put punt-to-host:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/v2/tohost/l3/IPv4/l4/UDP/port/9000 {"l3_protocol":"IPv4","l4_protocol":"UDP","port":9000,"socket_path":"/tmp/socket/path"}
```

---

**REST**

API Reference: [VPP Punt Socket][vpp-punt-socket]

GET punt sockets configuration data:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/punt/sockets
```

---

**gRPC**

Prepare data:
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

---

Prepare configuration:
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

---

Update the VPP configuration using the client of the `ConfigurationClient` type:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```
You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see: 

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)


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


Example IP redirect data:
```json
{  
    "l3_protocol":"IPv4",
    "tx_interface":"tap1",
    "next_hop":"192.168.0.1"
}
```

Put IP redirect:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/v2/ipredirect/l3/IPv4/tx/tap1 '{"l3_protocol":"IPv4","tx_interface":"tap1","next_hop":"192.168.0.1"}'
```

---

**REST** 

!!! Note
    The punt plugin does not support a punt IP redirect REST API. 

---

**gRPC**

Prepare data:
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

---

Prepare data:
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

---

Update the VPP configuration using the client of the `ConfigurationClient` type:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```
You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see: 

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)

---

### Limitations

Current limitations for punt-to-host:

- You cannot show or configure UDP using the VPP CLI.
- VPP does not provide an API to dump the configuration. The VPP agent cannot read existing entries, and this may cause resync problems.
- VPP agent supports the TCP protocol to filter incoming traffic; VPP data plane does not.
* You cannot remove punt-to-host entries because the VPP does not support this option. Any attempt to do so exits with an error.

Current limitations for punt-to-host via unix domain socket:

- You cannot show or configure entries using the VPP CLI.
- VPP agent cannot read registered entries since the VPP does not provide an API to do so.
- VPP startup configuration punt section requires a defined unix domain socket path. VPP can only define one path at any one time.

Current limitations for IP redirect:

- VPP does not provide an API to dump existing IP redirect entries. The VPP agent cannot read existing entries, and this may cause resync problems.

---

### Known Issues

If you define the Unix domain socket path in the startup config, the path _must exist_. Otherwise, VPP will fail to start. 

---

## ACL Plugin

Access Control Lists (ACL) filter network traffic by applying "match-action" rules to ingress or egress packets. The match function classifies the packets; the action defines packet processing functions to perform on matching packets. 

Actions you can define in an ACL:

- deny
- permit 
- reflect

**References:**

- [VPP ACL proto][acl-proto]
- [VPP ACL model][acl-model]

The VPP agent ACL plugin uses the binary API of the [VPP data plane ACL plugin](https://docs.fd.io/vpp/18.01/dir_9e97495e78c4182e4b3c22b8c511d67b.html). The VPP agent displays the VPP ACL plugin version at startup. Every ACL includes match rules defining a match criteria, and `action` rules defining actions to perform on those packets.

The VPP agent associates each ACL with a unique name. VPP generates an index; VPP agent manages the ACL-index binding. You must define match rules, and action rules, for each ACL. 

IP match rules let you specify interfaces, different IP protocols, IP address, port numbers, and L4 protocol, each with their own parameters.For example, you could define your IP rule to match on, ingress interface foo IPv4, port 6231, TCP, source address **A**, and destination address **B**. IP match rules support TCP, UDP, and ICMP.

You cannot mix IP rules and macip rules in the same ACL.

---

**ACL Configuration Examples**

**KV Data Store**

Example ACL data:
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

Put ACL:
```
agentctl kvdb put /vnf-agent/vpp1/config/vpp/acls/v2/acl/acl1 '{"name":"acl1","interfaces":{"egress":["tap1","tap2"],"ingress":["tap3","tap4"]},"rules":[{"action":1,"ip_rule":{"ip":{"destination_network":"10.20.1.0/24","source_network":"192.168.1.2/32"},"tcp":{"destination_port_range":{"lower_port":1150,"upper_port":1250},"source_port_range":{"lower_port":150,"upper_port":250},"tcp_flags_mask":20,"tcp_flags_value":10}}}]}'
```

---

**REST**

API References:

- [VPP IP ACL][vpp-ip-acl]
- [VPP MACIP ACL][vpp-macip-acl]

GET all IP ACLs:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/acl/ip
```

GET all MACIP ACLs:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/acl/macip
```
---

**gRPC**

Prepare data:
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

Prepare configuration:
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

Update the VPP configuration using the client of the `ConfigurationClient` type::
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see: 

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)

---

## ABF Plugin

The ACL-based forwarding (ABF) plugin implements policy-based routing (PBR). With ABF, you forward packets using ACL rules, rather than a destination longest match lookup into a routing table performed in normal IP forwarding.  

**References:**

- [VPP ABF proto][abf-proto]
- [VPP ABF model][abf-model]

The plugin defines an ABF entry with a numeric index. ABF data includes a list of interfaces the ABF attaches to, the forwarding paths, and the associated ACL name. 

The ACL represents a dependency for the given ABF; The KV Scheduler will cache the ABF configuration until you create the ACL. The same applies for ABF interfaces.   

---

**ABF Configuration Examples**

**KV Data Store**

Example ABF data:
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

Put ABF:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/abfs/v2/abf/1 '{"index":1,"acl_name":"aclip1","attached_interfaces":[{"input_interface":"tap1","priority":40},{"input_interface":"memif1","priority":60}],"forwarding_paths":[{"next_hop_ip":"10.0.0.10","interface_name":"loop1","weight":20,"preference":25}]}'
```

---

**REST**

API Reference: [VPP ABF][vpp-abf] 

GET ABF configuration data:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/abf
```

---

**gRPC**

Prepare data:
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

Prepare configuration:
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

Update the VPP configuration using the client of the `ConfigurationClient` type:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```
You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see: 

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)


---

## NAT Plugin

Network address translation (NAT) translates IP addresses belonging to different address domains by modifying the address information in the packet header. The NAT plugin provides control plane functionality for the VPP NAT44 data plane implementation. 

---

### NAT Global Configuration

The NAT global configuration groups configuration data under single global key. You require configured NAT-enabled interfaces in VPP, but if not present, the KV scheduler caches the configuration for later use when the interfaces becomes available. 

**References:**

- [VPP NAT global proto][nat-proto]
- [VPP NAT global model ][nat-model]


---

**KV Data Store**

Example NAT global data:
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

Put NAT global:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/nat/v2/nat44-global/settings {"nat_interfaces":[{"name":"tap1"},{"name":"tap2","output_feature":true},{"name":"tap3","is_inside":true}],"address_pool":[{"address":"192.168.0.1"},{"address":"175.124.0.1"},{"address":"10.10.0.1"}],"virtual_reassembly":{"timeout":10,"max_reassemblies":20,"max_fragments":10,"drop_fragments":true}}
```

Delete NAT global:
```bash
agentctl kvdb del /vnf-agent/vpp1/config/vpp/nat/v2/nat44-global/settings
```

---

**REST**

API Reference: [NAT Global][vpp-nat-global]

GET NAT global:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/nat/global
```

---

**gRPC**

Prepare data:
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

Prepare configuration:
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

Update the VPP configuration using the client of the `ConfigurationClient` type:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```
You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see: 

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)


---

### DNAT44

Destination network address translation (DNAT) translates the destination IP address of a packet in one direction, and performs the inverse NAT function for packets  returning in the opposite direction. 

The DNAT configuration contains a list of static and/or identity mappings labelled under a single key. 

**References:**

- [VPP DNAT44 proto][nat-proto]
- [VPP DNAT44 model][nat-model]

The DNAT44 NAT proto defines static mappings and identity mappings.

Static mapping maps a single external interface, IP address, and port number to one or more local IP address and port numbers. When packets originating from an external network arrive on the external interface, DNAT44 can load balance packets across two or more local IP address and port number entries by invoking the [VPP NAT load balancer][vpp-nat-lb].

Identity mapping defines the interface and IP address from which packets originated from.

DNAT44 contains a unique label serving as an identifier. You can define an arbitrary number of static and identity mappings under a single label.

---

**Twice NAT**

You can specify the source and destination addresses in a single rule using twice NAT. For example, you might require:

- source **A** translates to **B** for destination **C**
<br></br>
- source **A** translates to **Y** for destination **Z**
 
In addition, you can configure address pool to support twice NAT using the `twice_nat` boolean. This lets you specify a source IP address to use in twice NAT processing. 

You can override the default behavior of selecting the first IP address in the address pool with at least one free available port. You may have use cases where multiple twice-nat translations require different source IP addresses.

!!! Note
    VPP does not support the twice NAT address pool feature with load balancing. You can either use static mappings without load balancing where you have only one local IP, or use twice NAT without the twice NAT address pool feature.
 
You can configure static mapping twice NAT function using:

- `twice-nat` defines the twice NAT mode options of enable, disable or self.
<br></br>
- `twice_nat_pool_ip` specifies the source IP address. 

Identity mapping defines the interface and IP address from which packets originated from.   

!!! Note
    In some corner DNAT44 cases, you might have the same source and destination after a DNAT44 translation. For more details on self Twice NAT and a VPP NAT plugin implementations, see [Contiv VPP NAT plugin](https://github.com/contiv/vpp/blob/master/docs/dev-guide/SERVICES.md#the-vpp-nat-plugin).

---

**DNAT44 Configuration Examples**

**KV Data Store**

Example DNAT44 data:
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

Put DNAT44:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/vpp/nat/v2/dnat44/dnat1 {"label":"dnat1","st_mappings":[{"external_interface":"tap1","external_ip":"192.168.0.1","external_port":8989,"local_ips":[{"local_ip":"172.124.0.2","local_port":6500,"probability":40},{"local_ip":"172.125.10.5","local_port":2300,"probability":40}],"protocol":"UDP","twice_nat":"ENABLED"}],"id_mappings":[{"ip_address":"10.10.0.1","port":2525}]}
```


---

**REST**

API Reference: [NAT DNAT44][vpp-nat-dnat] 

GET NAT DNAT44 configuration data:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/nat/dnat
```

---

**gRPC**

Prepare data:
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

Prepare configuration:
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

Update the VPP configuration using the client of the `ConfigurationClient` type::
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

You can combine this configuration data with any other VPP or Linux configuration.

For more details and examples using gRPC, see:

- [GRPC tutorial][grpc-tutorial]
- [grpc_vpp_remote_client example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client)
- [grpc_vpp_notifications example](https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications)


---

## SR Plugin

The `srplugin` supports VPP Segment Routing for IPv6 (SRv6) configurations.

**References:**

- [VPP SRv6 proto][srv6-proto]
- [VPP SRv6 model][srv6-model]


SRv6-global key: 
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/srv6-global
```

---

### Configuring Local SIDs

SRv6 local SID configuration key:
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/localsid/<SID>
```
```<SID>``` notes:
 
 - Unique ID identifying the individual local segment identifier in an SRv6 path.
 - Must be a valid IPv6 address. 

---

### Configuring Policy

SRv6 policy configuration key:
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/policy/<bsid>
```
```<bsid>``` notes:

- Unique binding SID of the policy. 
- Must be a valid IPv6 address. 

You can define an SR policy inside multiple segment lists. The VPP SR implementation does not allow a policy without at least one segment list. Your SR policy configuration will fail, if it does not contain at least one segment list.

!!! Note
   An SR policy that does not contain at least one segment list will still put to the KV data store, but its application to VPP will result in a validation error. 

---

### Configuring Steering

SRv6 steering configuration key:
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/steering/<name>
```
Note that ```<name>``` is a unique name for the steering configuration.

---

## STN Plugin

In some deployments, you might deploy a host with two NICs: one controlled by VPP, and the other controlled by the host stack. 

You can implement the steal-the-NIC (STN) plugin when your host only supports a single NIC. In this [single-NIC setup][stn-contiv-vpp], the plugin steals the interface and its IP address from the host network stack just before VPP startup. Note that the VPP agent uses the same VPP interface IP address on the host-to-VPP interconnect TAP interface.  

**References:**

- [VPP STN proto][stn-proto]
- [VPP STNmodel][stn-model] 

---

## Telemetry Plugin

The Telemetry plugin exports telemetry statistics from VPP to [Prometheus][prometheus]. It publishes stats via the registry path `/vpp` on port `9191`. Updates occur every 30 seconds.

**References:**

- [Telemetry plugin folder](https://github.com/ligato/vpp-agent/tree/master/plugins/telemetry)
- [Telemetry conf file](../user-guide/config-files.md#telemetry)

**API References:**

- [Telemetry][telemetry-plugin]
- [Telemetry Memory][telemetry-memory]
- [Telemetry Runtime][telemetry-runtime]
- [Telemetry Nodecount][telemetry-nodecount]

**Example exported data**

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
    
**Conf File**

You can change the polling interval, disable export to prometheus, and disable/enable the telemetry plugin using the telemetry plugin conf file:
 
 - `polling-interval` time in nanoseconds between reads from the VPP. 
 - `disabled` set to `true` to disable the telemetry plugin.
 - `prometheus-disabled` set to `false` to export stats to prometheus.

[abf-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/abf/models.go
[abf-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/abf/abf.proto
[acl-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/acl/models.go
[acl-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/acl/acl.proto
[arp-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/arp.proto
[agentctl]: ../user-guide/agentctl.md
[afpacket-control-flow]: ../developer-guide/control-flows.md#example-af-packet-interface
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
[ipfix-rfc]: https://www.ietf.org/rfc/rfc7011.txt
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
[vrrp-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/vrrp.proto
[vpp-interface-rest-api]: ../api/api-vpp-agent.md#vpp-interfaces
[vpp-bd-rest-api]: ../api/api-vpp-agent.md#vpp-l2-bridge-domain
[vpp-l2fib-rest-api]: ../api/api-vpp-agent.md#vpp-l2-fib
[vpp-l2xc-rest-api]: ../api/api-vpp-agent.md#vpp-l2-x-connect
[vpp-l3-routes]: ../api/api-vpp-agent.md#vpp-l3-routes
[vpp-l3-arps]: ../api/api-vpp-agent.md#vpp-l3-arps
[vpp-l3-ip-scan-neighbor]: ../api/api-vpp-agent.md#vpp-l3-ip-scan-neighbor
[vpp-proxy-arp-ranges]: ../api/api-vpp-agent.md#vpp-l3-proxy-arp-ranges
[vpp-proxy-arp-interfaces]: ../api/api-vpp-agent.md#vpp-l3-proxy-arp-interfaces
[vpp-nat-global]: ../api/api-vpp-agent.md#vpp-nat-global
[vpp-nat-dnat]: ../api/api-vpp-agent.md#vpp-nat-dnat
[vpp-nat-interfaces]: ../api/api-vpp-agent.md#vpp-nat-interfaces
[vpp-nat-pool]: ../api/api-vpp-agent.md#vpp-nat-pool
[vpp-ipsec-spd]: ../api/api-vpp-agent.md#vpp-ipsec-spd
[vpp-ipsec-sa]: ../api/api-vpp-agent.md#vpp-ipsec-sa
[vpp-ipsec-sp]: ../api/api-vpp-agent.md#vpp-ipsec-sp
[vpp-punt-socket]: ../api/api-vpp-agent.md#vpp-punt-socket
[vpp-ip-acl]: ../api/api-vpp-agent.md#vpp-acl-ip
[vpp-macip-acl]: ../api/api-vpp-agent.md#vpp-acl-macip
[vpp-abf]: ../api/api-vpp-agent.md#vpp-abf
[vpp-ipfix-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/ipfix/ipfix.proto
[vpp-flowprobe-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/ipfix/flowprobe.proto
[vpp-ipfix-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/ipfix/models.go
[vpp-telemetry]: ../api/api-vpp-agent.md#vpp-telemetry
[vpp-telemetry-memory]: ../api/api-vpp-agent.md#vpp-telemetrymemory
[vpp-telemetry-runtime]: ../api/api-vpp-agent.md#vpp-telemetryruntime
[vpp-telemetry-nodecount]: ../api/api-vpp-agent.md#vpp-telemetrynodecount
[linux-interfaces]: ../api/api-vpp-agent.md#linux-interfaces
[linux-l3-arps]: ../api/api-vpp-agent.md#linux-l3-arps
[linux-interface-stats]: ../api/api-vpp-agent.md#linux-interface-stats
[linux-l3-routes]: ../api/api-vpp-agent.md#linux-l3-routes
[stats-configurator]: ../api/api-vpp-agent.md#statsconfigurator
[vpp-cli-command]: ../api/api-vpp-agent.md#vpp-cli-command
[vrf-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/vrf.proto
[telemetry-plugin]: ../api/api-vpp-agent.md#vpp-telemetry
[telemetry-memory]: ../api/api-vpp-agent.md#vpp-telemetrymemory
[telemetry-runtime]: ../api/api-vpp-agent.md#vpp-telemetryruntime
[telemetry-nodecount]: ../api/api-vpp-agent.md#vpp-telemetrynodecount
[agentctl-config]: ../user-guide/agentctl.md#config
[wireguard]: https://www.wireguard.com/
[wireguard-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/wireguard/wireguard.proto
[wireguard-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/wireguard/models.go 

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
