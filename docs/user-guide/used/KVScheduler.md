**Related article:** [Implement your own KV Descriptor](../../developer-guide/implementing-your-own-kvdescriptor.md)

A major enhancement that led to the increase of the agent's major version number from 1.x to 2.x, is an introduction of a new framework, called **KVScheduler**, providing transaction-based configuration processing with a generic mechanism for dependency resolution between configuration items, which in effect simplifies and unifies the configurators. KVScheduler is shipped as a separate [plugin], even though it is now a core component around which all the VPP and Linux configurators have been re-build.

## Motivation

KVScheduler is a reaction to a series of drawbacks of the original design, which gradually arose with the set of supported configuration items growing:
* plugins `vpp` and `linux` became bloated and complicated, suffering with race conditions and a lack of visibility
* the `configurators` - i.e. components of `vpp` and `linux` plugins, each processing a specific configuration item type (e.g. interface, route, etc.) - had to be effectively built from scratch, solving the same set of problems again and duplicating lot's of code
* configurators would communicate with each other only through notifications, and react to changes asynchronously to ensure proper operation ordering - i.e. the problem of dependency resolution was effectively distributed across all configurators, making it very difficult to understand, predict and stabilize the system behavior from the developer's global viewpoint
* a culmination of all the issues above was an unreliable and unpredictable re-synchronization (or resync for short), also known as state reconciliation, between northbound (desired state) and southbound (actual state) - something that was meant to be marketed as the main feature of the VPP-Agent.

## Basic concepts and terminology

KVScheduler solves problems common to all configurators through generalization - moving away from specific configuration items and describing the system with abstract terms:
* **Model** is a description for a single item type, defined as Protobuf Message (for example Bridge Domain model can be found [here][bd-model-example])
* **Value** is a run-time instance of a given model
* **Key** identifies specific value (it is built from key template defined for the model, e.g. [L2 FIB key template][fib-key-template])
* **Label** can be optionally defined to provide shorter identifier unique only across values of the same type (e.g. interface name)
* **Metadata** is extra run-time information (of undefined type) assigned to a value, which may be updated after a CRUD operation or an agent restart (for example [sw_if_index][vpp-iface-idx] of every interface is kept in the metadata for its key-value pair)
* **Metadata Map**, also known as index-map, implements mapping between value label and its metadata for a given value type - typically exposed in read-only mode to allow other plugins to read and reference metadata (for example, interface plugin exposes its metadata map [here][vpp-iface-map], which is then used by ARP, route plugin etc. to read sw_if_index of target interfaces).  Metadata maps are automatically created and updated by the scheduler (he is the owner), and exposed to other plugins only in the read-only mode. 
* **[Value origin][value-origin]** defines where the value came from - whether it was received from NB to be configured or whether it was created in the SB plane automatically (e.g. default routes, loop0 interface, etc.)
* Key-value pairs are operated with through CRUD operations, where **Add** is used to denote **Create**, **Dump** is basically Read, Update is denoted by the scheduler as **Modify** and **Delete** is borrowed unchanged
* **Dependency** defined for a value, references another key-value pair that must exist (be created), otherwise the associated value cannot be Added and must remain cached in the state **Pending** - value is allowed to have defined multiple dependencies and all must be satisfied for the value to be considered ready for creation.
* **Derived value**, in future release to be renamed to **attribute** for more clarity, is typically a single field of the original value or its property, manipulated separately - i.e. with possibly its own dependencies (dependency on the source value is implicit), custom implementations for CRUD operations and potentially used as target for dependencies of other key-value pairs - for example, every [interface to be assigned to a bridge domain][bd-interface] is treated as a [separate key-value pair][bd-derived-vals], dependent on the [target interface to be created first][bd-iface-deps], but otherwise not blocking the rest of the bridge domain to be applied
* **Graph** of values is a kvscheduler-internal in-memory storage for all configured and pending key-value pairs, with edges representing inter-value relations, such as "depends-on" and "is-derived-from" - configurators no longer have to implement their own caches for pending values
* **KVDescriptor** assigns implementations of CRUD operations and defines derived values and dependencies to a single value type - this is what configurators basically boil down to - to learn more, please read how to [implement your own KVDescriptor](Implementing-your-own-KVDescriptor)

## Dependencies

The idea behind scheduler is based on the Mediator pattern - configurators do not communicate directly, but instead interact through the mediator. This reduces the dependencies between communicating objects, thereby reducing coupling.

The values are described for scheduler by registered KVDescriptor-s.
The scheduler learns two kinds of relations between values that have to be respected by the scheduling algorithm:
1. `A` **depends on** `B`:
   - `A` cannot exist without `B`
   - request to add `A` without `B` existing must be postponed by marking `A` as `pending` (value with unmet dependencies) in the in-memory graph 
   - if `B` is to be removed and `A` exists, `A` must be removed first and set to `pending` state in case `B` is restored in the future
   - Note: values pushed from SB are not checked for dependencies
