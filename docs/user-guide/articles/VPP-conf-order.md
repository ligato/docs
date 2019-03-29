# VPP configuration order

One of the VPP biggest drawbacks is that the dependency relations of various configuration items are very strict, and ability of the VPP to manage it is limited, or not present at all. There are two problems to be solved:
1. A configuration item dependent on any other configuration item cannot be created "in advance", the dependency must be fulfilled first
2. If the dependency item is removed, the VPP behavior is not consistent here. Sometimes, also the dependent item is removed as expected but there are cases where it becomes unmanageable or in any other kind of invalid state where it cannot be directly handled.

### Configuration order using the VPP CLI

The base kind of the VPP configuration are interfaces. After the interface creation, the VPP generates its index which is serves as a reference for other configuration items which plan to use it (bridge domains, routes, ARP entries, ...). Other items, like FIBs may have even more complicated dependencies on several configuration types.

Example:

1. Start with the empty VPP and configure an interface:
```bash
vpp# create loopback interface 
loop0
vpp# 
```

The interface `loop0` was added and interface index (and also name) was generated for it. The index can be shown by `show interfaces`. 

2. Set the interface up:
```bash
vpp# set interface state loop0 up
vpp#
``` 

The CLI command requires interface name to know which interface should be set to UP state. 

3. Now let's create the bridge domain:
```bash
vpp# create bridge-domain 1 learn 1 forward 1 uu-flood 1 flood 1 arp-term 1   
bridge-domain 1
vpp# 
```

4. The bridge domain is currently empty, so let's assign our interface to it:
```bash
vpp# set interface l2 bridge loop0 1 
vpp#
``` 

We set `loop0` to bridge domain with index 1. Again, we cannot use non-existing interface (since the names are generated) but we also cannot use non-existing bridge domain.

5. At last, let's configure the FIB table entry:
```bash
vpp# l2fib add 52:54:00:53:18:57 1 loop0
vpp#
```

The call contains dependencies on the `loop0` interface and on our bridge domain with ID=1. From the example steps above, we see that the order of CLI commands must follow in this way (with some exceptions like the bridge domain creation (third step) which can be called earlier).

6. Now remove the interface:
```bash
vpp# delete loopback interface intfc loop0
vpp#
```

If we check the FIB table now with `show l2fib details`, we can see that the interface name was changed from `loop0` to `Stale`.

7. Remove bridge domain. The command is:
```bash
vpp# create bridge-domain 1 del
vpp#
```

Here we get into trouble, since the output of the `show l2fib verbose` is still the same:
```bash
vpp# sh l2fib verbose
    Mac-Address     BD-Idx If-Idx BSN-ISN Age(min) static filter bvi         Interface-Name        
 52:54:00:53:18:57    1      1      0/0      no      *      -     -               Stale                     
L2FIB total/learned entries: 2/0  Last scan time: 6.6309e-4sec  Learn limit: 4194304 
vpp#
``` 
But attempt to remove the FIB entry will not pass:
```bash
vpp# l2fib del 52:54:00:53:18:57 1 
l2fib del: bridge domain ID 1 invalid
vpp#
```

So we ended up with the configuration which cannot be removed (until the bridge domain is re-created). To avoid similar scenarios and provide more freedom in configuration order, the vpp-agent uses automatic configuration order mechanism which handles this one and similar cases.

### Configuration order using the Ligato vpp-agent

The vpp-agent uses northbound API definition of every supported configuration item, while the single proto-modelled dataset may call multiple binary API calls to the VPP. For example NB interface configuration creates the interface itself, sets its state, MAC address, IP addresses and other. However, the vpp-agent goes even further - it allows to "configure" VPP items with not-yet-existing references. Such an item is not really configured (since it cannot be), but the agent "remembers" it and puts to the VPP when possible, without any other intervention, thus removing strict VPP ordering.

**Note:** if you want to follow, this part expects to have the basic setup prepared (vpp-agent + VPP + ETCD). If you need help with the setup, [here is the guide](Quickstart.md).

