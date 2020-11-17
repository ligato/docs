# KV Scheduler

---

This section describes the KV Scheduler.

Package references: [kvscheduler](https://godoc.org/github.com/ligato/vpp-agent/plugins/kvscheduler), [api](https://godoc.org/github.com/ligato/vpp-agent/plugins/kvscheduler/api), [graph](https://godoc.org/github.com/ligato/vpp-agent/plugins/kvscheduler/internal/graph) 

---

### Introduction

The KV Scheduler core plugin supports dependency resolution, and computes the proper programming sequence for multiple interdependent configuration items. It works with VPP and Linux agents on the southbound (SB) side, and external data stores and rpc clients on the northbound (NB) side. 

---

### Motivation

The KV Scheduler addresses several challenges observed in the original VPP agent design. These challenges arose as the variety and complexity of different configuration items increased.

* VPP and Linux plugins became bloated, complicated, and suffered from race conditions and a lack of visibility.
</br>
</br>

* VPP and Linux plugin `configurator` components, each processing a specific configuration item type, were built from scratch, solving the same set of problems, with frequent code duplication.
</br>
</br>

* Plugin configurators would communicate with each other through notifications, and react to changes asynchronously to ensure a proper operational sequence. Dependency resolution processing was distributed across all configurators, making it difficult to understand, predict, and stabilize the system behavior.

Re-synchronization (resync) occurring between the desired northbound (NB) configuration state and the southbound (SB) runtime configuration state became unreliable and unpredictable.

---

!!! Terminology
    `Northbound (NB)` describes the desired or intended configuration state. `Southbound (SB)` describes the actual runtime configuration state. `Resync` is also referred to as state reconciliation. `CRUD` stands for create, read, update and delete. KV Descriptors are referred to as `descriptors`.

---

### Basic Concepts

Internally, the KV Scheduler builds a graph to model system state. The vertices represent configuration items; the edges represent relationships between the configuration items. The KV Scheduler walks the tree to mark dependencies, and compute the sequence of programming actions to perform. In addition, it builds transaction plans, refreshes state, and performs resync. 

The KV Scheduler generates a transaction plan that drives CRUD operations to the VPP or Linux agents in the SB direction. It will cache configuration items until outstanding dependencies are resolved. It coordinates and performs partial or full state synchronization for NB and SB. 

To abstract away from the details of specific configuration items, graph vertices are "described" to the KV Scheduler using KV Descriptors. You can think of a KV Descriptor as a handler, each supporting a distinct subset of graph vertices. KV Descriptors provide the KV Scheduler with pointers to callbacks that implement CRUD operations. 
</br>

![kvs-system](../img/user-guide/2components-ligato-framework-arch-KVS2.svg)

</br>You can retrieve information for the KV Scheduler using [agentctl][agentctl] and [REST APIs][kvs-rest-api].

Note that you can use the KV Scheduler system for any application containing an object with other dependent objects. 


---

### Mediator Pattern

KV Descriptors employ a [mediator pattern](https://en.wikipedia.org/wiki/Mediator_pattern), where plugins are decoupled and no longer communicate directly with each other. Instead, any interactions between plugins occur through the KV Scheduler mediator. 

Plugins provide CRUD callbacks, and describe their dependencies on other plugins through one or more KV Descriptors. The KV Scheduler can then plan operations without knowing what the graph vertices stand for in the system. 

Furthermore, the number and variety of configuration items can grow without altering the transaction processing engine, or increasing the complexity of the components in your application. You just need to implement and register new KV Descriptors.

---

### Terminology

The KV Scheduler's graph-based representation uses the following terminology: 

* **Model** describes a single configuration item type such as an interface, route, or bridge domain. For more details, see the  [User Guide model section](../user-guide/concepts.md#what-is-a-model). To look over an implementation, see the [bridge domain model][bd-model-example].
</br>
</br>
  
* **Value** (`proto.Message`) is a runtime instance of a given model. For more details, see the [proto.Message section ](../user-guide/concepts.md#protomessage) of the User Guide.
</br>
</br>

* **Key** (`string`) identifies a specific indentifier built from the model specification and value attributes. For more information, see the [User Guide keys section](../user-guide/concepts.md#keys).
</br>
</br>

* **Label** (`string`) identifies a unique value associated with the same type such as interface name. 
</br>
</br>

* **Value State** (`enum ValueState`) defines the operational state of a value. For example, a value can be successfully `CONFIGURED`, `PENDING` due to unmet dependencies, or `FAILED` after the last CRUD operation returns an error. [Value state proto][value-states] lists the possible value states.
</br>
</br>

* **Value Status** (`struct BaseValueStatus`) contains details on the executed operation, the last returned error, and the list of unmet dependencies. The KV Scheduler API can read the value status, and watch for updates.
</br>
</br>

* **Metadata** (`interface{}`) consists of additional runtime information of an undefined type, and assigned to a value. Metadata updates follow a CRUD operation or agent restart. As an example, the KV Scheduler graph contains [sw_if_index][vpp-iface-idx] metadata for each interface. 
</br>
</br>

* **Metadata Map**, also known as index-map, implements the mapping between a value label and its metadata for a given value type. The KV Scheduler creates and updates metadata maps. Other plugins can read and reference the metadata in read-only mode. </br></br>For example, the interface plugin exposes its [interface metadata map][vpp-iface-map] containing the `sw_if-index` for each interface. The ARP plugin, the route plugin, and other plugins use the `sw_if_index` to reference specific interfaces. 
</br>
</br>

* **[Value origin][value-origin]** defines the source of the value. The value origin options are NB, SB and unknown. 
</br>
</br>

* **Key-value pairs** specify a key and associated value. CRUD operations such as `CREATE` manipulate key-value pairs. Every key-value pair must have at most one KV Descriptor.   
</br>
* **[Dependency](#dependencies)** represents a dependency value. It references another key-value pair that must be created and exist before the dependent value is created. </br></br>If a dependency is not satisfied, the dependent value remains cached in the `PENDING` state. Multiple dependencies can be defined for a dependent value. All dependencies must be satisfied before creating the dependent value.
</br>
</br>

* **Derived value** is single field associated with an original value or its property. Custom CRUD operations manipulate the derived value, possibly using its own dependencies. A derived value can serve as a target for dependencies of other key-value pairs. </br></br>For example, every [interface assigned to a bridge domain][bd-interface] is treated as a [separate key-value pair][bd-derived-vals], dependent on the [existing target interface][bd-iface-deps]. Additionally, it does not block the remaining bridge domain programming. See the [bridge domain control-flow][bd-cfd] demonstrating the order of operations required to create a bridge domain.
</br>
</br>

* **Graph** of values contains all configured and pending key-value pairs inside KV Scheduler-internal in-memory. Graph edges represent inter-value relations, such as "depends-on" or "is-derived-from". Graph nodes represent the key-value pairs.
</br>
</br>
  
* **Graph Refresh** updates the graph content to reflect the real SB state. This process calls the `Retrieve()` function of every descriptor supporting this operation. It adds or updates graph vertices with the retrieved values. Refresh executes just prior to [full or downstream resync](#resync), or after a CRUD operation failure impacts any vertices.  
</br>
</br>

* **KV Descriptor** implements CRUD operations and defines derived values and dependencies for a single value type. </br></br>To learn more, read how to [implement your own KVDescriptor](kvdescriptor.md). For an in-depth discussion, see the [KV Descriptors section](kvdescriptor.md).

To retrieve KV Scheduler graph details, use the [graph snapshot API](../api/api-kvs.md#graph-snapshot), or the [agentctl dump](../user-guide/agentctl.md#dump) command. 

---


- check can be and ed
- add active voice examples or comments
- update text from concepts section
- look over rest pointer in kvs troubleshooting

### Dependencies

The KV Scheduler must learn about two types of relationships between values when scheduling configuration updates for your application. 

1. `A` **depends on** `B`:

- `A` cannot exist without `B`.
</br>
</br>

- Your request to _create_ `A`, without the existence of `B`, must be postponed. `A` is marked `PENDING`.
</br>
</br>

- If `A` exists, and you need to remove `B`, you must first remove `A`.</br>`A` is marked `PENDING` in case you later restore `B`. 

---

2. `B` **is derived from** `A`:

- value `B` is _not_ created directly by either NB or SB. Rather, it is derived from a base value `A` using the `DerivedValues()` method of the `A` descriptor.
</br>
</br>

- a derived value `B` exists for only as long as its base `A` exists. </br>`B` is removed immediately when the base value `A` disappears.
</br>
</br>

- a derived value's descriptor can differ from the base value descriptor. You might have a base value property that other values depend on. Or you have an extra action to perform when additional dependencies are met.

You will use dependencies to implement descriptors and plugin lookup in your application.  

!!! note
    Values obtained from SB via notifications are not checked for dependencies

---

### Diagram

The diagram below illustrates the interactions between the KV Scheduler and the layers above and below. Using [Bridge Domain][bd-cfd] as an example, it depicts the dependency and the derivation relationships. It also shows a cached pending value of unspecified type waiting for the system to first create the interface.

![KVScheduler diagram](../img/developer-guide/kvscheduler.svg)

If you wish to re-create the transactions shown in the graph, use a combination of agentctl put, dump and config history commands. The [Quick Start](../user-guide/quickstart.md), [Plugin](../plugins/plugin-overview.md) and [Agentctl][agentctl] sections of this documentation provide examples for configuring interfaces and bridge domains. 

---

### Resync

You don't have to implement plugin-specific resync functions. By providing a KV Descriptor with CRUD operation callbacks to the KV Scheduler, your plugin "teaches" the KV Scheduler how to handle the plugin's configuration items. The KV Scheduler can then determine and execute the set of operations that perform a complete resynchronization.

---

The KV Scheduler further enhances the concept of state reconciliation by defining three resync types: full resync, upstream resync, and downstream resync.

- **Full resync:**
    - Re-reads intended configuration from NB.
    - Refreshes the SB view using one or more `Retrieve` operations.
    - Resolves inconsistencies using `Create`\\`Delete`\\`Update` operations.
</br>
</br>

- **Upstream resync:**
    - Partial resync, similar to the full resync.
    - _DOES NOT_ refresh SB view, and assumes the SB view is up-to-date and/or not required in the resync. Upstream resync excludes the SB refresh because it is easier to re-calculate the intended state, rather than determine the minimal difference.
</br>
</br>

- **Downstream resync:** 
     - Partial resync, similar to the full resync
     - _DOES NOT_ re-read intended configuration from NB, and assumes it is up-to-date  
     -  Used periodically to resync, even without interacting with the NB.

You can initiate a downstream resync using the [agentctl config resync](../user-guide/agentctl.md#config) command, or the [KV Scheduler resync REST API](../api/api-kvs.md#downstream-resync) 

---

### Transactions

The KV Scheduler groups and applies related changes as a transaction. However, not all NB interfaces support this function. For example, changes from the etcd data store are received one at a time. 

To leverage the transaction support, you must use the [localclient](../user-guide/concepts.md#client-v2) for the same process, or [gRPC](https://github.com/ligato/cn-infra/tree/master/rpc/grpc) for remote access.  

The KV Scheduler queues and asynchronously executes transactions. This simplifies the algorithm and avoids concurrency issues. 

---

The KV Scheduler splits [transaction processing](https://github.com/ligato/vpp-agent/blob/9e4fee3138c423ed33af72ca5ea5d41b868324c1/plugins/kvscheduler/txn_process.go#L117) into several stages. 

The primary functions of the simulation and execution stages consist of the following:

- **Simulation:** 
    - generates the transaction plan consisting of a set of _planned operations_.
    - _DOES NOT_ perform CRUD callbacks to descriptors.
    - Assumes no failures. 
</br>
</br>
- **Execution:**
    - Executes operations sorted in the correct order. 
    - If any operation fails, it reverts to applied changes, unless the `BestEffort` mode is enabled. In the best effort case, the KV Scheduler attempts to apply the maximum possible set of required changes. `BestEffort` is the default for resync.

Following simulation, The KV Scheduler generates transaction metadata, and the transaction plan. This process happens before execution to inform you of pending operations. Note this process occurs even if any of the operations cause the agent to crash. 

After the transaction has executed, the set of completed operations and any errors prints to a log. You can view the transaction logs using the [agentctl config history](../user-guide/agentctl.md#config) command, or the [KV Scheduler transaction history REST API](../api/api-kvs.md#transaction-history).

The example below shows an abbreviated transaction log. Observe that the planned operations match the executed operations. This indicates no errors. 
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

Multiple files describe interfaces that constitute the KV Scheduler API. The table below contains the file name and brief description. 

For more details on the specific files, click on the file name. 

| File  |  Description |
|---|---|
|[errors.go][errors-go]|defines errors returned from the KV Scheduler. `InvalidValueError` error wrapper allows plugins to specify the specific reason for a validation error.|
|[kv_scheduler_api.go][kv-scheduler-api]  |NB uses for transaction commit, or read and watch for value status updates; SB uses to push notifications.|
|[kv_descriptor_api.go][kv-descriptor-api] | defines the KV Descriptor interface. |
|[txn_options.go][txn-options-go] |Set of available options for transactions.  |
|[txn_record.go][txn-record-go] | Type definition for storing processed transaction records.  |
|[value_status.proto][value-status-proto]|Operational value status proto. [API][value-states-api] can read the current status of one or more values, and watch for updates through a channel|
|[value_status.pb.go][value-status-pb]| Go code generated from value status proto.  |
|[value_status.go][value-status] | extends `value_status.pb.go` to implement proper (un)marshalling for proto.Messages. |
 

---

### REST API

You can retrieve the KV Scheduler system state through a set of REST APIs. 

For more information, see the [KV Scheduler REST API Section][kv-scheduler-rest-api].

---

### Agentctl Dump

You can use the agentctl dump command to dump the running state of the KV Scheduler. 

For more information, see the [agentctl dump](../user-guide/agentctl.md#dump) command section of the User Guide. 



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
[value-status-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/kvscheduler/value_status.proto
[value-status-pb]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/kvscheduler/value_status.pb.go
[value-status]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/kvscheduler/value_status.go
[agentctl]: ../user-guide/agentctl.md
[kvs-rest-api]: ../api/api-kvs.md
  

*[ARP]: Address Resolution Protocol
*[REST]: Representational State Transfer
*[VPP]: Vector Packet Processing