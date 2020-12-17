# Linux Plugins

This section describes the VPP agent's Linux plugins. The information provided for each plugin includes:

- Short description.
- Pointers to the `.proto` files containing configuration/NB protobuf API definitions, the `models.go` file defining the model, and conf file if available.  
- Configuration programming examples, if available, using an etcd data store, REST and gPRC. 

---

**Agentctl** 

You can manage Linux configurations using [agentctl][agentctl].
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

## Linux Interface Plugin

The Linux interface plugin manages Linux-based host OS interfaces. It supports VETH and TAP interface types.

**References:**

- [Linux interface proto][linux-interface-proto] 
- [Linux interface model][linux-interface-model]
- [Linux interface conf file][linux-interface-conf-file] 

The Linux interface plugin watches for changes to the supported interface types. The VPP interface plugin removes "redundant" configuration. The Linux interface plugin does not. Instead, it holds on to the state of all VETH and TAP interfaces present in the default namespace.

Namespaces are supported but handled separately by the [Namespace plugin](#namespace-plugin). Interfaces in non-default namespaces remain unknown and not listed in the NB configuration, if not registered during a resync. The VPP agent uses an [external library][netlink-repo] to manage and control OS interfaces.   

The Linux interface proto describes the interface including type, name, namespace, host OS name, IP addresses, and items specific to TAP or VETH links. You must include an logical interface `name`, and can define a `host_if_name`. If you do not define one, the host interface name will match the logical interface `name`. 

---

### VETH

The Linux interface plugin supports a [VETH interface][veth-man-page]. The characteristics of a VETH interface: 
 
- VETH device is a local Ethernet tunnel 
- Devices are created in pairs. 
- Each device can send and receive packets to each other.
- When either device is down, the link state of the pair is down. 

You need a VETH device to communicate between namespaces, including the host namespace.

!!! Note
    At different times, VETHs are characterized as devices, interfaces, local tunnels, and connection endpoints. The VPP agent views a VETH as an interface.


You create VETH interfaces after defining both VETH devices, and making each known to the VPP agent. You can configure a single VETH device, but the KV Scheduler will cache the configuration until you define the other peer VETH device.

You must include the `peer_if_name` in your VETH configuration. It defines the name of the peer VETH device. You can also enable checksum offloading of transmitted and received packets. 
 

---

### TAP

The TAP kernel interface simulates a link-layer device. The VPP agent does not directly create a TAP interface. Instead, VPP puts the TAP interface into the host OS default namespace once you create a VPP-TAP interface. 

The [VPP interface proto][vpp-interface-proto] for the TAP interface includes the `host_if_name`. This field defines the TAP interface name as known to the host. 
 
After you create a VPP-TAP interface, the plugin lets you configure TAP interface IP and MAC addresses, or the move the interface to a different namespace. 
 
If a Linux TAP delete action is invoked, the interface is reverted to the original state.  All configured fields are stripped, and it is moved back to the default namespace. The interface disappears when its VPP counterpart is removed. 

If you delete a TAP interface, it reverts to its original state. This process strips all configured fields, and moves it back to the default namespace. The interface disappears after you remove its VPP counterpart.  

---

**Linux Interface Configuration Examples**

**KV Data Store**
 
Example VETH configuration. Make sure you define each interface with the correct `peer_if_name`. 

First VETH:
```json
{  
    "name":"veth1",
    "type":"VETH",
    "namespace":{  
        "type":"NSID",
        "reference":"ns1"
    },
    "enabled":true,
    "ip_addresses":[  
        "192.168.22.1/24",
        "10.0.2.2/24"
    ],
    "phys_address":"D2:74:8C:12:67:D2",
    "mtu":1500,
    "veth":{  
        "peer_if_name":"veth2"
    }
}
```

Second VETH:
```json
{  
    "name":"veth2",
    "type":"VETH",
    "namespace":{  
        "type":"NSID",
        "reference":"ns2"
    },
    "enabled":true,
    "ip_addresses":[  
        "192.168.22.5/24"
    ],
    "phys_address":"92:C7:42:67:AB:CD",
    "mtu":1500,
    "veth":{  
        "peer_if_name":"veth1"
    }
}
```

Tap interface: 
```json
{  
    "name":"tap1",
    "type":"TAP_TO_VPP",
    "namespace":{  
        "type":"NSID",
        "reference":"ns2"
    },
    "host_if_name":"tap-host",
    "enabled":true,
    "ip_addresses":[  
        "172.52.45.127/24"
    ],
    "phys_address":"BC:FE:E9:5E:07:04",
    "mtu":1500,
    "tap":{  
        "vpp_tap_if_name":"tap-host"
    }
}
```

Put interfaces:
```bash
# VETH1
agentctl kvdb put /vnf-agent/vpp1/config/linux/interfaces/v2/interface/veth1 '{"name":"veth1","type":"VETH","namespace":{"type":"NSID","reference":"ns1"},"enabled":true,"ip_addresses":["192.168.22.1/24","10.0.2.2/24"],"phys_address":"D2:74:8C:12:67:D2","mtu":1500,"veth":{"peer_if_name":"veth2"}}'

# VETH 2
agentctl kvdb put /vnf-agent/vpp1/config/linux/interfaces/v2/interface/veth2 '{"name":"veth2","type":"VETH","namespace":{"type":"NSID","reference":"ns2"},"enabled":true,"ip_addresses":["192.168.22.5/24"],"phys_address":"92:C7:42:67:AB:CD","mtu":1500,"veth":{"peer_if_name":"veth1"}}'

# TAP
agentctl kvdb put /vnf-agent/vpp1/config/linux/interfaces/v2/interface/tap1 '{"name":"tap1","type":"TAP_TO_VPP","namespace":{"type":"NSID","reference":"ns2"},"host_if_name":"tap-host","enabled":true,"ip_addresses":["172.52.45.127/24"],"phys_address":"BC:FE:E9:5E:07:04","mtu":1500,"tap":{"vpp_tap_if_name":"tap-host"}}'
```

---

**REST**

API References: 

- [Linux Interfaces][linux-interfaces]
- [Linux Interface Stats][linux-interface-stats]

GET all Linux interfaces:
```bash
curl -X GET http://localhost:9191/dump/linux/v2/interfaces
```

---

**gRPC**

Prepare data:
```go
linuxTap := &linuxIf.Interface{
		Name:        "tap1",
		HostIfName:  "tap-host",
		Type:        linuxIf.Interface_TAP_TO_VPP,
		Enabled:     true,
		PhysAddress: "BC:FE:E9:5E:07:04",
		Namespace: &linux_namespace.NetNamespace{
			Reference: "ns2",
			Type:      linux_namespace.NetNamespace_NSID,
		},
		Mtu: 1500,
		IpAddresses: []string{
			"172.52.45.127/24",
		},
		Link: &linuxIf.Interface_Tap{
			Tap: &linuxIf.TapLink{
				VppTapIfName: "tap-host",
			},
		},
	}
```

Prepare configuration:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/linux"
)

