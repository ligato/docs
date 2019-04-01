# Access Control Lists plugin

**Written for: v2.0-vpp18.10**

Access control lists (ACLs) provide a means to filter packets by allowing a user to permit or deny specific IP traffic at defined interfaces. Access lists filter network traffic by controlling whether packets are forwarded or blocked at the router’s interfaces based on the criteria you specified within the access list.

- [Overview](#overview)
- [Model](#model)
- [Configuration](#conf)

## <a name="overview">Overview</a>

The VPP-Agent acl plugin uses binary API of the VPP access control list plugin. The version of the VPP ACL plugin is displayed at the Agent startup (currently at 1.3). Every ACL consists from match (rules packet have to fulfill in order to be received) and action (what is done with the matched traffic). The current implementation rules for packet are 'ALLOW', 'DENY' and 'REFLECT'.  

## <a name="model">Model</a>

The ACL is defined in the VPP-Agent northbound API [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/acl/acl.proto). The generated object is defined as the `ACL`.

The Agent defines every access list with unique name. The VPP generates index, but the association is handled fully by the Agent. Every ACL must consist from single action and single type of match (IP match or MAC-IP match). 
The IP match (called IP rule) can be specified for variety of protocols, each with its own parameters. For example, the IP rule for IP protocol can define source and destination network the packet must match in order to execute defined action. Other supported protocols are TCP, UDP and ICMP. Single ACL rule can define multiple protocol-based rules.
The MAC-IP match (MACIP rule) defines IP address + mask and MAC address + mask as filtering rules. 
Remember, that the IP rules and MACIP rules cannot be combined in the single ACL.

## <a name="conf">Configuration</a>

How to configure the access list:

**1. Using Key-value database:** use proto-modeled data with the correct ACL key. 

Key:
```
/vnf-agent/vpp1/config/vpp/acls/v2/acl/<name>
```

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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

**The VPP CLI access list commands:**

The VPP does not support any CLI commands related to access list. In order to retrieve ACL configuration, use `vat#` console and direct binary API call `acl_dump`.

# IPSec plugin

**Written for: v2.0-vpp18.10**

The IPSec plugin allows to configure **security policy databases** and **security associations** to the VPP, and also handles relations between the SPD and SA or between SPD and an assigned interface. Note that the IPSec tunnel interfaces are not a part of IPSec plugin (their configuration is handled in [VPP interface plugin](used/VPP-Interface-plugin.md)).

- [Security policy database](#spd)
  * [Model](#spd-model)
  * [Configuration](#spd-config)
- [Security association](#sa)
  * [Model](#sa-model)
  * [Configuration](#sa-config)
  
## <a name="spd">Security policy database</a>

Security policy database (SPD) specifies policies that determine the disposition of all the inbound or outbound IP traffic from either the host or the security gateway. The SPD is bound to an SPD interface and contains a list of policy entries (security table). Every policy entry points to VPP security association.

### <a name="spd-model">Model</a>

Security policy database is defined in the common IPSec [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/ipsec/ipsec.proto). The generated object is defined as `SecurityPolicyDatabase`. 

Key format:
```
config/vpp/v2/ipsec/spd/<index>
```

The SPD defines its own unique index within VPP. The user has an option to set its own index (it is not generated in the VPP, as for example for access lists) in `uint32` range. The index is a mandatory field in the model because it serves as a unique identifier also in the vpp-agent and is a part of the SPD key as well. The attention has to be paid defining an index in the model. Despite the field is a `string` format, it only accepts plain numbers. Attempt to set any non-numeric characters ends up with an error.     

Every policy entry has field `sa_index`. This is the security association reference in the security policy database. The field is mandatory, missing value causes an error during configuration.

The SPD defines two bindings: the security association (for every entry) and the interface. The interface binding is important since VPP needs to create an empty SPD first, which cannot be done without it. All the policy entries are configured after that, where the SA is expected to exist.
Since every security policy database entry is configured independently, vpp-agent can configure IPSec SPD only partially (depending on available binding).

All the binding can be resolved by the vpp-agent. 

### <a name="spd-config">Configuration</a>

How to configure the security policy database:

**1. Using Key-value database:** put proto-modeled data with the correct key for SPD. 

Key:
```
/vnf-agent/vpp1/config/vpp/ipsec/v2/spd/<index>
```

Example value:
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

IPSec currently does not support REST.

**3. Using GRPC:**

* Prepare the IPSec SPD data:
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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

**The VPP CLI IPSec SPD commands:**

The VPP cli has a command to show the SPD IPSec configuration: `sh ipsec`

## <a name="sa">Security associations</a>

The VPP security association (SA) is a set of IPSec specifications negotiated between devices establishing and IPSec relationship. The SA includes preferences for the authentication type, IPSec protocol or encryption used when the IPSec connection is established.

### <a name="sa-model">Model</a>

Security association is defined in the common IPSec [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/ipsec/ipsec.proto). The generated object is defined as `SecurityAssociation`. 

Key format:
```
/vnf-agent/vpp1/config/vpp/ipsec/v2/sa/<index>
```

The SA uses the same indexing system as SPD. The index is a user-defined unique identifier of the security association in `uint32` range. Similarly to SPD, the SA index is also defined as `string` type field in the model but can be set only to numerical values. Attempt to set other values causes processing errors.

The SA has no dependencies on other configuration types.

### <a name="sa-config">Configuration</a>

How to configure the security association:

**1. Using Key-value database:** use proto-modeled data with the correct security association key. 

Key:
```
/vnf-agent/vpp1/config/vpp/ipsec/v2/sa/<sa-index>
```  

Example value:
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

IPSec currently does not support REST.

**3. Using GRPC:**

* Prepare the IPSec SA data:
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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

**The VPP CLI IPSec SA commands:**

Show the IPSec configuration with the VPP cli command: `sh ipsec`

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

# Punt plugin

**Written for: v2.0-vpp18.10**

The punt plugin provides several options for how to configure the VPP to allow a specific IP traffic to be punted to the host TCP/IP stack. The plugin supports **punt to the host** (either directly, or **via Unix domain socket**) and registration of **IP punt redirect** rules.

- [Punt to host stack](#pths)
  * [Model](#pths-model)
  * [Requirements](#pths-req)
  * [Configuration](#pths-config)
  * [Limitations](#pths-limit)
  * [Known issues](#pths-issues)
- [IP redirect](#ipr)
  * [Model](#ipr-model)
  * [Configuration](#ipr-config)
  * [Limitations](#ipr-limit)

## <a name="pths">Punt to host stack</a>

All the incoming traffic matching one of the VPP interface addresses, and also matching defined L3 protocol, L4 protocol, and port - and would be otherwise dropped - will be instead punted to the host. If a Unix domain socket path is defined (optional), traffic will be punted via socket. All the fields which serve as a traffic filter are mandatory.

### <a name="pths-model">Model</a>

The punt plugin defines the following [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/punt/punt.proto) which grants support for two main configuration items defined by different northbound keys.

The punt to host is defined as `ToHost` object in the generated proto model. 

L3/L4 protocol in the key is defined as a `string` value, however, the value is transformed to numeric representation in the VPP binary API.

The usage of L3 protocol `ALL` is exclusive for IP punt to host (without socket registration) in the VPP API. If used for the IP punt with socket registration, the vpp-agent calls the binary API twice with the same parameters for both, IPv4 and IPv6.

### <a name="pths-req">Requirements</a>

**Important note:** in order to configure a punt to host via Unix domain socket, a specific VPP startup-config is required. The attempt to set punt without it results in errors in VPP. Startup-config:

```
punt {
  socket /tmp/socket/punt
}
```

The path has to match with the one in the northbound data. 

### <a name="pths-config">Configuration</a>

How to configure punt to host:

**1. Using Key-value database:** put proto-modeled data to the database under the correct key for punt to host configuration.

Key (three fields need to be filled: L3 protocol, L4 protocol and port):
```
/vnf-agent/vpp1/config/vpp/v2/tohost/l3/<l3_protocol>/l4/<l4_protocol>/port/<port>
```  

Example value:
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

Punt currently does not support REST.

**3. Using GRPC:**

* Prepare the Punt data:
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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

### <a name="pths-limit">Limitations</a>

Current limitations for a punt to host:
* The UDP configuration cannot be shown (or even configured) via the VPP cli.
* The VPP does not provide API to dump configuration, which takes the vpp-agent the opportunity to read existing entries and may case certain issues with resync.
* Although the vpp-agent supports the TCP protocol as the L4 protocol to filter incoming traffic, the current VPP version don't.
* Configured punt to host entry cannot be removed since the VPP does not support this option. The attempt to do so exits with an error.

Current limitations for a punt to host via unix domain socket:
* The configuration cannot be shown (or even configured) in the VPP cli.
* The vpp-agent cannot read registered entries since the VPP does not provide an API to do so.
* The VPP startup config punt section requires unix domain socket path defined. The VPP limitation is that only one path can be defined at the same time.

### <a name="pths-issues">Known issues</a>

* VPP issue: if the Unix domain socket path is defined in the startup config, the path has to exist, otherwise the VPP fails to start. The file itself can be created by the VPP.

## <a name="ipr">IP redirect</a>

Defined as the IP punt, IP redirect allows a traffic matching given IP protocol to be punted to the defined TX interface and next hop IP address. All those fields have to be defined in the northbound proto-modeled data. Optionally, the RX interface can be also defined as an input filter.  

### <a name="ipr-model">Model</a>

IP redirect is defined as `IpRedirect` object in the generated proto model. L3 protocol is defined as `string` value (transformed to numeric in VPP API call). The table is the same as before.

If L3 protocol is set to `ALL`, the respective API is called for IPv4 and IPv6 separately.

### <a name="ipr-config">Configuration</a>

How to configure IP redirect:

**1. Using the key-value database:** put proto-modeled data to the database under the correct key for IP redirect configuration.

Key: 
```
/vnf-agent/vpp1/config/vpp/v2/ipredirect/l3/<l3_protocol>/tx/<tx_interface>
```

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

Punt currently does not support REST.

**3. Using GRPC:**

* Prepare the IP redirect data:
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

* Prepare the GRPC config data:
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

* Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

The VPP cli command (for configuration verification) is `show ip punt redirect `.

### <a name="ipr-limit">Limitations</a>

* The VPP does not provide API calls to dump existing IP redirect entries. It may cause resync not to work properly.