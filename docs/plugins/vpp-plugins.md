# VPP Plugins

---

## GoVPPMux plugin

The `govppmux` plugin allows other plugins to access VPP independently by means of connection multiplexing.

Any plugin that interacts with VPP can ask `govppmux` to obtain its own communication channel to access a running VPP instance. Behind the scenes, all channels share the same connection created during the plugin initialization using `govpp` core function.

### Connection

Tho GoVPP multiplexer supports two connection types:

  - socket client
  - shared memory
    
By default the GoVPP connects using the `socket client`. This option can be changed using the environment variable `GOVPPMUX_NOSOCK` with a fallback to `shared memory`. In that case, the plugin connects to that VPP instance which uses the default shared memory segment prefix.
 
The default behaviour assumes that there is only a single VPP running in a sand-boxed environment together with the VPP agent. In the case where VPP runs with a customized SHM prefix, or there are several VPP instances running side-by-side, GoVPP must know the prefix in order to connect to the desired VPP 
instance. The prefix must be included in the GoVPPMux configuration file `govpp.conf` with the key `shm-prefix` and the value matching the VPP shared memory prefix name.

### Multiplexing

The `NewAPIChannel` call returns a new API channel for communication with VPP via the `GoVPP` core. It uses default buffer sizes for the request and reply go channels. By default both are 100 messages long.

A user plugin could blast configuration requests in bulk mode which could overload VPP. To avoid these situations, it is recommended one use `NewAPIChannelBuffered` and increase the buffer size for requests. This is useful since the buffer for responses is also used to carry VPP notifications and statistics which can burst in size and frequency on occassion. By increasing the reply channel buffer size, the probability of dropping messages from VPP decreases at the cost of an increased memory footprint. 


### Trace
 
The duration of a VPP binary API call can be measured using the trace feature. Data is logged after every event (e.g. resynchronization, new interfaces, bridge domains, fib entries etc.). 

Enable trace in the `govpp.conf`: 
 
`trace-enabled: true` or  `trace-enabled: false`
  
The trace feature is disabled by default. 

The trace plugin supports several configuration items for the VPP health-check probe:

  - `health check probe interval`: time between health check probes
  - `health check reply timeout`: if this timer pops, probe is considered failed
  - `health check threshold`: number of consecutive failed health checks until an error is reported
  - `reply timeout`: if the reply from a channel does not arrive until timeout elapses, the request fails
  - `shm-prefix`: used for connection to a VPP instance not using default shared memory prefix
  - `resync-after-reconnect`: resync after the VPP reconnection
  
## Interface plugin

The VPP interface plugin is used to setup `VPP Interfaces`, manage interface status, and interface and DHCP lease notifications. VPP can support multiple interface types with the following supported by the VPP agent:

- UNDEFINED_TYPE
- SUB_INTERFACE
- SOFTWARE_LOOPBACK
- DPDK
- MEMIF
- TAP
- AF_PACKET
- VXLAN_TUNNEL
- IPSEC_TUNNEL = 8 [deprecated=true]; // Deprecated in VPP 20.01+. Use IPIP_TUNNEL + ipsec.TunnelProtection instead.
- VMXNET3_INTERFACE
- BOND_INTERFACE
- GRE_TUNNEL
- GTPU_TUNNEL
- IPIP_TUNNEL


Other types are specified as undefined.

All the interface types except DPDK (physical) can be created or removed directly in the VPP if all necessary conditions are met. For example an Af-packet interface requires a VEth as a host in order to attach to it. The DPDK interfaces (PCI, physical) can be only configured. They cannot be added or removed.
  
VPP uses multiple commands and/or binary APIs to create and configure shared binaries. The VPP agent simplifies this processes by providing a single model with specific extensions depending on the interface type. 
The northbound configuration demands are translated to a sequence of binary API calls using the GoVPP library for eventual programming of the interface VPP.

The interface plugin defines following [interface model][interface-model] and is divided into two parts:
 
 - data common to all interface types.
 - data for a specific interface type.
  
Note that there are exceptions in the common part, where not all fields can be defined on every interface type. For example, a physical address cannot be set to Af-packet or to DPDK interface despite the presence of the field in the definition. If set, the value is silently ignored.

Mandatory fields are `interface type` and `logical name`. The name is limited to 63 characters. The interface may define unique fields for the given type.

A key is defined for every interface type. Here is an example of a VPP interface key with `<name>` being unique for a particular instance of the interface:

```
/vnf-agent/vpp1/config/vpp/v2/interfaces/<name>

```

See the [key reference][key-reference] to learn more about the interface keys. 

The interface's logical name is for use by the VPP agent. It serves as a reference for other models (e.g. bridge domains). VPP works with indexes, which are generated when the new interface instance is created. 

The index is a unique integer for identification and future VPP internal references. In order to parse the name and index, the VPP agent uses a corresponding tag in the VPP binary API. This "name-to-index" mapping entry enables the VPP agent to locate the interface name and and the corresponding index. 

### Unnumbered interfaces

An interface can be assigned with IPv4 or IPv6 addresses or some combination thereof. A VPP 
interface can also be set as unnumbered. It this scenario it "borrows" the IP address of another interface, the so called "target" interface. 