config := &configurator.Config{
		LinuxConfig: &linux.ConfigData{
			Interfaces: []*interfaces.Interface{
				memoryInterface,
			},
		}
	}
```

Update the configuration using the client of the `ConfigurationClient` type:
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


The Linux plugin namespace management requires that docker run in privileged mode with the `--privileged`) flag set. Otherwise, the VPP agent can fail to start.

---
 
### Disable the Linux Interface Plugin

You can disable the Linux interface plugin by setting `disabled: true` in the [Linux interface][linux-interface-conf-file] and [Linux L3 plugin][linux-l3-conf-file] conf files respectively.

---

## IP Tables Plugin

Linux kernel networking supports a firewall function using the [iptables][iptables-digital-ocean-blog] utility. Iptables use rulechains to define firewall policies.

Rulechain concepts:

- Chains in the kernel packet processing pipeline where a specific table is triggered. There are predefined chains such as `prerouting` and `postrouting`. You can create and instantiate custom chains.
<br>
</br> 
- Table(s) defining how packets should be evaluated at each chain as they traverse their way through the stack. 
 
A sequence of tables and their respective chains define the rulechains that in turn, make up an iptables firewall policy.
 
The Linux iptables proto include the configurable options for defining and managing rulechains.        

**References:**

- [Linux iptables proto][linux-iptables-proto]
- [Linux iptables model][linux-iptables-model]
- [Linux iptables conf file][linux-iptables-conf-file]

---

## L3 Plugin

The Linux L3 plugin supports the configuration of Linux ARP entries and Linux routes. The [Linux interface plugin](#linux-interface-plugin) is a dependency. L3 configuration items support the Linux namespace in the same manner as the Linux interface.  

### Linux ARP

The address resolution protocol (ARP) discovers the MAC address associated with a given IP address. An interface and IP address define a unique ARP entry in the ARP table.

**References**

- [Linux ARP proto][linux-arp-proto]
- [Linux ARP model][linux-l3-arp-route-models]
- [Linux L3 conf file][linux-l3-conf-file]

The Linux ARP proto defines an interface, IP address and MAC address fields. All fields are mandatory.

The ARP entry is dependent on the associated Linux interface. The KV Scheduler postpones ARP configuration updates until the interface appears. The target namespace is derived from the interface identified by the unique name. You must use a unique host interface name within the namespace. 

---

**Linux ARP Configuration Examples**

**KV Data Store**
 
Example data:
```json
{  
    "interface":"veth1",
    "ip_address":"130.0.0.1",
    "hw_address":"46:06:18:DB:05:3A"
}
```

Put linux ARP entry:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/linux/l3/v2/arp/veth1/130.0.0.1 '{"interface":"veth1","ip_address":"130.0.0.1","hw_address":"46:06:18:DB:05:3A"}'
```

