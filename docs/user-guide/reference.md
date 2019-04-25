# Reference

This page is an overview of all keys supported for the VPP-Agent

---

Parts of the key in `<>` must be set with the same value as in a model. The microservice label is set to `vpp1` in every mentioned key, but if different value is used, it needs to be replaced in the key as well.

Link in key title redirects to the associated proto definition.

### VPP keys

**[Access control lists:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/acl/acl.proto)**

Key:
```
/vnf-agent/vpp1/config/vpp/acls/v2/acl/<name>
```

**[VPP interfaces:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/interface.proto)**

Key:
```
/vnf-agent/vpp1/config/vpp/v2/interfaces/<name>
```

**[VPP interface status (read only):](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/state.proto)**

Key:
```
/vnf-agent/vpp1/vpp/status/v2/interface/<name>
```

**[Bridge domains:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto)**

Key:
```
/vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/<name>
```

**[Forwarding table (FIBs):](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/fib.proto)**

Key:
```
/vnf-agent/vpp1/config/vpp/l2/v2/fib/<bridge_domain>/mac/<phys_address>
```

**[Cross connects:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/xconnect.proto)**

Key:
```
/vnf-agent/vpp1/config/vpp/l2/v2/xconnect/<receive-interface>
```

**[VPP ARP entries:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/arp.proto)**

Key:
```
/vnf-agent/vpp1/config/vpp/v2/arp/<interface>/<ip_address>
```

**[VPP proxy ARP entries:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/l3.proto)**

Key (single for the entire proxy-arp configuration):
```
/vnf-agent/vpp1/config/vpp/v2/proxyarp-global/settings
```

**[VPP routes:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/route.proto)**

Key (dst_network is with mask in format <ip>/<mask>, gw can be empty if not set):
```
/vnf-agent/vpp1/config/vpp/v2/route/vrf/<vrf_id>/dst/<dst_network>/gw/<next_hop_addr>
```

**[IP scan neighbor:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/l3.proto)**

Key (single for the entire ip-scan-neighbor configuration):
```
/vnf-agent/vpp1/config/vpp/v2/ipscanneigh-global/settings
```

**[NAT global settings:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/nat/nat.proto)**

Key (single for the NAT global configuration):
```
/vnf-agent/vpp1/config/vpp/nat/v2/nat44-global/settings
```

**[DNAT:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/nat/nat.proto)**

Key:
```
/vnf-agent/vpp1/config/vpp/nat/v2/dnat44/<label>
```

**[IPSec security policy database:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/ipsec/ipsec.proto)**

Key:
```
/vnf-agent/vpp1/config/vpp/ipsec/v2/spd/<index>
```

**[IPSec security association:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/ipsec/ipsec.proto)**

Key:
```
/vnf-agent/vpp1/config/vpp/ipsec/v2/sa/<index>
```

**[Punt to host stack:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/punt/punt.proto)**

Key (l3_protocol and l4_protocol are enum values, not indexes (like IPv4, IPv6, TCP, etc.)):
```
/vnf-agent/vpp1/config/vpp/v2/tohost/l3/<l3_protocol>/l4/<l4_protocol>/port/<port>
```

**[IP redirect:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/punt/punt.proto)**

Key (l3_protocol is an enum value, not index (like IPv4, IPv6 etc.)):
```
/vnf-agent/vpp1/config/vpp/v2/ipredirect/l3/<l3_protocol>/tx/<tx_interface>
```

**[STN:](https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/stn/stn.proto)**

Key:
```
/vnf-agent/vpp1/config/vpp/stn/v2/rule/<interface>/ip/<ip_address>
```

### Linux keys

**[Linux interfaces:](https://github.com/ligato/vpp-agent/blob/master/api/models/linux/interfaces/interface.proto)**

Key:
```
/vnf-agent/vpp1/config/linux/interfaces/v2/interface/<name>
```

**[Linux ARP entries:](https://github.com/ligato/vpp-agent/blob/master/api/models/linux/l3/arp.proto)**

Key:
```
/vnf-agent/vpp1/config/linux/l3/v2/arp/<interface>/<ip_address>
```

**[Linux routes:](https://github.com/ligato/vpp-agent/blob/master/api/models/linux/l3/route.proto)**

Key (dst_network is with mask in format <ip>/<mask>):
```
/vnf-agent/vpp1/config/linux/l3/v2/route/<dst_network>/<outgoing_interface>
```