To set an interface as unnumbered, omit `ip_addresses` and set `interface_with_ip`. The target interface 
must exist and if it does not, the unnumbered interface will be postponed and configured later when the required target interface appears. The target interface must be configured with at least one IP address. 

### Maximum transmission unit

The MTU is the size of the largest protocol data unit that can be sent on the "wire". A custom MTU size for the interface can be set using the `mtu` value in the interface definition. If the field is left empty, VPP uses a default MTU size.

The VPP agent provides an option to automatically set the MTU size for an interface using a [config file] [interface-config-file]. If the global MTU size is set to a non-zero value, but interface definition contains 
a different local value, the local value take precedence over the global value.

!!! note
    MTU is not available for VxLan or IPSec tunnel interfaces.

### Interface types

!!! Note
    In the brief descriptions below, reference will be made to interface `links` or `structures` along with the contents defining various properties and functions specific to the respective interface type. Those details are contained in a file called [`interface.proto`][links].

Interface type-specific fields supported by the VPP agent are defined in special structures called [links][links]. Observations on specific interface types follow:

- **Bond** was created to support the Link Aggregation Control Protocol (LACP). Network bonding is the process of combining or joining two or more network interfaces together into to form a single interface. It can support various modes of operation and load balancing across the different member interfaces of the bond interface. 

- **Sub-interface** is derived from other interfaces by adding VLAN ID to it. The sub-interface link of type `SubInterface` requires a name of the parent interface, and the ID of the VLAN.

- **Software** loopback does not have any special fields.

- **DPDK** and physical interfaces cannot be created or removed directly by the VPP agent. The PCI interface must be connected to the VPP and should be configured for use by the Linux kernel. The VPP agent can configure the interface IP and MAC address information or set it as enabled. No special configuration fields are defined.

- **Memory interface** or shared memory packet interface (memif for short) types provides high-performance send and receive functions between VPP instances. Additional fields are defined in `MemifLink`. The most important of these is the socket filename. The default uses a default socket filename with zero index which can be used to create memif interfaces. Note that the VPP agent needs to be resynced to register the new memif interface. 

- **Tap interface** exists in two versions, TAPv1 and TAPv2 but only the latter can be configured. TAPv2 is based on [virtio][virtio] which means it knows that it runs in a virtual environment and cooperate with the hypervisor. The `TapLink` provides several setup options available for TAPv2. 

!!! danger "important"
    The first version of the Tap interface is no longer supported by VPP. It cannot be managed by the VPP agent. 


- **Af-packet interface** is the VPP "host" interface. Its primary function in life is to connect to the host via the OS VEth interface. Because of this, the Af-packet interface cannot exist without its host interface. If the desired VEth is missing, the configuration is postponed. If the host interface was removed, the VPP agent "un-configures" and caches related Af-packet information 
The `AfpacketLink` contains only one field with the name of the host interface as defined in the [Linux interface plugin][linux-interface-plugin-guide].

- **VxLan tunnel** endpoint (VTEP) defines a `VxlanLink` with source and destination addresses, a VXLan network identifier (VNI) and a name of the optional multicast interface.

- **IPSec virtual tunnel** interface (VTI) provides a routable interface type for terminating IPSec tunnels. `IPSecLink` describes IPsec tunnel setup options including local and remote IP addresses, local and remote security parameter index (SPI) values and crypto algorithms for encryption and authentication. The choice of crypto algorithms are determined by those supported by VPP.

- **VmxNet3** interface is a virtual network adapter for delivering high performance in virtual environments. 
VmxNet3 requires VMware tools to be used. Setup options are described in `VmxNet3Link`.

**Configuration example**

**1. Using key-value database** (etcd in our example) put the interface config data using the correct [interface key][key-reference]).

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

Use `etcdctl` to put the compacted software_loopback interface key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/interfaces/loop1 '{"name":"loop1","type":"SOFTWARE_LOOPBACK","enabled":true,"phys_address":"7C:4E:E7:8A:63:68","ip_addresses":["192.168.25.3/24","172.125.45.1/24"],"mtu":1478}'
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

Use `etcdctl` to put the compacted memif interface key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/interfaces/memif1 '{"name":"memif1","type":"MEMIF","enabled":true,"phys_address":"4E:93:2A:38:A7:77","ip_addresses":["172.125.40.1/24"],"mtu":1478,"memif":{"master":true,"id":1,"socket_filename":"/tmp/memif1.sock","secret":"secret"}}'
```

**2. Using REST:**

Currently REST only supports retrieval of the existing configuration. The following command can be used to read all interfaces via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces
```

The interface type-specific calls:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/loopback
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/ethernet
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/memif
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/tap
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/vxlan
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/afpacket
```

**3. Using GRPC:**

Prepare the interface data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/interfaces"
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

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Interfaces: []*interfaces.Interface{
				memoryInterface,
			},
			
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Interface status

Interface status is a **collection of interface data** that can be stored in an external database or sent as a notification.

The VPP agent waits on several types of VPP events such as object creation, deletion or counter updates. The event is processed and all data extracted. Using the collected information, the VPP agent builds a notification which is sent to all registered publishers (databases). Any errors that occurred are also stored. The statistics readout is performed every second.

!!! danger "Important"
    The interface status is disabled by default, since no default publishers are defined. Use the interface plugin config file to enable interface status. (see the [example config file][interface-config-file])   

The [interface state model][interface-state-model] is represented by two objects: `InterfaceState` containing status data, and `InterfaceNotification` which is a wrapper around the state with additional information required to send a status notification.

Keys used for interface status and errors:

```
// Interface status
/vnf-agent/<label>/vpp/status/v2/interface/<name>

