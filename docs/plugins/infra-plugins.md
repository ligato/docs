# Infra Plugins

This section describes the set of [Ligato infrastructure][cn-infra-github] plugins.

---

## Status Check

The status check plugin monitors the overall status of a cn-infra based application by collecting and aggregating the status of agent plugins. The status is exposed to external clients via [etcd - datasync][datasync-plugin] and [REST][rest-plugin]. 


![status check][status-check-image]
<p style="text-align: center; font-weight: bold">StatusCheck API</p>

**References:**

- [Ligato cn-infra repo][cn-infra-github]
- [Status check repo folder][status-check-repo-folder]
- [Status check proto][status-check-proto]
- [Status check model][status-check-model]

---

### Overall Agent Status

The overall agent status is aggregated from all plugins' status. Conceptually, this is a  `logical AND` for each plugin's status.

The agent's current status can be retrieved from the etcd data store using this key: 
```
/vnf-agent/<agent-label>/check/status`

```

Example:
```
$ etcdctl get /vnf-agent/<agent-label>/check/status/v1/agent
/vnf-agent/<agent-label>/check/status/v1/agent
{"build_version":"e059fdfcd96565eb976a947b59ce56cfb7b1e8a0","build_date":"2017-06-16.14:59","state":1,"start_time":1497617981,"last_change":1497617981,"last_update":1497617991}
```

To verify the agent status via HTTP, use the `/liveness` and `/readiness` URL endpoints:
```
$ curl -X GET http://localhost:9191/liveness
{"build_version":"e059fdfcd96565eb976a947b59ce56cfb7b1e8a0","build_date":"2017-06-16.14:59","state":1,"start_time":1497617981,"last_change":1497617981,"last_update":1497617991}
$ curl -X GET http://localhost:9191/readiness
{"build_version":"e059fdfcd96565eb976a947b59ce56cfb7b1e8a0","build_date":"2017-06-16.14:59","state":1,"start_time":1497617981,"last_change":1497617981,"last_update":1497617991}
```

To change the HTTP server port (default `9191`), use the `http-port` flag:
```
$ vpp-agent -http-port 9090
```

---

### Plugin Status

Plugins may use the `PluginStatusWriter.ReportStateChange` API to push status information at any time. For optimal performance, `statuscheck` will then propagate the status report to external clients, if and only if, it has changed since the last update. 

 

![status check push][status-check-push-image]
<p style="text-align: center; font-weight: bold">Plugin Status using Push approach</p>

---

Alternatively, plugins may choose to use the pull-based approach, and define the `probe` function passed to `PluginStatusWriter.Register` API. Statuscheck will then periodically probe the plugin for the current status. Once again, the status is propagated only if it has changed since the last enquiry. 


![status check pull][status-check-pull-image]
<p style="text-align: center; font-weight: bold">Plugin Status using Pull approach</p>

It is recommended `NOT` to mix the push and the pull approaches within the same plugin.

To retrieve the current status of a plugin from etcd, use this key template:
```
/vnf-agent/<agent-label>/check/status/v1/plugin/<PLUGIN_NAME>
```
 
For example, to retrieve the status of the GoVPPMUX plugin, use this command: 

```
$ etcdctl get /vnf-agent/<agent-label>/check/status/v1/plugin/GOVPP
/vnf-agent/<agent-label>/check/status/v1/plugin/GOVPP
{"state":2,"last_change":1496322205,"last_update":1496322361,"error":"VPP disconnected"}
```

---

## Index Map

The idxmap package provides a mapping structure to support the following use cases:

* Exposing read-only access to plugin local data for other plugins
* Secondary indexing
* Data caching for a KV data store such as etcd

**References:**

- [Ligato cn-infra repo][cn-infra-github]
- [Index Map repo folder][idxmap-repo-folder]
- [Index Map GoDoc description][idxmap-godoc]

---

### Exposing Local Data Use Case

App plugins often need to expose some structured information to other plugins inside the agent. Structured data stored in the idxmap is available for read access to other plugins:

- via lookup using primary keys or secondary indices
- via watching for data changes in the map using channels or callbacks. Subscribe for changes and receive a notification once an item is added, changed or removed.

![idxmap local][idx-map-local-image]
<p style="text-align: center; font-weight: bold">Using idxmap to expose local data</p>

---

### Caching Use Case

It is useful to cache data from a KV data store when you need to:

- minimize the number of lookups into the KV data store
- execute lookups by secondary indexes for KV data stores that do not necessarily support secondary indexing. Not that etcd does not support secondary indexing

**CacheHelper** turns idxmap, injected as field `IDX`, into an indexed local copy of remotely stored key-value data. CacheHelper watches the target KV data store for data changes and resync events. Received key-value pairs are transformed into the name-value (+ secondary indices if defined) pairs and stored in the injected idxmap instance. 

See the `idxmap caching` figure below.
![idxmap cache][idx-map-cache-image]
<p style="text-align: center; font-weight: bold">idxmap caching</p>

---
  
## Log Manager


The log manager plugin is used to view and modify the log levels of loggers using a REST API.

**References:**

- [Ligato cn-infra repo][cn-infra-github]
- [Log manager repo folder][log-manager-repo-folder]
- [Log manager conf file][log-manager-conf-file]

**APIs**

List all registered loggers:
```
curl -X GET http://<host>:<port>/log/list
```
    
Set log level for a registered logger:
```
curl -X PUT http://<host>:<port>/log/<logger-name>/<log-level>
```
 
where `<log-level>` is one of the following: `debug`,`info`,`warning`,`error`,`fatal`, and `panic`.
   
**Conf File**

The logger conf file is composed of several parts: 

- default level applied for all plugins
- map defining individual loggers and associated log levels:
- Hooks defining a list of one or more external links to external logs such as [Logstath][logstash] and [Fluentd][fluentd].

Conf file template:
```json
default-level: debug
loggers:
# - name: "defaultLogger"
#   level: info
 - name: "example"
   level: debug
 - name: "examplechildLogger"
   level: debug