---

**REST**

API Reference: [Linux L3 ARPs][linux-l3-arps]

GET all Linux ARP entries:
```bash
curl -X GET http://localhost:9191/dump/linux/v2/arps
```

---

**gRPC**

Prepare data:
```go
linuxArp := &linuxL3.ARPEntry{
		Interface: "linuxIf1",
		IpAddress: "130.0.0.1",
		HwAddress: "46:06:18:DB:05:3A",
	}
```

Prepare the gRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/linux"
)

config := &configurator.Config{
		LinuxConfig: &linux.ConfigData{
			ArpEntries []*l3.ARPEntry {    
			    linuxArp,
			},	
		}
	}
```


Update the configuration using the client of the `ConfigurationClient` type:
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

### Linux Routes

Linux routing table entries contain destination network prefixes, a corresponding next_hop IP address and the outbound interface.

**References:**

- [Linux routes proto][linux-route-proto]
- [Linux routes model][linux-l3-arp-route-models]
- [Linux L3 conf file][linux-l3-conf-file]

You must have an interface for the route. The namespace is derived from the interface in the same manner as Linux ARP. If you configure a destination prefix and next_hop gateway, you must use valid IP addresses.  

You can include a scope for each route, that defines the network domain where the route is applicable.  

The scope options are:

- Global 
- Site
- Link
- Host

---   

**Linux Route Configuration Examples**

**KV Data Store**

Example configuration:
```json
{  
    "outgoing_interface":"linuxIf1",
    "scope":"GLOBAL",
    "dst_network":"10.0.2.0/24",
    "metric":100
}
```

Put Linux route:
```bash
agentctl kvdb put /vnf-agent/vpp1/config/linux/l3/v2/route/10.0.2.0/24/veth1 '{"outgoing_interface":"veth1","scope":"GLOBAL","dst_network":"10.0.2.0/24","metric":100}'
```

---

**REST**

API Reference: [Linux L3 routes][linux-l3-routes]

GET Linux routing table entries:
```bash
curl -X GET http://localhost:9191/dump/linux/v2/routes
```

---

**gRPC**

Prepare data:
```go
linuxRoute := &linuxL3.Route{
		DstNetwork:        "10.0.2.0/24",
		OutgoingInterface: "veth1",
		Metric:            100,
		Scope:             linuxL3.Route_HOST,
	}
```

Prepare data:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"github.com/ligato/vpp-agent/proto/ligato/linux"
)

config := &configurator.Config{
		LinuxConfig: &linux.ConfigData{
			Routes: []*l3.Route  {    
			    linuxRoute,
			},	
		}
	}
```

Update the configuration using the client of the `ConfigurationClient` type:
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

## Namespace Plugin

Linux plugins use the auxiliary namespace plugin to handle namespaces and microservices. 

**References:**

- [Linux namespace proto][linux-namespace-proto]
- [Linux namespace model][linux-namespace-model]
- [Linux namespace conf file][linux-namespace-conf-file]

The namespace proto contains two fields: `type` and `reference`. 

