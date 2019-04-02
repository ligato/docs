# Interface plugin

**Written for: v2.0-vpp18.10**

The VPP interface plugin is a base plugin able to setup various types of **VPP interfaces** and also manages **the interface status** (state data, optionally published back to the database) and **interface and DHCP lease notifications**. The VPP provides multiple interface types, following are supported in the VPP-agent:

  - Sub-interface
  - Software loopback
  - DPDK (physical) interface
  - Memory interface (memif)
  - Tap (version 1 and 2)
  - Af-packet interface
  - VxLAN tunnel
  - IPSec tunnel
  - VmxNet3 interface

Other types are specified as undefined for the vpp-agent purpose.
  
All the interfaces except DPDK (physical) can be created or removed directly in the VPP if all necessary conditions are met (for example an Af-packet interface requires a VEth in the host in order to attach to it. The DPDK interfaces (PCI, physical) can be only configured, cannot be added or removed.
  
The VPP uses multiple commands and/or binary APIs to create and configure shared boundaries. The vpp-agent is simplifying this processes, providing the single model with specific extensions depending on required interface type. The northbound configuration is translated to a sequence of binary API calls using GOVPP library. All interface types except the DPDK (physical) interface can be directly created or removed in the VPP. Physical interfaces can be only configured. The Af-packet interface is dependent on host-interface of the VEth type, and cannot exist without it. 

The interface plugin defines following [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/interface.proto). The model is divided to two parts - the first, which contains fields common for all interface types, and the second, exclusive for given interface type. Note that there are even exceptions in the common part, where not all fields can be defined on every interface type. For example, a physical address cannot be set to Af-packet or to DPDK interface despite the field exists in the definition. If this is done, the value is silently ignored.

Mandatory fields are **interface type** and **logical name**. The name is limited to 63 characters. The interface may or may not define unique fields for the given type.

In the generated proto model, the interface is referred to the `Interface` object.

All interfaces use the same key for every type. It means the name has to be incomparable among all interfaces. The key format:

```
vpp/config/v2/config/vpp/v2/interface/<name>
```
 
The interface logical name is for the vpp-agent use only. It serves as a reference for other models (bridge domains for instance). The VPP works with indexes, which are generated when the new interface instance is created. The index is a unique integer for identification and future references. In order to parse name and index, the vpp-agent stores a tag to the VPP via binary API - it's a name-to-index entry which helps the vpp-agent to classify interface name and assign the correct index to it. 

### Unnumbered interfaces

Every interface can have assigned one or more IPv4 or IPv6 addresses or their combination. Besides this, the VPP interface can be set as unnumbered; it takes the IP address of another interface and performs as it has the same IP address as the target instance. 

To set an interface as unnumbered, omit `ip_addresses` and set `interface_with_ip`. The target interface has to exist (if not, the unnumbered interface will be postponed and configured later when the required interface appears) and must contain at least one IP address. 

### Maximum transmission unit

The MTU is the size of the largest protocol data unit that can be communicated in a single network layer transaction. The VPP interfaces allow setting custom MTU size. For this purpose, there is a field called `mtu` available in the interface model. If the field is left empty, the VPP uses a default value of the MTU.

The vpp-agent offers an option to automatically set the MTU of specific size to every created interface via *vpp-ifplugin* config file (example [here](https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/ifplugin/vpp-ifplugin.conf)). If the global MTU value is set to non-zero value, but interface definition contains its own different value, the local value is preferred before the global one.

**Note:** MTU is not available for VxLan tunnels and IPSec tunnel interfaces.

### Interface types

Type-specific interface fields in vpp-agent are defined in special structures called links. Type of the interface's link must match the type of the interface itself. Description of main differences of interface types:

**Sub-interface**

The Sub-interface is an interface type derived from other interfaces adding VLAN ID to it. The sub-interface link of type `SubInterface` requires a name of the parent (super) interface, and ID of the VLAN.

**Software loopback**

The loopback interface type does not have any special fields.

**DPDK interface**

The physical interface cannot be created or removed directly by the VPP agent. The PCI interface has to be connected to the VPP - they should be configured for use by the Linux kernel and shut down. Vpp-agent can set the interface IP, MAC or set it as enabled. No special configuration fields are defined.

**Memory interface**

Shared memory packet interface (memif) provides high-performance packet transmit and reception between VPPs. The memory interface defines additional fields in `MemifLink`. The most important is a socket filename - the VPP by default contains default socket filename with zero index which can be used to create memif interfaces (the vpp-agent needs to be resynced to register it). 

**Tap interface**

Note: The first version of the Tap interface is no longer supported by the VPP, thus cannot be managed by the vpp-agent. 

The Tap interface exists in two versions, only the latter can be configured. TAPv2 is based on virtio (it knows that runs in a virtual environment, and cooperates with the hypervisor). The `TapLink` provides several setup options available for the TAPv2. 

Field `version` sets the desired version (should be set only to 2), `host_if_name` allows to set a custom name for the TAP interface in the host OS (optional, if not set it will be autogenerated). The option `to_microservice` puts the host TAP to the desired namespace right after creation. Please refer the [interface model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/interface.proto) to know which fields are defined for specific TAP interface version.

**Af-packet interface**

The Af-packet is the VPP "host" interface, its main purpose is to connect it to the host via the OS VEth interface. Because of this, the af-packet cannot exist without its host interface. If the desired VEth is missing, the configuration is postponed. If the host interface was removed, vpp-agent un-configures and caches related Af-packets. The `AfpacketLink` contains only one field with the name of the host interface (as defined in the [Linux interface plugin](linux-plugins.md#linux-interface-plugin)).

**VxLan tunnel**

VxLan tunnel endpoint (VTEP) defines `VxlanLink` with source and destination address, a VXLan network identifier (VNI) and a name of the optional multicast interface.

**IPSec tunnel**

IPSec virtual tunnel interface (VTI) provides a routable interface type for terminating IPSec tunnels and an easy way to define protection between sites to form an overlay network. The IPSec tunnel defines `IPSecLink` with various options like local and remote IP address, local and remote security parameter index (SPI) or cryptographic algorithms for encryption and authentication. Available cryptographic algorithms are determined from types supported by the VPP.

**VmxNet3 interface**

VmxNet3 is a virtual network adapter designed to deliver high performance in virtual environments. VmxNet3 requires VMware tools to be used. Type-specific fields are defined in `VmxNet3Link` structure.

**Configuration**

How to configure an interface:

**1. Using key-value database** put the proto-modelled data with the correct key for interfaces. 

Key:
```
/vnf-agent/vpp1/config/vpp/v2/interfaces/<name>
```

Example value of common interface configuration, since loopback type does not use any type specific fields.
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

Use `etcdctl` to put compacted loop interface key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/interfaces//loop1 '{"name":"loop1","type":"SOFTWARE_LOOPBACK","enabled":true,"phys_address":"7C:4E:E7:8A:63:68","ip_addresses":["192.168.25.3/24","172.125.45.1/24"],"mtu":1478}'
```

Another example with memif type interface. Note memif-specific fields in `memif` section:
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

Use `etcdctl` to put compacted memif interface key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/interfaces/memif1 '{"name":"memif1","type":"MEMIF","enabled":true,"phys_address":"4E:93:2A:38:A7:77","ip_addresses":["172.125.40.1/24"],"mtu":1478,"memif":{"master":true,"id":1,"socket_filename":"/tmp/memif1.sock","secret":"secret"}}'
```

**2. Using REST:**

The REST currently supports only retrieval of the existing configuration. The following command can be used to read all interfaces via cURL:

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

* Prepare the interface data:
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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

**The VPP CLI interface commands:** 

The VPP cli has several CLI commands to verify configuration:
 - to show all configured interfaces: `show interface`
 - to show interface IP addresses: `show interface address`
 - show connected PCI interfaces: `show PCI`
 - various information about the state: `show hardware`
 - show VxLan tunnel details: `show vxlan tunnel`
 - show IPSec tunnel details: `show ipsec`

### <a name="ifstate">Interface status</a>

The interface status is a collection of interface data and identification (tagged name, internal name, index) which can be stored to an external database or send a notification.

The vpp-agent waits on several types of VPP events, like creation, deletion or counter update. The event is processed, all the data extracted and using interface index provided in the event data, vpp-agent reads the rest of the interface data from the VPP. Using all the collected information, vpp-agent builds a notification which is sent to all provided publishers (databases). All errors occurred are also stored.

The statistics readout is performed every second.

**Important note:** the interface status is disabled by default, since no default publishers are defined. Use vpp-ifplugin config file to define it (see the [example conf file](https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/ifplugin/vpp-ifplugin.conf))   

The interface state [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/state.proto) is represented by two objects: `InterfaceState` with all the status data, and `InterfaceNotification` which is a wrapper around the state with additional information required to send status as a notification.

Keys used for interface status and errors:

```
// Interface status
vpp/config/v2/status/vpp/v2/interface/<name>

// Errors
vpp/config/v2/status/vpp/v2/interface/errors/<name>
```

The interface status is meant to be read-only, published only by the VPP agent.

Data provided by interface status:
    * identification (logical and internal name, software interface index, interface type)
    * administrative and operational status
    * physical address, MTU value
    * used duplex 
    * statistics (received, transmitted or dropped packets, bytes, error count, etc)

### Interface status usage

To read status data, use any tool with access to a given database (etcdctl for the ETCD, redis-cli for Redis, boltbrowser for BoltDB, etc). 

Let's read interface status data with `etcdctl`:
```
etcdctl get --prefix /vnf-agent/vpp1/vpp/status/v2/interface/
```

The command above returns status for all interfaces, which have it currently stored inside the ETCD. To read the status just for single interface, add its name to the end:
```
etcdctl get --prefix /vnf-agent/vpp1/vpp/status/v2/interface/loop1
etcdctl get --prefix /vnf-agent/vpp1/vpp/status/v2/interface/memif1
```

Do not forget to enable status in the config file, otherwise, the status will not be published to the database.

# L2 plugin

**Written for: v2.0-vpp18.10**

The VPP L2 plugin is a base plugin which can be used to configure link-layer related configuration items to the VPP, notably **bridge domains**, **forwarding tables** (FIBs) and **VPP cross connects**. The L2 plugin is strongly dependent on [interface plugin](default-vpp-plugins.md#interface-plugin) since every configuration item has some kind of dependency on it. 
  
### Bridge domains

The L2 bridge domain is a set of interfaces which share which share the same flooding or broadcasting characteristics. Every bridge domain contains several attributes (MAC learning, unicast forwarding, flooding, ARP termination table) which can be enabled or disabled. The bridge domain is identified by unique ID, which is managed by the plugin and cannot be defined from outside.   

The L2 plugin defines individual proto definition for every configuration item, so the bridge domain is also defined by its own [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto). The definition consists of common bridge domain fields (forwarding, learning...), list of assigned interfaces and ARP termination table. 
Bridge domain interfaces are only referenced - the configuration of the interface itself and its configuration (IP address, ...) lays on the interface plugin. Every referenced interface also contains bridge domain-specific fields (BVI and split horizon group). Note that only one BVI interface is allowed per bridge domain.

The bridge domain does not have any mandatory fields except the logical name. In that case, an empty bridge domain with no interface or ARP entries is created.

In the generated proto model, the bridge domain is referred to the `BridgeDomain` object. 

The bridge domain logical name is for the plugin-use and also as a reference in other configuration types dependent on the specific bridge domain. The bridge domain index can be obtained via retrieve, but the value is only informative since it cannot be changed, set or referenced.

**Configuration**

How to configure a bridge domain:

**1. Using the key-value database** put the proto-modelled data with the correct key for bridge domains to the database. 

Key:
```
/vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/<name>
```

Example value:

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

The REST currently supports only retrieving of the existing configuration. The following command can be used to read all bridge domains via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/bd
```

**3. Using GRPC:**

* Prepare the bridge domain data:
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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

**The VPP CLI bridge domain commands:** 

The VPP cli has following CLI commands to verify configuration:
 - show list of all configured bridge domains: `show bridge-domain`
 - show details (interfaces, ARP entries, ...) for any bridge domain: `show bridge-domain <index> details`

### <a name="fts">Forwarding tables</a>  

An L2 forwarding information base (FIB) (also known as a forwarding table) can be used in network bridging or routing to find the output interface to which the incoming packet should be forwarded. FIB entries are created also by the VPP itself (like for interfaces within the bridge domain with enabled MAC learning). The Agent allows to configuring static FIB entries for a combination of interface and bridge domain.

A FIB entry is defined in the following [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/fib.proto). To successfully configure FIB entry, a MAC address, interface and bridge domain must be provided. Also, the condition that the interface is a part of the bridge domain has to be fulfilled. The interface and the bridge domain are referenced with logical names.  
Note that the FIB entry will not appear in the VPP until all conditions are met.
The FIB entry is referred to the `FIBEntry` object in the generated proto. 

**Configuration** 

How to configure a FIB:

Key (composed from the bridge domain name and a MAC address as unique identifier):
```
/vnf-agent/vpp1/config/vpp/l2/v2/fib/<bridge_domain>/mac/<phys_address>
```

Example value:
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

The REST currently supports only retrieving of the existing configuration. The following command can be used to read all FIBs via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/fib
```

**3. Using GRPC:**

* Prepare the FIB data:
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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

**The VPP CLI FIB commands:**

The VPP cli has following CLI command to verify FIB configuration:
 - show list of all configured FIB entries: `show l2fib`

### Cross connects

The Agent supports L2 cross connect feature, which sets interface pair to cross connect mode. All packets received on the first interface are transmitted to the second interface. This mode is not bidirectional by default, both interfaces have to be set.

The cross-connect mode uses a very simple [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/xconnect.proto) which defines references to transmit and receive interface. Both interfaces are mandatory fields and must exist in order to set the mode successfully. If one or both of them are missing, the configuration is postponed. 
The cross-connect is referred as `XConnectPair` in generated code. 

**Configuration**

How to configure cross-connect:

Key (note that the interface is the receive interface):
```
/vnf-agent/vpp1/config/vpp/l2/v2/xconnect/<receive-interface>
```

Example value:
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

The REST currently supports only retrieving of the existing configuration. The following command can be used to read all cross-connect entries via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/xc
```

**3. Using GRPC:**

* Prepare the cross-connect data:
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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

**The VPP CLI cross-connect commands:**

The cross-connect mode can be shown on the VPP with command `show mode`

# L3 plugin

**Written for: v2.0-vpp18.10**

The VPP L3 plugin is capable of configuring **ARP** entries (including **proxy ARP**), **VPP routes** and **IP neighbor** feature. The L3 plugin is dependent on [interface plugin](default-vpp-plugins.md#interface-plugin) in many aspects since several configuration items require the interface to be already present.
  
### ARP entries  

The L3 plugin defines descriptor managing the VPP implementation of the address resolution protocol. ARP is a communication protocol for discovering the MAC address associated with a given IP address. In terms of the plugin, the ARP entry is uniquely identified by the interface and IP address.

The northbound plugin API defines ARP in the following [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/arp.proto). Every ARP entry is defined with MAC address, IP address,  and an Interface. The two latter are a part of the key.

In the generated proto model, the ARP is referred to the `ARPEntry` object.

**Configuration**

How to configure ARP table entry:

**1. Using a key-value database** put the proto-modeled data with the correct key for the ARP to the database. 

Key:
```
/vnf-agent/vpp1/config/vpp/v2/arp/<interface>/<ip_address>
```

Example value:
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

The REST currently supports only retrieving of the existing configuration. The following command can be used to read all ARP entries via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/arps
```

**3. Using GRPC:**

* Prepare the ARP data:
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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

**The VPP CLI ARP commands:**

The VPP cli has following CLI commands to verify configuration:
 - show list of all configured ARPs: `show ip arp`

### Proxy ARP

Support for the VPP proxy ARP - a technique by which a proxy device an a given network answers the ARP queries for an IP address from a different network. Proxy ARP IP address ranges are defined via the respective binary API call. Besides this, desired interfaces have to be enabled for the proxy ARP feature.    

The Proxy ARP [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/l3.proto) is located in the common L3 model proto file. The model defines a list of IP address ranges and another list of interfaces. Naturally, the interface has to exist in order to be set as enabled for ARP proxy. 

The proxy ARP type is referred to `ProxyARP`.

**Configuration**  

How to configure the proxy ARP:

**1. Using a key-value database** put proto-modeled data under correct proxy ARP key to the KVDB. Note that only one key is defined for proxy ARP (there is no unique field as name, so all IP ranges and interfaces are grouped):

Key:
```
/vnf-agent/vpp1/config/vpp/v2/proxyarp-global/settings
```

Example value:
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

The REST currently supports only retrieving of the existing configuration. The Proxy ARP uses two cURL commands to obtain interfaces and ranges: 

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/proxyarp/interfaces
curl -X GET http://localhost:9191/dump/vpp/v2/proxyarp/ranges

```

**3. Using GRPC:**

* Prepare the Proxy-ARP data:
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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

**The VPP CLI Proxy ARP commands:**

To verify ARP configuration, use the same call as for the ordinary ARP:
 - show list of all configured proxy ARPs: `show ip arp`

### Routes 

The VPP routing table lists routes to particular network destinations. Routes are grouped in virtual routing and forwarding (VRF) tables, with default table of index 0. A route can be assigned to different VRF table directly in the modeled data.

The VPP route [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/route.proto) defines all the base route fields (destination IP, next hop, interface) and other specifications like weight or preference. The model also distinguishes amidst several route types:

**1. Intra-VRF route:** defines a route where the forwarding is done only in the specific VRF or according to the outgoing interface.
**2. Inter-VRF route:** forwarding is being done by a lookup into different VRF routes. This kind of route does not expect outgoing interface being defined. 
**3. Drop route:** such a route drops network communication designed for IP address specified.

**Configuration**

How to configure a VPP route:

**1. Using the key-value database** put the proto-modeled data with the correct key for the route to the database. 

Key (destination address is in format <ip>/<mask>, gateway is only IP address and does not need to be specified if not used in data. Also make sure the VRF matches the one in the model data - otherwise the value from key is being used):
```
/vnf-agent/vpp1/config/vpp/v2/route/vrf/<vrf_id>/dst/<dst_network>/gw/<next_hop_addr>
```

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

The REST currently supports only retrieving of the existing configuration. The following command can be used to read all routes via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/routes
```

**3. Using GRPC:**

* Prepare the VPP route data:
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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

**The VPP CLI route commands:**

The route configuration can be verified with:
 - show all rotes: `show ip fib`
 - show routes from the given VRF table: `show ip fib table <table-ID>` 
 
### IP neighbor

The VPP IP scan-neighbor feature is used to enable or disable periodic IP neighbor scan.

The [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/l3.proto) allows to set various scan intervals, timers or delays and can be set to IPv4 mode, IPv6 mode or both of them.

**Configuration**  

How to set IP scan neighbor feature:

**1. Using a key-value database** put the proto-modeled data with the correct key to the database. The key is global (as for proxy ARP) - it means only one can be present in the KVDB at a time.

Key:
```
/vnf-agent/vpp1/config/vpp/v2/ipscanneigh-global/settings
``` 

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

IP scan neighbor configuration type does not support REST.

**3. Using GRPC:**

* Prepare the IP scan neighbor data:
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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

**The VPP CLI IP scan neighbor commands:**

The IP scan neighbor configuration can be verified with:
 - show IP scan neighbor: `show ip scan-neighbor`