hooks:
  syslog:
    levels:
    - panic
    - error
    - warn
#  custom:
#    addres: localhost
#    levels:
#    - panic
```

    
Initial log levels can be set using the environmental variable `INITIAL_LOGLVL`. This overrides `default-level` defined in the [conf file][log-manager-conf-file].   
 

--- 
  
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
where `entity` is a string representation of a measured object and `start` is a start time.  

Tracer can measure a repeating event, as might occur in a loop for example. Every event will be stored with the particular index.

Use `t.Get()` to read all measurements. The Trace object contains a list of entries and overall time duration.

The last method,  `t.Clear()`, removes all entries from the internal database.  

---

## Messaging/Kafka

This package provides clients for publishing synchronous/asynchronous messages, and for consuming selected topics.

The mux package uses these clients and allows them to share their access to Kafka brokers among multiple entities. This package also implements the generic messaging API defined in the parent package.

**References:**

- [Ligato cn-infra repo][cn-infra-github]
- [Kafka repo folder][kafka-repo-folder]
- [Kafka conf file][kafka-conf-file]

---

### Requirements

The minimum supported version of Kafka is determined by the [Sarama][sarama] library.  

If Kafka is not installed locally, you can use docker image for testing:
```
sudo docker run -p 2181:2181 -p 9092:9092 --name kafka --rm \
 --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 spotify/kafka
