# KV Scheduler

---

The KV Scheduler plugin provides transaction-based configuration processing based on a generic mechanism for dependency resolution between different configuration items. The KV Scheduler is a core component employed by all VPP and Linux plugin configurators.

Reference: [KV Scheduler Discussion in the Developer Guide][kvs-dev-guide]

---

### Motivation

The KV Scheduler addresses several challenges encountered in the original VPP agent design. These became apparent as the variety and complexity of different configuration items increased.   

* VPP and Linux plugins became bloated and complicated, suffering from race conditions and a lack of visibility.

* VPP and Linux plugin configurator components, each processing a specific configuration item type such as an interface or route, were built from scratch, solving the same set of problems with frequent code duplication.

* Configurator components would communicate with each through notifications, and react to changes asynchronously to ensure proper operational ordering. Dependency resolution was not distributed across all configurators, making it difficult to understand, predict and stabilize system behavior from the developer's viewpoint.

The result was an unreliable and unpredictable re-synchronization (resync), also known as state reconciliation, between the desired northbound (NB) configuration state, and the actual run-time southbound (SB) configuration state. 


!!! Note
    Northbound (NB) describes the desired or intended configuration state. Southbound (SB) describes the run-time configuration state.

---

### Basic Concepts 

The [KV Scheduler Discussion in the Developer Guide][kvs-dev-guide] provides more detail on the KV Scheduler plugin. 

Here, we will provide a brief explanations of key concepts: 

- Applies graph theory concepts to manage dependencies between configuration items and their proper operational ordering.
- System state is modeled as a graph; configuration items are represented as vertices and the relationship between them are represented as edges. 
- Graph is then walked through to generate transaction plans, refresh the state, execute state reconciliation, and so on.
- Transaction plan is a series of CRUD operations performed on graph vertices, where vertices represent configuration items.
- Configuration items and how to deal with them are "abstracted away" by the notion of a [KV Descriptors][kvdescriptor-dev-guide] that "describe" the graph vertices to the KV Scheduler.
- KV Descriptors are basically handlers, each assigned to a distinct subset of graph vertices, providing the KV Scheduler with pointers to callbacks that implement CRUD operations. 
- Plugins are decoupled and no longer communicate directly, but instead interact with each other through a mediator. The KV Scheduler is the mediator.
- Plugins provide CRUD callbacks, and describe their dependencies on other plugins through one or more KV Descriptors.
- KV Scheduler is then able to plan operations without knowing what the graph vertices represent in the configured system.


---

### Dependencies

The idea behind the KV Scheduler is based on the mediator pattern. Configurators do not communicate directly, but instead interact through a mediator. This reduces the dependencies between communicating objects, thereby reducing coupling.

The values are described to the KV Scheduler by registered KV Descriptors. The KV Scheduler learns two types of relationships between values processed by the scheduling algorithm:

**1.** `A` **depends on** `B`:

   - `A` cannot exist without `B`.
   - Request to add `A` without `B` existing must be postponed by marking `A` as `pending` in the in-memory graph. `Pending` means that a value has dependencies that have not been resolved.
   - If `B` is to be removed and `A` exists, `A` must be removed first and set to `pending` state in case `B` is restored in the future.
   
!!! note  
    Values pushed from SB are not checked for dependencies
    
**2.** `B` **is derived from** `A`:

   - value `B` is not added directly by NB or SB. Instead, it is derived from base value `A` using the DerivedValues() method of the base value's descriptor.
   - A derived value exists only as long as its base value does. It is removed once the base value disappears. It is NOT placed in a `pending` state.
   - A derived value may be described by a descriptor that is different from the base value. It usually represents the property of the base value that other values may depend on, or an extra action to be taken when additional dependencies are met.

---

### Resync 

Configurators no longer need to implement resync on their own. In effect, they "teach" the KV Scheduler how to operate on configuration items by providing callbacks to CRUD operations through KV Descriptors. The KV Scheduler now has all it needs to determine and execute the set of operations needed to reach state synchronization following a transaction or restart.

Furthermore, the KV Scheduler enhances the concept of state reconciliation, and defines three types of the resync:

- **Full resync**: desired configuration is re-read from NB, the view of SB is refreshed via Dump operations and inconsistencies are resolved via Add/Delete/Modify operations.
- **Upstream resync**: partial resync, same as full resync except the SB view is assumed to be up-to-date and will not be refreshed. It can be used by NB to recalculate the desired state which may be simpler than exerting effort to compute the minimal deltas.
- **Downstream resync**: partial resync, same as full resync except the desired configuration is assumed to be up-to-date and will not be re-read from NB. It can be used periodically to resync, even without interacting with NB.

---

### Transactions

The KV Scheduler can group related changes and apply them as transactions. This is not supported, however, by all agent NB interfaces. For example, changes from the etcd data store are always received one at a time. To leverage the transaction support, localclient consisting of the same process, or remote access using a gRPC API is required.

Inside the KV Scheduler, transactions are queued and executed synchronously to simplify the algorithm and avoid concurrency issues. 

Transaction processing is divided into two stages:

- **Simulation**: the set of operations to execute, their order determined by the `transaction plan`, without actually initiating CRUD descriptor callbacks.

