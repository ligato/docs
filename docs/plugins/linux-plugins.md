# Linux Plugins

This section describes the VPP Agent Linux plugins. Each plugin section provides:

- Short description
- Pointers to the `*.proto` containing configuration/NB protobuf API definitions, the `models.go` file defining the model, and the conf file if one is available.  
- Example configuration interactions using an etcd data store, REST and gPRC.

**Agentctl** 

Linux configuration data can also be managed using `agentctl config` commands.
```
Usage:	agentctl config COMMAND

Manage agent configuration

COMMANDS
  get         Get config from agent
  history     Retrieve config history
  resync      Run config resync
  retrieve    Retrieve currently running config
  update      Update config in agent
```

Reference: [agentctl config commands][agentctl-config]

!!! Note
    For Linux plugins, REST supports the retrieval of the existing configuration. REST cannot be used to add, modify or delete configuration data.

---

## Linux Interface Plugin

The Linux interface plugin manages Linux-based host OS interfaces. It supports VETH and TAP interface types.

**References:**

- [Linux interface proto][linux-interface-proto] 
- [Linux interface model][linux-interface-model]
- [Linux interface conf file][linux-interface-conf-file] 

The Linux interface plugin watches for changes to the supported interface types. The difference between the VPP and Linux interface plugin is that the latter does not remove any "redundant" configuration. Instead, the plugin holds on to the state of all VETH and TAP interfaces present in the default namespace.

Namespaces are also supported but handled separately by the [Namespace plugin](#namespace-plugin). Interfaces in non-default namespaces remain unknown if not registered during resync, or in other words not listed in the NB configuration. The VPP agent uses an [external library][netlink-repo] to manage and control OS interfaces.   

The Linux interface proto describes the interface including type, name, namespace, host OS name, IP addresses, and sections specific to TAP or VETH links. The interface name is required, and the host OS name can be specified. If not specified, the host name will be the same as the logical name.

---

### VETH

The Linux interface plugin supports a [VETH interface][veth-man-page]. The characteristics of a VETH interface are: 
 
- VETH device is a local Ethernet tunnel 
- Devices are created in pairs. 
- Packets transmitted on one device in the pair are immediately received by the other device in the pair.
- When either device is down, the link state of the pair is down. 

The VETH device is needed when namespaces need to communicate to the main host namespace or between each other.

!!! Note
    At different times, VETHs are characterized as `devices`, `interfaces`, and `local tunnels`, and `connection endpoint`. Note that the VPP agent considers VETH to be an interface.


VETH interfaces are created after both ends are defined and made known to the VPP agent. The single VETH can be conveyed to the VPP agent, but it is cached until its counterpart configuration appears. 

The VETH configuration is of type `VethLink` and contains a mandatory field `peer_if_name`.  This is the name of the adjacent VETH endpoint. Optionally, checksum offloading of the received and transmitted packets can be defined.  

---

### TAP

The TAP interface is a virtual network kernel interface that simulates a link-layer device. TAP interfaces are not directly created by the VPP agent. They are automatically put to the host OS default namespace by VPP after the VPP-TAP interface is created. The [VPP interface proto][vpp-interface-proto] for the TAP interface includes the `host_if_name` that defines the host name of the Linux TAP interface that is created.
 
After the VPP-TAP interface is created, the Linux interface can be modified by the Linux interface plugin. IP and MAC addresses can be set, or the interface can be moved to a different namespace.
 
If a Linux TAP delete action is invoked, the interface is reverted to the original state.  All configured fields are stripped, and it is moved back to the default namespace. The interface disappears when its VPP counterpart is removed. 

---

**Linux Interface Configuration Examples**

**KV Data Store**
 
Put the interface configuration data into an etcd data store using the correct [interface key][linux-key-reference].

Example configuration for VETH. Keep in mind that the two interfaces must be defined with the correct peer name.

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

Use `etcdctl` to put both interface key-value entries:
```bash
# VETH1
etcdctl put /vnf-agent/vpp1/config/linux/interfaces/v2/interface/veth1 '{"name":"veth1","type":"VETH","namespace":{"type":"NSID","reference":"ns1"},"enabled":true,"ip_addresses":["192.168.22.1/24","10.0.2.2/24"],"phys_address":"D2:74:8C:12:67:D2","mtu":1500,"veth":{"peer_if_name":"veth2"}}'

# VETH 2
etcdctl put /vnf-agent/vpp1/config/linux/interfaces/v2/interface/veth2 '{"name":"veth2","type":"VETH","namespace":{"type":"NSID","reference":"ns2"},"enabled":true,"ip_addresses":["192.168.22.5/24"],"phys_address":"92:C7:42:67:AB:CD","mtu":1500,"veth":{"peer_if_name":"veth1"}}'

# TAP
etcdctl put /vnf-agent/vpp1/config/linux/interfaces/v2/interface/tap1 '{"name":"tap1","type":"TAP_TO_VPP","namespace":{"type":"NSID","reference":"ns2"},"host_if_name":"tap-host","enabled":true,"ip_addresses":["172.52.45.127/24"],"phys_address":"BC:FE:E9:5E:07:04","mtu":1500,"tap":{"vpp_tap_if_name":"tap-host"}}'
```

---

**REST**

API References: 

- [Linux Interfaces][linux-interfaces]
- [Linux Interface Stats][linux-interface-stats]

Use this cURL command to GET all Linux interfaces:

```bash
curl -X GET http://localhost:9191/dump/linux/v2/interfaces
```

---

**gRPC**

Prepare the interface data (TAP):
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

Prepare the gRPC configuration data:
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

The configuration data can be combined with any other Linux and VPP configuration.

Update data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

### Limitations

The Linux plugin namespace management requires that docker run in privileged mode with the `--privileged`) flag set. Otherwise, the VPP agent can fail to start.
 
