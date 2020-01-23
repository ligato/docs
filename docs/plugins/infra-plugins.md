# Infra plugins

!!! Note
    The documentation in some cases may refer to the Ligato Infra as Cloud Native - Infra or CN-infra for short.

---

## Status Check

The `statuscheck` infra plugin monitors the overall status of a cn-infra based app by collecting and aggregating the status of agents plugins. The status is exposed to external clients via [ETCD - datasync][datasync-plugin] and [HTTP][rest-plugin]. 

See the `statuscheck API` figure below:

![status check][status-check-image]
<p style="text-align: center; font-weight: bold">StatusCheck API</p>

### Overall Agent Status

The overall agent status is aggregated from all plugins' status. Conceptually this is a  `logical AND` for each plugin's status.

The agent's current status can be retrieved from etcd using this key: 
```
/vnf-agent/<agent-label>/check/status`

```

Example:
```
$ etcdctl get /vnf-agent/<agent-label>/check/status/v1/agent
/vnf-agent/<agent-label>/check/status/v1/agent
{"build_version":"e059fdfcd96565eb976a947b59ce56cfb7b1e8a0","build_date":"2017-06-16.14:59","state":1,"start_time":1497617981,"last_change":1497617981,"last_update":1497617991}
```

To verify the agent status via HTTP, use the `/liveness` and `/readiness` URLs:
```
$ curl -X GET http://localhost:9191/liveness
{"build_version":"e059fdfcd96565eb976a947b59ce56cfb7b1e8a0","build_date":"2017-06-16.14:59","state":1,"start_time":1497617981,"last_change":1497617981,"last_update":1497617991}
$ curl -X GET http://localhost:9191/readiness
{"build_version":"e059fdfcd96565eb976a947b59ce56cfb7b1e8a0","build_date":"2017-06-16.14:59","state":1,"start_time":1497617981,"last_change":1497617981,"last_update":1497617991}
```

To change the HTTP server port (default `9191`), use the `http-port` option of the vpp-agent:
```
$ vpp-agent -http-port 9090
```

### Plugin Status

Plugins may use the `PluginStatusWriter.ReportStateChange` API to `push` status information at any time. For optimum performance, `statuscheck` will then propagate the status report to external clients if and only if it has changed since the last update. 

See the `Push Status` figure below. 

![status check push][status-check-push-image]
<p style="text-align: center; font-weight: bold">Push Status</p>

Alternatively, plugins may choose to use the `PULL`-based approach and define the `probe` function passed to `PluginStatusWriter.Register` API. `statuscheck` will then periodically probe the plugin for the current status. Once again, the status is propagated only if it has changed since the last enquiry. See the 'Push Status' figure below.


![status check pull][status-check-pull-image]
<p style="text-align: center; font-weight: bold">PullStatus</p>

It is recommended `NOT` to mix the `push` and the `pull` approaches within the same plugin.

To retrieve the current status of a plugin from etcd, use the following key template: `/vnf-agent/<agent-label>/check/status/v1/plugin/<PLUGIN_NAME>`
 
For example, to retrieve the status of the GoVPP plugin, use: 

```
$ etcdctl get /vnf-agent/<agent-label>/check/status/v1/plugin/GOVPP
/vnf-agent/<agent-label>/check/status/v1/plugin/GOVPP
{"state":2,"last_change":1496322205,"last_update":1496322361,"error":"VPP disconnected"}
```





## Index Map

The idxmap package provides a mapping structure to support the following use-cases:

* Exposing read-only access to plugin local data for other plugins
* Secondary indexing
* Data caching for a key-value store (such as etcd)

For more detailed description see the godoc reference [here][idxmap-godoc].

### Exposing plugin local data Use-Case

App plugins often need to expose some structured information to other plugins inside the agent (see the `Using idxmap to expose local data` figure below). Structured data stored in idxmap is available for read access to other plugins:

- via lookup using primary keys or secondary indices;
- via watching for data changes in the map using channels or callbacks. Subscribe for changes and receive notification once an item is added, changed or removed.

![idxmap local][idx-map-local-image]
<p style="text-align: center; font-weight: bold">Using idxmap to expose local data</p>

### Caching Use-Case

It is useful to cache data from a key-value store when you need to:

- minimize the number of lookups into the key-value store
- execute lookups by secondary indexes for key-value stores that do not necessarily support secondary indexing (e.g. etcd does not support this)

**CacheHelper** turns `idxmap` (injected as field `IDX`) into an indexed local copy of remotely stored key-value data. `CacheHelper` watches the target key-value store for data changes and resync events. Received key-value pairs are transformed into the name-value (+ secondary indices if defined) pairs and stored into the injected idxmap instance. 

See the `idxmap caching` figure below.
![idxmap cache][idx-map-cache-image]
<p style="text-align: center; font-weight: bold">idxmap caching</p>

  
## Log Manager


The log manager plugin allows one to view and modify the log levels of loggers using a REST API.

**APIs**

List all registered loggers:
```
curl -X GET http://<host>:<port>/log/list
```
    
Set log level for a registered logger:
```
curl -X PUT http://<host>:<port>/log/<logger-name>/<log-level>
```
 
where `<log-level>` is one of the following: `debug`,`info`,`warning`,`error`,`fatal`,`panic`
   
and `<host>` and `<port>` are determined by configuration of the REST plugin.
 
**Config file**

The logger config file is composed of two parts: 

- default level applied for all plugins.
- map where every logger can have its own log level defined.
 
!!! note    
    Initial log levels can be set using the environmental variable `INITIAL_LOGLVL` which overrides `default-level` defined in the [configuration file][log-level-config].   
  
### Tracer

Trace is a utility for logging events across time periods.

To create a new tracer, call:
```
t := NewTracer(name string, log logging.Logger)
```

Tracer object can store a new entry with: 

```
t.LogTime(entity string, start Time)
``` 
where `entity` is a string representation of a measured object (name of a function, structure or just simple string) and `start` is a start time. 

Tracer can measure a repeating event (in a loop for example). Every event will be stored with the particular index.

Use `t.Get()` to read all measurements. The Trace object contains a list of entries and overall time duration.

The last method is `t.Clear()` which removes all entries from the internal database.  

## Messaging/Kafka

This package provides clients for publishing synchronous/asynchronous messages and for consuming selected topics.

The mux package uses these clients and allows them to share their access to Kafka brokers among multiple entities. This package also implements the generic messaging API defined in the parent package.

### Requirements

The minimal supported version of Kafka is determined by the [sarama][sarama] library.  

If you don't have Kafka installed locally you can use docker image for testing:
```
sudo docker run -p 2181:2181 -p 9092:9092 --name kafka --rm \
 --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 spotify/kafka