// Errors
/vnf-agent/<label>/vpp/status/v2/interface/errors/<name>
```

The interface status is meant to be read-only and published only by the VPP agent.

Data provided by interface status:

  * identification (logical and internal name, software interface index, interface type)
  * administrative and operational status
  * physical address, MTU value
  * used duplex 
  * statistics (received, transmitted or dropped packets, bytes, error count, etc.)

**Interface status usage**

To read status data, use any tool with access to a given database (**etcdctl** for etcd, **redis-cli** for Redis, **boltbrowser** for BoltDB, etc). 

Let's read interface status data with `etcdctl`:
```
etcdctl get --prefix /vnf-agent/vpp1/vpp/status/v2/interface/
```

The command above returns status for all interfaces that published to and stored in etcd. To read the status for just a single interface, append its name to the end of etcdctl command as so:
```
etcdctl get --prefix /vnf-agent/vpp1/vpp/status/v2/interface/loop1
etcdctl get --prefix /vnf-agent/vpp1/vpp/status/v2/interface/memif1
```
!!! note
    Do not forget to enable status in the config file. Otherwise, the status will not be published to the database.

## L2 plugin

The VPP L2 plugin is used to configure VPP link-layer configuration items, notably **bridge domains**, **forwarding tables** (FIBs) and **VPP cross connects**.  

### Bridge domains

The L2 bridge domain is a set of interfaces that belong to the same flooding or broadcast domain. Every bridge domain contains several attributes (MAC learning, unicast forwarding, flooding, ARP termination table) which can be enabled or disabled. The bridge domain is identified by a unique ID, which is managed by the plugin and cannot be externally configured.   

The L2 plugin defines an individual model for every configuration item and the [bridge domain][bd-model] is no exception. The definition consists of common bridge domain fields (forwarding, learning...), a list of assigned interfaces and an ARP termination table. Bridge domain interfaces are only referenced - the configuration of the interface itself is handled by the interface plugin. Every referenced interface also contains bridge domain-specific fields (BVI and split horizon group). Note that **only one BVI interface is allowed per bridge domain**.

The bridge domain does not have any mandatory fields except a logical name. The bridge domain logical name can be used as a reference for other configuration types configured as part of  the specific bridge domain. 

The bridge domain index can be obtained via the usual retrievel methods (e.g. etcdctl get, REST, etc.), but the value is only informative since it cannot be changed, set or referenced.

**Configuration example**

**1. Using the key-value database** (etcd in our example) put the interface config data using the correct [bridge domain key][key-reference]).

Example data:

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

Use `etcdctl` to put compacted key-value entry:

```bash
etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1 '{"name":"bd1","learn":true,"flood":true,"forward":true,"unknown_unicast_flood":true,"arp_termination":true,"interfaces":[{"name":"if1","split_horizon_group":0,"bridged_virtual_interface":true},{"name":"if2","split_horizon_group":0},{"name":"if2","split_horizon_group":0}],"arp_termination_table":[{"ip_address":"192.168.10.10","phys_address":"a7:65:f1:b5:dc:f6"},{"ip_address":"10.10.0.1","phys_address":"59:6C:45:59:8E:BC"}]}'
```

To remove the configuration:

```bash
etcdctl del /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1
```

**2. Using REST:**

Currently REST only supports the retrieval of the existing configuration. The following command can be used to read all bridge domains via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/bd
```

**3. Using GRPC:**

Prepare the bridge domain data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l2"
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

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			BridgeDomains: []*l2.BridgeDomain{
				bd,
			},
			
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Forwarding tables  

An L2 forwarding information base (FIB) can be used in network bridging or routing. Inbound packets are forwarded to the output interface defined in the FIB table. FIB entries are created by VPP just as are interfaces within the bridge domain enabled with MAC learning. The VPP agent enables configuration of static FIB entries with a combination of interface and bridge domain values.

A FIB entry is defined in the following [model][fib-model]. To configure a FIB entry, a MAC address, interface and bridge domain must be provided. The condition that the interface is part of the bridge domain must be fulfilled. The interface and the bridge domain are referenced with logical names. Note that the FIB entry will not appear in the VPP until all conditions are met.  

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct FIB key to the database ([key reference][key-reference]).

Example data:
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

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/fib/bd1/mac/62:89:C6:A3:6D:5C '{"phys_address":"62:89:C6:A3:6D:5C","bridge_domain":"bd1","outgoing_interface":"tap1","action":"FORWARD","static_config":true,"bridged_virtual_interface":false}'
```

**2. Using REST:**

Currently REST only supports the retrieval of the existing configuration. The following command can be used to read all FIBs via cURL:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/fib
```

**3. Using GRPC:**