### Disable the Linux Interface Plugin

Use the [Linux interface][linux-interface-conf-file] and [Linux L3 plugin][linux-l3-conf-file] conf files respectively and set both to `disabled: true`.

---

## IP Tables Plugin

Linux kernel networking supports a firewall function using the [iptables][iptables-digital-ocean-blog] utility. Iptables work on the notion of rulechains that consist of:

- Chains in the kernel packet processing pipeline where a specific table is triggered. There are predefined chains such `prerouting` and `postrouting`. Custom chains can be created and instantiated as well. 
- Table(s) defining how packets should be evaluated at each chain as they traverse their way through the stack. 
 
A sequence of tables and their respective chains define the rulechains that in turn, make up an iptables firewall policy.
 
The Linux iptables proto defines the configurable options to define and manage iptables rulechains.        

**References:**

- [Linux iptables proto][linux-iptables-proto]
- [Linux iptables model][linux-iptables-model]
- [Linux iptables conf file][linux-iptables-conf-file]

---

## L3 Plugin

The Linux L3 plugin is used to configure Linux ARP entries and Linux routes. The Linux L3 plugin is dependent on the [Linux interface plugin](#linux-interface-plugin). L3 configuration items support the Linux namespace in the same manner as the Linux interface.  

### Linux ARP

The address resolution protocol (ARP) is a communication protocol for discovering the MAC address associated with a given IP address.

**References**

- [Linux ARP proto][linux-arp-proto]
- [Linux ARP model][linux-l3-arp-route-models]
- [Linux L3 conf file][linux-l3-conf-file]

The Linux ARP proto defines an interface, IP address and MAC address fields. All fields are mandatory.

The ARP entry is dependent on the associated Linux interface. An ARP configuration is postponed until the interface appears. The target namespace is derived from the interface identified by the unique name. The interface host name must be unique within the namespace 
scope.   

---

**Linux ARP Configuration Examples**

**KV Data Store**
 
Put the ARP entry configuration data into an etcd data store using the correct [Linux ARP key][linux-key-reference].

Example data:
```json
{  
    "interface":"veth1",
    "ip_address":"130.0.0.1",
    "hw_address":"46:06:18:DB:05:3A"
}
```

Use this `etcdctl` command to put the key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/linux/l3/v2/arp/veth1/130.0.0.1 '{"interface":"veth1","ip_address":"130.0.0.1","hw_address":"46:06:18:DB:05:3A"}'
```

To remove the configuration:
```bash
etcdctl del /vnf-agent/vpp1/config/linux/l3/v2/arp/veth1/130.0.0.1
```

---

**REST**

API Reference: [Linux L3 ARPs][linux-l3-arps]

Use this cURL command to GET all Linux ARP entries:

```bash
curl -X GET http://localhost:9191/dump/linux/v2/arps
```

---

**gRPC**

Prepare the Linux ARP data:
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

The configuration data can be combined with any other Linux or VPP configuration.

Update data with gRPC by using the client of the `ConfigurationClient` type. Read more about how to prepare the gRPC connection, and other CRUD methods in the [GRPC tutorial][grpc-tutorial]:

```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

### Linux Routes

The linux routing table contains destination network prefixes, a corresponding next_hop IP address and the outbound interface.

**References:**

- [Linux routes proto][linux-route-proto]
- [Linux routes model][linux-l3-arp-route-models]
- [Linux L3 conf file][linux-l3-conf-file]

The interface for the route is mandatory. The namespace is derived from the interface in the same manner as Linux ARP. If the source and gateway (next_hop router) IP addresses are set, they must be valid, and the gateway IP address must be evaluated as reachable. 

Every route includes a scope which is the network domain where the route is applicable. The scope is defined in the route configuration.

The scope options are:

- Global 
- Site
- Link
- Host

---   

**Linux Route Configuration Examples**

**KV Data Store**

Put the route data into an etcd data store using the [Linux route key][linux-key-reference].

Example configuration:
```json
{  
    "outgoing_interface":"linuxIf1",
    "scope":"GLOBAL",
    "dst_network":"10.0.2.0/24",
    "metric":100
}
```

Use this `etcdctl` command to put the key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/linux/l3/v2/route/10.0.2.0/24/veth1 '{"outgoing_interface":"veth1","scope":"GLOBAL","dst_network":"10.0.2.0/24","metric":100}'
```

To remove the configuration:
```bash
etcdctl del /vnf-agent/vpp1/config/linux/l3/v2/route/10.0.2.0/24/veth1
```

---

**REST**

API Reference: [Linux L3 routes][linux-l3-routes]

Use this cURL command to GET Linux routing table entries:


```bash
curl -X GET http://localhost:9191/dump/linux/v2/routes
```

---

**gRPC**

Prepare the Linux route data:
```go
linuxRoute := &linuxL3.Route{
		DstNetwork:        "10.0.2.0/24",
		OutgoingInterface: "veth1",
		Metric:            100,
		Scope:             linuxL3.Route_HOST,
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
			Routes: []*l3.Route  {    
			    linuxRoute,
			},	
		}
	}
