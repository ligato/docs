# Infra plugins

---

## Status Check

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

## Index Map

The idxmap package provides an enhanced mapping structure to help in the following use cases:

* Exposing read-only access to plugin local data for other plugins
* Secondary indexing
* Data caching for key-value store (such as ETCD)

For more detailed description see the godoc.

### Exposing plugin local information Use Case

App plugins often need to expose some structured information to other plugins inside the agent (see the following diagram). Structured data stored in idxmap are available for (possibly concurrent) read access to other plugins:

1. either via lookup using primary keys or secondary indices;
2. or via watching for data changes in the map using channels or callbacks (subscribe for changes and receive notification once an item is added, changed or removed).

![idxmap local][idx-map-local-image]

### Caching Use Case

It is useful to have the data from a key-value store cached when you need to:

- minimize the number of lookups into the key-value store
- execute lookups by secondary indexes for key-value stores that do not necessarily support secondary indexing (e.g. ETCD)

**CacheHelper** turns `idxmap` (injected as field `IDX`) into an indexed local copy of remotely stored key-value data. `CacheHelper` watches the target key-value store for data changes and resync events. Received key-value pairs are transformed into the name-value (+ secondary indices if defined) pairs and stored into the injected idxmap instance. 
For a visual explanation, see the diagram below:

![idxmap cache][idx-map-cache-image]

The constructor that combines `CacheHelper` with `idxmap` to build the cache from the example can be found there as well.
  
## Log Manager

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
 
!!! note    
    Initial log level can be set using environmental variable `INITIAL_LOGLVL`. The variable replaces default-level from configuration file. However, loggers (partial definition) replace default value set by environmental variable for specific loggers defined.  
  
### Tracer

A simple utility able to log measured time periods various events. To create a new tracer, call:

`t := NewTracer(name string, log logging.Logger)`

Tracer object can store a new entry with `t.LogTime(entity string, start Time)` where `entity` is a string representation of a measured object (name of a function, structure or just simple string) and `start` is a start time. Tracer can measure repeating event (in a loop for example). Every event will be stored with the particular index.

Use `t.Get()` to read all measurements. The Trace object contains a list of entries and overall time duration.

Last method is `t.Clear()` which removes all entries from the internal database.  

## Messaging/Kafka

The client package provides single purpose clients for publishing synchronous/asynchronous messages and for consuming selected topics.

The mux package uses these clients and allows to share their access to Kafka brokers among multiple entities. This package also implements the generic messaging API defined in the parent package.

### Requirements

Minimal supported version of Kafka is determined by [sarama][sarama] library - Kafka 0.10 and 0.9, although older releases are still likely to work.

If you don't have Kafka installed locally you can use docker image for testing:
```
sudo docker run -p 2181:2181 -p 9092:9092 --name kafka --rm \
 --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 spotify/kafka
```

### Kafka plugin

Kafka plugin provides access to Kafka brokers.

**Configuration**

- Location of the Kafka configuration file can be defined either by command line flag `kafka-config` or set via `KAFKA_CONFIG` env variable.

**Status Check**

- Kafka plugin has a mechanism to periodically check a connection status of the Kafka server. 

### Multiplexer

The multiplexer instance has an access to Kafka brokers. To share the access it allows to create connections. There are available two connection types one support message of type `[]byte` and the other `proto.Message`. Both of them allows to create several SyncPublishers and AsyncPublishers that implements `BytesPublisher` interface or `ProtoPubliser` 
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

## Process manager

The process manager plugin provides a set of methods to create a plugin-defined processes instance implementing a set of methods to manage and monitor them. There are several ways how to obtain a process instance via `ProcessManager` 
API:

**New process with options:** using method `NewProcess(<cmd>, <options>...)` which requires a command and a set of optional parameters. 
**New process from template:** using method `NewProcessFromTemplate(<tmp>)` which requires template as a parameter
**Attach to existing process:** using method `AttachProcess(<pid>)`. The process ID is required to order to attach.

### Management

!!! note
    Since application (management plugin) is parent of all processes, application termination causes all started processes to stop as well. This can be changed with *Detach* option (see process options).

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

**Writer** allows to define custom `stdOut` and `stdErr` values. 
!!! note  
    Usability is limited when defined via template (only standard `os.stdout` and `os.stderr` can be used)

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

## Supervisor

The supervisor is the infrastructure plugin providing mechanisms to handle and manage processes and process hooks. Unlike the [process manager][process-manager], supervisor defines a configuration file with a simplified process definition where multiple program instances can be specified. The supervisor is considered a process manager extension with the convenient way how to define and manage various processes. 

### Config File

The config file is split into two main categories - processes or **programs**, and **hooks**. Each of these may contain multiple entries (more programs or hooks in a single file). The program definition is as follows:

- **Name** is a unique program name, and also an optional parameter which is derived from the executable name if omitted.
- **Executable path** is a mandatory field containing a path to the executable for a given program.
- **Executable arguments** is a list of strings which is passed to the command as executable arguments. An arbitrary count of arguments can be defined.
- **Logfile path** should be added if it is required to put a program output log to the file. The log file is created if missing, and also every program can use its own file. In case the log file is not specified, the program log is written to standard output.
- **Restarts** makes use of the process manager auto-restart feature. The field parameter defines the maximum auto-restarts of the given program. Note that any termination hooks are executed also when the program is restarted since the operating system sends events in order termination -> starting -> sleeping/idle.

All spawned processes are bound to the supervisor and cannot exist without it. Terminating supervisor exists all running instances.

### Hooks

Hooks are special commands which are executed when some specific event related to the programs occurs. The config file may define as many hooks as needed. Basically hooks are not bound to any specific program instance or event - instead, every hook is called for every program event and it is up to the called script to decide required actions.

- **Cmd** is a command called for the given hook.
- **CmdArgs** is a set of parameters for the command above.

Usually, the hook is expected to run under certain conditions (a specific process, event, etc.). The executable is started within a specific environment (executed hook never uses the current process environment). 

Default environment variables set before any hook is started:

- **SUPERVISOR_PROCESS_NAME** is set to the program name and can be used to start hooks bound to a specific process.
- **SUPERVISOR_PROCESS_STATE** is a label marking what happened with the process in given event (started, idle, sleeping, terminated, ...)
- **SUPERVISOR_EVENT_TYPE** defines event type (currently only for process status)

Every hook is started with all those variables set.
 
## Service Label

The `servicelabel` is a small Core Agent Plugin, which other plugins can use to obtain the microservice label, i.e. the string used to identify the particular VNF. The label is primarily used to prefix keys in ETCD data store so that the configurations of different VNFs do not get mixed up.

**Configuration**

- the serviceLabel can be set either by commandline flag `microservice-label` or environment variable `MICROSERVICE_LABEL`

**Example**

Example of retrieving and using the microservice label:
```
plugin.Label = servicelabel.GetAgentLabel()
dbw.Watch(dataChan, cfg.SomeConfigKeyPrefix(plugin.Label))
```





[datasync-plugin]: db-plugins.md#datasync-plugin
[idx-map-cache-image]: ../img/user-guide/idxmap_cache.png
[idx-map-local-image]: ../img/user-guide/idxmap_local.png
[kubernetes-probes]: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
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