Prepare the FIB data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l2"
)

fib := &l2.FIBEntry{
		PhysAddress: "a7:35:45:55:65:75",
		BridgeDomain: "bd1",
		Action: l2.FIBEntry_FORWARD,
		OutgoingInterface: "if1",
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Fibs: []*l2.FIBEntry {
				bd,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Cross connects

The VPP agent supports the L2 cross connect feature which sets an interface pair to cross connect (xconnect) mode. All packets received on the first interface are transmitted out the second interface. This mode is not bidirectional by default. If required both interfaces in each direction must be set.

The cross-connect mode uses a very simple [xconnect model][xc-model] that references the transmit and receive interfaces. Both interfaces are mandatory fields and must exist in order to set the mode successfully. If one or both are missing, the configuration is postponed. 

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for cross connect to the database ([key reference][key-reference]).

Example data:
```json
{
    "receive_interface": "if1",
    "transmit_interface": "if2"
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/xconnect/if1 '{"receive_interface":"if1","transmit_interface":"if2"}'
```
**2. Using REST:**

Currently REST only supports the retrieval of the existing configuration. The following command can be used to read all cross-connects via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/xc
```

**3. Using GRPC:**

Prepare the cross-connect data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l2"
)

xc := &l2.XConnectPair{
		ReceiveInterface: "if1",
		TransmitInterface: "if2",
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			XconnectPairs: []*l2.XConnectPair {
				xc,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

## L3 plugin

The VPP L3 plugin is capable of configuring **ARP** entries (including **proxy ARP**), **VPP routes** and the **IP neighbor** feature. 

### ARP entries  

The L3 plugin works with the VPP implementation of the address resolution protocol. **ARP is a communication protocol for discovering the MAC address associated with a given IP address**. In terms of the plugin, the ARP entry is uniquely identified by the interface and IP address.

The northbound plugin API defines ARP in the following [model][arp-model]. Every ARP entry includes a MAC address, IP address and an interface. The two latter values are a part of the key.

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for ARP to the database ([key reference][key-reference]).

Example data:
```json
{  
    "interface":"if1",
    "ip_address":"192.168.10.21",
    "phys_address":"59:6C:45:59:8E:BD",
    "static":true
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/arp/tap1/192.168.10.21 '{"interface":"tap1","ip_address":"192.168.10.21","phys_address":"59:6C:45:59:8E:BD","static":true}'
```

To remove configuration:

```bash
etcdctl del /vnf-agent/vpp1/config/vpp/v2/arp/tap1/192.168.10.21
```

**2. Using REST:**

REST currently only supports the retrieval of the existing configuration. The following command can be used to read all ARP entries via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/arps
```

**3. Using GRPC:**

Prepare the ARP data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l3"
)

arp := &l3.ARPEntry{
		Interface: "if1",
		IpAddress: "10.20.30.40/24",
		PhysAddress: "55:65:75:a7:35:45",
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Arps: []*l3.ARPEntry {    
			    arp,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial])):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Proxy ARP

Proxy ARP is a technique where a proxy device on a subnet answers ARP queries for an IP address of a host on a different network. Proxy ARP IP address ranges are defined via the respective binary API call. In addition, the desired interfaces must be enabled for the proxy ARP feature.    

The Proxy ARP model is defined in the [common L3 model][l3-model] file. It lays out a list of IP address ranges and another list of interfaces. Naturally, the interface must exist in order for it to be enabled for proxy ARP. 

**Configuration example**  

**1. Using the key-value database** put the proto-modelled data with the correct key for proxy ARP to the database ([key reference][key-reference]).

Example data:
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

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/proxyarp-global/settings '{"interfaces":[{"name":"tap1"},{"name":"tap2"}],"ranges":[{"first_ip_addr":"10.0.0.1","last_ip_addr":"10.0.0.3"}]}'
```

**2. Using REST:**

Currently REST only supports the retrieval of the existing configuration. Proxy ARP uses two cURL commands to obtain interfaces and ranges: 

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/proxyarp/interfaces
curl -X GET http://localhost:9191/dump/vpp/v2/proxyarp/ranges

```

**3. Using GRPC:**

Prepare the Proxy-ARP data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l3"
)

