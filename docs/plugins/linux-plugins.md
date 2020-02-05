# Linux plugins

---

## Interface plugin

The Linux interface plugin manages linux-based host OS interfaces. Currently it only supports VETH and TAP interface types.  

The plugin watches for changes to the supported interface types. The difference between the VPP and Linux interface plugin is that the latter does not remove any "redundant" configuration. Instead, the plugin holds on to the state of all VETH and TAP interfaces present in the default namespace.

Namespaces are also supported, however they are handled by a different plugin. Interfaces in namespaces other than default remain unknown if not listed in the NB configuration (i.e. they are not registered during resync). The vpp-agent uses an external library to fetch data about OS interfaces and manage them.  

[The Linux interface model][linux-interface-model] describes the interface and is composed of two parts. The first part describes those items common to the supported interface types. The second part describes items specific to each interface type. The interface may have an optional namespace field. The interface name is required, and the host name can be specified. If not specified, the host name will be the same as the logical name.

### VETH

As described in a document available on the [redhat developers page][redhat-veth-page],
 
- [VETH (virtual Ethernet)][veth] device is a local Ethernet tunnel 
- Devices are created in pairs. 
- Packets transmitted on one device in the pair are immediately received on the other device.
- When either device is down, the link state of the pair is down. The VETH device is needed when namespaces need to communicate to the main host namespace or between each other.

!!! Note
    At different times, VETHs are characterized as `devices`, `interfaces`, and `local tunnels`. All surely apply but if a single abstracted term is needed, all of those terms can be lumped under `connection endpoint` or some variation thereof. Note that the vpp-agent considers VETH to be an interface.


VETH interfaces are created after both ends are defined and known to the vpp-agent. The single VETH can be put to the vpp-agent, but it is cached until its counterpart config appears. 

The type-specific configuration is of type `VethLink` and contains a mandatory field `peer_if_name`.  This is the name of the adjacent VETH endpoint. Optionally, checksum offloading of the received and transmitted packets can be defined.  

### TAP

The TAP interface is a virtual network kernel interface that simulates a link-layer device. TAP interfaces are not directly created by the vpp-agent. Rather they are automatically put to the host OS default namespace by VPP after the VPP TAP interface is created. The [VPP interface plugin model][vpp-interface-model] for TAPs defines a special field used to define the host name of the Linux TAP interface that is created.
 
After that, the Linux interface can be modified by the linux plugin. IP or MAC addresses can be set, or the interface can be moved to a different namespace.
 
The Linux tap delete means that the interface is reverted to the original state, all configured fields are stripped and it is moved back to default namespace. The interface disappears when its VPP counterpart is removed. 

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for the Linux interface to the database ([key reference][key-reference]).

Example configuration for VETH. Keep in mind that the two interfaces must be defined with the correct peer name:

The first VETH:
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

The second VETH:
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

Use `etcdctl` to put both compacted interface key-value entries:
```bash
# VETH1
etcdctl put /vnf-agent/vpp1/config/linux/interfaces/v2/interface/veth1 '{"name":"veth1","type":"VETH","namespace":{"type":"NSID","reference":"ns1"},"enabled":true,"ip_addresses":["192.168.22.1/24","10.0.2.2/24"],"phys_address":"D2:74:8C:12:67:D2","mtu":1500,"veth":{"peer_if_name":"veth2"}}'

# VETH 2
etcdctl put /vnf-agent/vpp1/config/linux/interfaces/v2/interface/veth2 '{"name":"veth2","type":"VETH","namespace":{"type":"NSID","reference":"ns2"},"enabled":true,"ip_addresses":["192.168.22.5/24"],"phys_address":"92:C7:42:67:AB:CD","mtu":1500,"veth":{"peer_if_name":"veth1"}}'

# TAP
etcdctl put /vnf-agent/vpp1/config/linux/interfaces/v2/interface/tap1 '{"name":"tap1","type":"TAP_TO_VPP","namespace":{"type":"NSID","reference":"ns2"},"host_if_name":"tap-host","enabled":true,"ip_addresses":["172.52.45.127/24"],"phys_address":"BC:FE:E9:5E:07:04","mtu":1500,"tap":{"vpp_tap_if_name":"tap-host"}}'
```

