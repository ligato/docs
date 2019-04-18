# NAT global config 

The global NAT configuration is a special case of data grouped under single key (it means no unique character is a part of the key, so there is always only one global NAT configuration). The data are used to enable NAT features (like forwarding), enable interfaces for NAT, define NAT IP addresses (address pools) or specify virtual reassembly. Interfaces marked to be enabled for NAT should be present in the VPP but if not, the Scheduler plugin caches the configuration for later use when the incriminated interface is available.

The global configuration is divided into several independent parts defining certain VPP NAT features:

  - **Forwarding** is a boolean field enabling or disabling forwarding.
  - **NAT interfaces** represent a list of interfaces which will be enabled for NAT. If the interface does not exist in   the VPP, it is cached and potentially configured later. Every interface is defined by its logical name and whether is   an inside or an outside interface. The output feature is also defined here.
  - **Address pools** is a list of IP addresses for given VRF table and with either enabled or disabled twice NAT.   Despite the name, only one address is defined in the single "pool" entry.
  - **Virtual reassembly** provides support for datagram fragmentation handling to allow correct recalculation of   higher-level checksums.

### Configuration example

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

The REST currently supports only retrieving of the existing configuration. The following command can be used to read the NAT global config via cURL:

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

# DNAT44

Destination network address translation (DNAT) allows transparently changing the destination IP address of an packet and performing the inverse function for any replies. Any router situated between two endpoints can perform this transformation of the packet. In the VPP-Agent, the DNAT configuration is a list of static and/or identity mappings 
labelled under single key.

The DNAT44 consists from two main parts - static mappings and identity mappings. The static mapping can be load balanced - if more than one local IP address is defined for single static mapping, the load balancer is automatically allowed for that mapping. 
THe DNAT44 contains a unique label serving as an identifier. However, the DNAT configuration is not limited, an arbitrary count of static and identity mappings can be listed under single label. 

### Configuration example  

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

The REST currently supports only retrieving of the existing configuration. The following command can be used to read the NAT global config via cURL:

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

[key-reference]: references.md
[grpc-tutorial]: ../tutorials/08_grpc-tutorial.md