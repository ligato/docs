# KV Scheduler

The **KVScheduler** is a major enhancement that warranted increasing the VPP Agent's major version  from 1.x to 2.x. It provides transaction-based configuration processing based on a generic mechanism for dependency resolution between different configuration items, which in effect simplifies and unifies the configurators. KVScheduler is shipped as a 
separate plugin, even though it is now a core component around which all the VPP and Linux configurators have been re-build.

### Motivation

The KVScheduler is the answer to a series of problems with the original VPP Agent design, which got gradually worse as the set of supported configuration items kept growing:

* plugins `vpp` and `linux` became bloated and complicated, suffering with race conditions and a lack of visibility

* the `configurators` - i.e. components of `vpp` and `linux` plugins, each processing a specific configuration item type (e.g. interface, route, etc.) - had to be effectively built from scratch, solving the same set of problems again and duplicating lot's of code

* configurators would communicate with each other only through notifications, and react to changes asynchronously to ensure proper operation ordering - i.e. the problem of dependency resolution was effectively distributed across all configurators, making it very difficult to understand, predict and stabilize the system behavior from the developer's 
global viewpoint

* The culmination of all the issues above was an unreliable and unpredictable re-synchronization (or resync for short), also known as state reconciliation, between NorthBound (the desired state) and SouthBound (the actual state) - something that was meant to be one of the main VPP-Agent value-adds.

### Basic concepts and terminology

KVScheduler solves problems common to all configurators through generalization - moving away from specific configuration items and describing the system with abstract terms:

* **Model** is a description for a single item type, defined as Protobuf Message (for example Bridge Domain model can be found [here][bd-model])

* **Value** is a run-time instance of a given model

* **Key** identifies specific value (it is built from key template defined for the model, e.g. L2 FIB key template)

* **Label** can be optionally defined to provide shorter identifier unique only across values of the same type (e.g. interface name)

* **Metadata** is extra run-time information (of undefined type) assigned to a value, which may be updated after a CRUD operation or an agent restart (for example [sw_if_index][vpp-iface-idx] of every interface is kept in the metadata for its key-value pair)

* **Metadata Map**, also known as index-map, implements mapping between value label and its metadata for a given value type - typically exposed in read-only mode to allow other plugins to read and reference metadata (for example, interface plugin exposes its metadata map [here][vpp-iface-map], which is then used by ARP, route plugin etc. to read 
sw_if_index of target interfaces).  Metadata maps are automatically created and updated by the scheduler (he is the owner), and exposed to other plugins only in the read-only mode. 

* **[Value origin][value-origin]** defines where the value came from - whether it was received from NB to be configured or whether it was created in the SB plane automatically (e.g. default routes, loop0 interface, etc.)

* Key-value pairs are operated with through CRUD operations, where **Add** is used to denote **Create**, **Dump** is basically Read, Update is denoted by the scheduler as **Modify** and **Delete** is borrowed unchanged

* **Dependency** defined for a value, references another key-value pair that must exist (be created), otherwise the associated value cannot be Added and must remain cached in the state **Pending** - value is allowed to have defined multiple dependencies and all must be satisfied for the value to be considered ready for creation.

* **Derived value**, in future release to be renamed to **attribute** for more clarity, is typically a single field of the original value or its property, manipulated separately - i.e. with possibly its own dependencies (dependency on the source value is implicit), custom implementations for CRUD operations and potentially used as target for dependencies 
of other key-value pairs - for example, every [interface to be assigned to a bridge domain][bd-interface] is treated as a [separate key-value pair][bd-derived-vals], dependent on the [target interface to be created first][bd-iface-deps], but otherwise not blocking the rest of the bridge domain to be applied

* **Graph** of values is a kvscheduler-internal in-memory storage for all configured and pending key-value pairs, with edges representing inter-value relations, such as "depends-on" and "is-derived-from" - configurators no longer have to implement their own caches for pending values

* **KVDescriptor** assigns implementations of CRUD operations and defines derived values and dependencies to a single value type - this is what configurators basically boil down to - to learn more, please read how to [implement your own KVDescriptor][kvdescriptor-dev-guide].

### Dependencies

The idea behind scheduler is based on the Mediator pattern - configurators do not communicate directly, but instead interact through the mediator. This reduces the dependencies between communicating objects, thereby reducing coupling.

