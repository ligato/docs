# Punt plugin

The punt plugin provides several options for how to configure the VPP to allow a specific IP traffic to be punted to the host TCP/IP stack. The plugin supports **punt to the host** (either directly, or **via Unix domain socket**) and registration of **IP punt redirect** rules.

# Punt to host stack

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

### Configuration example

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

# IP redirect

Defined as the IP punt, IP redirect allows a traffic matching given IP protocol to be punted to the defined TX interface and next hop IP address. All those fields have to be defined in the northbound proto-modeled data. Optionally, the RX interface can be also defined as an input filter.  

IP redirect is defined as `IpRedirect` object in the generated proto model. L3 protocol is defined as `string` value (transformed to numeric in VPP API call). The table is the same as before.

If L3 protocol is set to `ALL`, the respective API is called for IPv4 and IPv6 separately.

### Configuration example

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

Current limitations for IP redirect:
* The VPP does not provide API calls to dump existing IP redirect entries. It may cause resync not to work properly.

### Known issues

* VPP issue: if the Unix domain socket path is defined in the startup config, the path has to exist, otherwise the VPP fails to start. The file itself can be created by the VPP.

[grpc-tutorial]: ../tutorials/08_grpc-tutorial.md
[key-reference]: references.md
[punt-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/punt/punt.proto