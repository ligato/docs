# Keys Reference

This page describes the VPP and Linux keys supported by the VPP agent.

---

The parts of the long form key in `<>` must be set with the same value as defined in the model. As examples, look over the structure and sample values used in the `etcdctl put` commands used in the VPP and Linux plugins sections. The microservice label is set to `vpp1` in the examples. 

For convenience, included below are the .proto and model files associated with each key.

References:

- [VPP Plugins](../plugins/vpp-plugins.md)
- [Linux Plugins](../plugins/linux-plugins.md)
- [Agentctl model ls command](../user-guide/agentctl.md#model)

---

### VPP Keys

**ACL-based forwarding (ABF)**

- [VPP ABF proto][abf-proto]
- [VPP ABF model][abf-model]

Key:
```json
/vnf-agent/vpp1/config/vpp/abfs/v2/abf/<index>
```

---

**Access control lists (ACL)**

- [VPP ACL proto][acl-proto]
- [VPP ACL model][acl-model]

Key:
```json
/vnf-agent/vpp1/config/vpp/acls/v2/acl/<name>
```
---


**VPP interfaces**

- [VPP interface proto][vpp-interface-proto] 
- [VPP interface models][vpp-interface-model]

Key:
```json
/vnf-agent/vpp1/config/vpp/v2/interfaces/<name>
```
---


**VPP interface status**

- [VPP interface status proto][vpp-interface-state-proto]
- [VPP interface status models][vpp-interface-model]

Key:
```json
/vnf-agent/vpp1/vpp/status/v2/interface/<name>
```
---

**VPP Interface Span**

- [VPP span proto][vpp-span-proto]
- [VPP span model][vpp-interface-model]

Key:
```json
/vnf-agent/vpp1/config/vpp/v2/span/<interface from>/to/<interface to>
```

**Bridge domains**

- [BD proto][bd-proto]
- [BD model][bd-model]

Key:
```json
/vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/<name>
```
---


**Forwarding table (L2FIBs)**

- [FIB proto][bd-proto]
- [FIB model][bd-model] 


Key:
```json
/vnf-agent/vpp1/config/vpp/l2/v2/fib/<bridge_domain>/mac/<phys_address>
```
---


**L2 x-connect**

- [xconnect proto][xconnect-proto]
- [xconnect model][bd-model]


Key:
```json
/vnf-agent/vpp1/config/vpp/l2/v2/xconnect/<receive-interface>
```
---


**VPP ARP**

- [VPP ARP proto][arp-proto]
- [VPP ARP model][L3-models]

Key:
```json
/vnf-agent/vpp1/config/vpp/v2/arp/<interface>/<ip_address>
```
---


**VPP proxy ARP**

- [VPP Proxy ARP proto][L3-proto]
- [VPP Proxy ARP model][L3-models] 

Key:
```json
/vnf-agent/vpp1/config/vpp/v2/proxyarp-global/<settings>
```
---


**VPP routes**

- [VPP routes proto][route-proto]
- [VPP routes model][L3-models]

Key (dst_network is with mask in format `<ip>/<mask>`):
```json
/vnf-agent/vpp1/config/vpp/v2/route/vrf/<vrf_id>/dst/<dst_network>/gw/<next_hop_addr>
```

Key if the gateway is `not set`:
```json
/vnf-agent/vpp1/config/vpp/v2/route/vrf/<vrf_id>/dst/<dst_network>
```
---


**IP scan neighbor**

- [VPP IP scan-neighbor proto][L3-proto]
- [VPP IP scan-neighbor][L3-models] 

Key:
```json
/vnf-agent/vpp1/config/vpp/v2/ipscanneigh-global/<settings>
```
---


**Virtual routing and forwarding (VRF)**

- [VPP VRF table proto][vrf-proto]
- [VPP VRF table model][L3-models]

Key (protocol is either 'IPV4' or 'IPV6'):
```json
/vnf-agent/vpp1/config/vpp/v2/vrf-table/id/<id>/protocol/<protocol>
```
---

**L3 x-connect**

- [VPP L3xc proto][l3xc-proto]
- [VPP l3xc model][l3-models]

Key:
```json
/vnf-agent/vpp1/config/vpp/v2/l3xconnect/<interface>/protocol/<protocol>
```

---

**NAT Global**

- [VPP NAT global proto][nat-proto]
- [VPP NAT global model ][nat-model]

Key:
```json
/vnf-agent/vpp1/config/vpp/nat/v2/nat44-global/<settings>
```
---

**DNAT44**

- [VPP DNAT44 proto][nat-proto]
- [VPP DNAT44 model][nat-model]

Key:
```json
/vnf-agent/vpp1/config/vpp/nat/v2/dnat44/<label>
```
---

**NAT Pool**

- [VPP NAT pool proto][nat-proto]
- [VPP NAT pool model ][nat-model]

Key:
```json
/vnf-agent/vpp1/config/vpp/nat/v2/nat44-pool/<settings>
```

---

**NAT Interface**

- [VPP NAT interface proto][nat-proto]
- [VPP NAT interface model][nat-model]

Key:
```json
/vnf-agent/vpp1/config/vpp/nat/v2/nat44-interface/<name>
```
---


**IPSec SPD (Security Policy Database)**

- [VPP SPD proto][ipsec-proto]
- [VPP SPD model][ipsec-model]

Key:
```json
/vnf-agent/vpp1/config/vpp/ipsec/v2/spd/<index>
```
---


**IPSec SA (Security Association)**

- [VPP SA proto][ipsec-proto]
- [VPP SA model][ipsec-model]

Key:
```json
/vnf-agent/vpp1/config/vpp/ipsec/v2/sa/<index>
```
---
**IPsec tunnel protection**

- [VPP tunnel protect proto][ipsec-proto]
- [VPP tunnel protect model][ipsec-model]

Key:
```json
/vnf-agent/vpp1/config/vpp/ipsec/v2/tun-protect/<interface>
```
---

**Punt to host stack**

- [VPP Punt proto][punt-proto]
- [VPP Punt model][punt-model]

Key (l3_protocol and l4_protocol are enum values, not indexes such as IPv4, IPv6 and TCP):
```json
/vnf-agent/vpp1/config/vpp/v2/tohost/l3/<l3_protocol>/l4/<l4_protocol>/port/<port>
```
Key (punt to host exceptions)
```json
/vnf-agent/vpp1/config/vpp/v2/exception/<reason>
```

---


**IP redirect**

- [VPP IP redirect proto][punt-proto]
- [VPP IP redirect model][punt-model] 

Key (l3_protocol is an enum value, not an index such as IPv4 and IPv6):
```json
/vnf-agent/vpp1/config/vpp/v2/ipredirect/l3/<l3_protocol>/tx/<tx_interface>
```

---

**DHCP Proxy**


- [VPP DHCP Proxy proto][l3-proto]
- [VPP DHCP Proxy model][L3-models] 

Key:
```json
/vnf-agent/vpp1/config/vpp/v2/dhcp-proxy/<source IP address>
```

---

**STN**

- [VPP STN proto][stn-proto]
- [VPP STN model][stn-model]

Key:
```json
/vnf-agent/vpp1/config/vpp/stn/v2/rule/<interface>/ip/<ip_address>
```
---

**SRv6**

- [VPP SRv6 proto][srv6-proto]
- [VPP SRv6 model][srv6-model]

Key - Global Configuration: 
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/srv6-global
```
Key - Local SID:
```json
/vnf-agent/<agent-label>/config/vpp/srv6/v2/localsid/<SID>
```
Key - Policy:
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/policy/<bsid>
```
Key - Steering:
```
/vnf-agent/<agent-label>/config/vpp/srv6/v2/steering/<name>
```

---

### Linux Keys

**Linux interfaces**

- [Linux interface proto][linux-interface-proto] 
- [Linux interface model][linux-interface-model]

Key:
```json
/vnf-agent/vpp1/config/linux/interfaces/v2/interface/<name>
```

---


**Linux ARP**

- [Linux ARP proto][linux-arp-proto]
- [Linux ARP model][linux-l3-arp-route-models]

Key:
```json
/vnf-agent/vpp1/config/linux/l3/v2/arp/<interface>/<ip_address>
```
---


**Linux IP tables**

- [Linux iptables proto][linux-iptables-proto]
- [Linux iptables model][linux-iptables-model]

Key:
```json
/vnf-agent/vpp1/config/linux/iptables/v2/rulechain/<name>
```
---


**Linux routes**

- [Linux routes proto][linux-route-proto]
- [Linux routes model][linux-l3-arp-route-models]

Key (dst_network is with mask in format `<ip>/<mask>`):
```json
/vnf-agent/vpp1/config/linux/l3/v2/route/<dst_network>/<outgoing_interface>
```

[abf-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/abf/abf.proto
[acl-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/acl/acl.proto
[arp-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/arp.proto
[arp-proto-linux]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/l3/arp.proto
[bridge-domain-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l2/bridge_domain.proto
[fib-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l2/fib.proto
[interface-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/interfaces/interface.proto
[interface-proto-linux]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/interfaces/interface.proto
[interface-state-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/interfaces/state.proto
[ipsec-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/ipsec/ipsec.proto
[iptables-proto-linux]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/iptables/iptables.proto
[l3-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/l3.proto
[l3-xconnect-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/l3xc.proto
[nat-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/nat/nat.proto
[punt-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/punt/punt.proto
[route-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/route.proto
[route-proto-linux]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/l3/route.proto
[srv6-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/srv6/srv6.proto
[stn-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/stn/stn.proto
[vrf-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/vrf.proto
[xconnect-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l2/xconnect.proto
[vrf-table-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/vrf.proto

[abf-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/abf/models.go
[abf-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/abf/abf.proto
[acl-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/acl/models.go
[acl-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/acl/acl.proto
[arp-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/arp.proto
[bd-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l2/models.go
[bd-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l2/bridge_domain.proto
[vpp-interface-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/interfaces/models.go
[vpp-interface-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/interfaces/interface.proto
[ipsec-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/ipsec/models.go
[ipsec-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/ipsec/ipsec.proto
[L3-models]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/models.go
[L3-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/l3.proto
[l3xc-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/l3xc.proto
[linux-key-reference]: ../user-guide/reference.md#linux-keys
[linux-interface-plugin-guide]: linux-plugins.md#interface-plugin
[nat-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/nat/models.go
[nat-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/nat/nat.proto
[punt-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/punt/models.go
[punt-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/punt/punt.proto
[route-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/route.proto
[srv6-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/srv6/models.go
[srv6-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/srv6/srv6.proto
[vpp-interface-state-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/interfaces/state.proto
[stn-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/stn/models.go
[linux-interface-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/interfaces/models.go
[linux-interface-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/interfaces/interface.proto
[linux-l3-arp-route-models]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/l3/models.go
[linux-iptables-model]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/iptables/models.go
[linux-iptables-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/iptables/iptables.proto
[linux-route-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/l3/route.proto
[linux-arp-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/l3/arp.proto
[vpp-span-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/interfaces/span.proto
[vpp-span-model]: 

*[ARP]: Address Resolution Protocol
*[NAT]: Network Address Translation