- **Execution**: executing the operations in the correct order. If any operation fails, the applied changes are reverted. If the `BestEffort` mode is enabled, the KV Scheduler attempts to apply the maximum possible set of required changes. BestEffort is the default for resync.  

Following simulation, transaction metadata including the sequence number printed as `#xxx`, description, and values to apply are printed, together with the transaction plan. This is done before execution, to ensure that the user has visibility into the operations, even if any cause the agent to crash. 

After the transaction has executed, the set of completed operations and any errors are printed. 

Here is an example of the transaction output printed to logs. In this case, there were no errors so the plan matches the executed operations.
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

---

### REST API

The KV Scheduler exposes the state of the system through a set of REST APIs.

Reference: [KV Scheduler REST API Docs Section][kv-scheduler-rest-api]

---

**Transaction History**: [GET /scheduler/txn-history][kvs-txn-history-api]

- GET a complete history of planned and executed transactions. Can also scope by sequence number and time window. 
- parameters:
    - `format=<json/text>`
    - `seq-num=<txn-seq-num>`: transaction sequence number
    - `since=<unix-timestamp>`: if undefined, the output starts with the oldest kept record
    - `until=<unix-timestamp>`: if undefined, the output end with the last executed transaction

----

**Key Timeline**: [GET /scheduler/key-timeline][kvs-key-timeline-api]

- GET the timeline of value changes for a `specific key`
- Parameters:
    - `key=<key-without-agent-prefix>`

----

**Graph Snapshot**: [GET /scheduler/graph-snapshot][kvs-graph-snapshot-api]

- GET a snapshot of the KV Scheduler internal graph at a point in time
- parameters:
    - `time=<unix-timestamp>`: if undefined, current state is returned
        
----
       
**Dump**: [GET /scheduler/dump][kvs-dump-api]

- GET list of the `descriptors` registered with the KV Scheduler, list of the `key prefixes` under watch in the NB direction, and `view` options from the perspective of the KV Scheduler
- parameters: none

---
 
**Dump (with parameters)**: **GET /scheduler/dump?`<parameters>`**

- GET key-value data filtered by parameters
- parameters: 
    - `descriptor=<descriptor-name>`
    - `key-prefix=<key prefix name>`
    - `view=<NB, SB, cached>`
    - [example][kvs-dump-parameters-example] with query parameters of `view=SB&key-prefix=key-prefix=config/vpp/v2/interfaces/` 

---

**Status**: [GET /scheduler/status][kvs-status-api]

- GET value state by descriptor, by key, or all
- parameters:
    - `descriptor=<descriptor-name>`
    - `key=<key name>`

---

**Flag-Stats**: [GET /schedular/flag-stats][kvs-flag-stats]

- GET total and per-value counts by value flag.
- parameters:
    - `last-update: last transaction that changed/updated the value`
    - `value-state: current state of the value`
    - `descriptor: used to look up values by descriptor`
    - `derived: mark derived values`
    - `unavailable: mark NB values which should not be considered when resolving dependencies of other values`
    - `Error: used to store error returned from the last operation, including validation errors` 
 
    
--- 
    
**Request Downstream Resync**: [POST /scheduler/downstream-resync][kvs-downstream-resync]

- Triggers downstream resync
- parameters:
    - `retry=< 1/true | 0/false >`: permit retry operations that failed during resync
    - `verbose=< 1/true | 0/false >`: print graph after refresh

---
    
**Graph Visualization**: **GET /scheduler/graph**

Used to generate a graph-based representation of the system state, used internally by the KV Scheduler, and can be displayed using any modern web browser supporting SVG. 

Reference: [How to visualize the graph][kvs-graph-api] section of the Developer Guide

----
[bd-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto
[bd-interface]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto#L19
[bd-derived-vals]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/l2plugin/descriptor/bridgedomain.go
[bd-iface-deps]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/l2plugin/descriptor/bd_interface.go
[kvs-dev-guide]: ../developer-guide/kvscheduler.md
[kv-scheduler-rest-api]: ../api/api-kvs.md
[kvs-txn-history-api]: ../api/api-kvs.md#transaction-history
[kvs-key-timeline-api]: ../api/api-kvs.md#key-timeline
[kvs-graph-snapshot-api]: ../api/api-kvs.md#graph-snapshot
[kvs-dump-api]: ../api/api-kvs.md#dump
[kvs-dump-parameters-example]: ../api/api-kvs.md#dump-viewkey-prefix
[kvs-downstream-resync]: ../api/api-kvs.md#downstream-resync
[kvs-status-api]: ../api/api-kvs.md#status
[kvs-flag-stats]: ../api/api-kvs.md#flag-stats
[kvs-graph-api]: ../developer-guide/kvs-troubleshooting.md#how-to-visualize-the-graph
[kvdescriptor-dev-guide]: ../developer-guide/kvdescriptor.md
[vpp-iface-idx]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/ifplugin/ifaceidx/ifaceidx.go
[vpp-iface-map]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/ifplugin/ifplugin_api.go#L26
[value-origin]: https://github.com/ligato/vpp-agent/blob/dev/plugins/kvscheduler/api/kv_descriptor_api.go#L53

*[ARP]: Address Resolution Protocol
*[FIB]: Forwarding Information Base
*[REST]: Representational State Transfer
*[VNF]: Virtual Network Function