# KV Scheduler

---

The KVScheduler plugin provides transaction-based configuration processing based on a generic mechanism for dependency resolution between different configuration items. KVScheduler is a core component around which all VPP and Linux configurator (plugin) have been built.

### Motivation

The KVScheduler addresses several challenges encountered in the original vpp-agent design, which became apparent as the variety and complexity of different configuration items increased.   

* `vpp` and `linux` plugins became bloated and complicated, suffering from race conditions and a lack of visibility.

* the `configurators` - i.e. components of `vpp` and `linux` plugins, each processing a specific configuration item type (e.g. interface, route, etc.) - were built from scratch, solving the same set of problems again and again with frequent code duplication.

* configurators would communicate with each through notifications, and react to changes asynchronously to ensure proper operation ordering. Dependency resolution wasn distributed across all configurators, making it very difficult to understand, predict and stabilize the system behavior from the developer's viewpoint.

The result was an unreliable and unpredictable re-synchronization (or resync for short), also known as state reconciliation, between the desired configuration state (referred as Northbound) and the actual configuration state (referred to as southbound.). 

Indeed an efficient and accurate resync function was meant to be one of the primary values offered by the Ligato vpp-agent. 

!!! Note
    Northbound (NB) describes the desired or intended configuration state. Southbound (SB) describes the actual running configuration state.

### Basic concepts and terminology

The KVScheduler addresses the aforementioned challenges by pivoting away from literal configuration item processing and moving to a system based on abstraction.

Some basic concepts and terminology to reinforce this notion:

- **Model** is a description for a single item type, defined as a protobuf Message (e.g. Bridge Domain model can be found [here][bd-model])

- **Value** is a run-time instance of a given model

- **Key** identifies a specific value. It is built from the key template defined for the model.

- **Label** can be optionally defined to provide a shorter identifier unique to values of the same type (e.g. interface name)

- **Metadata** is extra run-time information (of undefined type) assigned to a value, which may be updated after a CRUD operation or an agent restart. For example the [sw_if_index][vpp-iface-idx] of every interface is kept in the metadata for its key-value pair)

- **Metadata Map**, also known as index-map, implements a mapping between value label and its metadata for a given value type - typically exposed in read-only mode to allow other plugins to read and reference metadata. For example, the interface plugin exposes its metadata map [here][vpp-iface-map], which is then used by ARP, route plugin etc. to read the sw_if_index of target interfaces. Metadata maps are automatically created and updated by the KVscheduler, and exposed to other plugins only in read-only mode. 

- **[Value origin][value-origin]** defines where the value came from - whether it was received from NB to be configured or whether it was created in the SB plane automatically (e.g. default routes, loop0 interface, etc.)

- **Key-value pairs** managed via CRUD operations, where `Add` = `Create`, `Dump` equals `Read`, `Update` equals `modify`, and `delete` equals `borrow unchanged`

- **Dependency** defined for a value, references another key-value pair that must exist (be created), otherwise the associated value cannot be added and must remain cached in the state **Pending** - value is allowed to have defined multiple dependencies and all must be satisfied for the value to be considered ready for creation.

- **Derived value**, in a future release will be renamed to **attribute** for more clarity. It is typically a single field of the original value or its property, manipulated separately. It could come with its own dependencies (dependency on the source value is implicit) or custom implementations for CRUD operations. Potentially it could used as target for dependencies of other key-value pairs. For example, every [interface to be assigned to a bridge domain][bd-interface] is treated as a [separate key-value pair][bd-derived-vals], dependent on the [target interface to be created first][bd-iface-deps], but otherwise not blocking the rest of the bridge domain to be applied

- **Graph** of values is a KVScheduler internal in-memory storage for all configured and pending key-value pairs, with edges representing inter-value relations, such as "depends-on" and "is-derived-from". Configurators no longer have to implement their own caches for pending values

- **KVDescriptor** assigns implementations of CRUD operations and defines derived values and dependencies to a single value type. This is what configurators basically boil down to. To learn more, please read how to [implement your own KVDescriptor][kvdescriptor-dev-guide].


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

**Transaction History**

```
GET /scheduler/txn-history
```
- returns the full history of executed transactions or only for a given time window

- args:
    - `format=<json/text>`
    - `seq-num=<txn-seq-num>`: transaction sequence number
    - `since=<unix-timestamp>`: if undefined, the output starts with the oldest kept record
    - `until=<unix-timestamp>`: if undefined, the output end with the last executed transaction

**Key Timeline** 
```
GET /scheduler/key-timeline
```
- args:
    - `key=<key-without-agent-prefix>`: key of the value to show changes over time


**Graph Snapshot**: 
```
GET /scheduler/graph-snaphost
```
- args:
    - `time=<unix-timestamp>`: if undefined, current state is returned
        
        
**Dump Values** 
```
GET /scheduler/dump
```
- args (without args prints Index page):
    - `descriptor=<descriptor-name>`
    - `state=<NB, SB, internal>` (in the next release will be renamed `view`): whether to dump desired, actual or the configuration state as known to the KVscheduler
    
    
    
    
**Request Downstream Resync**

```
POST /scheduler/downstream-resync
```
- args:
    - `retry=< 1/true | 0/false >`: allow to retry operations that failed during resync
    - `verbose=< 1/true | 0/false >`: print graph after refresh (Dump)


[bd-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto
[bd-interface]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto#L19
[bd-derived-vals]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/l2plugin/descriptor/bridgedomain.go
[bd-iface-deps]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/l2plugin/descriptor/bd_interface.go
[kvdescriptor-dev-guide]: ../developer-guide/kvdescriptor.md
[vpp-iface-idx]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/ifplugin/ifaceidx/ifaceidx.go
[vpp-iface-map]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/ifplugin/ifplugin_api.go#L26
[value-origin]: https://github.com/ligato/vpp-agent/blob/dev/plugins/kvscheduler/api/kv_descriptor_api.go#L53

*[ARP]: Address Resolution Protocol
*[FIB]: Forwarding Information Base
*[REST]: Representational State Transfer
*[VNF]: Virtual Network Function