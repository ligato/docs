# ARP entries  

The L3 plugin defines descriptor managing the VPP implementation of the address resolution protocol. **ARP is a communication protocol for discovering the MAC address associated with a given IP address**. In terms of the plugin, the ARP entry is uniquely identified by the interface and IP address.

The northbound plugin API defines ARP in the following [model][arp-model]. Every ARP entry is defined with MAC address, IP address and an Interface. The two latter are a part of the key.

### Configuration example

**1. Using the key-value database** put the proto-modelled data with the correct key for ARP to the database ([key reference][key-reference]).

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

Prepare the ARP data:
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

Prepare the GRPC config data:
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

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial])):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

# Proxy ARP

Support for the VPP proxy ARP - a technique by which a proxy device an a given network answers the ARP queries for an IP address from a different network. Proxy ARP IP address ranges are defined via the respective binary API call. Besides this, desired interfaces have to be enabled for the proxy ARP feature.    

The Proxy ARP [model][l3-model] is located in the common L3 model proto file. The model defines a list of IP address ranges and another list of interfaces. Naturally, the interface has to exist in order to be set as enabled for ARP 
proxy. 

### Configuration example  

**1. Using the key-value database** put the proto-modelled data with the correct key for proxy ARP to the database ([key reference][key-reference]).

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

Prepare the Proxy-ARP data:
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

Prepare the GRPC config data:
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

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

# Routes 

The VPP routing table lists routes to particular network destinations. Routes are grouped in virtual routing and forwarding (VRF) tables, with default table of index 0. A route can be assigned to different VRF table directly in the modeled data.

The VPP route [model][route-model] defines all the base route fields (destination IP, next hop, interface) and other specifications like weight or preference. The model also distinguishes amidst several route types:

  **1. Intra-VRF route:** defines a route where the forwarding is done only in the specific VRF or according to the outgoing interface.
  **2. Inter-VRF route:** forwarding is being done by a lookup into different VRF routes. This kind of route does not  expect outgoing interface being defined. 
  **3. Drop route:** such a route drops network communication designed for IP address specified.

### Configuration example

**1. Using the key-value database** put the proto-modelled data with the correct key for routes to the database 
([key reference][key-reference]).

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

Prepare the VPP route data:
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

Prepare the GRPC config data:
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

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```
 
# IP scan-neighbor

The VPP IP scan-neighbor feature is used to enable or disable periodic IP neighbor scan. The [model][l3-model] allows to set various scan intervals, timers or delays and can be set to IPv4 mode, IPv6 mode or both of them.

### Configuration example  

**1. Using the key-value database** put the proto-modelled data with the correct key for IP scan neighbor to the database ([key reference][key-reference]).

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

Prepare the IP scan neighbor data:
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

Prepare the GRPC config data:
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

Update the data via the GRPC using the client of the `ConfigurationClient` type (read more about how to prepare the GRPC connection and about other CRUD methods in the [GRPC tutorial][grpc-tutorial]):
```go
import (
	"github.com/ligato/vpp-agent/api/configurator"
)

response, err := client.Update(context.Background(), &configurator.UpdateRequest{Update: config, FullResync: true})
```

[arp-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/arp.proto
[key-reference]: references.md
[grpc-tutorial]: ../tutorials/08_grpc-tutorial.md
[l3-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/l3.proto
[route-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/route.proto