# KV Scheduler

---

The KVScheduler plugin provides transaction-based configuration processing based on a generic mechanism for dependency resolution between different configuration items. KVScheduler is a core component which all VPP and Linux plugin configurators are now using.

More Detail: [KVScheduler Discussion in the Developer Guide][kvs-dev-guide]

### Motivation

The KVScheduler addresses several challenges encountered in the original vpp-agent design, which became apparent as the variety and complexity of different configuration items increased.   

* `vpp` and `linux` plugins became bloated and complicated, suffering from race conditions and a lack of visibility.

* the `configurators` - i.e. components of `vpp` and `linux` plugins, each processing a specific configuration item type (e.g. interface, route, etc.) - were built from scratch, solving the same set of problems again and again with frequent code duplication.

* configurators would communicate with each through notifications, and react to changes asynchronously to ensure proper operation ordering. Dependency resolution wasn distributed across all configurators, making it very difficult to understand, predict and stabilize the system behavior from the developer's viewpoint.

The result was an unreliable and unpredictable re-synchronization (or resync for short), also known as state reconciliation, between the desired configuration state (referred as Northbound) and the actual configuration state (referred to as southbound.). 

Indeed an efficient and accurate resync function was meant to be one of the primary values offered by the Ligato vpp-agent. 

!!! Note
    Northbound (NB) describes the desired or intended configuration state. Southbound (SB) describes the actual run-time configuration state.

### Basic concepts 

The [KVScheduler Discussion in the Developer Guide][kvs-dev-guide] provides a good deal more detail on this powerful plugin. 

Here we will provide a brief explanations of some of the key concepts: 

- applies graph theory concepts to manage dependencies between configuration items and their proper operational ordering
- state of the system is modeled as a graph; configuration items are represented as vertices and relations between them are represented as edges. 
- graph is then walked through to generate transaction plans, refresh the state, execute state reconciliation, etc
- transaction plan is a series of CRUD operations performed on some of the graph verticies (i.e. configuration items)
- configuration items and how to deal with them are "abstracted away" by the notion of a [KVDescriptors][kvdescriptor-dev-guide] that "describe" the graph vertices to the KVScheduler
- KVDescriptors are basically handlers, each assigned to a distinct subset of graph vertices, providing the scheduler with pointers to callbacks that implement CRUD operations. 
- plugins are decoupled and no longer communicate directly, but instead interact with each other through a mediator (the KVScheduler)
- Plugins only provide CRUD callbacks and describe their dependencies on other plugins through one or more KVDescriptors.
- KVScheduler is then able to plan operations without even knowing what the graph vertices actually represent in the configured system.


### Dependencies

The idea behind the scheduler is based on the mediator pattern. Configurators do not communicate directly, but instead interact through the mediator. This reduces the dependencies between communicating objects, thereby reducing coupling.

The values are described for the KVScheduler by registered KVDescriptors. The KVScheduler learns two types of relationships between values that must be respected by the scheduling algorithm:

1. `A` **depends on** `B`:

   - `A` cannot exist without `B`
   - request to add `A` without `B` existing must be postponed by marking `A` as `pending` (value with unmet dependencies) in the in-memory graph 
   - if `B` is to be removed and `A` exists, `A` must be removed first and set to `pending` state in case `B` is restored in the future
   
!!! note  
    Values pushed from SB are not checked for dependencies
    