xc := &l2.XConnectPair{
		ReceiveInterface: "if1",
		TransmitInterface: "if2",
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
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

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Routes 

The VPP routing table contains destination network prefixes, a corresponding next_hop IP address and the outbound interface. Routes are grouped in virtual routing and forwarding (VRF) tables, with a default table of index 0.

The VPP [route model][route-model] explains the basic routing table fields (i.e. destination network, next_hop, etc.) as well as additional fields including weight and preference. 

The model also defines several route types:

- Intra-VRF route: forwarding is done by a lookup in the local VRF table.

- Inter-VRF route: forwarding is done by a lookup into an external VRF table. 
 
- Drop route: drops network communication destined for specified IP address.
  
**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for routes to the database 
([key reference][key-reference]).

Example data (intra-VRF route):
```json
{  
    "type": "INTRA_VRF",
    "dst_network":"10.1.1.3/32",
    "next_hop_addr":"192.168.1.13",
    "outgoing_interface":"tap1",
    "weight":6
}
```

Use `etcdctl` to put compacted intra-VRF route:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/route/vrf/0/dst/10.1.1.3/32/gw/192.168.1.13 '{"type":"INTRA_VRF","dst_network":"10.1.1.3/32","next_hop_addr":"192.168.1.13","outgoing_interface":"tap1","weight":6}'
```

Another example (inter-VRF route)
```json
{  
    "type":"INTER_VRF",
    "dst_network":"1.2.3.4/32",
    "via_vrf_id":1
}
```

Use `etcdctl` to put compacted inter-VRF route:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/route/vrf/0/dst/1.2.3.4/32/gw '{"type":"INTER_VRF","dst_network":"1.2.3.4/32","via_vrf_id":1}'
```

**2. Using REST:**

The following command can be used to read all routes via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/routes
```

**3. Using GRPC:**

Prepare the VPP route data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l3"
)

route := &l3.Route{
		VrfId:             0,
		DstNetwork:        "10.1.1.3/32",
		NextHopAddr:       "192.168.1.13",
		Weight:            60,
		OutgoingInterface: "if1",
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Routes: []*l3.Route {}
				route,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```
 
### IP scan-neighbor

The VPP IP scan-neighbor feature is used to enable or disable periodic IP neighbor scans. The [model][l3-model] supports scan intervals, timers, delays, IPv4 mode, IPv6 mode or dual-stack. 

**Configuration example**  

**1. Using the key-value database** put the proto-modelled data with the correct key for IP scan neighbor to the database ([key reference][key-reference]).

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

Use `etcdctl` to put compacted data:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/ipscanneigh-global/settings '{"mode":"BOTH","scan_interval":11,"max_proc_time":36,"max_update":5,"scan_int_delay":16,"stale_threshold":26}'
```

**2. Using REST:**

The following command can be used to read IP scan neighbor configuration via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/ipscanneigh
```

**3. Using GRPC:**

Prepare the IP scan neighbor data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l3"
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

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			IpscanNeighbor: *l3.IPScanNeighbor {  
				ipScanNeigh,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

## IPSec plugin

The IPSec plugin handles the configuration of **security policy databases** (SPD) and **security associations** (SA) in VPP. Note that the IPSec tunnel interfaces are not part of the IPSec plugin but instead are supported by the [VPP interface plugin][interface-model].

### Security policy database

Security policy database (SPD) specifies policies that determine the disposition of all the inbound or outbound IP traffic from either the host or the security gateway. The SPD is bound to an SPD interface and contains a list of policy entries in a table. Every policy entry points to a VPP security association. SPD is defined in the common [IPSec model][ipsec-model]. 

The SPD defines its own unique index within VPP. The user has an option to set their own index in the `uint32` range. The index is a mandatory field in the model because it serves as a unique identifier for the VPP agent as it is a part of the SPD key. 

!!! Note
    Pay special attention when defining an index in the model. Despite the fact the field uses a `string` format, it only accepts plain numbers. Attempts to set any non-numeric characters will result in an error.     

Every policy entry has field `sa_index`. This is the SA reference in the SPD. The field is mandatory and a missing value will generate a configuration error.

The SPD defines two bindings: SA and the interface. The interface binding is important since VPP needs to create an empty SPD first and this cannot be done without it. All the policy entries are configured after that, where the SA is expected to exist. 
 

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for security policy databases to the database ([key reference][key-reference]).

Example data:
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

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/ipsec/v2/spd/1 '{"index":"1","interfaces":[{"name":"tap1"}],"policy_entries":[{"priority":10,"is_outbound":false,"remote_addr_start":"10.0.0.1","remote_addr_stop":"10.0.0.1","local_addr_start":"10.0.0.2","local_addr_stop":"10.0.0.2","action":3,"sa_index":"20"},{"priority":10,"is_outbound":true,"remote_addr_start":"10.0.0.1","remote_addr_stop":"10.0.0.1","local_addr_start":"10.0.0.2","local_addr_stop":"10.0.0.2","action":3,"sa_index":"10"}]}'
```

**2. Using REST:**

REST currently only supports the retrieval of the existing configuration. The following command can be used to read all security policy databases via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/ipsec/spds
```

**3. Using GRPC:**

Prepare the IPSec SPD data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/ipsec"
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

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			IpsecSpds: []*ipsec.SecurityPolicyDatabase {
				spd,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Security associations

The VPP security association (SA) is a set of IPSec specifications negotiated between devices establishing a secure IPSec connection. The SA includes preferences for the authentication type, IPSec protocol (AH or ESP) and the encryption methods used when the IPSec connection is established.

SA is defined in the common [IPSec model][ipsec-model]. 

The SA uses the same indexing system as SPD. The index is a user-defined unique identifier in the `uint32` range. Like the SPD, the SA index is defined as `string` type field in the model but can be set only to numerical values. Attempts to use values other than numerical will cause errors. The SA has no dependencies on other configuration types.

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for security associations to the database ([key reference][key-reference]).

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

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/ipsec/v2/sa/1 '{"index":"1","spi":1001,"protocol":1,"crypto_alg":1,"crypto_key":"4a506a794f574265564551694d653768","integ_alg":2,"integ_key":"4339314b55523947594d6d3547666b45764e6a58"}'
```

**2. Using REST:**

REST currently only supports the retrieval of the existing configuration. The following command can be used to read all security associations via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/ipsec/sas
```

**3. Using GRPC:**

Prepare the IPSec SA data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/ipsec"
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

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			IpsecSas: []*ipsec.SecurityAssociation {
				sa,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

## Punt plugin

The punt plugin is used to configure VPP to punt (re-direct) packets to the TCP/IP host stack. The plugin supports `punt to the host` either directly, or via a Unix domain socket.

### Punt to host stack

All incoming traffic:
 
- matching one of the VPP interface addresses
- matching a defined L3 protocol, L4 protocol, and port
- and would otherwise be dropped

will be punted to the host. If an optional Unix domain socket path is defined, traffic will be punted via that socket. All the fields which serve as a traffic filter are mandatory.

The punt plugin conform with the [punt model][punt-model] which grants support for two main configuration items defined by different northbound keys. L3/L4 protocol in the key is defined as a `string` value. However that value is transformed to a numeric representation in the VPP binary API call. 

The usage of the L3 protocol `ALL` is exclusive for IP punt to host (without socket registration) in the VPP binary VPP API. If used for the IP punt with socket registration, the VPP agent calls the VPP binary API twice with the same parameters for both, IPv4 and IPv6.

!!! danger "Important"
    In order to configure a punt to host via a Unix domain socket, a specific VPP startup-config is required. The attempt to set punt without it results in errors in VPP. This is an example startup-config `punt { socket /tmp/socket/punt }`. The path must match with the one in the northbound data. 

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for punt to host to the database ([key reference][key-reference]). 

Example data:
```json
{  
    "l3_protocol":"IPv4",
    "l4_protocol":"UDP",
    "port":9000,
    "socket_path":"/tmp/socket/path"
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/tohost/l3/IPv4/l4/UDP/port/9000 {"l3_protocol":"IPv4","l4_protocol":"UDP","port":9000,"socket_path":"/tmp/socket/path"}
```

**2. Using REST:** 

REST currently only supports the retrieval of the registered punt to host entries with socket. Passive punt to host cannot be retrieved:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/punt/sockets
```

**3. Using GRPC:**

Prepare the Punt data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/punt"
)

punt := &punt.ToHost{
    L3Protocol: punt.L3Protocol_IPv4,
    L4Protocol: punt.L4Protocol_UDP,
    Port:       9000,
    SocketPath: "/tmp/socket/path",
}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
    VppConfig: &vpp.ConfigData{
        PuntTohosts: []*punt.ToHost {
            punt,
        },	
    }
}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### IP redirect

IP redirect enables traffic matching a given IP protocol (e.g. v4) to be punted to the defined TX interface and next_hop IP address. The IP protocol, TX and next_hop fields must be defined in the northbound API. Optionally, the RX interface can be defined as an input filter.  

IP redirect is defined as the `IpRedirect` object in the generated proto model. L3 protocol is defined as a `string` value (transformed to numeric in the VPP binary API call). The table is the same as before.

If L3 protocol is set to `ALL`, the respective API is called for IPv4 and IPv6 separately.

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for the IP redirect to the database ([key reference][key-reference]). 

Example value:
```json
{  
    "l3_protocol":"IPv4",
    "tx_interface":"tap1",
    "next_hop":"192.168.0.1"
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/ipredirect/l3/IPv4/tx/tap1 '{"l3_protocol":"IPv4","tx_interface":"tap1","next_hop":"192.168.0.1"}'
```

**2. Using REST:** 

Punt IP redirect currently does not support REST.

**3. Using GRPC:**

Prepare the IP redirect data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/punt"
)

punt := &punt.IPRedirect{
    L3Protocol:  punt.L3Protocol_IPv4,
    TxInterface: "if1",
    NextHop:     "192.168.0.1",
}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
    VppConfig: &vpp.ConfigData{
        PuntTohosts: []*punt.ToHost {
            punt,
        },	
    }
}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Limitations

Current limitations for a punt to host:

- UDP configuration cannot be shown (or even configured) via the VPP CLI.
- VPP does not provide an API to dump configuration. Thus the VPP agent does not have the opportunity to read existing entries and this may cause certain issues with resync.
* Although the VPP agent supports the TCP protocol as the L4 protocol to filter incoming traffic, the current VPP dataplane version does not.
* Configured punt to host entry cannot be removed since VPP does not support this option. Any attempt to do so exits with an error.

Current limitations for a punt to host via unix domain socket:

- Configuration cannot be shown (or even configured) in the VPP CLI.
- VPP agent cannot read registered entries since the VPP does not provide an API to do so.
- The VPP startup config punt section requires a defined unix domain socket path. The VPP limitation is that only one path can be defined at any one time.

Current limitations for IP redirect:

- VPP does not provide API calls to dump existing IP redirect entries. This may cause resync problems.

### Known issues

- VPP issue: if the Unix domain socket path is defined in the startup config, the path `must exist`, otherwise VPP fails to start. 

## Access Control Lists plugin

Access Control Lists (ACL) filter network traffic by controlling whether packets are forwarded (permitted) or blocked (deny) at the router’s interfaces based on the criteria specified in the access list.

The VPP agent ACL plugin uses the binary API of the VPP (dataplane) ACL plugin. The version of the VPP ACL plugin is displayed at VPP agent startup (currently at 1.3). Every ACL consists of `match` rules that classify the packets to be acted upon and  `action` rules (or actions) to be applied to those packets. The ACL is defined in the VPP agent northbound API [model][acl-model].

The VPP agent defines an access list with a unique name. VPP generates an index, but the association is under the purview of the VPP agent. Every ACL must contain match rules and action rules. 

The IP match (called IP rule) can be specified for a variety of protocols, each with their own parameters. For example, the IP rule for the IP protocol can define the source and destination network addresses the packet must match in order to execute the defined action. Other supported protocols are TCP, UDP and ICMP. 

A single ACL rule can define multiple protocol-based rules. The MAC-IP match (MACIP rule) defines IP address + mask and MAC address + mask as filtering rules. Note that the IP rules and MACIP rules cannot be combined in the same ACL.

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for ACL entries to the database ([key reference][key-reference]).

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

Use `etcdctl` to put compacted key-value entry:
```
etcdctl put /vnf-agent/vpp1/config/vpp/acls/v2/acl/acl1 '{"name":"acl1","interfaces":{"egress":["tap1","tap2"],"ingress":["tap3","tap4"]},"rules":[{"action":1,"ip_rule":{"ip":{"destination_network":"10.20.1.0/24","source_network":"192.168.1.2/32"},"tcp":{"destination_port_range":{"lower_port":1150,"upper_port":1250},"source_port_range":{"lower_port":150,"upper_port":250},"tcp_flags_mask":20,"tcp_flags_value":10}}}]}'
```

**2. Using REST:**

Use following commands to obtain IP or MACIP based Access lists:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/acl/ip
curl -X GET http://localhost:9191/dump/vpp/v2/acl/macip

```

**3. Using GRPC:**

Prepare the ACL data:
```go
import acl "github.com/ligato/vpp-agent/api/models/vpp/acl"

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

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		VppConfig: &vpp.ConfigData{
			Acls: []*acl.ACL {
				acl,
			},	
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

## ACL-based forwarding (ABF) plugin

The ACL-based forwarding plugin is an implementation of policy-based routing (PBR). With ABF, packets are forwarded based on user-defined fields expressed by ACLs rather than by the results of a longest match lookup in a routing table. 

The ABF entry is defined by a numeric index. The data consists of a list of interfaces the ABF is attached to, the forwarding paths and the name of the associated ACL. The ACL represents a dependency for the given ABF - if the ACL is not present, the ABF configuration will be cached until created. Same applies for ABF interfaces.   

**Configuration example**

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

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/abfs/v2/abf/1 '{"index":1,"acl_name":"aclip1","attached_interfaces":[{"input_interface":"tap1","priority":40},{"input_interface":"memif1","priority":60}],"forwarding_paths":[{"next_hop_ip":"10.0.0.10","interface_name":"loop1","weight":20,"preference":25}]}'
```

To remove the configuration:
```bash
etcdctl del /vnf-agent/vpp1/config/vpp/abfs/v2/abf/1
``` 

**2. Using REST:**

Use following commands to obtain configured ABF entry:
```bash
curl -X GET http://localhost:9191/dump/vpp/v2/abf
```

**3. Using GRPC:**
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	vpp_abf "github.com/ligato/vpp-agent/api/models/vpp/abf"
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

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
		Abfs: &vpp.ConfigData{
			Abfs: []*vpp.ABF{ {
				natGlobal,
			},
		}
	}
```

The config data can be combined with any other VPP or Linux configuration.

## NAT plugin

Network address translation, or NAT is a method for translating IP addresses belonging to different address domains by modifying the address information in the packet header. The VPP agent NAT plugin provides control plane functionality for the VPP dataplane NAT implementation of NAT44. The NAT plugin is dependent on the [interface plugin][interface-plugin-guide].

### NAT global config 

The global NAT configuration is a special case of data grouped under single key. This means there is no unique character as a part of the key, so there is only one global NAT configuration. Interfaces marked as NAT-enabled  should be present in the VPP but if not, the KV scheduler plugin caches the configuration for later use when the interface becomes available.


The NAT global configuration is divided into several independent parts pertaining to specific VPP NAT features:

  - **Forwarding** is a boolean field enabling or disabling forwarding.
  - **NAT interfaces** represents a list of interfaces which will be enabled for NAT. If the interface does not exist in the VPP, it is cached and potentially configured later. Every interface is defined by its logical name and whether it is an `inside` or an `outside` interface. The output feature is also defined here.
  - **Address pools** is a list of "NAT-able" IP addresses for a given VRF table. Despite the name, only one address is defined in the single "pool" entry.
  - **Virtual reassembly** provides support for datagram fragmentation handling to allow correct recalculation of higher-level checksums.

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for nat global config to the database ([key reference][key-reference]).

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

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/nat/v2/nat44-global/settings {"nat_interfaces":[{"name":"tap1"},{"name":"tap2","output_feature":true},{"name":"tap3","is_inside":true}],"address_pool":[{"address":"192.168.0.1"},{"address":"175.124.0.1"},{"address":"10.10.0.1"}],"virtual_reassembly":{"timeout":10,"max_reassemblies":20,"max_fragments":10,"drop_fragments":true}}
```

To remove the configuration:
```bash
etcdctl del /vnf-agent/vpp1/config/vpp/nat/v2/nat44-global/settings
```

**2. Using REST:**

REST currently only supports retrieval of the existing configuration. The following command can be used to read the NAT global config via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/nat/global
```

**3. Using GRPC:**

Prepare the global NAT data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/nat"
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

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
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

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### DNAT44

Destination network address translation (DNAT) translates the destination IP address of a packet in one direction and performs the inverse NAT function for packets returning in the opposite direction. In the VPP agent, the DNAT configuration is composed of a list of static and/or identity mappings labelled under a single key.

The DNAT44 consists from two main parts: static mappings and identity mappings.

- static mapping is the case where a single external interface, IP address and port number is mapped to one or more local IP address and port numbers in a static configuration. When packets arrive on the external interface, the DNAT44 can load balance packets across two or more local IP address and port number entries present in the static mapping (by load probability if needed). In other words, if more than one local IP address is defined for single static mapping, the [VPP NAT load balancer][vpp-nat-lb] is automatically allowed to kick in. 


THe DNAT44 contains a unique label serving as an identifier. However, the DNAT configuration is not limited. An arbitrary number of static and identity mappings can be listed under a single label. 

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for nat44 to the database ([key reference][key-reference]).

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

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/nat/v2/dnat44/dnat1 {"label":"dnat1","st_mappings":[{"external_interface":"tap1","external_ip":"192.168.0.1","external_port":8989,"local_ips":[{"local_ip":"172.124.0.2","local_port":6500,"probability":40},{"local_ip":"172.125.10.5","local_port":2300,"probability":40}],"protocol":"UDP","twice_nat":"ENABLED"}],"id_mappings":[{"ip_address":"10.10.0.1","port":2525}]}
```

To remove the configuration:

```bash
etcdctl del /vnf-agent/vpp1/config/vpp/nat/v2/dnat44/dnat1
```

**2. Using REST:**

REST currently only supports the retrieval of the existing configuration. The following command can be used to read the NAT global config via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/nat/dnat
```

**3. Using GRPC:**

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

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/vpp"
)

config := &configurator.Config{
    VppConfig: &vpp.ConfigData{
        Dnat44S: []*nat.DNat44 {
            natGlobal,
        },
        
    }
}
```

The config data can be combined with any other VPP or Linux configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

## SR plugin

The `srplugin` is designed to configure Segment Routing for IPv6 (SRv6) in VPP.
Configuration managed by this plugin is modelled in the [srv6 proto file][src6-model].

All configuration data must be stored in etcd using the srv6 key:
 
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/
```

### Configuring Local SIDs
The local SID can be configured using this key:
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/localsid/<SID>
```
where ```<SID>``` (Segment ID) is a unique ID of a local sid and it must be a valid IPv6 address. 

### Configuring Policy
The segment routing policy can be configured using this key:
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/policy/<bsid>
```
where ```<bsid>``` is the unique binding SID of the policy. As with any other SRv6 SIDs, it must be a valid IPv6 address. 

The policy can be defined inside multiple segment lists. The VPP implementation does not allow a policy without at least one segment list. Therefore inserting(updating with) policy that has not defined at least one segment list will fail. Note that the value can be written to etcd, but its application to VPP will result in a validation error).

### Configuring Steering
The steering (the VPP's policy for steering traffic into SR policy) can be configured using this key:
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/steering/<name>
```
where ```<name>``` is a unique name of steering.

## Telemetry

The `telemetry` plugin is used for exporting telemetry statistics from the VPP to [Prometheus][prometheus]. Statistics are published via the registry path `/vpp` on port `9191` and updated every 30 seconds.

**Exported data**

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

- VPP memory (`show memory`)

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
    
**Configuration file**

The telemetry plugin configuration file allows one to change the polling interval, or turn polling off. The `polling-interval` is the time in nanoseconds between reads from the VPP. The parameter `disabled` can be set to `true` in order to disable the telemetry plugin.

[acl-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/acl/acl.proto
[arp-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/arp.proto
[bd-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto
[fib-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/fib.proto
[grpc-tutorial]: ../tutorials/08_grpc-tutorial.md
[interface-config-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/ifplugin/vpp-ifplugin.conf
[interface-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/interfaces/interface.proto
[interface-state-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/state.proto
[ipsec-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/ipsec/ipsec.proto
[key-reference]: ../user-guide/reference.md
[l3-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/l3.proto
[linux-interface-plugin-guide]: linux-plugins.md#interface-plugin
[punt-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/punt/punt.proto
[route-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/route.proto
[src6-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/srv6/srv6.proto
[xc-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/xconnect.proto
[links]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/interface.proto
[vpp-nat-lb]: https://jira.fd.io/browse/VPP-954
[prometheus]: https://prometheus.io/
[virtio]: https://wiki.libvirt.org/page/Virtio  

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
