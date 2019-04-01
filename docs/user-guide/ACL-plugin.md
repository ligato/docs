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

