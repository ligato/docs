# KV Scheduler

---

This section describes the KV Scheduler.

Package references: [kvscheduler](https://godoc.org/github.com/ligato/vpp-agent/plugins/kvscheduler), [api](https://godoc.org/github.com/ligato/vpp-agent/plugins/kvscheduler/api), [graph](https://godoc.org/github.com/ligato/vpp-agent/plugins/kvscheduler/internal/graph) 

---

### Introduction

The KV Scheduler is a transaction-based configuration processing system. It includes a generic mechanism for dependency resolution between configuration items. The KV Scheduler ships as a separate plugin, and is used by all VPP and Linux plugins.

---

### Motivation

The KV Scheduler addresses several challenges encountered in the original VPP agent design. These challenges arose as the variety and complexity of different configuration items increased.

* VPP and Linux plugins became bloated, complicated, and suffered from race conditions and a lack of visibility.
</br>
</br>

* `configurator` components of VPP and Linux plugins, each processing a specific configuration item type (e.g. interface, route, etc.) were built from scratch, solving the same set of problems, with frequent code duplication.
</br>
</br>

* Plugin configurators would communicate with each other through notifications, and react to changes asynchronously to ensure proper operational sequence. Dependency resolution was distributed across all configurators, making it difficult to understand, predict, and stabilize the system behavior.

Re-synchronization (resync) occurring between the desired northbound (NB) configuration state and the southbound (SB) runtime configuration state became unreliable and unpredictable.

---

!!! Terminology
    `Northbound (NB)` describes the desired or intended configuration state. `Southbound (SB)` describes the actual run-time configuration state. `Resync` is also referred to as state reconciliation. `CRUD` stands for create, read, update and delete. KV Descriptors are referred to as just `descriptors`.

---

### Basic Concepts

The KV Scheduler uses graph theory concepts to manage dependencies between configuration items, and the order they are programmed into the network. It applies a level of abstraction on top of the plugins, and models the state of the system as a graph. Vertices represent configuration items; edges represent the relationship between configuration items. The KV Scheduler walks through the graph to generate transaction plans, refresh the state, and perform resync.

A transaction plan consists of a series of CRUD operations to perform on the graph vertices. To abstract away from specific configuration items and accompanying details, graph vertices are "described" to the KV Scheduler using [KV Descriptors][kvdescriptor-guide]. 

You can think of a KV Descriptor as a handler, each assigned to a distinct subset of graph vertices. They provide the KV Scheduler with pointers to callbacks that implement CRUD operations.

---

### Mediator Pattern

KV Descriptors are based on the [mediator pattern](https://en.wikipedia.org/wiki/Mediator_pattern), where plugins are decoupled and no longer communicate directly with each other. Instead, any interactions between plugins occur through the KV Scheduler mediator. 

Plugins need only provide CRUD callbacks, and describe their dependencies on other plugins through one or more KV Descriptors. The KV Scheduler can then plan operations without knowing what the graph vertices represent in the configured system. 

Furthermore, the set of supported configuration items can grow without altering the transaction processing engine, or increasing the complexity of any of the components. You just need to implement and register new KV Descriptors.

---

### Terminology

The KV Scheduler's graph-based representation uses the following terminology to describe the higher level abstractions of specific objects:

* **Model** represents a single configuration item type such as interface, route, or bridge domain. For more details, see the  [User Guide model section](../user-guide/concepts.md#what-is-a-model). To look over an implementation, see the [bridge domain model][bd-model-example].
</br>
</br>
  
* **Value** (`proto.Message`) is a run-time instance of a given model. For more details, see the [proto.Message section ](../user-guide/concepts.md#protomessage) of the User Guide.
</br>
</br>

* **Key** (`string`) identifies a specific and value built from the model specification and value attributes. For more information, see the [keys section](../user-guide/concepts.md#keys) of User Guide.
</br>
</br>

* **Label** (`string`) identifies a unique value associated with the same type such as interface name. 
</br>
</br>

* **Value State** (`enum ValueState`) defines the operational state of a value. For example, a value can be successfully `CONFIGURED`, `PENDING` due to unmet dependencies, or `FAILED` after the last CRUD operation returned an error. [Value state proto][value-states] defines the possible value states.
</br>
</br>

* **Value Status** (`struct BaseValueStatus`) provides details on the executed operation, the last returned error, and the list of unmet dependencies. The status of one or more values and their updates can be read and watched for via the [KVScheduler API][value-states-api]. More information on this API can be found [below](#api).
</br>
</br>

* **Metadata** (`interface{}`) is additional run-time information of an undefined type assigned to a value. It is updated following a CRUD operation or agent restart. An example of metadata is the [sw_if_index][vpp-iface-idx], which is maintained for every interface alongside its value.
</br>
</br>

* **Metadata Map**, also known as index-map, implements the mapping between a value label and its metadata for a given value type. It is exposed in read-only mode to allow other plugins to read and reference metadata. For example, the interface plugin exposes its metadata map [here][vpp-iface-map]. It is used by the ARP plugin, the route plugin, and other plugins to read the sw_if_index of target interfaces. KV Scheduler creates and updates Metadata maps.
</br>
</br>

* **[Value origin][value-origin]** defines the source of the value. For example, it could be configuration data received from the NB, or data that was automatically created in the SB such as default routes and the loop0 interface.
</br>
</br>

* **Key-value pairs** are manipulated by CRUD operations.
</br>
</br>

* **Dependency** is defined for a value, and it references another key-value pair that must be created and exist before the dependent value can be created. If a dependency is not satisfied, the dependent value must remain cached in the `PENDING` state. Multiple dependencies can be defined for a dependent value; All dependencies must be satisfied before the dependent value can be created.
</br>
</br>

* **Derived value** is a single field of the original value or its property. It is manipulated separately (possibly using its own dependencies) through custom CRUD operations. It can be used as a target for dependencies of other key-value pairs. For example, every [interface to be assigned to a bridge domain][bd-interface] is treated as a [separate key-value pair][bd-derived-vals], dependent on the [target interface to be created first][bd-iface-deps], but otherwise not blocking the rest of the bridge domain to be programmed. See this [control-flow][bd-cfd] demonstrating the order of operations needed to create a bridge domain.
</br>
</br>

* **Graph** of values is KV Scheduler-internal in-memory storage for all configured and pending key-value pairs.  Graph edges represent inter-value relations, such as "depends-on" or "is-derived-from", and graph nodes are the key-value pairs themselves.


!!! note    
    Plugin configurators no longer need to implement their own caches for pending values.
  

* **Graph Refresh** is the process of updating the graph content to reflect the real SB state. This is achieved by calling the `Retrieve` function of every descriptor that supports this operation, and adding/updating graph vertices with the retrieved values. Refresh is performed just before the [Full or Downstream resync](#resync). Or after a failed CRUD operation for only those vertices impacted by the failure.
</br>
</br>

* **KV Descriptor** implements CRUD operations and defines derived values and dependencies for a single value type. To learn more, please read how to [implement your own KVDescriptor](kvdescriptor.md).

---

### Dependencies

The KV Scheduler must learn about two types of relationships between values utilized by the scheduling algorithm.

1. `A` **depends on** `B`:

- `A` cannot exist without `B`.
</br>
</br>

- a request to _create_ `A` without the existence of `B` must be postponed by marking `A` as `PENDING` in the in-memory graph.
</br>
</br>

- If `B` needs to be removed and `A` exists, `A` must be removed first and set to the `PENDING` state in case `B` is restored in the future.

!!! note
    Values obtained from SB via notifications are not checked for dependencies

---

2. `B` **is derived from** `A`:

- value `B` is not created directly by either NB or SB. Rather, it is derived from a base value `A` using the `DerivedValues()` method of the descriptor for `A`.
</br>
</br>

- a derived value `B` exists for only as long as its base `A` exists. It is removed immediately when the base value `A` disappears.
</br>
</br>

- a derived value may be represented by a descriptor different from the base value. It usually represents some property of the base value that other values may depend on, or an extra action to be taken when additional dependencies are met.

---

### Diagram

The diagram below shows the interactions between the KV Scheduler and the layers above and below. Using [Bridge Domain][bd-cfd] as an example, it depicts both the dependency and the derivation relationships, together with a pending value (of unspecified type) waiting for an interface to be created first.

![KVScheduler diagram](../img/developer-guide/kvscheduler.svg)

---

### Resync

Plugins are not required to implement their own resync mechanisms. By providing a KV Descriptor with CRUD operation callbacks to the KV Scheduler, a plugin "teaches" the KV Scheduler how to handle the plugin's configuration items. The KV Scheduler can determine and execute the set of operations needed to perform complete resynchronization.

---

The KV Scheduler further enhances the concept of state reconciliation by defining three resync types: full resync, upstream resync, and downstream resync.

- **Full resync:**
    - Re-reads intended configuration from NB.
    - Refreshes SB view via one or more `Retrieve` operations.
    - Resolves inconsistencies using `Create`\\`Delete`\\`Update` operations.
</br>
</br>

- **Upstream resync:**
    - Partial resync, similar to the full resync.
    - _DOES NOT_ refresh SB view, and assumes SB view is up-to-date and/or not required in the resync. This is because it may be easier to re-calculate the intended state rather than determine the minimal difference.
</br>
</br>

- **Downstream resync:** 
     - Partial resync, similar to the full resync
     - _DOES NOT_ re-read intended configuration from NB, and assumes it is up-to-date  
     -  Used periodically to resync, even without interacting with the NB.

---

### Transactions

The KV Scheduler can group related changes and apply them as a transaction. This is not supported by all NB interfaces. For example, changes from the etcd data store are received one at a time. 

To leverage the transaction support, you must use the [localclient](../user-guide/concepts.md#client-v2) for the same process, or [gRPC](https://github.com/ligato/cn-infra/tree/master/rpc/grpc) for remote access.  

Inside the KV Scheduler, transactions are queued and executed synchronously. This simplifies the algorithm and avoids concurrency issues. 

---

The KV Scheduler splits transaction processing into two stages: simulation and execution.

- **Simulation:** 
    - Set of sequenced operations referred to as the *transaction plan*.
    - _DOES NOT_ perform CRUD callbacks to descriptors.
    - Assumes no failures. 
</br>
</br>
- **Execution:**
    - Set of executed operations in the correct order. 
    - If any operation fails, reverts applied changes, unless the `BestEffort` mode is enabled. In this case, the KV Scheduler attempts to apply the maximum possible set of required changes. `BestEffort` is the default for resync.

Following simulation, the KV Scheduler generates transaction metadata and the transaction plan. This process happens before execution to inform you of pending operations. Note this process occurs even if any of the operations cause the agent to crash. After the transaction has executed, the set of completed operations and encountered errors is printed.

The example below shows a transaction output log. Observe that the planned operations match the executed operations which indicates no errors . 
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

// here you would potentially see logs from C(R)UD operations           
          
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

### KV Scheduler API

A separate [sub-package "api"][kvscheduler-api-dir] of the [KV Scheduler plugin][plugin] defines the KV Scheduler API. 

Multiple files define interfaces that constitute the KV Scheduler API. The table below includes file name, repo pointer along with a brief description. 

| File  |  Description |
|---|---|
|[errors.go][errors-go]|defines errors returned from the KV Scheduler. `InvalidValueError` error wrapper allows plugins to specify the specific reason for a validation error.|
|[kv_scheduler_api.go][kv-scheduler-api]  |NB uses for transaction commit, or read and watch for value status updates; SB uses to push notifications.|
|[kv_descriptor_api.go][kv-descriptor-api] | defines the KV Descriptor interface. |
|[txn_options.go][txn-options-go] |Set of available options for transactions.  |
|[txn_record.go][txn-record-go] | Type definition used to store processed transaction records.  |
|value_status.proto  |Operational value status proto. [API][value-states-api] can read the current status of one or more values, and watch for updates through a channel|
|value_status.pb.go  | Go code generated from value status proto.  |
|value_status.go | extends `value_status.pb.go` to implement proper (un)marshalling for proto.Messages. |

 * `errors.go`: defines errors that can be returned from within the KV Scheduler; additionally the `InvalidValueError` error wrapper is defined to allow plugins to further specify the reason for a validation error, which is then exposed through the [value status][value-states].

 * `kv_scheduler_api.go`: the KV Scheduler API used by: NB to commit a transaction, or to read and watch for value status updates; SB to push notifications regarding values that are, automatically or externally, created and/or updated.

 * `kv_descriptor_api.go`: defines the KV Descriptor interface. The detailed description can be found [here][kvdescriptor-guide].

 * `txn_options.go`: a set of available options for transactions. For example, a transaction can be customized to permit retries of failed operations. Retry delay period and maximum retry attempts can be configured.

 * `txn_record.go`: type definition used to store a record of a processed transaction. These records can be obtained using the [GetTransactionHistory][get-history-api] method, or through the [REST API](#rest-api).

 * `value_status.proto`: operational value status defined using protobuf. The [API][value-states-api] can read the current status of one or more values, and watch for updates through a channel.

 * `value_status.pb.go`: Go code generated from `value_status.proto`.

 * `value_status.go`: further extends `value_status.pb.go` to implement proper (un)marshalling for proto.Messages.

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

[plugin]: https://github.com/ligato/vpp-agent/tree/master/plugins/kvscheduler
[bd-cfd]: control-flows.md#example-bridge-domain
[bd-model-example]: https://github.com/ligato/vpp-agent/blob/e8e54ef67b666e57ffef1bca555c8ce5585f215f/api/models/vpp/l2/keys.go#L27-L31
[vpp-iface-idx]: https://github.com/ligato/vpp-agent/blob/e8e54ef67b666e57ffef1bca555c8ce5585f215f/plugins/vpp/ifplugin/ifaceidx/ifaceidx.go#L62
[vpp-iface-map]: https://github.com/ligato/vpp-agent/blob/e8e54ef67b666e57ffef1bca555c8ce5585f215f/plugins/vpp/ifplugin/ifplugin_api.go#L27
[value-origin]: https://github.com/ligato/vpp-agent/blob/e8e54ef67b666e57ffef1bca555c8ce5585f215f/plugins/kvscheduler/api/kv_descriptor_api.go#L53
[bd-interface]: https://github.com/ligato/vpp-agent/blob/e8e54ef67b666e57ffef1bca555c8ce5585f215f/api/models/vpp/l2/bridge-domain.proto#L19-L24
[bd-derived-vals]: https://github.com/ligato/vpp-agent/blob/e8e54ef67b666e57ffef1bca555c8ce5585f215f/plugins/vpp/l2plugin/descriptor/bridgedomain.go#L242-L251
[bd-iface-deps]: https://github.com/ligato/vpp-agent/blob/e8e54ef67b666e57ffef1bca555c8ce5585f215f/plugins/vpp/l2plugin/descriptor/bd_interface.go#L120-L127
[contiv-vpp]: https://github.com/contiv/vpp/
[graph-example]: ../img/developer-guide/large-graph-example.svg
[model-registration]: model-registration.md
[clientv2]: https://github.com/ligato/vpp-agent/tree/master/clientv2
[value-states]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/kvscheduler/value_status.proto
[value-states-api]: https://github.com/ligato/vpp-agent/blob/e8e54ef67b666e57ffef1bca555c8ce5585f215f/plugins/kvscheduler/api/kv_scheduler_api.go#L233-L239
[kvscheduler-api-dir]: https://github.com/ligato/vpp-agent/tree/master/plugins/kvscheduler/api
[kvdescriptor-guide]: kvdescriptor.md
[get-history-api]: https://github.com/ligato/vpp-agent/blob/e8e54ef67b666e57ffef1bca555c8ce5585f215f/plugins/kvscheduler/api/kv_scheduler_api.go#L241-L244
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
[errors-go]: https://github.com/ligato/vpp-agent/blob/master/plugins/kvscheduler/api/errors.go
[txn-options-go]: https://github.com/ligato/vpp-agent/blob/master/plugins/kvscheduler/api/txn_options.go
[txn-record-go]: https://github.com/ligato/vpp-agent/blob/master/plugins/kvscheduler/api/txn_record.go
[kv-descriptor-api]: https://github.com/ligato/vpp-agent/blob/master/plugins/kvscheduler/api/kv_descriptor_api.go
[kv-scheduler-api]: https://github.com/ligato/vpp-agent/blob/master/plugins/kvscheduler/api/kv_scheduler_api.go
  

*[ARP]: Address Resolution Protocol
*[REST]: Representational State Transfer
*[VPP]: Vector Packet Processing