The values are described for scheduler by registered KVDescriptor-s. The scheduler learns two kinds of relations between values that have to be respected by the scheduling algorithm:

1. `A` **depends on** `B`:
   - `A` cannot exist without `B`
   - request to add `A` without `B` existing must be postponed by marking `A` as `pending` (value with unmet dependencies) in the in-memory graph 
   - if `B` is to be removed and `A` exists, `A` must be removed first and set to `pending` state in case `B` is restored in the future
   - Note: values pushed from SB are not checked for dependencies
2. `B` **is derived from** `A`:
   - value `B` is not added directly (by NB or SB) but gets derived from base value `A` (using the DerivedValues() method of the base value's descriptor)
   - a derived value exists only as long as its base does and gets removed (immediately, not pending) once the base value goes away
   - a derived value may be described by a different descriptor than the base and usually represents the property of the base value (that other values may depend on) or an extra action to be taken when additional dependencies are met.

### Resync 

Configurators no longer have to implement resync on their own. As they "teach" KVScheduler how to operate with configuration items by providing callbacks to CRUD operations through KVDescriptors, the scheduler has all it needs to determine and execute the set of operation needed to get the state in-sync after a transaction or restart.

Furthermore, KVScheduler enhances the concept of state reconciliation, and defines three types of the resync:

* **Full resync**: the desired configuration is re-read from NB, the view of SB is refreshed via Dump operations and inconsistencies are resolved via Add/Delete/Modify operations
* **Upstream resync**: partial resync, same as Full resync except for the view of SB is assumed to be up-to-date and will not get refreshed - can be used by NB when it is easier to re-calculate the desired state than to determine the (minimal) difference
* **Downstream resync**: partial resync, same as Full resync except the desired configuration is assumed to be up-to-date and will not be re-read from NB - can be used periodically to resync, even without interacting with NB

### Transactions

The scheduler allows to group related changes and applies them as transactions. This is not supported, however, by all agent NB interfaces - for example, changes from `etcd` datastore are always received one a time. To leverage the transaction support, localclient (the same process) or GRPC API (remote access) have to be used instead.

Inside the scheduler, transactions are queued and executed synchronously to simplify the algorithm and avoid concurrency issues. The processing of a transaction is split into two stages:

* **Simulation**: the set of operations to execute and their order is determined (so-called *transaction plan*), without actually calling any CRUD callback from descriptors - assuming no failures.

* **Execution**: executing the operations in the right order. If any operation fails, the already applied changes are reverted, unless the so called `BestEffort` mode is enabled, in which case the scheduler tries to apply the maximum possible set of required changes. `BestEffort` is the default for resync.  

Right after simulation, transaction metadata (sequence number printed as `#xxx`, description, values to apply, etc.) are printed, together with transaction plan. This is done before execution, to ensure that the user is informed about the operations that were going to be executed even if any of the operations cause the agent to crash. After the 
transaction has executed, the set of actually executed operations and potentially some errors are printed to finalize the output for the transaction.

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
        - `state=<NB, SB, internal>` (in the next release will be renamed to `view`): whether to dump desired, actual 
        or the configuration state as known to kvscheduler
* **request downstream resync**: `POST /scheduler/downstream-resync`
    - args:
        - `retry=< 1/true | 0/false >`: allow to retry operations that failed during resync
        - `verbose=< 1/true | 0/false >`: print graph after refresh (Dump)

This page describes how to use the VPP-Agent with the gRPC

# Status Check

The `statuscheck` infrastructure plugin monitors the overall status of a CN-Infra based app by collecting and aggregating partial statuses of agents plugins. The status is exposed to external clients via [ETCD - datasync][datasync-plugin] and [HTTP][rest-plugin], as shown in the following diagram:

![status check][status-check-image]

### Overall Agent Status

The overall Agent Status is aggregated from all Plugins' Status (logical AND for each Plugin Status success/error).

The agent's current overall status can be retrieved from ETCD from the following key: 
`/vnf-agent/<agent-label>/check/status`

```
$ etcdctl get /vnf-agent/<agent-label>/check/status/v1/agent
/vnf-agent/<agent-label>/check/status/v1/agent
{"build_version":"e059fdfcd96565eb976a947b59ce56cfb7b1e8a0","build_date":"2017-06-16.14:59","state":1,"start_time":1497617981,"last_change":1497617981,"last_update":1497617991}
```

To verify the agent status via HTTP (e.g. for Kubernetes 
[liveness and readiness probes][kubernetes-probes], use the `/liveness` and `/readiness` URLs:
```
$ curl -X GET http://localhost:9191/liveness
{"build_version":"e059fdfcd96565eb976a947b59ce56cfb7b1e8a0","build_date":"2017-06-16.14:59","state":1,"start_time":1497617981,"last_change":1497617981,"last_update":1497617991}
$ curl -X GET http://localhost:9191/readiness
{"build_version":"e059fdfcd96565eb976a947b59ce56cfb7b1e8a0","build_date":"2017-06-16.14:59","state":1,"start_time":1497617981,"last_change":1497617981,"last_update":1497617991}
```

To change the HTTP server port (default `9191`), use the `http-port` option of the agent, e.g.:
```
$ vpp-agent -http-port 9090
```

### Plugin Status

Plugin may use `PluginStatusWriter.ReportStateChange` API to **PUSH** the status information at any time. For optimum performance, 'statuscheck' will then propagate the status report further to external clients only if it has changed since the last update.

Alternatively, plugin may chose to use the **PULL** based approach and define the `probe` function passed to `PluginStatusWriter.Register` API. `statuscheck` will then periodically probe the plugin for the current status. Once again, the status is propagated further only if it has changed since the last enquiry.

It is recommended not to mix the PULL and the PUSH based approach within the same plugin.

To retrieve the current status of a plugin from ETCD, use the following key template: `/vnf-agent/<agent-label>/check/status/v1/plugin/<PLUGIN_NAME>`
 
For example, to retrieve the status of the GoVPP plugin, use: 

```
$ etcdctl get /vnf-agent/<agent-label>/check/status/v1/plugin/GOVPP
/vnf-agent/<agent-label>/check/status/v1/plugin/GOVPP
{"state":2,"last_change":1496322205,"last_update":1496322361,"error":"VPP disconnected"}
```

Push Plugin Status:
![status check pull][status-check-pull-image]

Pull Plugins Status - PROBING:
![status check push][status-check-push-image]

# Index Map

The idxmap package provides an enhanced mapping structure to help in the following use cases:

* Exposing read-only access to plugin local data for other plugins
* Secondary indexing
* Data caching for key-value store (such as etcd)

For more detailed description see the godoc.

### Exposing plugin local information Use Case

App plugins often need to expose some structured information to other plugins inside the agent (see the following diagram). Structured data stored in idxmap are available for (possibly concurrent) read access to other plugins:

1. either via lookup using primary keys or secondary indices;
2. or via watching for data changes in the map using channels or callbacks (subscribe for changes and receive notification once an item is added, changed or removed).

![idxmap local][idx-map-local-image]

### Caching Use Case

It is useful to have the data from a key-value store cached when you need to:

- minimize the number of lookups into the key-value store
- execute lookups by secondary indexes for key-value stores that do not necessarily support secondary indexing (e.g. etcd)

**CacheHelper** turns `idxmap` (injected as field `IDX`) into an indexed local copy of remotely stored key-value data. `CacheHelper` watches the target key-value store for data changes and resync events. Received key-value pairs are transformed into the name-value (+ secondary indices if defined) pairs and stored into the injected idxmap instance. 
For a visual explanation, see the diagram below:

![idxmap cache][idx-map-cache-image]

The constructor that combines `CacheHelper` with `idxmap` to build the cache from the example can be found there as well.

### Examples

* Isolated and simplified examples can be found here: 
  * [lookup][idx-map-lookup-example]
  * [watch][idx-map-watcher-example]
  
# Log Manager

Log manager plugin allows to view and modify log levels of loggers using REST API.

**API**
- List all registered loggers:

    ```curl -X GET http://<host>:<port>/log/list```
- Set log level for a registered logger:
   ```curl -X PUT http://<host>:<port>/log/<logger-name>/<log-level>```
 
   `<log-level>` is one of `debug`,`info`,`warning`,`error`,`fatal`,`panic`
   
`<host>` and `<port>` are determined by configuration of rest.Plugin.
 
**Config file**

- Logger config file is composed of two parts: the default level applied for all plugins, and a map where every logger can have its own log level defined.
  
**Note:** initial log level can be set using environmental variable `INITIAL_LOGLVL`. The variable replaces default-level from configuration file. However, loggers (partial definition) replace default value set by environmental variable for specific loggers defined.  
  
### Tracer

A simple utility able to log measured time periods various events. To create a new tracer, call:

`t := NewTracer(name string, log logging.Logger)`

Tracer object can store a new entry with `t.LogTime(entity string, start Time)` where `entity` is a string representation of a measured object (name of a function, structure or just simple string) and `start` is a start time. Tracer can measure repeating event (in a loop for example). Every event will be stored with the particular index.

Use `t.Get()` to read all measurements. The Trace object contains a list of entries and overall time duration.

Last method is `t.Clear()` which removes all entries from the internal database.  

# Messaging/Kafka

The client package provides single purpose clients for publishing synchronous/asynchronous messages and for consuming selected topics.

The mux package uses these clients and allows to share their access to kafka brokers among multiple entities. This package also implements the generic messaging API defined in the parent package.

### Requirements

Minimal supported version of kafka is determined by [sarama][sarama] library - Kafka 0.10 and 0.9, although older releases are still likely to work.

If you don't have kafka installed locally you can use docker image for testing:
```
sudo docker run -p 2181:2181 -p 9092:9092 --name kafka --rm \
 --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 spotify/kafka
```

### Kafka plugin

Kafka plugin provides access to kafka brokers.

**Configuration**

- Location of the Kafka configuration file can be defined either by command line flag `kafka-config` or set via `KAFKA_CONFIG` env variable.

**Status Check**

- Kafka plugin has a mechanism to periodically check a connection status of the Kafka server. 

### Multiplexer

The multiplexer instance has an access to kafka Brokers. To share the access it allows to create connections. There are available two connection types one support message of type `[]byte` and the other `proto.Message`. Both of them allows to create several SyncPublishers and AsyncPublishers that implements `BytesPublisher` interface or `ProtoPubliser` 
respectively. The connections also provide API for consuming messages implementing `BytesMessage` interface or `ProtoMessage` respectively.

```
   
    +-----------------+                                  +---------------+
    |                 |                                  |               |
    |  Kafka brokers  |        +--------------+     +----| SyncPublisher |
    |                 |        |              |     |    |               |
    +--------^--------+    +---| Connection   <-----+    +---------------+
             |             |   |              |
   +---------+----------+  |   +--------------+
   |  Multiplexer       |  |
   |                    <--+
   | SyncProducer       <--+   +--------------+
   | AsyncProducer      |  |   |              |
   | Consumer           |  |   | Connection   <-----+    +----------------+
   |                    |  +---|              |     |    |                |
   |                    |      +--------------+     +----| AsyncPublisher |
   +--------------------+                                |                | 
                                                         +----------------+

```

# Process manager

The process manager plugin provides a set of methods to create a plugin-defined processes instance implementing a set of methods to manage and monitor them. There are several ways how to obtain a process instance via `ProcessManager` 
API:

**New process with options:** using method `NewProcess(<cmd>, <options>...)` which requires a command and a set of optional parameters. 
**New process from template:** using method `NewProcessFromTemplate(<tmp>)` which requires template as a parameter
**Attach to existing process:** using method `AttachProcess(<pid>)`. The process ID is required to order to attach.

### Management

Note: since application (management plugin) is parent of all processes, application termination causes all started processes to stop as well. This can be changed with *Detach* option (see process options).

Process management methods:

* `Start()` starts the plugin-defined process, stores the instance and does initial status file read
* `Restart()` tries to gracefully stop (force stop if fails) the process and starts it again. If the instance is not running, it is started.
* `Stop()` stops the instance using SIGTERM signal. Process is not guaranteed to be stopped. Note that child processes (not detached) may end up as defunct if stopped this way. 
* `StopAndWait()` stops the instance using SIGTERM signal and waits until the process completes. 
* `Kill()` force-stops the process using SIGKILL signal and releases all the resources used.
* `Wait()` waits until the process completes.
* `Signal()` allows user to send custom signal to a process. Note that some signals may cause unexpected behavior in process handling.

Process monitor methods:

* `IsAlive()` returns true if process is running
* `GetNotificationChan()` returns channel where process status notifications will be sent. Useful when a process is created via template with 'notify' field set to true. In other cases, the channel is provided by user.
* `GetName` returns process name as defined in status file
* `GetPid()` returns process ID
* `UpdateStatus()` updates internal status of the plugin and returns the actual status file
* `GetCommand()` returns original process command. Always empty for attached processes.
* `GetArguments()` returns original arguments the process was run with. Always empty for attached processes.
* `GetStartTime()` returns time stamp when the process was started for the last time
* `GetUpTime()` returns process up-time in nanoseconds

### Status watcher

Every process is watched for status changes (it does not matter which way it was crated) despite the process is running or not. The watcher uses standard statuses (running, sleeping, idle, etc.). The state is read from process status file and every change is reported. The plugin also defines two plugin-wide statues:
* **Initial** - only for newly created processes, means that the process command was defined but not started yet
* **Terminated** - if the process is not running or does not respond
* **Unavailable** - if the process is running but the status cannot be obtained
The process status is periodically polled and notifications can be sent to the user defined channel. In case process was crated via template, channel was initialized in the plugin and can be obtained via `GetNotificationChan()`.

### Process options

Following options are available for processes. All options can be defined in the API method as well as in the template. All of them are optional.

**Args:** takes string array as parameter, process will be started with given arguments. 

**Restarts:** takes a number as a parameter, defines count of automatic restarts when the process state becomes terminated.

**Writer** allows to define custom `stdOut` and `stdErr` values. Note: usability is limited when defined via template (only standard `os.stdout` and `os.stderr` can be used)

**Detach:** no parameters, started process detaches from the parent application and will be given to current user. This setting allows the process to run even after the parent was terminated.

**EnvVar** can be used to define environment variables (for example `os.Environ` for all)

**Template:** requires name, and run-on-startup flag. This setup creates a template on process creation. The template path has to be set in the plugin.

**Notify:** allows user to provide a notification channel for status changes.

### Templates

The template is a file which defines process configuration for plugin manager. All templates should be stored in the path defined in the plugin config file.

```
./process-manager-plugin -process-manager-config=<path-to-file>
```

The template can be either written by hand using proto model, or generated with the *Template* option while creating a new process. 

On the plugin init, all templates are read, and those with *run-on-startup* set to 'true' are also immediately started. The template contains several fields defining process name, command, arguments and all the other fields from options.

The plugin API allows to read templates directly with `GetTemplate(<name)` or `GetAllTmplates()`. The template object can be used as parameter to start a new process from it. 

# Service Label

The `servicelabel` is a small Core Agent Plugin, which other plugins can use to obtain the microservice label, i.e. the string used to identify the particular VNF. The label is primarily used to prefix keys in ETCD data store so that the configurations of different VNFs do not get mixed up.

**Configuration**

- the serviceLabel can be set either by commandline flag `microservice-label` or environment variable `MICROSERVICE_LABEL`

**Example**

Example of retrieving and using the microservice label:
```
plugin.Label = servicelabel.GetAgentLabel()
dbw.Watch(dataChan, cfg.SomeConfigKeyPrefix(plugin.Label))
```

[bd-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto
[bd-interface]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/model/l2/bd.proto#L14
[bd-derived-vals]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/l2plugin/descriptor/bridgedomain.go#L225
[bd-iface-deps]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/l2plugin/descriptor/bd_interface.go#L128
[datasync-plugin]: db-plugins.md#datasync-plugin
[idx-map-cache-image]: ../img/user-guide/idxmap_cache.png
[idx-map-local-image]: ../img/user-guide/idxmap_local.png
[idx-map-lookup-example]: https://github.com/ligato/vpp-agent/tree/master/examples/idx_mapping_lookup
[idx-map-watcher-example]: https://github.com/ligato/vpp-agent/tree/master/examples/idx_mapping_watcher
[kubernetes-probes]: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
[kvdescriptor-dev-guide]: ../developer-guide/kvdescriptor.md
[rest-plugin]: connection-plugins.md#vpp-agent-rest
[sarama]: https://github.com/Shopify/sarama
[status-check-image]: ../img/user-guide/status_check.png
[status-check-push-image]: ../img/user-guide/status_check_push.png
[status-check-pull-image]: ../img/user-guide/status_check_pull.png
[vpp-iface-idx]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/ifplugin/ifaceidx/ifaceidx.go#L62
[vpp-iface-map]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/ifplugin/ifplugin_api.go#L26
[value-origin]: https://github.com/ligato/vpp-agent/blob/dev/plugins/kvscheduler/api/kv_descriptor_api.go#L53