2. `B` **is derived from** `A`:

   - value `B` is not added directly (by NB or SB) but instead is derived from base value `A` (using the DerivedValues() method of the base value's descriptor)
   - a derived value exists only as long as its base does and is removed (immediately, not pending) once the base value disappears.
   - a derived value may be described by a descriptor different from the base and usually represents the property of the base value that other values may depend on or, an extra action to be taken when additional dependencies are met.

### Resync 

Configurators no longer need to implement resync on their own. As they "teach" the KVScheduler how to operate with configuration items by providing callbacks to CRUD operations through KVDescriptors, it now has all it needs to determine and execute the set of operations needed to reach state synchronization following a transaction or restart.

Furthermore, the KVScheduler enhances the concept of state reconciliation, and defines three types of the resync:

- **Full resync**: the desired configuration is re-read from NB, the view of SB is refreshed via Dump operations and inconsistencies are resolved via Add/Delete/Modify operations
- **Upstream resync**: partial resync, same as Full resync except the SB view is assumed to be up-to-date and will not be refreshed. It can be used by NB when it is easier to re-calculate the desired state than to determine the (minimal) deltas.
- **Downstream resync**: partial resync, same as Full resync except the desired configuration is assumed to be up-to-date and will not be re-read from NB. It can be used periodically to resync, even without interacting with NB

### Transactions

The KVScheduler can group related changes and apply them as transactions. This is not supported, however, by all agent NB interfaces - for example, changes from the `etcd` datastore are always received one a time. To leverage the transaction support, localclient (the same process) or GRPC API (remote access) must be used instead.

Inside the KVScheduler, transactions are queued and executed synchronously to simplify the algorithm and avoid concurrency issues. 

The processing of a transaction is split into two stages:

- **Simulation**: the set of operations to execute and their order is determined by the `transaction plan`, without actually initiating CRUD descriptor callbacks.

- **Execution**: executing the operations in the correct order. If any operation fails, the already applied changes are reverted. If the `BestEffort` mode is enabled, the KVScheduler attempts to apply the maximum possible set of required changes. `BestEffort` is the default for resync.  

Right after simulation, transaction metadata (sequence number printed as `#xxx`, description, values to apply, etc.) are printed, together with the transaction plan. This is done before execution, to ensure that the user is informed about the operations that were going to be executed even if any of the operations cause the agent to crash. 

After the transaction has executed, the set of completed operations and and any errors are printed. 

An example transaction output printed to logs (in this case there were no errors, therefore the plan matches the executed operations):
```
+======================================================================================================================+
| Transaction #5                                                                                        NB transaction |
+======================================================================================================================+
  * transaction arguments:
      - seq-num: 5
      - type: NB transaction
      - Description: example transaction
      - values:
          - key: vpp/config/v2/nat44/dnat/default/kubernetes
            value: { label:"default/kubernetes" st_mappings:<external_ip:"10.96.0.1" external_port:443 local_ips:<local_ip:"10.3.1.10" local_port:6443 > twice_nat:SELF >  }
          - key: vpp/config/v2/ipneigh
            value: { mode:IPv4 scan_interval:1 stale_threshold:4  }
  * planned operations:
      1. ADD:
          - key: vpp/config/v2/nat44/dnat/default/kubernetes
          - value: { label:"default/kubernetes" st_mappings:<external_ip:"10.96.0.1" external_port:443 local_ips:<local_ip:"10.3.1.10" local_port:6443 > twice_nat:SELF >  }
      2. MODIFY:
          - key: vpp/config/v2/ipneigh
          - prev-value: { scan_interval:1 max_proc_time:20 max_update:10 scan_int_delay:1 stale_threshold:4  }
          - new-value: { mode:IPv4 scan_interval:1 stale_threshold:4  }
o----------------------------------------------------------------------------------------------------------------------o
  * executed operations (2019-01-21 11:29:27.794325984 +0000 UTC m=+8.270999232 - 2019-01-21 11:29:27.797588466 +0000 UTC m=+8.274261700, duration = 3.262468ms):
     1. ADD:
         - key: vpp/config/v2/nat44/dnat/default/kubernetes
         - value: { label:"default/kubernetes" st_mappings:<external_ip:"10.96.0.1" external_port:443 local_ips:<local_ip:"10.3.1.10" local_port:6443 > twice_nat:SELF >  }
      2. MODIFY:
          - key: vpp/config/v2/ipneigh
          - prev-value: { scan_interval:1 max_proc_time:20 max_update:10 scan_int_delay:1 stale_threshold:4  }
          - new-value: { mode:IPv4 scan_interval:1 stale_threshold:4  }
x----------------------------------------------------------------------------------------------------------------------x
x #5                                                                                                          took 3ms x
x----------------------------------------------------------------------------------------------------------------------x
```

### REST API

KVScheduler also exposes the state of the system and the history of operations through a set of REST APIs:

- **Transaction History**: `GET /scheduler/txn-history`
    - returns the full history of executed transactions or only for a given time window
    - args:
        - `format=<json/text>`
        - `seq-num=<txn-seq-num>`: transaction sequence number
        - `since=<unix-timestamp>`: if undefined, the output starts with the oldest kept record
        - `until=<unix-timestamp>`: if undefined, the output end with the last executed transaction

----

- **Key Timeline**: `GET /scheduler/key-timeline` 
    - args:
        - `key=<key-without-agent-prefix>`: key of the value to show changes over time

----

- **Graph Snapshot**: `GET /scheduler/graph-snaphost`
    - args:
        - `time=<unix-timestamp>`: if undefined, current state is returned
        
----        
- **Dump Values**: `GET /scheduler/dump`  
    - args (without args prints Index page):
        - `descriptor=<descriptor-name>`
        - `state=<NB, SB, internal>` (in the next release will be renamed `view`): whether to dump desired, actual or the configuration state as known to the KVscheduler
    
    
----    
- **Request Downstream Resync**: `POST /scheduler/downstream-resync`
    - args:
        - `retry=< 1/true | 0/false >`: allow to retry operations that failed during resync
        - `verbose=< 1/true | 0/false >`: print graph after refresh (Dump)

----
[bd-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto
[bd-interface]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto#L19
[bd-derived-vals]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/l2plugin/descriptor/bridgedomain.go
[bd-iface-deps]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/l2plugin/descriptor/bd_interface.go
[kvs-dev-guide]: ../developer-guide/kvscheduler.md
[kvdescriptor-dev-guide]: ../developer-guide/kvdescriptor.md
[vpp-iface-idx]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/ifplugin/ifaceidx/ifaceidx.go
[vpp-iface-map]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/ifplugin/ifplugin_api.go#L26
[value-origin]: https://github.com/ligato/vpp-agent/blob/dev/plugins/kvscheduler/api/kv_descriptor_api.go#L53

*[ARP]: Address Resolution Protocol
*[FIB]: Forwarding Information Base
*[REST]: Representational State Transfer
*[VNF]: Virtual Network Function