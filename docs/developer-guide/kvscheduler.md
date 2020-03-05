# KV Scheduler

---

### Introduction

The KV Scheduler is a transaction-based configuration processing system. It includes a generic mechanism for dependency resolution between configuration items. The KV Scheduler is shipped as a separate [plugin], and is a core component used by all VPP and Linux plugins.

### Motivation

The KV Scheduler addresses several challenges encountered in the original vpp-agent design, which became apparent as the variety and complexity of different configuration items increased.

* `vpp` and `linux` plugins became bloated and complicated, suffering from race conditions and a lack of visibility.

* `configurator` components of `vpp` and `linux` plugins, each processing a specific configuration item type (e.g. interface, route, etc.) were built from scratch, solving the same set of problems again and again with frequent code duplication.

* plugin configurators would communicate with each other through notifications, and react to changes asynchronously to ensure proper operational sequence. Dependency resolution was distributed across all configurators, making it difficult to understand, predict and stabilize the system behavior.

The result was an unreliable and unpredictable re-synchronization (resync) occurring between the desired configuration state and the runtime configuration state.


!!! Terminology
    `Northbound (NB)` describes the desired or intended configuration state. `Southbound (SB)` describes the actual run-time configuration state. `Resync` is also referred to as state reconciliation. `CRUD` stands for create, read, update and delete. It standard nomenclature describing the basic actions performed by APIs. The KV Scheduler is referred to as the scheduler. KV Descriptors are also referred to as just `descriptors`.

### Basic concepts

The KV Scheduler uses graph theory concepts to manage dependencies between configuration items, and the order by which they are programmed into the network. A level of abstraction is built on top of the plugins, where the state of the system is modeled as a graph. Configuration items are represented as vertices and the relationship between them are represented as edges. The graph is then walked through to generate transaction plans, refresh the state, and perform resync.

The transaction plan that is prepared using the graph representation consists of a series of CRUD operations that can be executed on graph vertices. To abstract away from specific configuration items and accompanying details, graph vertices are "described" to the KV Scheduler using [KVDescriptors][kvdescriptor-guide]. KVDescriptors are basically handlers, each assigned to a distinct subset of graph vertices. They provide the KV Scheduler with pointers to callbacks that implement CRUD operations.

### Mediator Pattern

