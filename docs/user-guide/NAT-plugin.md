# Network Address Translation plugin

**Written for: v2.0-vpp18.10**

Network address translation, or NAT is a method of remapping IP address space into another IP address space modifying address information in the packet header. The VPP-Agent Network address translation is control plane plugin for the VPP NAT implementation of NAT44. The NAT plugin is dependent on [interface plugin](VPP-Interface-plugin).

- [NAT global config](#global)
  * [Model](#gmodel)
  * [Configuration](#gconf)
- [DNAT44](#dnat44)
  * [Model](#dnat44model)
  * [Configuration](#dnat44conf)
  
## <a name="global">NAT global config</a>  

The global NAT configuration is a special case of data grouped under single key (it means no unique character is a part of the key, so there is always only one global NAT configuration). The data are used to enable NAT features (like forwarding), enable interfaces for NAT, define NAT IP addresses (address pools) or specify virtual reassembly.  
Interfaces marked to be enabled for NAT should be present in the VPP but if not, the Scheduler plugin caches the configuration for later use when the incriminated interface is available.

### <a name="gmodel">Model</a>  

The global configuration is divided into several independent parts defining certain VPP NAT features.

**Forwarding** is a boolean field enabling or disabling forwarding.

**NAT interfaces** represent a list of interfaces which will be enabled for NAT. If the interface does not exist in the VPP, it is cached and potentially configured later. Every interface is defined by its logical name and whether is an inside or an outside interface. The output feature is also defined here.

**Address pools** is a list of IP addresses for given VRF table and with either enabled or disabled twice NAT. Despite the name, only one address is defined in the single "pool" entry.

**Virtual reassembly** provides support for datagram fragmentation handling to allow correct recalculation of higher-level checksums.

### <a name="gconf">Configuration</a>  

How to configure the global NAT status:

**1. Using the key-value database** put the proto-modelled data with the correct key for global NAT to the database. 

Key:
```
/vnf-agent/vpp1/config/vpp/nat/v2/nat44-global/settings
```

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

The REST currently supports only retrieving of the existing configuration. The following command can be used to read the NAT global config via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/nat/global
```

**3. Using GRPC:**

* Prepare the global NAT data:
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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

**The VPP CLI NAT44 global commands:** 

The VPP cli has following CLI commands to verify configuration:
 - show list of all NAT44 addresses: `show nat44 addresses`
 - show list of all NAT44 interfaces: `show nat44 interfaces`
 - show list of all NAT44 interface addresses: `show nat44 interface address`

## <a name="dnat44">DNAT44</a>  

Destination network address translation (DNAT) allows transparently changing the destination IP address of an packet and performing the inverse function for any replies. Any router situated between two endpoints can perform this transformation of the packet.
In the VPP-Agent, the DNAT configuration is a list of static and/or identity mappings labelled under single key.

### <a name="dnat44model">Model</a>  

The DNAT44 consists from two main parts - static mappings and identity mappings. The static mapping can be load balanced - if more than one local IP address is defined for single static mapping, the load balancer is automatically allowed for that mapping. 
THe DNAT44 contains a unique label serving as an identifier. However, the DNAT configuration is not limited, an arbitrary count of static and identity mappings can be listed under single label. 

### <a name="dnat44conf">Configuration</a>  

How to configure the DNAT:

**1. Using the key-value database** put the proto-modelled data with the correct key for the DNAT to the database. 

Key:
```
/vnf-agent/vpp1/config/vpp/nat/v2/dnat44/<label>
```

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

The REST currently supports only retrieving of the existing configuration. The following command can be used to read the NAT global config via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/nat/dnat
```

**3. Using GRPC:**

* Prepare the DNAT data:
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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

**The VPP CLI DNAT44 commands:** 

The VPP cli has following CLI commands to verify configuration:
 - show static mappings: `show nat44 static mappings`


