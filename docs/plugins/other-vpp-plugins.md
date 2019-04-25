# Other VPP plugins

---

## Access Control Lists plugin

Access control lists (ACLs) provide a means to filter packets by allowing a user to permit or deny specific IP traffic at defined interfaces. Access lists filter network traffic by controlling whether packets are forwarded or blocked at the router’s interfaces based on the criteria you specified within the access list.

The VPP-Agent acl plugin uses binary API of the VPP access control list plugin. The version of the VPP ACL plugin is displayed at the Agent startup (currently at 1.3). Every ACL consists from match (rules packet have to fulfill in order to be received) and action (what is done with the matched traffic). The current implementation rules for packet are 'ALLOW', 'DENY' and 'REFLECT'. The ACL is defined in the VPP-Agent northbound API [model][acl-model].

The Agent defines every access list with unique name. The VPP generates index, but the association is handled fully by the Agent. Every ACL must consist from single action and single type of match (IP match or MAC-IP match). The IP match (called IP rule) can be specified for variety of protocols, each with its own parameters. For example, the IP rule for IP protocol can define source and destination network the packet must match in order to execute defined action. Other supported protocols are TCP, UDP and ICMP. Single ACL rule can define multiple protocol-based rules. The MAC-IP match (MACIP rule) defines IP address + mask and MAC address + mask as filtering rules. Remember, that the IP rules and MACIP rules cannot be combined in the single ACL.

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

## IPSec plugin

The IPSec plugin allows to configure **security policy databases** and **security associations** to the VPP, and also handles relations between the SPD and SA or between SPD and an assigned interface. Note that the IPSec tunnel interfaces are not a part of IPSec plugin (their configuration is handled in the [VPP interface plugin][interface-plugin-guide]).

### Security policy database

Security policy database (SPD) specifies policies that determine the disposition of all the inbound or outbound IP traffic from either the host or the security gateway. The SPD is bound to an SPD interface and contains a list of policy entries (security table). Every policy entry points to VPP security association. Security policy database is defined in the common IPSec [model][ipsec-model]. 

The SPD defines its own unique index within VPP. The user has an option to set its own index (it is not generated in the VPP, as for example for access lists) in `uint32` range. The index is a mandatory field in the model because it serves as a unique identifier also in the vpp-agent and is a part of the SPD key as well. The attention has to be paid 
defining an index in the model. Despite the field is a `string` format, it only accepts plain numbers. Attempt to set any non-numeric characters ends up with an error.     

Every policy entry has field `sa_index`. This is the security association reference in the security policy database. The field is mandatory, missing value causes an error during configuration.

The SPD defines two bindings: the security association (for every entry) and the interface. The interface binding is important since VPP needs to create an empty SPD first, which cannot be done without it. All the policy entries are configured after that, where the SA is expected to exist. Since every security policy database entry is configured independently, vpp-agent can configure IPSec SPD only partially (depending on available binding).

All the binding can be resolved by the vpp-agent. 

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for security policy databases to the database ([key reference][key-reference]).

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

The VPP security association (SA) is a set of IPSec specifications negotiated between devices establishing and IPSec relationship. The SA includes preferences for the authentication type, IPSec protocol or encryption used when the IPSec connection is established.

Security association is defined in the common IPSec [model][ipsec-model]. 

The SA uses the same indexing system as SPD. The index is a user-defined unique identifier of the security association in `uint32` range. Similarly to SPD, the SA index is also defined as `string` type field in the model but can be set only to numerical values. Attempt to set other values causes processing errors. The SA has no dependencies on other 
configuration types.

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for security associations to the database ([key reference][key-reference]).

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

## NAT plugin

Network address translation, or NAT is a method of remapping IP address space into another IP address space modifying address information in the packet header. The VPP-Agent Network address translation is control plane plugin for the VPP NAT implementation of NAT44. The NAT plugin is dependent on [interface plugin][interface-plugin-guide].

### NAT global config 

The global NAT configuration is a special case of data grouped under single key (it means no unique character is a part of the key, so there is always only one global NAT configuration). The data are used to enable NAT features (like forwarding), enable interfaces for NAT, define NAT IP addresses (address pools) or specify virtual reassembly. Interfaces marked to be enabled for NAT should be present in the VPP but if not, the Scheduler plugin caches the configuration for later use when the incriminated interface is available.