```

### Kafka plugin

The Kafka plugin provides access to Kafka brokers.

**Configuration**

- Location of the Kafka configuration file can be defined either by CLI flag `kafka-config` or set via `KAFKA_CONFIG` env variable.

**Status Check**

- Kafka plugin has a mechanism to periodically check the connection status of the Kafka server. 

### Multiplexer

The multiplexer instance provides access to Kafka brokers through connections. There are two connection types with one supporting messages of type `[ ]byte` and the other `proto.Message`.

Both are allowed to create SyncPublishers and AsyncPublishers that implement the `BytesPublisher` or `ProtoPubliser` interfaces. The connections provide an API for consuming messages implementing the said `BytesMessage` interface or `ProtoMessage` respectively.

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

## Process manager

The process manager plugin enables the creation of processes to  manage and monitor plugins. 

There are several possible ways to obtain a process instance via the `ProcessManager` 
API:

- New process with options: using method `NewProcess(<cmd>, <options>...)` which requires a command and set of optional parameters. 
- New process from template: using method `NewProcessFromTemplate(<tmp>)` which requires a parameter template.
- Attach to existing process: using method `AttachProcess(<pid>)`. The process ID is required to order to attach.

### Management

!!! note
    Since the application is parent to all processes, application termination causes all started processes to stop as well. This can be changed with the `Detach` option (see process options).

Process management methods:

- `Start()` starts the plugin-defined process, stores the instance and performs an initial status file read.
- `Restart()` attempts to gracefully stop (force stop if it fails) and then (re)start the process. 
- `Stop()` stops the instance using a SIGTERM signal. Process is not guaranteed to be stopped. Note that child processes (not detached) may be killed if stopped this way. 
- `StopAndWait()` stops the instance using a SIGTERM signal and waits until the process completes. 
- `Kill()` force-stops the process using a SIGKILL signal and releases all used resources used.
- `Wait()` waits until the process completes.
- `Signal()` allows the user to send a custom signal to a process. Note that some signals may cause unexpected behavior in process handling.

Process monitor methods:

- `IsAlive()` returns true if process is running
- `GetNotificationChan()` returns a channel where process status notifications will be sent. Useful when a process is created via template with 'notify' field set to true. In other cases, the channel is provided by a user.
- `GetName` returns process name as defined in the status file.
- `GetPid()` returns process ID.
- `UpdateStatus()` updates the plugin's internal status and returns the actual status file
- `GetCommand()` returns the original process command. This will always be empty for attached processes.
- `GetArguments()` returns original arguments the process was started with. This will always be empty for attached processes.
- `GetStartTime()` returns the time stamp when the process was started for the last time
- `GetUpTime()` returns process up-time in nanoseconds

### Status watcher

Every process is watched for status changes independed of how it was created and if the process is running or not. Status such as running, sleeping, idle, etc. are used. The state is read from a process status file and every change is reported. 

The plugin defines several system-wide status types:

- Initial: only for newly created processes, meaning the process command was defined but not started yet
- Terminated: if the process is not running or does not respond
- Unavailable: if the process is running but the status cannot be obtained

The process status is periodically polled and notifications can be sent to the user defined channel. In case the process was created via template, the channel initialized in the plugin and can be obtained via `GetNotificationChan()`.

### Process options

The following options are available for processes. All options can be defined in the API method or template. All of them are optional.

**Args:** takes string array as a parameter and process will be started with given arguments. 

**Restarts:** takes a number as a parameter that defines the count of automatic restarts when the process state becomes terminated.

**Writer:** can define custom `stdOut` and `stdErr` values.
 
!!! note  
    Usability is limited when defined via template. Only standard `os.stdout` and `os.stderr` can be used.

**Detach:** no parameters applicable. Started process detaches from the parent application and will be given to the current user. This setting allows the process to run even after the parent has been terminated.

**EnvVar:** define environment variables (for example `os.Environ` for all)

**Template:** requires name `run-on-startup` flag. This creates a template upon process creation. The template path must be set in the plugin.

**Notify:** to provide a notification channel for status changes.

### Templates

The template is a file which defines the process configuration for the plugin manager. All templates should be stored in the path defined in the plugin config file.

```
./process-manager-plugin -process-manager-config=<path-to-file>
```

The template can be written "by hand" using a proto model, or generated with the `Template` option while creating a new process. 

Upon plugin init, all templates are read, and those with *run-on-startup* set to `true` are started. The template contains several fields defining process name, command, arguments and additional options fields.

The plugin API is used to read templates directly with `GetTemplate(<name)` or `GetAllTmplates()`. The template object can be used as parameter to start a new process.  

## Supervisor

The supervisor is the infrastructure plugin providing mechanisms to handle and manage processes and process hooks. Unlike the [process manager][process-manager], supervisor defines a configuration file with a simplified process definition where multiple program instances can be specified. The supervisor is considered a process manager extension with the convenient way how to define and manage various processes. 

### Config File

The config file is split into two main categories - processes or **programs**, and **hooks**. Each of these may contain multiple entries (more programs or hooks in a single file). The program definition is as follows:

- **name** is a unique program name, and also an optional parameter which is derived from the executable name if it is omitted.
- **executable-path** is a mandatory field containing a path to the executable for a given program.
- **executable-args** is a list of strings which is passed to the command as executable arguments. An arbitrary count of arguments can be defined.
- **logfile-path** should be added if it is required to put a program output log to the file. The log file is created if missing, and also every program can use its own file. In case the log file is not specified, the program log is written to standard output.
- **restarts** make use of the process manager auto-restart feature. The field parameter defines the maximum auto-restarts of the given program. Note that any termination hooks are executed also when the program is restarted since the operating system sends events in order termination -> starting -> sleeping/idle.
- **cpu-affinity-mask** allows to bind a process to a given set of CPUs. Value is the same hexadecimal format as for taskset command. Invalid value prints error message but it does not terminate the process. Use only when you know what you are doing, do not try to outsmart OS CPU scheduling. If a program has its own config file to manage CPUs, prioritize it. Keep in mind that incorrect use may slow down certain applications or that the application may contain its own CPU manager which overrides this value. Locking process to CPU does NOT keep other processes off that CPU.
- **cpu-affinity-setup-delay** postpones CPU affinity setup for a given time. Some processes may manipulate CPU scheduling during startup, this option allows to "bypass" it, waiting until the process is fully loaded and then lock it to the given value.

All spawned processes are bound to the supervisor and cannot exist without it. Terminating supervisor exists all running instances.

### Hooks

Hooks are special commands which are executed when some specific event related to the programs occurs. The config file may define as many hooks as needed. Basically hooks are not bound to any specific program instance or event - instead, every hook is called for every program event and it is up to the called script to decide required actions.

- **cmd** is a command called for the given hook.
- **cmd-args** is a set of parameters for the command above.

Usually, the hook is expected to run under certain conditions (a specific process, event, etc.). The executable is started within a specific environment (executed hook never uses the current process environment). 

Default environment variables set before any hook is started:

- **SUPERVISOR_PROCESS_NAME** is set to the program name and can be used to start hooks bound to a specific process.
- **SUPERVISOR_PROCESS_STATE** is a label marking what happened with the process in given event (started, idle, sleeping, terminated, ...)
- **SUPERVISOR_EVENT_TYPE** defines event type (currently only for process status)

Every hook is started with all those variables set.
 
## Service Label

The `servicelabel` is a plugin, available to other plugins that require the use of the `microservice label`. The label is primarily used to prefix keys in the etcd data store so that the configurations of different VPPs are not mixed up in a [multiple vpp-agent environment][multi-vpp-agent].

**Configuration**

- the serviceLabel can be set by the config file flag `microservice-label` or environment variable `MICROSERVICE_LABEL`

**Example**

Example of retrieving and using the microservice label:
```
plugin.Label = servicelabel.GetAgentLabel()
dbw.Watch(dataChan, cfg.SomeConfigKeyPrefix(plugin.Label))
```





[datasync-plugin]: db-plugins.md#datasync-plugin
[idx-map-cache-image]: ../img/user-guide/idxmap_cache.png
[idx-map-local-image]: ../img/user-guide/idxmap_local.png
[idxmap-godoc]: https://godoc.org/github.com/ligato/cn-infra/idxmap
[kubernetes-probes]: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
[log-level-config]: ../user-guide/config-files.md#log-manager
[multi-vpp-agent]: ../user-guide/get-vpp-agent.md#how-to-use-multiple-vpp-agents
[rest-plugin]: connection-plugins.md#vpp-agent-rest
[process-manager]: infra-plugins.md#process-manager
[rest-plugin]: connection-plugins.md#rest-plugin
[sarama]: https://github.com/Shopify/sarama
[status-check-image]: ../img/user-guide/status_check.png
[status-check-push-image]: ../img/user-guide/status_check_push.png
[status-check-pull-image]: ../img/user-guide/status_check_pull.png
[vpp-iface-idx]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/ifplugin/ifaceidx/ifaceidx.goL62
[vpp-iface-map]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/ifplugin/ifplugin_api.go#L26
[value-origin]: https://github.com/ligato/vpp-agent/blob/dev/plugins/kvscheduler/api/kv_descriptor_api.go#L53

*[FIB]: Forwarding Information Base
*[HTTP]: Hypertext Transfer Protocol
*[REST]: Representational State Transfer
*[VNF]: Virtual Network Function
*[VPP]: Vector Packet Processing