```

---

### Kafka Plugin

The Kafka plugin provides access to Kafka brokers.

**Configuration**

The location of the Kafka configuration file can be defined with the flag of `kafka-config`, or set via the `KAFKA_CONFIG` env variable.

The configuration options are described in the [Kafka conf file][kafka-conf-file] section of user guide. 

**Status Check**

The Kafka plugin has a mechanism to periodically check the connection status of the Kafka server using the status check plugin.  

### Multiplexer

The [multiplexer][kafka-mux] instance provides access to Kafka brokers through connections. There are two connection types with one supporting messages of type `[ ]byte`, and the other `proto.Message`.

Both can create SyncPublishers and AsyncPublishers that implement the `BytesPublisher` or `ProtoPubliser` interfaces. The connections provide an API for consuming messages implementing the `BytesMessage` interface, or `ProtoMessage` interface respectively.

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

---

## Process Manager

The process manager plugin enables the creation of processes to manage and monitor plugins. 

**References:**

- [Ligato cn-infra repo][cn-infra-github]
- [Process manager repo folder][pm-repo-folder]
- [Process manager conf file][pm-conf-file]

There are several ways to obtain a process instance via the `ProcessManager` 
API:

- New process with options: using method `NewProcess(<cmd>, <options>...)`. This requires a command and set of optional parameters. 
- New process from template: using method `NewProcessFromTemplate(<tmp>)`. This requires a parameter template.
- Attach to existing process: using method `AttachProcess(<pid>)`. The process ID is required to order to attach.

---

### Management

!!! note
    Since the application is parent to all processes, application termination causes all started processes to stop as well. This can be changed with the `Detach` option in the process options.

**Process management methods:**

- `Start()` starts the plugin-defined process, stores the instance, and performs an initial status file read.
- `Restart()` attempts to gracefully stop, force stop if it fails, and then (re)start the process. 
- `Stop()` stops the instance using a SIGTERM signal. Process is not guaranteed to be stopped. Note that child processes, that is those not detached, may be killed if stopped this way. 
- `StopAndWait()` stops the instance using a SIGTERM signal, and waits until the process completes. 
- `Kill()` force-stops the process using a SIGKILL signal, and releases all used resources used.
- `Wait()` waits until the process completes.
- `Signal()` allows the user to send a custom signal to a process. Note that some signals may cause unexpected behavior in process handling.

---

**Process monitor methods:**

- `IsAlive()` returns true if process is running
- `GetNotificationChan()` returns a channel where process status notifications will be sent. Useful when a process is created via template with `notify` field set to true. In other cases, the channel is provided by a user.
- `GetName` returns process name as defined in the status file.
- `GetPid()` returns process ID.
- `UpdateStatus()` updates the plugin's internal status and returns the actual status file.
- `GetCommand()` returns the original process command. This will always be empty for attached processes.
- `GetArguments()` returns original arguments the process was started with. This will always be empty for attached processes.
- `GetStartTime()` returns the time stamp when the process was started for the last time.
- `GetUpTime()` returns process up-time in nanoseconds.

---

### Status Watcher

Every process is watched for status changes independent of how it was created, and if the process is running or not. Status including running, sleeping, and idle are used. The state is read from a process status file and all changes are reported. 

The plugin defines several system-wide status types:

- **Initial:** only for newly created processes, meaning the process command was defined, but not started yet.
- **Terminated:** if the process is not running or does not respond.
- **Unavailable:** if the process is running but the status cannot be obtained.

The process status is periodically polled and notifications can be sent to the user defined channel. In case the process was created via template, the channel initialized in the plugin can be obtained using `GetNotificationChan()`.

---

### Process Options

The following options are available for processes. All options can be defined in the API method or template. All are optional.

- **Args:** string array as a parameter, and the process will be started with the given arguments. 
- **Restarts:** count of automatic restarts following process state detected as terminated.
- **Writer:** define custom `stdOut` and `stdErr` values.
- **Detach:** no parameters applicable. Started process detaches from the parent application and will be given to the current user. This setting allows the process to run even after the parent has been terminated.
- **EnvVar:** define environment variables. For example, `os.Environ` for all.
- **Template:** requires name `run-on-startup` flag. This creates a template upon process creation. The template path must be set in the plugin.
- **Notify:** provide a notification channel for status changes.

!!! note  
    Usability is limited when defined with a template. Only standard `os.stdout` and `os.stderr` can be used.

---

### Templates

The template is a file which defines the process configuration for the process manager. All templates should be stored in the path defined in the conf file.

```
./process-manager-plugin -process-manager-config=<path-to-file>
```

The template can be written "by hand" using a proto model, or generated with the `Template` option while creating a new process. 

Upon plugin init, all templates are read, and those with *run-on-startup* set to `true` are started. The template contains several fields defining process name, command, arguments, and additional options fields.

The plugin API is used to read templates directly with `GetTemplate(<name)` or `GetAllTmplates()`. The template object can be used as parameter to start a new process.

---  

## Supervisor

**References:**

- [Ligato cn-infra repo][cn-infra-github]
- [Supervisor repo folder][supervisor-repo-folder]
- [Super conf file][supervisor-conf-file]


The supervisor is an infrastructure plugin providing mechanisms to handle and manage processes and process hooks. Unlike the [process manager][process-manager], the supervisor defines a conf file with a simplified process definition, whereby multiple program instances can be specified. The supervisor is considered a process manager extension with the convenient method to define and manage various processes. 

---

### Supervisor Conf File

The conf file is split into two main categories:

 - **programs** or processes
 - **hooks**
 
 Each of these may contain multiple entries so more programs or hooks can be contained in a single file. 
 
 The program definition is as follows:

- **name** is a unique program name, and also an optional parameter which is derived from the executable name if it is omitted.
- **executable-path** is a mandatory field containing a path to the executable for a given program.
- **executable-args** is a list of strings which is passed to the command as executable arguments. An arbitrary count of arguments can be defined.
- **logfile-path** should be added if it is required to put a program output log to the file. The log file is created, if missing, and every program can use its own file. In case the log file is not specified, the program log is written to standard output.
- **restarts** makes use of the process manager auto-restart feature. The field parameter defines the maximum auto-restarts of the given program. Note that any termination hooks are executed when the program is restarted. This is because the operating system sends events in order of: termination -> starting -> sleeping/idle.
- **cpu-affinity-mask** allows one to bind a process to a given set of CPUs. Value is in the same hexadecimal format as the taskset command. Invalid value prints the error message, but it does not terminate the process. If a program has its own configuration file to manage CPUs, treat that as priority. Keep in mind that incorrect use may slow down certain applications, or that the application may contain its own CPU manager which overrides this value. Locking a process to a CPU does NOT keep other processes off that CPU. __`NOTE`: Use with care! Do not try to outsmart OS CPU scheduling.__ 
- **cpu-affinity-setup-delay** postpones CPU affinity setup for a given time period. Some processes may manipulate CPU scheduling during startup. This option allows one to "bypass" that, and wait until the process is fully loaded before locking it to the given value.

All spawned processes are bound to the supervisor and cannot exist without it. Terminating supervisor exits all running instances.

---

### Hooks

Hooks are special commands which are executed when some specific event related to the program occurs. The conf file may define as many hooks as needed. Hooks are not bound to any specific program instance or event. Instead, every hook is called for every program event, and it is up to the called script to decide the required actions.

- **cmd** is a command called for the given hook.
- **cmd-args** is a set of parameters for the command above.

Usually, the hook is expected to run under certain conditions including a specific process, or event. The executable is started within a specific environment. The executed hook never uses the current process environment.

Here is the example [supervisor.conf file][supervisor-conf-file] located in the repo. It includes programs and hooks.
```json
# Example supervisor config file starting vpp and agent,
# and defining hook for the vpp process which runs 'test.sh'
# if terminated
# See `taskset` man page to learn about how the cpu affinity
# mask should be set.
# ---
#
# programs:
#  - name: "vpp"
#    executable-path: "/usr/bin/vpp"
#    executable-args: ["-c", "/etc/vpp/base-startup.conf"]
#    logfile-path: "/tmp/supervisor.log"
#    restarts: 4
#    cpu-affinity-mask: 4
#    cpu-affinity-setup-delay: 1000000000
#  - name: "agent"
#    executable-path: "/usr/local/bin/vpp-agent"
#    executable-args: ["--config-dir=/tmp/config"]
#    logfile-path: "/tmp/supervisor.log"
#hooks:
#  - program-name: "vpp"
#    event-type: "terminated"
#    cmd: "/tmp/test.sh"
``` 

The following default env variables are set before any hook is started:

- **SUPERVISOR_PROCESS_NAME** is set to the program name and can be used to start hooks bound to a specific process.
- **SUPERVISOR_PROCESS_STATE** is a label marking what happened with the process in given event (started, idle, sleeping, terminated, ...)
- **SUPERVISOR_EVENT_TYPE** defines event type (currently only for process status)

Every hook is started with all env variables set.

---

## Service Label

**References:**

- [Ligato cn-infra repo][cn-infra-github]
- [Service label repo folder][service-label-repo-folder]
- [Service label conf file][service-label-conf-file]


The Service Label is a plugin, available to other plugins, that require the use of the [microservice label][microservice-label]. The label is primarily used to prefix keys in the etcd data store so that the configurations of different VPPs are not mixed up in a [multiple VPP agent environment][vpp-multiple-agents].

**Conf File**

The service label can be set by the flag of `microservice-label`, or using the env variable of `MICROSERVICE_LABEL`

The configuration options are described in the [service label conf file][service-label-conf-file] section of the user guide.


**Example**

Retrieving and using the microservice label:
```
plugin.Label = servicelabel.GetAgentLabel()
dbw.Watch(dataChan, cfg.SomeConfigKeyPrefix(plugin.Label))
```




[cn-infra-github]: https://github.com/ligato/cn-infra
[datasync-plugin]: db-plugins.md#datasync-plugin
[fluentd]: https://www.fluentd.org/
[godoc-idxmap]: https://godoc.org/github.com/ligato/cn-infra/idxmap
[idx-map-cache-image]: ../img/user-guide/idxmap_cache.png
[idx-map-local-image]: ../img/user-guide/idxmap_local.png
[idxmap-godoc]: https://pkg.go.dev/github.com/ligato/cn-infra/idxmap?tab=doc
[idxmap-repo-folder]: https://github.com/ligato/cn-infra/tree/master/idxmap
[kafka-conf-file]: ../user-guide/config-files.md#kafka
[kafka-mux]: https://github.com/ligato/cn-infra/tree/master/messaging/kafka/mux
[kafka-repo-folder]: https://github.com/ligato/cn-infra/tree/master/messaging/kafka
[kubernetes-probes]: https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-probes/
[log-manager-conf-file]: ../user-guide/config-files.md#log-manager
[log-manager-repo-folder]: https://github.com/ligato/cn-infra/tree/master/logging/logmanager
[logstash]: https://www.elastic.co/logstash
[microservice-label]: ../user-guide/concepts.md#keys-and-microservice-label
[multi-vpp-agent]: ../user-guide/get-vpp-agent.md#how-to-use-multiple-vpp-agents
[rest-plugin]: connection-plugins.md#vpp-agent-rest
[pm-conf-file]: ../user-guide/config-files.md#process-manager
[pm-repo-folder]: https://github.com/ligato/cn-infra/tree/master/exec/processmanager
[process-manager]: infra-plugins.md#process-manager
[rest-plugin]: connection-plugins.md#rest-plugin
[sarama]: https://github.com/Shopify/sarama
[service-label-conf-file]: ../user-guide/config-files.md#service-label
[service-label-repo-folder]: https://github.com/ligato/cn-infra/tree/master/servicelabel
[status-check-image]: ../img/user-guide/status_check.png
[status-check-model]: https://github.com/ligato/cn-infra/blob/master/health/statuscheck/model/status/keys_agent_status.go
[status-check-proto]: https://github.com/ligato/cn-infra/blob/master/health/statuscheck/model/status/status.proto
[status-check-push-image]: ../img/user-guide/status_check_push.png
[status-check-pull-image]: ../img/user-guide/status_check_pull.png
[status-check-repo-folder]: https://github.com/ligato/cn-infra/tree/master/health/statuscheck
[supervisor-repo-folder]: https://github.com/ligato/cn-infra/tree/master/exec/supervisor
[supervisor-conf-file]: https://github.com/ligato/cn-infra/blob/master/exec/supervisor/supervisor.conf
[vpp-iface-idx]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/ifplugin/ifaceidx/ifaceidx.goL62
[vpp-iface-map]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/ifplugin/ifplugin_api.go#L26
[vpp-multiple-agents]: ../user-guide/get-vpp-agent.md#using-multiple-vpp-agents
[value-origin]: https://github.com/ligato/vpp-agent/blob/dev/plugins/kvscheduler/api/kv_descriptor_api.go#L53

*[FIB]: Forwarding Information Base
*[HTTP]: Hypertext Transfer Protocol
*[REST]: Representational State Transfer
*[VNF]: Virtual Network Function
*[VPP]: Vector Packet Processing