**2. Using REST:**

REST currently supports only retrieval of the existing configuration. The following command can be used to read all Linux interfaces via cURL:

```bash
curl -X GET http://localhost:9191/dump/linux/v2/interfaces
```
**3. Using GRPC:**

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

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/linux"
)

config := &configurator.Config{
		LinuxConfig: &linux.ConfigData{
			Interfaces: []*interfaces.Interface{
				memoryInterface,
			},
		}
	}
```

The config data can be combined with any other Linux and VPP configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Limitations

The linux plugin namespace management requires that docker run in privileged mode with the `--privileged`) flag set. Otherwise, the vpp-agent can fail to start. 

### Disable the Linux interface plugin

Use the config files for Linux interfaces and the Linux l3 plugins and set both to `disabled` and `true`.

## L3 plugin

The Linux L3 plugin is used to configure **Linux ARP** entries and **Linux routes**. The Linux L3 plugin is dependent on the [Linux interface plugin][linux-interface-guide]. L3 configuration items support the Linux namespace in the same manner as the Linux interfaces.  

### ARP entries

ARP is a network protocol that assists in the discovery of a MAC address associated with the given IP address. The plugin makes a use of a simple proto model with interface, IP address and MAC address fields. All fields are mandatory.

The namespace resolution is automatic and is of no concern to the user. Since the ARP entry is dependent on the associated Linux interface, the configuration is postponed until  the moment the interface appears. Target namespace is derived from the interface identified by the unique name. The interface host name must be unique within the namespace 
scope.   

The northbound Linux plugin API defines ARP in the [linux arp model][linux-arp-model]. Every ARP entry is defined with a MAC address, IP address, and an interface. The IP address and the interface are a part of the key.

In the generated proto model, the ARP is referred to as the `ARPEntry` object.

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for Linux ARP entries to the database ([key reference][key-reference]).

Example data:
```json
{  
    "interface":"veth1",
    "ip_address":"130.0.0.1",
    "hw_address":"46:06:18:DB:05:3A"
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/linux/l3/v2/arp/veth1/130.0.0.1 '{"interface":"veth1","ip_address":"130.0.0.1","hw_address":"46:06:18:DB:05:3A"}'
```

To remove the configuration:
```bash
etcdctl del /vnf-agent/vpp1/config/linux/l3/v2/arp/veth1/130.0.0.1
```

**2. Using REST:**

REST currently supports only the retrieval of the existing configuration. The following command can be used to read all ARP entries via cURL:

```bash
curl -X GET http://localhost:9191/dump/linux/v2/arps
```

**3. Using GRPC:**

Prepare the Linux ARP data:
```go
linuxArp := &linuxL3.ARPEntry{
		Interface: "linuxIf1",
		IpAddress: "130.0.0.1",
		HwAddress: "46:06:18:DB:05:3A",
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/linux"
)

config := &configurator.Config{
		LinuxConfig: &linux.ConfigData{
			ArpEntries []*l3.ARPEntry {    
			    linuxArp,
			},	
		}
	}
```

The config data can be combined with any other Linux or VPP configuration.

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### Routes

A routing table lists the routes to network destinations, metrics (i.e. distances, link costs, etc.) associated with said routes and an outbound interface pointing to the gateway (next_hop) router. The route interface is mandatory (the namespace is derived from the interface, the same principle as for the Linux ARP). If the source and gateway (next_hop router) IP addresses are set, they must be valid and the gateway IP address must be evaluated as reachable. 

Every route includes a scope which is the network domain where the route is applicable. The options are `Global`, `Site`, `Link` and `Host`. The scope is defined in the route config.  

The northbound Linux plugin API defines route in the [linux route model][linux-route-model].

**Configuration example**

**1. Using the key-value database** to put the proto-modelled data with the correct key for Linux routes to the database ([key reference][key-reference]).

Example configuration:
```json
{  
    "outgoing_interface":"linuxIf1",
    "scope":"GLOBAL",
    "dst_network":"10.0.2.0/24",
    "metric":100
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/linux/l3/v2/route/10.0.2.0/24/veth1 '{"outgoing_interface":"veth1","scope":"GLOBAL","dst_network":"10.0.2.0/24","metric":100}'
```

To remove the configuration:
```bash
etcdctl del /vnf-agent/vpp1/config/linux/l3/v2/route/10.0.2.0/24/veth1
```

**2. Using REST:**

REST currently supports only the retrieval of the existing configuration. The following command can be used to read all linux route entries via cURL:

```bash
curl -X GET http://localhost:9191/dump/linux/v2/routes
```

**3. Using GRPC:**

Prepare the Linux route data:
```go
linuxRoute := &linuxL3.Route{
		DstNetwork:        "10.0.2.0/24",
		OutgoingInterface: "veth1",
		Metric:            100,
		Scope:             linuxL3.Route_HOST,
	}
```

Prepare the GRPC config data:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
	"github.com/ligato/vpp-agent/api/models/linux"
)

config := &configurator.Config{
		LinuxConfig: &linux.ConfigData{
			Routes: []*l3.Route  {    
			    linuxRoute,
			},	
		}
	}
```

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]:
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

## Namespace plugin

The namespace plugin is an auxiliary plugin used by other Linux plugins to handle namespaces and microservices.

The [namespace model][linux-namespace-model] contains two fields; type and reference. The namespace can be of type:
 
- named
- process ID
- file handle reference
- docker container running a microservice. 

The reference is a namespace identifier (namespace ID, specific PID, file path or a microservice label).

The namespace is imported in the [Linux interface plugin][linux-interface-guide]. By itself it does not define any keys (except notifications). 

### Namespaces

The vpp-agent supports Linux network namespaces. It is possible to attach a Linux interface/ARP/route into a new, existing or even yet-to-be-created network namespace via the `Namespace` configuration section inside the data model.

Namespace can be referenced in multiple ways. 

- The most low-level link to a namespace is a file descriptor associated with the symbolic link automatically created in the `proc` filesystem, pointing to the definition of the namespace used by a given process (`/proc/<PID>/ns/net`) or by a task of a given process (`/proc/<PID>/task/<TID>/ns/net`). 
- A more common approach is to use just the PID of the process whose namespace we want to attach to, or to create a bind-mount of the symbolic link into `/var/run/netns` directory and use the filename of that mount. The latter is called `named` namespace and it is created and managed, for example, by the `ip netns` command line tool from the `iproute2` package. The advantage of `named` namespace is that it can outlive the process it was originally created by.

A `namespace` configuration section can be seen as a union of values. First, set the type and then store the reference into the appropriate field (`pid` vs. `name` vs `microservice`). The vpp-agent supports both PID-based references as well as `named` namespaces.

### Microservices

Additionally, we provide a non-standard namespace reference, denoted as `MICROSERVICE_REF_NS`, which is specific to ecosystems with microservices. It is possible to attach interface/ARP/route into the namespace of a container that runs a microservice with a given label. 

To simplify further, it is not required to start the microservice before the configured item is pushed. The vpp-agent will postpone interface (re)configuration until the referenced microservice is launched. Behind the scenes, the vpp-agent communicates with the docker daemon to construct and maintain an up-to-date map of microservice labels to PIDs and the IDs of their corresponding containers. Whenever a new microservice is detected, all pending interfaces are moved to its namespace.

[grpc-tutorial]: ../tutorials/08_grpc-tutorial.md
[key-reference]: ../user-guide/reference.md
[linux-arp-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/linux/l3/arp.proto
[linux-interface-guide]: linux-plugins.md#interface-plugin
[linux-interface-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/linux/interfaces/interface.proto
[linux-namespace-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/linux/namespace/namespace.proto
[linux-route-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/linux/l3/route.proto
[redhat-veth-page]: http://man7.org/linux/man-pages/man4/veth.4.html
[veth]: http://man7.org/linux/man-pages/man4/veth.4.html
[vpp-interface-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/interface.proto

*[ARP]: Address Resolution Protocol
