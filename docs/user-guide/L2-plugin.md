# L2 plugin

**Written for: v2.0-vpp18.10**

The VPP L2 plugin is a base plugin which can be used to configure link-layer related configuration items to the VPP, notably **bridge domains**, **forwarding tables** (FIBs) and **VPP cross connects**. The L2 plugin is strongly dependent on [interface plugin](VPP-Interface-plugin.md) since every configuration item has some kind of dependency on it. 

- [Bridge domains](#bds)
  * [Model](#bdmodel)
  * [Configuration](#bdconf)
- [Forwarding tables](#fts)
  * [Model](#ftmodel)
  * [Configuration](#ftconf)
- [Cross connects](#cc)
  * [Model](#ccmodel)
  *[Configuration](#ccconf)
  
## <a name="bds">Bridge domains</a>  

The L2 bridge domain is a set of interfaces which share which share the same flooding or broadcasting characteristics. Every bridge domain contains several attributes (MAC learning, unicast forwarding, flooding, ARP termination table) which can be enabled or disabled. The bridge domain is identified by unique ID, which is managed by the plugin and cannot be defined from outside.   

### <a name="bdmodel">Model</a>  

The L2 plugin defines individual proto definition for every configuration item, so the bridge domain is also defined by its own [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto). The definition consists of common bridge domain fields (forwarding, learning...), list of assigned interfaces and ARP termination table. 
Bridge domain interfaces are only referenced - the configuration of the interface itself and its configuration (IP address, ...) lays on the interface plugin. Every referenced interface also contains bridge domain-specific fields (BVI and split horizon group). Note that only one BVI interface is allowed per bridge domain.

The bridge domain does not have any mandatory fields except the logical name. In that case, an empty bridge domain with no interface or ARP entries is created.

In the generated proto model, the bridge domain is referred to the `BridgeDomain` object. 

The bridge domain logical name is for the plugin-use and also as a reference in other configuration types dependent on the specific bridge domain. The bridge domain index can be obtained via retrieve, but the value is only informative since it cannot be changed, set or referenced.

### <a name="bdconf">Configuration</a> 

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

## <a name="fts">Forwarding tables</a>  

An L2 forwarding information base (FIB) (also known as a forwarding table) can be used in network bridging or routing to find the output interface to which the incoming packet should be forwarded. FIB entries are created also by the VPP itself (like for interfaces within the bridge domain with enabled MAC learning). The Agent allows to configuring static FIB entries for a combination of interface and bridge domain.

### <a name="ftmodel">Model</a>  

A FIB entry is defined in the following [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/fib.proto). To successfully configure FIB entry, a MAC address, interface and bridge domain must be provided. Also, the condition that the interface is a part of the bridge domain has to be fulfilled. The interface and the bridge domain are referenced with logical names.  
Note that the FIB entry will not appear in the VPP until all conditions are met.
The FIB entry is referred to the `FIBEntry` object in the generated proto. 

### <a name="ftconf">Configuration</a> 

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

## <a name="cc">Cross connects</a>  

The Agent supports L2 cross connect feature, which sets interface pair to cross connect mode. All packets received on the first interface are transmitted to the second interface. This mode is not bidirectional by default, both interfaces have to be set.

### <a name="ccmodel">Model</a>  

The cross-connect mode uses a very simple [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/xconnect.proto) which defines references to transmit and receive interface. Both interfaces are mandatory fields and must exist in order to set the mode successfully. If one or both of them are missing, the configuration is postponed. 
The cross-connect is referred as `XConnectPair` in generated code. 

### <a name="ccconf">Configuration</a>

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

 
 