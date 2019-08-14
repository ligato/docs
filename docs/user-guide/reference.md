# Reference

This page is an overview of all keys supported for the vpp-agent.

---

Parts of the key in `<>` must be set with the same value as in a model. The microservice label is set to `vpp1` in every mentioned key, but if different value is used, it needs to be replaced in the key as well.

Link in key title redirects to the associated proto definition.

### VPP keys

**[ACL-based forwarding:][abf-proto]**

Key:
```text
/vnf-agent/vpp1/config/vpp/abfs/v2/abf/<index>
```

**[Access control lists:][acl-proto]**

Key:
```text
/vnf-agent/vpp1/config/vpp/acls/v2/acl/<name>
```

**[VPP interfaces:][interface-proto]**

Key:
```text
/vnf-agent/vpp1/config/vpp/v2/interfaces/<name>
```

**[VPP interface status (read only):][interface-state-proto]**

Key:
```text
/vnf-agent/vpp1/vpp/status/v2/interface/<name>
```

**[Bridge domains:][bridge-domain-proto]**

Key:
```text
/vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/<name>
```

**[Forwarding table (FIBs):][fib-proto]**

Key:
```text
/vnf-agent/vpp1/config/vpp/l2/v2/fib/<bridge_domain>/mac/<phys_address>
```

**[Cross connects:][xconnect-proto]**

Key:
```text
/vnf-agent/vpp1/config/vpp/l2/v2/xconnect/<receive-interface>
```

**[VPP ARP entries:][arp-proto]**

Key:
```text
/vnf-agent/vpp1/config/vpp/v2/arp/<interface>/<ip_address>
```

**[VPP proxy ARP entries:][l3-proto]**

Key (single for the entire proxy-arp configuration):
```text
/vnf-agent/vpp1/config/vpp/v2/proxyarp-global/settings
```

**[VPP routes:][route-proto]**

Key (dst_network is with mask in format <ip>/<mask>, gw can be empty if not set):
```text
/vnf-agent/vpp1/config/vpp/v2/route/vrf/<vrf_id>/dst/<dst_network>/gw/<next_hop_addr>
```

**[IP scan neighbor:][l3-proto]**

Key (single for the entire ip-scan-neighbor configuration):
```text
/vnf-agent/vpp1/config/vpp/v2/ipscanneigh-global/settings
```

**[Virtual routing and forwarding (VRF):][vrf-proto]**

Key (protocol is either 'IPV4' or 'IPV6'):
```text
/vnf-agent/vpp1/config/vpp/v2/vrf-table/id/<id>/protocol/<protocol>
```

**[NAT global settings:][nat-proto]**

Key (single for the NAT global configuration):
```text
/vnf-agent/vpp1/config/vpp/nat/v2/nat44-global/settings
```

**[DNAT:][nat-proto]**

Key:
```text
/vnf-agent/vpp1/config/vpp/nat/v2/dnat44/<label>
```

**[IPSec security policy database:][ipsec-proto]**

Key:
```text
/vnf-agent/vpp1/config/vpp/ipsec/v2/spd/<index>
```

**[IPSec security association:][ipsec-proto]**

Key:
```text
/vnf-agent/vpp1/config/vpp/ipsec/v2/sa/<index>
```

**[Punt to host stack:][punt-proto]**

Key (l3_protocol and l4_protocol are enum values, not indexes (like IPv4, IPv6, TCP, etc.)):
```text
/vnf-agent/vpp1/config/vpp/v2/tohost/l3/<l3_protocol>/l4/<l4_protocol>/port/<port>
```

**[IP redirect:][punt-proto]**

Key (l3_protocol is an enum value, not index (like IPv4, IPv6 etc.)):
```text
/vnf-agent/vpp1/config/vpp/v2/ipredirect/l3/<l3_protocol>/tx/<tx_interface>
```

**[STN:][stn-proto]**

Key:
```text
/vnf-agent/vpp1/config/vpp/stn/v2/rule/<interface>/ip/<ip_address>
```

### Linux keys

**[Linux interfaces:][interface-proto-linux]:**

Key:
```text
/vnf-agent/vpp1/config/linux/interfaces/v2/interface/<name>
```

**[Linux ARP entries:][arp-proto-linux]**

Key:
```text
/vnf-agent/vpp1/config/linux/l3/v2/arp/<interface>/<ip_address>
```

**[Linux IP tables:][iptables-proto-linux]**

Key:
```text
/vnf-agent/vpp1/config/linux/iptables/v2/rulechain/<name>
```

**[Linux routes:][route-proto-linux]**

Key (dst_network is with mask in format `<ip>/<mask>`):
```text
/vnf-agent/vpp1/config/linux/l3/v2/route/<dst_network>/<outgoing_interface>
```

[abf-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/abf/abf.proto
[acl-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/acl/acl.proto
[arp-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/arp.proto
[arp-proto-linux]: https://github.com/ligato/vpp-agent/blob/master/api/models/linux/l3/arp.proto
[bridge-domain-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto
[fib-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/fib.proto
[interface-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/interface.proto
[interface-proto-linux]: https://github.com/ligato/vpp-agent/blob/master/api/models/linux/interfaces/interface.proto
[interface-state-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/interfaces/state.proto
[ipsec-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/ipsec/ipsec.proto
[iptables-proto-linux]: https://github.com/ligato/vpp-agent/blob/master/api/models/linux/iptables/iptables.proto
[l3-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/l3.proto
[nat-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/nat/nat.proto
[punt-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/punt/punt.proto
[route-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/route.proto
[route-proto-linux]: https://github.com/ligato/vpp-agent/blob/master/api/models/linux/l3/route.proto
[stn-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/stn/stn.proto
[vrf-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l3/vrf.proto
[xconnect-proto]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/xconnect.proto

*[ARP]: Address Resolution Protocol
*[NAT]: Network Address Translation

