# VPP CLI

---

**The VPP CLI interface commands:** 

The VPP cli has several CLI commands to verify configuration:

- to show all configured interfaces: `show interface`
- to show interface IP addresses: `show interface address`
- show connected PCI interfaces: `show PCI`
- various information about the state: `show hardware`
- show VxLan tunnel details: `show vxlan tunnel`
- show IPSec tunnel details: `show ipsec`
- bond

**The VPP CLI bridge domain commands:** 

The VPP cli has following CLI commands to verify configuration:

- show list of all configured bridge domains: `show bridge-domain`
- show details (interfaces, ARP entries, ...) for any bridge domain: `show bridge-domain <index> details`

**The VPP CLI FIB commands:**

The VPP cli has following CLI command to verify FIB configuration:

- show list of all configured FIB entries: `show l2fib`

**The VPP CLI cross-connect commands:**

The cross-connect mode can be shown on the VPP with command `show mode`

**The VPP CLI ARP commands:**

The VPP cli has following CLI commands to verify configuration:
- show list of all configured ARPs: `show ip arp`

**The VPP CLI Proxy ARP commands:**

To verify ARP configuration, use the same call as for the ordinary ARP:

- show list of all configured proxy ARPs: `show ip arp`
 
**The VPP CLI route commands:**
 
The route configuration can be verified with:

- show all rotes: `show ip fib`
- show routes from the given VRF table: `show ip fib table <table-ID>` 

**The VPP CLI IP scan neighbor commands:**

The IP scan neighbor configuration can be verified with:

- show IP scan neighbor: `show ip scan-neighbor`

**The VPP CLI access list commands:**

The VPP does not support any CLI commands related to access list. 
In order to retrieve ACL configuration, use `vat#` console and direct binary API call `acl_dump`.    

**The VPP CLI IPSec SPD commands:**

The VPP cli has a command to show the SPD IPSec configuration: `sh ipsec`

**The VPP CLI IPSec SA commands:**

Show the IPSec configuration with the VPP cli command: `sh ipsec`

**The VPP CLI NAT44 global commands:** 

The VPP cli has following CLI commands to verify configuration:

- show list of all NAT44 addresses: `show nat44 addresses`
- show list of all NAT44 interfaces: `show nat44 interfaces`
- show list of all NAT44 interface addresses: `show nat44 interface address`

**The VPP CLI DNAT44 commands:** 

The VPP cli has following CLI commands to verify configuration:

- show static mappings: `show nat44 static mappings`

The VPP cli command (for configuration verification) is `show ip punt redirect`. 