The global configuration is divided into several independent parts defining certain VPP NAT features:

  - **Forwarding** is a boolean field enabling or disabling forwarding.
  - **NAT interfaces** represent a list of interfaces which will be enabled for NAT. If the interface does not exist in   the VPP, it is cached and potentially configured later. Every interface is defined by its logical name and whether is   an inside or an outside interface. The output feature is also defined here.
  - **Address pools** is a list of IP addresses for given VRF table and with either enabled or disabled twice NAT.   Despite the name, only one address is defined in the single "pool" entry.
  - **Virtual reassembly** provides support for datagram fragmentation handling to allow correct recalculation of   higher-level checksums.

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

### DNAT44

Destination network address translation (DNAT) allows transparently changing the destination IP address of an packet and performing the inverse function for any replies. Any router situated between two endpoints can perform this transformation of the packet. In the VPP-Agent, the DNAT configuration is a list of static and/or identity mappings 
labelled under single key.

The DNAT44 consists from two main parts - static mappings and identity mappings. The static mapping can be load balanced - if more than one local IP address is defined for single static mapping, the load balancer is automatically allowed for that mapping. 
THe DNAT44 contains a unique label serving as an identifier. However, the DNAT configuration is not limited, an arbitrary count of static and identity mappings can be listed under single label. 

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

## Punt plugin

The punt plugin provides several options for how to configure the VPP to allow a specific IP traffic to be punted to the host TCP/IP stack. The plugin supports **punt to the host** (either directly, or **via Unix domain socket**) and registration of **IP punt redirect** rules.

### Punt to host stack

All the incoming traffic matching one of the VPP interface addresses, and also matching defined L3 protocol, L4 protocol, and port - and would be otherwise dropped - will be instead punted to the host. If a Unix domain socket path is defined (optional), traffic will be punted via socket. All the fields which serve as a traffic filter are mandatory.

The punt plugin defines the following [model][punt-model] which grants support for two main configuration items defined by different northbound keys. L3/L4 protocol in the key is defined as a `string` value, however, the value is transformed to numeric representation in the VPP binary API. The usage of L3 protocol `ALL` is exclusive for IP punt to 
host (without socket registration) in the VPP API. If used for the IP punt with socket registration, the vpp-agent calls the binary API twice with the same parameters for both, IPv4 and IPv6.

**Important note:** in order to configure a punt to host via Unix domain socket, a specific VPP startup-config is required. The attempt to set punt without it results in errors in VPP. Startup-config:

```
punt {
  socket /tmp/socket/punt
}
```

The path has to match with the one in the northbound data. 

**Configuration example**

**1. Using the key-value database** put the proto-modelled data with the correct key for punt to host to the database ([key reference][key-reference]). 

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

Defined as the IP punt, IP redirect allows a traffic matching given IP protocol to be punted to the defined TX interface and next hop IP address. All those fields have to be defined in the northbound proto-modeled data. Optionally, the RX interface can be also defined as an input filter.  

IP redirect is defined as `IpRedirect` object in the generated proto model. L3 protocol is defined as `string` value (transformed to numeric in VPP API call). The table is the same as before.

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

Punt currently does not support REST.

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
* The UDP configuration cannot be shown (or even configured) via the VPP cli.
* The VPP does not provide API to dump configuration, which takes the vpp-agent the opportunity to read existing entries and may case certain issues with resync.
* Although the vpp-agent supports the TCP protocol as the L4 protocol to filter incoming traffic, the current VPP version don't.
* Configured punt to host entry cannot be removed since the VPP does not support this option. The attempt to do so exits with an error.

Current limitations for a punt to host via unix domain socket:
* The configuration cannot be shown (or even configured) in the VPP cli.
* The vpp-agent cannot read registered entries since the VPP does not provide an API to do so.
* The VPP startup config punt section requires unix domain socket path defined. The VPP limitation is that only one path can be defined at the same time.

Current limitations for IP redirect:
* The VPP does not provide API calls to dump existing IP redirect entries. It may cause resync not to work properly.

### Known issues

* VPP issue: if the Unix domain socket path is defined in the startup config, the path has to exist, otherwise the VPP fails to start. The file itself can be created by the VPP.

## Telemetry

The `telemetry` plugin is a core Agent Plugin for exporting telemetry statistics from the VPP to the Prometheus. Statistics are published via registry path `/vpp` on port `9191` and updated every 30 seconds.

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

The telemetry plugin configuration file allows to change polling interval, or turn the polling off. The `polling-interval` is time in nanoseconds between reads from the VPP. Parameter `disabled` can be set to `true` in order to disable the telemetry plugin.

[acl-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/acl/acl.proto
[grpc-tutorial]: ../tutorials/08_grpc-tutorial.md
[ipsec-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/ipsec/ipsec.proto
[key-reference]: ../features/references.md
[punt-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/punt/punt.proto