KV Descriptors are based on the [mediator pattern](https://en.wikipedia.org/wiki/Mediator_pattern), where plugins are decoupled and no longer communicate directly with each other. Instead, interactions between plugins are handled through the KV Scheduler mediator.

Plugins need only provide CRUD callbacks, and describe their dependencies on other plugins through one or more KVDescriptors. The KV Scheduler is then able to plan operations without knowing what the graph vertices represent in the configured system. Furthermore, the set of supported configuration items can be extended without altering the transaction processing engine or increasing the complexity of any of the components. All that is required is to implement and register new KVDescriptors.

### Terminology

The graph-based representation uses the following terminology to describe the higher level abstractions of specific objects.

* **Model** builds a representation of a single configuration item type such as interface, route, or bridge domain. Details on models and the model specification that includes proto. Messages can be found [here](../user-guide/concepts.md#what-is-a-model). The [bridge domain model][bd-model-example] is an example. Details regarding model registration can be found [here][model-registration].
  
* **Value** (`proto.Message`) is a run-time instance of a given model. Details on proto.Message can be found [here](../user-guide/concepts.md).

* **Key** (`string`) identifies a specific value that is built using the model specification and value attributes that identify a unique instance. More on keys can be found  [here](../user-guide/concepts.md#keys).

* **Label** (`string`) provides an identifier unique only across the value of the same type (e.g. interface name). It is generated from the model specification and value fields.

* **Value State** (`enum ValueState`) is the operational state of a value. For example, a value can be successfully `CONFIGURED`, `PENDING` due to unmet dependencies, or `FAILED` after the last CRUD operation returned an error. The set of value states is defined using the protobuf enumerated type [here][value-states].

* **Value Status** (`struct BaseValueStatus`) includes additional details such as the last executed operation, the last returned error, and the list of unmet dependencies. The status of one or more values and their updates can be read and watched for via the [KVScheduler API][value-states-api]. More information on this API can be found [below](#api).

* **Metadata** (`interface{}` is additional run-time information of an undefined type that is assigned to a value. It is updated following a CRUD operation or agent restart. An example of metadata is the [sw_if_index][vpp-iface-idx], which is maintained for every interface alongside its value.

* **Metadata Map**, also known as index-map, implements the mapping between a value label and its metadata for a given value type. It is exposed in read-only mode to allow other plugins to read and reference metadata. For example, the interface plugin exposes its metadata map [here][vpp-iface-map]. It is used by the ARP plugin, the route plugin, and other plugins to read the sw_if_index of target interfaces. Metadata maps are automatically created and updated by the KV Scheduler.

* **[Value origin][value-origin]** defines the source of the value. For example, it could be configuration data received from the NB, or data that was automatically created in the SB such as default routes and the loop0 interface.

* **Key-value pairs** are manipulated by CRUD operations.

* **Dependency** is defined for a value, and it references another key-value pair that must be created and exist before the dependent value can be created. If a dependency is not satisfied, the dependent value must remain cached in the `PENDING` state. Multiple dependencies can be defined for a dependent value; All dependencies must be satisfied before the dependent value can be created.

* **Derived value** is a single field of the original value or its property. It is manipulated separately (possibly using its own dependencies) through custom CRUD operations. It can be used as a target for dependencies of other key-value pairs. For example, every [interface to be assigned to a bridge domain][bd-interface] is treated as a [separate key-value pair][bd-derived-vals], dependent on the [target interface to be created first][bd-iface-deps], but otherwise not blocking the rest of the bridge domain to be programmed. See this [control-flow][bd-cfd] demonstrating the order of operations needed to create a bridge domain.

* **Graph** of values is KV Scheduler-internal in-memory storage for all configured and pending key-value pairs.  Graph edges represent inter-value relations, such as "depends-on" or "is-derived-from", and graph nodes are the key-value pairs themselves.

!!! note    
    Plugin configurators no longer need to implement their own caches for pending values.
  
* **Graph Refresh** is the process of updating the graph content to reflect the real SB state. This is achieved by calling the `Retrieve` function of every descriptor that supports this operation, and adding/updating graph vertices with the retrieved values. Refresh is performed just before the [Full or Downstream resync](#resync). Or after a failed CRUD operation for only those vertices impacted by the failure.

* **KV Descriptor** implements CRUD operations and defines derived values and dependencies for a single value type. To learn more, please read how to [implement your own KVDescriptor](kvdescriptor.md).

### Dependencies

The KV Scheduler must learn about two types of relationships between values utilized by the scheduling algorithm.

1. `A` **depends on** `B`:

- `A` cannot exist without `B`.
- a request to _create_ `A` without the existence of `B` must be postponed by marking `A` as `PENDING` in the in-memory graph.
- If `B` needs to be removed and `A` exists, `A` must be removed first and set to the `PENDING` state in case `B` is restored in the future.

!!! note
    Values obtained from SB via notifications are not checked for dependencies
2. `B` **is derived from** `A`:

- value `B` is not created directly by either NB or SB. Rather, it is derived from a base value `A` using the `DerivedValues()` method of the descriptor for `A`.
- a derived value `B` exists for only as long as its base `A` exists. It is removed immediately when the base value `A` disappears.
- a derived value may be represented by a descriptor different from the base value. It usually represents some property of the base value that other values may depend on, or an extra action to be taken when additional dependencies are met.

### Diagram

The following diagram shows the interactions between the KV Scheduler and the layers above and below. Using [Bridge Domain][bd-cfd] as an example, it depicts both the dependency and the derivation relationships, together with a pending value (of unspecified type) waiting for an interface to be created first.

![KVScheduler diagram](../img/developer-guide/kvscheduler.svg)

### Resync

Plugins no longer need to implement their own resync mechanisms. By providing a KV Descriptor with CRUD operation callbacks to the KV Scheduler, a plugin "teaches" the KV Scheduler how to handle the plugin's configuration items. The KV Scheduler can determine and execute the set of operations needed to perform complete resynchronization.

The KV Scheduler further enhances the concept of state reconciliation by defining three resync types:

* **Full resync**: the intended configuration is re-read from NB, the view of SB is refreshed via one or more `Retrieve` operations. Inconsistencies are resolved using `Create`\\`Delete`\\`Update` operations.
* **Upstream resync**: a partial resync, similar to the Full resync, except the view of SB state is not refreshed. It is either assumed to be up-to-date and/or is not required in the resync. This is because it may be easier to re-calculate the intended state rather than determine the minimal difference.
* **Downstream resync**: a partial resync, similar to the Full resync, except the intended configuration is assumed to be up-to-date and will not be re-read from the NB. Downstream resync can be used periodically to resync, even without interacting with the NB.

### Transactions

The KV Scheduler can group related changes and apply them as a transaction. This is not supported, however, by all agent NB interfaces. For example, changes from the `etcd` data store are always received one at a time. To leverage the transaction support, localclient (the same process) or gRPC API (remote access) must be used instead. Both are defined [here][clientv2].

Inside the KV Scheduler, transactions are queued and executed synchronously to simplify the algorithm and avoid concurrency issues. The processing of a transaction is split into two stages:

* **Simulation**: set of sequenced operations (so-called *transaction plan*), without actually calling any CRUD callbacks from the descriptors. This assumes no failures.
* **Execution**: executing the operations in the correct order. If any operation fails, applied changes are reverted, unless the `BestEffort` mode is enabled. In this case, the KV Scheduler attempts to apply the maximum possible set of required changes. `BestEffort` is the  default for resync.

Following simulation, transaction metadata (sequence number printed as `#xxx`, description, values to apply, etc.) is generated, together with the transaction plan. This is done before execution, so that the user is informed of pending operations. This occurs even if any of the operations cause the agent to crash. After the transaction has executed, the set of completed operations and encountered errors is printed.

An example transaction output log is shown below. There are no errors, therefore the planned operations match the executed operations:
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

### API

The KV Scheduler API is defined in a separate [sub-package "api"][kvscheduler-api-dir] of the [plugin][plugin]. The interfaces that constitute the KV Scheduler API are defined in multiple files:

 * `errors.go`: definitions of errors that can be returned from within the KV Scheduler; additionally the `InvalidValueError` error wrapper is defined to allow plugins to further specify the reason for a validation error, which is then exposed through the [value status][value-states].

 * `kv_scheduler_api.go`: the KV Scheduler API used by: NB to commit a transaction, or to read and watch for value status updates; SB to push notifications regarding values that are, automatically or externally, created and/or updated.

 * `kv_descriptor_api.go`: defines the KV Descriptor interface. The detailed description can be found [here][kvdescriptor-guide].

 * `txn_options.go`: a set of available options for transactions. For example, a transaction can be customized to permit retries of failed operations. Retry delay period and maximum retry attempts can be configured.

 * `txn_record.go`: type definition used to store a record of a processed transaction. These records can be obtained using the [GetTransactionHistory][get-history-api] method, or through the [REST API](#rest-api).

 * `value_status.proto`: operational value status defined using protobuf. The [API][value-states-api] can read the current status of one or more values, and watch for updates through a channel.

 * `value_status.pb.go`: Go code generated from `value_status.proto`.

 * `value_status.go`: further extends `value_status.pb.go` to implement proper (un)marshalling for proto.Messages.


### REST API

The KV Scheduler exposes the state of the system and the history of operations through formatted logs and a set of REST APIs.

* **graph visualization**: `GET /scheduler/graph`
    - returns a graph visualization plotted into an SVG image using [Graphviz](http://graphviz.org/). This can be displayed using any modern web browser.
    - example: [graph][graph-example] rendered for the [Contiv-VPP][contiv-vpp] project.
    - *requirements*: `dot` renderer; can be installed on Ubuntu with: `apt-get install graphviz`. This not needed when argument `format=dot` is used.
    - args:
        - `format=dot`: if defined, the API returns a plain graph description in the DOT format, which is available for further processing and customized rendering.
        - `txn=<txn-number>`: visualize the state of the graph as it was at the time when the given transaction had just completed. Vertices updated by the transaction are highlighted using a colored border.
* **transaction history**: `GET /scheduler/txn-history`
    - returns the full history of executed transactions, or for a given window of time.
    - args:
        - `format=<json/text>`
        - `seq-num=<txn-seq-num>`: transaction sequence number
        - `since=<unix-timestamp>`: if undefined, the output starts with the oldest kept record
        - `until=<unix-timestamp>`: if undefined, the output ends with the last executed transaction
* **key timeline**: `GET /scheduler/key-timeline`
     - args:
        - `key=<key-without-agent-prefix>`: key of the value to show changes over time
* **graph snapshot**: `GET /scheduler/graph-snaphost`
    - this is an internal representation that is not using `dot`
    - args:
        - `time=<unix-timestamp>`: if undefined, current state is returned
* **dump values**: `GET /scheduler/dump`
    - Index page is printed if args are `NOT` present
    - args:
        - `descriptor=<descriptor-name>`: dump values in the scope of the given descriptor
        - `key-prefix=<key-prefix>`: dump values with the given key prefix
        - `view=<NB, SB, internal>`: whether to dump intended, actual or the configuration state as known to  the KV Scheduler
* **request downstream resync**: `POST /scheduler/downstream-resync`
    - args:
        - `retry=< 1/true | 0/false >`: retry operations that failed
          during resync
        - `verbose=< 1/true | 0/false >`: print graph after refresh (Retrieve)

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

*[ARP]: Address Resolution Protocol
*[REST]: Representational State Transfer
*[VPP]: Vector Packet Processing