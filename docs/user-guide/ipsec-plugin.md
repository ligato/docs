# Security policy database

Security policy database (SPD) specifies policies that determine the disposition of all the inbound or outbound IP traffic from either the host or the security gateway. The SPD is bound to an SPD interface and contains a list of policy entries (security table). Every policy entry points to VPP security association. Security policy database is defined in the common IPSec [model][ipsec-model]. 

The SPD defines its own unique index within VPP. The user has an option to set its own index (it is not generated in the VPP, as for example for access lists) in `uint32` range. The index is a mandatory field in the model because it serves as a unique identifier also in the vpp-agent and is a part of the SPD key as well. The attention has to be paid 
defining an index in the model. Despite the field is a `string` format, it only accepts plain numbers. Attempt to set any non-numeric characters ends up with an error.     

Every policy entry has field `sa_index`. This is the security association reference in the security policy database. The field is mandatory, missing value causes an error during configuration.

The SPD defines two bindings: the security association (for every entry) and the interface. The interface binding is important since VPP needs to create an empty SPD first, which cannot be done without it. All the policy entries are configured after that, where the SA is expected to exist. Since every security policy database entry is configured independently, vpp-agent can configure IPSec SPD only partially (depending on available binding).

All the binding can be resolved by the vpp-agent. 

### Configuration example

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

# Security associations

The VPP security association (SA) is a set of IPSec specifications negotiated between devices establishing and IPSec relationship. The SA includes preferences for the authentication type, IPSec protocol or encryption used when the IPSec connection is established.

Security association is defined in the common IPSec [model][ipsec-model]. 

The SA uses the same indexing system as SPD. The index is a user-defined unique identifier of the security association in `uint32` range. Similarly to SPD, the SA index is also defined as `string` type field in the model but can be set only to numerical values. Attempt to set other values causes processing errors. The SA has no dependencies on other 
configuration types.

### Configuration example

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

[ipsec-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/ipsec/ipsec.proto
[key-reference]: references.md
[grpc-tutorial]: ../tutorials/08_grpc-tutorial.md