2. `B` **is derived from** `A`:
   - value `B` is not added directly (by NB or SB) but gets derived from base value `A` (using the DerivedValues() method of the base value's descriptor)
   - a derived value exists only as long as its base does and gets removed (immediately, not pending) once the base value goes away
   - a derived value may be described by a different descriptor than the base and usually represents the property of the base value (that other values may depend on) or an extra action to be taken when additional dependencies are met.

## Resync 

Configurators no longer have to implement resync on their own. As they "teach" KVScheduler how to operate with configuration items by providing callbacks to CRUD operations through KVDescriptors, the scheduler has all it needs to determine and execute the set of operation needed to get the state in-sync after a transaction or restart.

Furthermore, KVScheduler enhances the concept of state reconciliation, and defines three types of the resync:
* **Full resync**: the desired configuration is re-read from NB, the view of SB is refreshed via Dump operations and inconsistencies are resolved via Add/Delete/Modify operations
* **Upstream resync**: partial resync, same as Full resync except for the view of SB is assumed to be up-to-date and will not get refreshed - can be used by NB when it is easier to re-calculate the desired state than to determine the (minimal) difference
* **Downstream resync**: partial resync, same as Full resync except the desired configuration is assumed to be up-to-date and will not be re-read from NB - can be used periodically to resync, even without interacting with NB

## Transactions

The scheduler allows to group related changes and applies them as transactions. This is not supported, however, by all agent NB interfaces - for example, changes from `etcd` datastore are always received one a time. To leverage the transaction support, localclient (the same process) or GRPC API (remote access) have to be used instead.

Inside the scheduler, transactions are queued and executed synchronously to simplify the algorithm and avoid concurrency issues. The processing of a transaction is split into two stages:
* **Simulation**: the set of operations to execute and their order is determined (so-called *transaction plan*), without actually calling any CRUD callback from descriptors - assuming no failures.
* **Execution**: executing the operations in the right order. If any operation fails, the already applied changes are reverted, unless the so called `BestEffort` mode is enabled, in which case the scheduler tries to apply the maximum possible set of required changes. `BestEffort` is the default for resync.  

Right after simulation, transaction metadata (sequence number printed as `#xxx`, description, values to apply, etc.) are printed, together with transaction plan. This is done before execution, to ensure that the user is informed about the operations that were going to be executed even if any of the operations cause the agent to crash. After the transaction has executed, the set of actually executed operations and potentially some errors are printed to finalize the output for the transaction.

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

## REST API

KVScheduler exposes the state of the system and the history of operations not only via formatted logs but also through a set of REST APIs:
* **transaction history**: `GET /scheduler/txn-history`
    - returns the full history of executed transactions or only for a given time window
    - args:
        - `format=<json/text>`
        - `seq-num=<txn-seq-num>`: transaction sequence number
        - `since=<unix-timestamp>`: if undefined, the output starts with the oldest kept record
        - `until=<unix-timestamp>`: if undefined, the output end with the last executed transaction
* **key timeline**: `GET /scheduler/key-timeline`
     - args:
        - `key=<key-without-agent-prefix>`: key of the value to show changes over time for
* **graph snapshot**: `GET /scheduler/graph-snaphost`
    - args:
        - `time=<unix-timestamp>`: if undefined, current state is returned
* **dump values**: `GET /scheduler/dump`
    - args (without args prints Index page):
        - `descriptor=<descriptor-name>`
        - `state=<NB, SB, internal>` (in the next release will be renamed to `view`): whether to dump desired, actual or the configuration state as known to kvscheduler
* **request downstream resync**: `POST /scheduler/downstream-resync`
    - args:
        - `retry=< 1/true | 0/false >`: allow to retry operations that failed during resync
        - `verbose=< 1/true | 0/false >`: print graph after refresh (Dump)

[plugin]: https://github.com/ligato/vpp-agent/tree/dev/plugins/kvscheduler
[bd-model-example]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/model/l2/bd.proto
[fib-key-template]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/model/l2/keys.go#L38
[vpp-iface-idx]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/ifplugin/ifaceidx/ifaceidx.go#L62
[vpp-iface-map]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/ifplugin/ifplugin_api.go#L26
[value-origin]: https://github.com/ligato/vpp-agent/blob/dev/plugins/kvscheduler/api/kv_descriptor_api.go#L53
[bd-interface]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/model/l2/bd.proto#L14
[bd-derived-vals]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/l2plugin/descriptor/bridgedomain.go#L225
[bd-iface-deps]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/l2plugin/descriptor/bd_interface.go#L128