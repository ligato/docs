# VPP CLI

The VPP CLI can be used to verify and troubleshoot VPP configurations. This section describes some of the VPP CLI commands to support that function.

**References:**

- [FD.io wiki CLI Guide][vpp-cli-guide]
- [How to start the VPP CLI in the VPP agent container](quickstart.md#53-vpp-cli)
- [How to execute VPP CLI commands through a REST API][vpp-cli-rest-api]
- [How to execute VPP CLI commands through Agentctl][agentctl-vpp-cli]
 

---

**Class VPP CLI terminal console:**
```
    _______    _        _   _____  ___ 
 __/ __/ _ \  (_)__    | | / / _ \/ _ \
 _/ _// // / / / _ \   | |/ / ___/ ___/
 /_/ /____(_)_/\___/   |___/_/  /_/    

vpp# 
```

**Interface** 

show all configured interfaces: 
```
show interface
```
show interface IP addresses: 
```json
show interface address
```
show connected PCI interfaces:
```
show PCI
```
show hardware state: 
```
show hardware
```
show vxLan tunnel details: 
```
show vxlan tunnel
```
show bond interface details: 
```
show bond `or` show bond details
```

---

**Bridge domain:** 

show list of all configured bridge domains:
```
show bridge-domain
```
show details (interfaces, ARP entries, ...) for any bridge domain:
```
show bridge-domain <index> details
```

---

**FIB:**

show list of all configured FIB entries: 
```
show l2fib
```

---

**Cross-connect:**

show cross-connect mode:
```
show mode
```

**ARP and Proxy ARP**

show list of all configured ARPs:
```
show ip arp
```

---
 
**L3 route:**
 
show all routes:
```
show ip fib
```
show routes from a given VRF table:
```
show ip fib table <table-ID>
``` 

---

**IP scan neighbor**

show IP scan neighbor:
```
show ip scan-neighbor
```

---

**ACL**

VPP does not support any CLI commands related to ACLs. In order to retrieve ACL configuration data, use:

- `vat#` console and a direct binary API call `acl_dump`, or
- call the [IP ACL](../api/api-vpp-agent.md#vpp-acl-ip) REST API or [MACIP ACL](../api/api-vpp-agent.md#vpp-acl-macip) REST API from the VPP Agent    

---

**IPSec**

show IPSec configuration:
```
sh ipsec
```
---

**NAT44 global:** 

show list of all NAT44 addresses:
```
show nat44 addresses
```
show list of all NAT44 interfaces:
```
show nat44 interfaces
```
show list of all NAT44 interface addresses:
```
show nat44 interface address
```

---

**DNAT44** 

show static mappings: 
```
show nat44 static mappings
```

---

**IP Punt Redirect**

Show IP punt redirect configuration: 
```
show ip punt redirect
``` 
[vpp-cli-guide]: https://wiki.fd.io/view/VPP/Command-line_Interface_(CLI)_Guide
[vpp-cli-rest-api]: ../api/api-vpp-agent.md#vpp-cli-command
[agentctl-vpp-cli]: ../user-guide/agentctl.md#vpp