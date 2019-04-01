# L3 plugin

**Written for: v2.0-vpp18.10**

The VPP L3 plugin is capable of configuring **ARP** entries (including **proxy ARP**), **VPP routes** and **IP neighbor** feature. The L3 plugin is dependent on [interface plugin](VPP-Interface-plugin.md) in many aspects since several configuration items require the interface to be already present.

- [ARP entries](#arps)
  * [Model](#arpmodel)
  * [Configuration](#arpconf)
- [Proxy ARP](#parp)
  * [Model](#parpmodel)
  * [Configuration](#parpconf)
- [Routes](#routes)
  * [Model](#routesmodel)
  * [Configuration](#routesconf)
- [IP neighbor](#ipneigh)
  * [Model](#ipneighmodel)
  * [Configuration](#ipneighconf)
  
## <a name="arps">ARP entries</a>  

The L3 plugin defines descriptor managing the VPP implementation of the address resolution protocol. ARP is a communication protocol for discovering the MAC address associated with a given IP address. In terms of the plugin, the ARP entry is uniquely identified by the interface and IP address.

## <a name="arpmodel">Model</a>  

The northbound plugin API defines ARP in the following [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/arp.proto). Every ARP entry is defined with MAC address, IP address,  and an Interface. The two latter are a part of the key.

In the generated proto model, the ARP is referred to the `ARPEntry` object.

## <a name="arpconf">Configuration</a>  

How to configure ARP table entry:

**1. Using a key-value database** put the proto-modeled data with the correct key for the ARP to the database. 

Key:
```
/vnf-agent/vpp1/config/vpp/v2/arp/<interface>/<ip_address>
```

Example value:
```json
{  
    "interface":"if1",
    "ip_address":"192.168.10.21",
    "phys_address":"59:6C:45:59:8E:BD",
    "static":true
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/arp/tap1/192.168.10.21 '{"interface":"tap1","ip_address":"192.168.10.21","phys_address":"59:6C:45:59:8E:BD","static":true}'
```

To remove configuration:

```bash
etcdctl del /vnf-agent/vpp1/config/vpp/v2/arp/tap1/192.168.10.21
```

**2. Using REST:**

The REST currently supports only retrieving of the existing configuration. The following command can be used to read all ARP entries via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/arps
```

**3. Using GRPC:**

* Prepare the ARP data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l3"
)

arp := &l3.ARPEntry{
		Interface: "if1",
		IpAddress: "10.20.30.40/24",
		PhysAddress: "55:65:75:a7:35:45",
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
			Arps: []*l3.ARPEntry {    
			    arp,
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

**The VPP CLI ARP commands:**

The VPP cli has following CLI commands to verify configuration:
 - show list of all configured ARPs: `show ip arp`

## <a name="parp">Proxy ARP</a>  

Support for the VPP proxy ARP - a technique by which a proxy device an a given network answers the ARP queries for an IP address from a different network. Proxy ARP IP address ranges are defined via the respective binary API call. Besides this, desired interfaces have to be enabled for the proxy ARP feature.   

## <a name="parpmodel">Model</a>  

The Proxy ARP [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/l3.proto) is located in the common L3 model proto file. The model defines a list of IP address ranges and another list of interfaces. Naturally, the interface has to exist in order to be set as enabled for ARP proxy. 

The proxy ARP type is referred to `ProxyARP`.

## <a name="parpconf">Configuration</a>  

How to configure the proxy ARP:

**1. Using a key-value database** put proto-modeled data under correct proxy ARP key to the KVDB. Note that only one key is defined for proxy ARP (there is no unique field as name, so all IP ranges and interfaces are grouped):

Key:
```
/vnf-agent/vpp1/config/vpp/v2/proxyarp-global/settings
```

Example value:
```json
{  
    "interfaces":[  
        {  
            "name":"tap1"
        },
        {  
            "name":"tap2"
        }
    ],
    "ranges":[  
        {  
            "first_ip_addr":"10.0.0.1",
            "last_ip_addr":"10.0.0.3"
        }
    ]
}
```

Use `etcdctl` to put compacted key-value entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/proxyarp-global/settings '{"interfaces":[{"name":"tap1"},{"name":"tap2"}],"ranges":[{"first_ip_addr":"10.0.0.1","last_ip_addr":"10.0.0.3"}]}'
```

**2. Using REST:**

The REST currently supports only retrieving of the existing configuration. The Proxy ARP uses two cURL commands to obtain interfaces and ranges: 

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/proxyarp/interfaces
curl -X GET http://localhost:9191/dump/vpp/v2/proxyarp/ranges

```

**3. Using GRPC:**

* Prepare the Proxy-ARP data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l3"
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

proxyArp := &l3.ProxyARP{
		Interfaces: []*l3.ProxyARP_Interface{
			{
				Name: "if1",
			},
		},
		Ranges: []*l3.ProxyARP_Range{
			{
				FirstIpAddr: "10.0.0.1",
				LastIpAddr: "10.0.0.255",
			},
		},
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

**The VPP CLI Proxy ARP commands:**

To verify ARP configuration, use the same call as for the ordinary ARP:
 - show list of all configured proxy ARPs: `show ip arp`

## <a name="routes">Routes</a>  

The VPP routing table lists routes to particular network destinations. Routes are grouped in virtual routing and forwarding (VRF) tables, with default table of index 0. A route can be assigned to different VRF table directly in the modeled data. 

## <a name="routesmodel">Model</a>  

The VPP route [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/route.proto) defines all the base route fields (destination IP, next hop, interface) and other specifications like weight or preference. The model also distinguishes amidst several route types:

**1. Intra-VRF route:** defines a route where the forwarding is done only in the specific VRF or according to the outgoing interface.
**2. Inter-VRF route:** forwarding is being done by a lookup into different VRF routes. This kind of route does not expect outgoing interface being defined. 
**3. Drop route:** such a route drops network communication designed for IP address specified.

## <a name="routesconf">Configuration</a> 

How to configure a VPP route:

**1. Using the key-value database** put the proto-modeled data with the correct key for the route to the database. 

Key (destination address is in format <ip>/<mask>, gateway is only IP address and does not need to be specified if not used in data. Also make sure the VRF matches the one in the model data - otherwise the value from key is being used):
```
/vnf-agent/vpp1/config/vpp/v2/route/vrf/<vrf_id>/dst/<dst_network>/gw/<next_hop_addr>
```

Example data (intra-VRF route):
```json
{  
    "type": "INTRA_VRF",
    "dst_network":"10.1.1.3/32",
    "next_hop_addr":"192.168.1.13",
    "outgoing_interface":"tap1",
    "weight":6
}
```

Use `etcdctl` to put compacted intra-VRF route:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/route/vrf/0/dst/10.1.1.3/32/gw/192.168.1.13 '{"type":"INTRA_VRF","dst_network":"10.1.1.3/32","next_hop_addr":"192.168.1.13","outgoing_interface":"tap1","weight":6}'
```

Another example (inter-VRF route)
```json
{  
    "type":"INTER_VRF",
    "dst_network":"1.2.3.4/32",
    "via_vrf_id":1
}
```

Use `etcdctl` to put compacted inter-VRF route:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/route/vrf/0/dst/1.2.3.4/32/gw '{"type":"INTER_VRF","dst_network":"1.2.3.4/32","via_vrf_id":1}'
```

**2. Using REST:**

The REST currently supports only retrieving of the existing configuration. The following command can be used to read all routes via cURL:

```bash
curl -X GET http://localhost:9191/dump/vpp/v2/routes
```

**3. Using GRPC:**

* Prepare the VPP route data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l3"
)

route := &l3.Route{
		VrfId:             0,
		DstNetwork:        "10.1.1.3/32",
		NextHopAddr:       "192.168.1.13",
		Weight:            60,
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
			Routes: []*l3.Route {}
				route,
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

**The VPP CLI route commands:**

The route configuration can be verified with:
 - show all rotes: `show ip fib`
 - show routes from the given VRF table: `show ip fib table <table-ID>` 
 
## <a name="ipneigh">IP neighbor</a> 

The VPP IP scan-neighbor feature is used to enable or disable periodic IP neighbor scan.
 
## <a name="ipneighmodel">Model</a>

The [model](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/l3.proto) allows to set various scan intervals, timers or delays and can be set to IPv4 mode, IPv6 mode or both of them.

## <a name="ipneighhconf">Configuration</a>  

How to set IP scan neighbor feature:

**1. Using a key-value database** put the proto-modeled data with the correct key to the database. The key is global (as for proxy ARP) - it means only one can be present in the KVDB at a time.

Key:
```
/vnf-agent/vpp1/config/vpp/v2/ipscanneigh-global/settings
``` 

Example data:
```json
{  
    "mode":"BOTH",
    "scan_interval":11,
    "max_proc_time":36,
    "max_update":5,
    "scan_int_delay":16,
    "stale_threshold":26
}
```

Use `etcdctl` to put compacted data:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/ipscanneigh-global/settings '{"mode":"BOTH","scan_interval":11,"max_proc_time":36,"max_update":5,"scan_int_delay":16,"stale_threshold":26}'
```

**2. Using REST:**

IP scan neighbor configuration type does not support REST.

**3. Using GRPC:**

* Prepare the IP scan neighbor data:
```go
import (
	"github.com/ligato/vpp-agent/api/models/vpp"
	"github.com/ligato/vpp-agent/api/models/vpp/l3"
)

ipScanNeigh := &l3.IPScanNeighbor{
		Mode:           l3.IPScanNeighbor_BOTH,
		ScanInterval:   11,
		MaxProcTime:    36,
		MaxUpdate:      5,
		ScanIntDelay:   16,
		StaleThreshold: 26,
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
			IpscanNeighbor: *l3.IPScanNeighbor {  
				ipScanNeigh,
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

**The VPP CLI IP scan neighbor commands:**

The IP scan neighbor configuration can be verified with:
 - show IP scan neighbor: `show ip scan-neighbor`