1. Let's start from the end and put configuration for the FIB entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/fib/bd1/mac/62:89:C6:A3:6D:5C '{"phys_address":"62:89:C6:A3:6D:5C","bridge_domain":"bd1","outgoing_interface":"if1","action":"FORWARD"}'
```

Have a look at the vpp-agent output of `planned operations`:
```bash
1. CREATE [NOOP IS-PENDING]:
  - key: config/vpp/l2/v2/fib/bd1/mac/62:89:C6:A3:6D:5C
  - value: { phys_address:"62:89:C6:A3:6D:5C" bridge_domain:"bd1" outgoing_interface:"if1" } 
```

There is one `CREATE` transaction planned - our FIB entry with interface `if1` and bridge domain `bd1`. None of these exists, so the transaction was postponed, highlighting it as `[NOOP IS-PENDING]`.

2. Configure the bridge domain:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1 '{"name":"bd1","interfaces":[{"name":"if1"}]}'
```

We have created a simple bridge domain with interface. Let's break down the output:
```bash
1. CREATE:
  - key: config/vpp/l2/v2/bridge-domain/bd1
  - value: { name:"bd1" interfaces:<name:"if1" >  } 
2. CREATE [DERIVED NOOP IS-PENDING]:
  - key: vpp/bd/bd1/interface/tap1
  - value: { name:"if1" }
```

Now there are two planned operations - the first is the bridge domain `bd1` which can be created since there are no restrictions. The second operation contains following highlight: `[DERIVED NOOP IS-PENDING]`. 'DERIVED' means that this item was processed as separate key within the vpp-agent (it is internal functionality so let's not to bother with it now). The rest of the flag is the same as before, meaning that the value `if1` is not present yet.

3. Next step is to add incriminated interface:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/interfaces/tap1 '{"name":"tap1","type":"TAP","enabled":true}
```

The output:
```bash
1. CREATE:
  - key: config/vpp/v2/interfaces/tap1
  - value: { name:"if1" type:TAP enabled:true } 
2. CREATE [DERIVED WAS-PENDING]:
  - key: vpp/bd/bd1/interface/tap1
  - value: { name:"if1" } 
3. CREATE [WAS-PENDING]:
  - key: config/vpp/l2/v2/fib/bd1/mac/62:89:C6:A3:6D:5C
  - value: { phys_address:"62:89:C6:A3:6D:5C" bridge_domain:"bd1" outgoing_interface:"if1" }
```

There are three `planned operations` present. The first one is the interface itself, which was created (and also enabled). 
The second operation is marked as `DERIVED` (the same meaning, not important here) and `WAS-PENDING` which means the cached value could be finally resolved. This operation means that the interface was added to the bridge domain, as defined in the bridge domain configuration.
The last statement is marked `WAS-PENDING` as well and represents our FIB entry, cached until now. Since all dependencies were fulfilled (the bridge domain and the interface as a part of it), the FIB was configured.

4. Now let's try to remove the bridge domain as before:
```bash
etcdctl del /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1
```

The output:
```bash
1. DELETE [IS-PENDING]:
  - key: config/vpp/l2/v2/fib/bd1/mac/62:89:C6:A3:6D:5C
  - value: { phys_address:"62:89:C6:A3:6D:5C" bridge_domain:"bd1" outgoing_interface:"if1" } 
2. DELETE [DERIVED]:
  - key: vpp/bd/bd1/interface/tap1
  - value: { name:"if1" } 
3. DELETE:
  - key: config/vpp/l2/v2/bridge-domain/bd1
  - value: { name:"bd1" interfaces:<name:"if1" > } 
```

Notice that the first performed operation was removal of the FIB entry, but the value was not discarded by the vpp-agent since it still exists in the ETCD. The vpp-agent put the value back to the cache. This is important step since the FIB entry is removed before the bridge domain, so it will not get mis-configured and stuck in the VPP. The value no longer exists on the VPP (since logically it cannot without all the dependencies met), but if the bridge domain reappears, the FIB will be added back without any action required from outside.
The second step means that the interface `if1` was removed from the bridge domain, before its removal in the last step of the transaction.

The ordering and caching of the configuration is performed by the vpp-agent KVScheduler component. For more information how it works, please refer [here](KVScheduler.md).


   
 