```

Update data with gRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

---

## Namespace Plugin

The namespace plugin is an auxiliary plugin used by other Linux plugins to handle namespaces and microservices.

**References:**

- [Linux namespace proto][linux-namespace-proto]
- [Linux namespace model][linux-namespace-model]
- [Linux namespace conf file][linux-namespace-conf-file]

The namespace proto contains two fields: type and reference. 

The namespace type options are:
 
- named - SSID
- process ID - PID
- file handle reference - FD
- docker container running a microservice. 

The reference is a namespace identifier composed of the namespace ID, specific PID, file path or a microservice label.

The namespace is imported into the [Linux interface plugin][linux-interface-guide]. It does not define any keys except for notifications. 

---

### Linux Network Namespaces

The VPP agent supports Linux network namespaces. It is possible to attach a Linux interface/ARP/route into a new, existing, or yet-to-be-created network namespace via the `Namespace` configuration section defined in the [Linux interface proto][linux-interface-proto].

**References:**

- [Linux namespace proto][linux-namespace-proto]
- [Linux namespace model][linux-namespace-model]
- [Linux namespace conf file][linux-namespace-conf-file]

Namespace can be referenced in multiple ways. The most low-level link to a namespace is a file descriptor associated with the symbolic link. This is automatically created in the `proc` filesystem. It points to the definition of the namespace used by a given process (`/proc/<PID>/ns/net`), or to a task of a given process (`/proc/<PID>/task/<TID>/ns/net`).
 
The more common approach is to use the PID of the process whose namespace we wish to attach to, or to create a bind-mount of the symbolic link into the `/var/run/netns` directory and use the filename of that mount. 

The latter is called the `named` namespace and it is created and managed, for example, by the `ip netns` command line tool from the `iproute2` package. The advantage of the `named` namespace is that it can survive, after the process that created it disappears. 

A namespace configuration section can be seen as a union of values. First, set the type, then store the reference into the appropriate field be it `pid`, `name` or `microservice`. The VPP agent supports PID-based references and `named` namespaces.

---

### Microservices

Additionally, we provide a non-standard namespace reference, denoted as `MICROSERVICE_REF_NS`, which is specific to ecosystems with microservices. It is possible to attach a Linux interface/ARP/route into the namespace of a container that runs a microservice with a given microservice label. 

It is not required to start the microservice before the configured item is pushed. The VPP agent will postpone interface (re)configuration until the referenced microservice is launched. Behind the scenes, the VPP agent communicates with the docker daemon to construct and maintain an up-to-date map of microservice labels to PIDs, and the IDs of their corresponding containers. Whenever a new microservice is detected, all pending interfaces are moved to its namespace.

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
