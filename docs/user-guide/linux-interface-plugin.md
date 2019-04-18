The Linux interface plugin is a base plugin managing unix-based host OS interfaces. Currently it only supports interfaces of type vETH and TAP, since these interface types mostly serve as counterparts to VPP interfaces. 

The plugin also watches interface changes but only supported types are considered relevant. The difference between the VPP and Linux interface plugin is that the Linux plugin does not remove any "redundant" (in terms of agent) configuration and leaves it intact. Instead, the plugin holds state of all vETH and TAP interfaces present in the default namespace.

Namespaces are also supported, however they are handled by a different plugin. Interfaces in namespaces other than default remain unknown if not listed in the NB configuration (they are not registered during resync).

The vpp-agent uses external library to fetch data about OS interfaces and manage them.  

The Linux interface model defines following [model][linux-interface-model] describing model with the first part common for all interface types, and the second part specific for every interface type. The interface have an optional namespace field. The interface name is required, and the host name can be specified as well. If not, the host name will 
be the same as the logical name.

### Veth

The VETH (virtual Ethernet) device is a local Ethernet tunnel. Devices are created in pairs. Packets transmitted on one device in the pair are immediately received on the other device. When either device is down, the link state of the pair is down. The VETH configuration when namespaces need to communicate to the main host namespace or between each other.

The VETH interfaces are created after both ends are defined and known to the agent. The single VTEP can be put to the vpp-agent, but it is cached until its counterpart config appears. 

The type-specific configuration is of type `VethLink` and contains a mandatory field `peer_if_name` which is the name of the adjacent VETH end. Optionally, the checksum offloading of the received and transmitted packets can be defined. The VETH interface is in the vpp-agent used as an target for the VPP af-packet interface. 

### TAP

The TAP interface is a virtual network kernel interface which simulates a link-layer device. TAP interfaces are not directly created by the vpp-agent - they are automatically put to the host OD default namespace by the VPP after the VPP TAP interface is created. The [VPP interface plugin model][vpp-interface-model] for TAPs defines a special field used to define the host name of the Linux TAP interface created.
 
After that, the Linux interface can be modified by the linux plugin - IP or MAC address can be set, or the interface can be moved to different namespace.
 
The Linux tap delete means that the interface is reverted to the original state, all configured fields are stripped and it is moved back to default namespace. The interface disappears when its VPP counterpart is removed. 

### Configuration example

**1. Using the key-value database** put the proto-modelled data with the correct key for Linux interface to the database ([key reference][key-reference]).

Example configuration for VETH. Keep in mind that two interfaces must be defined with correct peer name:

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

The REST currently supports only retrieval of the existing configuration. The following command can be used to read all Linux interfaces via cURL:

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

The linux plugin namespace management requires docker ran in the privileged mode (with the `--privileged`) parameter. Otherwise, the vpp-agent can fail to start. 

### Disable the Linux interface plugin

Use the Linux config file for interfaces and for l3 plugin and set `disabled` na `true` (both of them must be disabled).

[grpc-tutorial]: ../tutorials/08_grpc-tutorial.md
[key-reference]: references.md
[linux-interface-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/linux/interfaces/interface.proto
[vpp-interface-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/interface.proto