Type options are:
 
- `NSID` - named namespace
- `PID` - namespace of a given process 
- `FD` - namespace referenced by file handle
- `MICROSERVICE` docker container running a microservice. 

The `reference` is a string with information specific to namespace type.

The namespace plugin imports the namespace into the [Linux interface plugin][linux-interface-guide]. It does not define any keys except for notifications. 

---

### Linux Network Namespaces

The VPP agent supports Linux network namespaces. You can attach a Linux interface/ARP/route into sa new, existing, or yet-to-be-created network namespace via the `Namespace` configuration section defined in the [Linux interface proto][linux-interface-proto].

**References:**

- [Linux namespace proto][linux-namespace-proto]
- [Linux namespace model][linux-namespace-model]
- [Linux namespace conf file][linux-namespace-conf-file]

You can reference a namespace in multiple ways. One technique uses a file descriptor associated with a symbolic link to a namespace. This link is automatically created in the `proc` filesystem. It points to the definition of the namespace used by a given process (`/proc/<PID>/ns/net`), or to a task of a given process (`/proc/<PID>/task/<TID>/ns/net`).
 
The more common approach uses the PID:

- Of the process's namespace you want to attach to, or
- To create a bind-mount of the symbolic link into the `/var/run/netns` directory, and use the filename of that mount.

The latter is called the `named` namespace. You can create a `named` namespace using the `ip netns` command line tool from the `iproute2` package. The `named` namespace can survive, after the process that created it disappears, which is a big advantage. 

You can view the namespace configuration section as a union of values. You first set the type, and then store the reference into the appropriate `pid`, `name` or `microservice` field. The VPP agent supports PID-based references and `named` namespaces.

---

### Microservices

The VPP agent also supports non-standard namespace reference, denoted as `MICROSERVICE_REF_NS`, which is specific to ecosystems with microservices. This lets you attach a Linux interface/ARP/route into the namespace of a container that runs a microservice with a given microservice label.

You are not required to start the microservice before you update the configured item. The VPP agent will postpone interface (re)configuration until you launch the referenced microservice. Behind the scenes, the VPP agent communicates with the docker daemon to construct and maintain an up-to-date map of microservice labels to PIDs, and the IDs of their corresponding containers. Whenever the VPP agent detects a new microservice, it moves all pending interfaces to the microservice namespace.

[grpc-tutorial]: ../tutorials/08_grpc-tutorial.md
[key-reference]: ../user-guide/reference.md
[linux-arp-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/l3/arp.proto
[linux-interface-conf-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/linux/ifplugin/linux-ifplugin.conf 
[linux-interface-guide]: linux-plugins.md#linux-interface-plugin
[linux-interface-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/interfaces/models.go
[linux-interface-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/interfaces/interface.proto
[linux-iptables-conf-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/linux/iptablesplugin/linux-iptablesplugin.conf
[linux-iptables-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/iptables/models.go
[linux-iptables-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/iptables/iptables.proto
[linux-key-reference]: ../user-guide/reference.md#linux-keys
[linux-l3-arp-route-models]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/l3/models.go
[linux-l3-conf-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/linux/l3plugin/linux-l3plugin.conf 
[linux-namespace-conf-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/linux/nsplugin/linux-nsplugin.conf
[linux-namespace-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/namespace/models.go 
[linux-namespace-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/namespace/namespace.proto
[linux-punt-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/punt/punt.proto
[linux-route-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/l3/route.proto
[linux-interfaces]: ../api/api-vpp-agent.md#linux-interfaces
[linux-l3-arps]: ../api/api-vpp-agent.md#linux-l3-arps
[linux-interface-stats]: ../api/api-vpp-agent.md#linux-interface-stats
[linux-l3-routes]: ../api/api-vpp-agent.md#linux-l3-routes
[iptables-digital-ocean-blog]: https://www.digitalocean.com/community/tutorials/a-deep-dive-into-iptables-and-netfilter-architecture
[netlink-repo]: https://github.com/vishvananda/netlink
[redhat-veth-page]: http://man7.org/linux/man-pages/man4/veth.4.html
[veth-man-page]: http://man7.org/linux/man-pages/man4/veth.4.html
[vpp-interface-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/interfaces/interface.proto 

*[ARP]: Address Resolution Protocol
