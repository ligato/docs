# Infra Plugins

This section describes the set of [Ligato infrastructure][cn-infra-github] plugins.

---

## Status Check

The status check plugin monitors the overall status of a cn-infra based application by collecting and aggregating the status of agent plugins. The plugin exposes the status to external clients using agentctl, etcd plugins, or [REST][rest-plugin]. 


![status check][status-check-image]
<p style="text-align: center; font-weight: bold">StatusCheck API</p>

**References:**

- [Ligato cn-infra repo][cn-infra-github]
- [Status check repo folder][status-check-repo-folder]
- [Status check proto][status-check-proto]
- [Status check model][status-check-model]

---

### Overall Agent Status

The overall agent status is aggregated from all plugins' status. Conceptually, this is a  `logical AND` for each plugin's status. The status check plugin aggregates each plugin's status, to arrive at the overall agent status. Conceptually, it performs a logical and on the collective plugin status to determine the overall agent status. 

The status check plugin can provide the overall status of the agent. It does so by performing a logical and on the plugin status  

Key to retrieve current agent status: 
```
/vnf-agent/<agent-label>/check/status`

```

Example:
```
$ agentctl kvdb get /vnf-agent/<agent-label>/check/status/v1/agent
/vnf-agent/<agent-label>/check/status/v1/agent
{"build_version":"e059fdfcd96565eb976a947b59ce56cfb7b1e8a0","build_date":"2017-06-16.14:59","state":1,"start_time":1497617981,"last_change":1497617981,"last_update":1497617991}
```

You can verify the agent status via HTTP, use the `/liveness` and `/readiness` URL endpoints:
```
$ curl -X GET http://localhost:9191/liveness
{"build_version":"e059fdfcd96565eb976a947b59ce56cfb7b1e8a0","build_date":"2017-06-16.14:59","state":1,"start_time":1497617981,"last_change":1497617981,"last_update":1497617991}
$ curl -X GET http://localhost:9191/readiness
{"build_version":"e059fdfcd96565eb976a947b59ce56cfb7b1e8a0","build_date":"2017-06-16.14:59","state":1,"start_time":1497617981,"last_change":1497617981,"last_update":1497617991}
```

You can change the HTTP server port (default `9191`), use the `http-port` flag:
```
$ vpp-agent -http-port 9090
```

---

### Plugin Status

Your plugins may use the `PluginStatusWriter.ReportStateChange` API to push status information at any time. For optimal performance, `statuscheck` propagates the status report to external clients, if and only if, it has changed since the last update. 

 

![status check push][status-check-push-image]
<p style="text-align: center; font-weight: bold">Plugin Status using Push approach</p>

---

Alternatively, your plugins can use the pull-based approach, and define the `probe` function passed to `PluginStatusWriter.Register` API. Statuscheck periodically probes the plugin for the current status. Once again, statuscheck propagates the status. only if it has changed since the last enquiry. 


![status check pull][status-check-pull-image]
<p style="text-align: center; font-weight: bold">Plugin Status using Pull approach</p>

!!! Note
    You should not NOT mix the push and the pull approaches within the same plugin.

Key template to retrieve the plugin's current status from etcd:
```
/vnf-agent/<agent-label>/check/status/v1/plugin/<PLUGIN_NAME>
```
 
Example of GoVPPMux plugin status retrieval from etcd: 
```
$ agentctl kvdb get /vnf-agent/<agent-label>/check/status/v1/plugin/GOVPP
/vnf-agent/<agent-label>/check/status/v1/plugin/GOVPP
{"state":2,"last_change":1496322205,"last_update":1496322361,"error":"VPP disconnected"}
```

---

## Index Map

The idxmap package provides a mapping structure to support the following use cases:

* Expose read-only access to plugin local data for other plugins.
* Secondary indexing.
* Data caching for a KV data store. 

**References:**

- [Ligato cn-infra repo][cn-infra-github]
- [Index Map repo folder][idxmap-repo-folder]
- [Index Map GoDoc description][idxmap-godoc]

---

### Exposing Local Data Use Case

App plugins often need to expose some structured data to other plugins inside the agent. You can provide read-only access to the structured plugin data stored in the idxmap using the following: 

- Lookup using primary keys or secondary indices.
- Watching for data changes in the map using channels or callbacks. You can subscribe to channels that provide notifications for changes by adding, deleting, or removing an item. 

![idxmap local][idx-map-local-image]
<p style="text-align: center; font-weight: bold">Using idxmap to expose local data</p>

---

### Caching Use Case

You will find it useful to cache data from a KV data store when you perform the following:

- Minimize the number of lookups into the KV data store.
- Execute lookups by secondary indexes for KV data stores that do not necessarily support secondary indexing. Not that etcd does not support secondary indexing.

[CacheHelper](https://github.com/ligato/cn-infra/blob/master/idxmap/mem/cache_helper.go) turns idxmap, injected as field `IDX`, into an indexed local copy of remotely stored key-value data. CacheHelper watches the target KV data store for data changes and resync events. It transforms the received key-value pairs into the name-value pairs, including secondary indices if defined. Cachehelper stores the transformed data in the injected idxmap instance. 

![idxmap cache][idx-map-cache-image]
<p style="text-align: center; font-weight: bold">idxmap caching</p>

---
  
## Log Manager


The log manager plugin allows you to view and modify the agent loggers and log levels using REST, agentctl, environment variables and conf files.

**References:**

- [Ligato cn-infra repo][cn-infra-github]
- [Log manager repo folder][log-manager-repo-folder]
- [Log manager conf file][log-manager-conf-file]

To read more on how to configure logging using REST, agentctl, environment variables and conf files, see [How to setup logging](../developer-guide/kvs-troubleshooting.md#how-to-set-up-logging).

You can change the agent's per-logger or global log levels using a conf file, environment variable, agentctl, or REST.

The log level choices consist of one of the following: 

- `debug`
- `info`
- `warning`
- `error`
- `fatal`
- `panic`

---
 
**Using the conf file or environment variable**

* To set a default for all loggers, per-logger levels, and external link hooks, use the [logs.conf](../user-guide/config-files.md#log-manager) file.
</br>
</br>
* To configure a list of external link hooks, set the `hooks:` fields in the logs.conf file. For example, you can define hooks as links to external logs such as [Logstath][logstash] and [Fluentd][fluentd].
<br>
</br>
* To override the entries in the `logs.conf` file, use the `INITIAL_LOGLVL=<level>` environment variable.


---

**Using agentctl log**

* To manage individual log levels, use [agentctl log](../user-guide/agentctl.md#log).

List individual loggers and their log levels:
```
agentctl log list
```

Example of setting the KV Scheduler log level to `debug`:
```
agentctl log set kvscheduler debug
```

---

**Using a REST API**

* List loggers and log levels:
```
curl -X GET http://localhost:9191/log/list
```

* Set an individual logger's log level:
```
curl -X POST http://localhost:9191/log/<logger-name>/<log-level>
```
   
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


--- 
  
### Tracer

You can log events across time periods using the tracer utility.

Create a new tracer:
```
t := NewTracer(name string, log logging.Logger)
```

Store a new tracer entry: 
```
t.LogTime(entity string, start Time)
``` 
where `entity` denotes a string representation of a measured object, and `start` is the start time.  

Tracer can measure a repeating event, as might occur in a loop for example. Tracer stores events with an index in an internal database. The trace object contains a list of entries and overall time duration.

You can read all tracer measurements using the following method:
```
t.Get()
```

You can remove all tracer entries from the internal database using the following method:
```
t.Clear()
```

---

## Messaging/Kafka

The client package provides single purpose clients for publishing synchronous/asynchronous messages and for consuming selected topics.

The mux component implements the session multiplexer that allows multiple plugins to share a single connection to a Kafka broker. The mux package also implements the generic messaging API defined in the parent package.

!!! Notes
    For more details on the topics covered in this section, including examples, see [Kafka package GoDoc](https://pkg.go.dev/github.com/ligato/cn-infra@v1.7.0/messaging/kafka#section-documentation)

**References:**

- [Ligato cn-infra repo][cn-infra-github]
- [Kafka repo folder][kafka-repo-folder]
- [Kafka conf file][kafka-conf-file]
- [Kafka package GoDoc](https://pkg.go.dev/github.com/ligato/cn-infra@v1.7.0/messaging/kafka#section-documentation)

---

### Requirements

The [Sarama][sarama] library determines the minimum supported version of Kafka you can use.  

If you do not have Kafka installed locally, you can use docker image for testing:
```
sudo docker run -p 2181:2181 -p 9092:9092 --name kafka --rm \
 --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 spotify/kafka
```

---

### Kafka Plugin

The Kafka plugin provides access to Kafka brokers.

**Configuration**

You can define the location of the Kafka conf file using the command line flag `kafka-config`, or by setting the `KAFKA_CONFIG` env variable. 

For a list of Kafka configuration options, see [Kafka conf file][etcd-conf-file].  

**Status Check**

The Kafka plugin uses the status check plugin to verify connection status.

### Multiplexer

The [multiplexer][kafka-mux] instance provides access to Kafka brokers through connections. You can implement a connection supporting two messages type: `[ ]byte`, or `proto.Message`.

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

- [Process proto](https://github.com/ligato/cn-infra/blob/master/exec/processmanager/template/model/process/process.proto)
- [Process manager conf file][pm-conf-file]
- [Process package GoDoc](https://pkg.go.dev/github.com/ligato/cn-infra@v1.7.0/process)

You can obtain a process instance with `ProcessManager` API using the one of the following techniques:

- New process with options: using method `NewProcess(<cmd>, <options>...)`. You need a command and set of optional parameters.
<br></br> 
- New process from template: using method `NewProcessFromTemplate(<tmp>)`. You need a parameter template.
<br></br>
- Attach to existing process: using method `AttachProcess(<pid>)`. You need a process id to attach to the existing process. 

---

### Management

!!! note
    The application is parent to all processes. Application termination causes all started processes to terminate. You can change this with the `Detach` option in the process options.

**Process management methods:**

- `Start()` starts the plugin-defined process, stores the instance, and performs an initial status file read.
<br></br>
- `Restart()` attempts to gracefully stop, force stop if it fails, and then start or re-start the process.
<br></br> 
- `Stop()` stops the instance using a SIGTERM signal. Process is not guaranteed to stop. Note this method could result in un-detached child processes termination.
<br></br> 
- `StopAndWait()` stops the instance using a SIGTERM signal, and waits until the process completes. 
<br></br>
- `Kill()` force-stops the process using a SIGKILL signal, and releases all used resources.
<br></br>
- `Wait()` waits until the process completes.
<br></br>
- `Signal()` allows you to send a custom signal to a process. Note that some signals may cause unexpected behavior in process handling.

---

**Process monitor methods:**

- `IsAlive()` returns true if process is running.
<br></br>
- `GetNotificationChan()` returns a channel for sending process status notifications. Use this method when you create a process via template with `notify` field set to true. In other cases, you will provide the channel. 
<br></br>
- `GetName` returns the process name as defined in the status file.
<br></br>
- `GetPid()` returns process id.
<br></br>
- `UpdateStatus()` updates the plugin's internal status and returns the actual status file.
<br></br>
- `GetCommand()` returns the original process command. Empty for attached processes.
<br></br>
- `GetArguments()` returns original arguments the process was started with. Empty for attached processes.
<br></br>
- `GetStartTime()` returns the time stamp when the process was started for the last time.
<br></br>
- `GetUpTime()` returns process up-time in nanoseconds.

---

### Status Watcher

The status watcher looks for process status changes independent of process creation method, and running state. The watcher uses the standard status labels includes running, sleeping, and idle. You can read state, including all reported changes, from a process status file.  

The following lists several system-wide status types supported by the plugin:

- **Initial** applies only for newly created processes. You defined the process command but did not start it.
<br></br>
- **Terminated** if the process is not running, or does not respond.
<br></br>
- **Unavailable** if the process is running, but cannot obtain status.

The status watcher periodically polls process status, and can send notifications to you over the notification channel. If you created the process with a template, you can obtain the notification channel using the `GetNotificationChan()` channel.

---

### Process Options

Process option characteristics consist of the following:
 
- Available to all processes.
- Defined in API method or template.
- Optional. 
 
The following lists the [process options](https://github.com/ligato/cn-infra/blob/master/exec/processmanager/process_options.go):

- **Args:** string array as a parameter, and the process will start with the given arguments.
<br></br> 
- **Restarts:** count of automatic restarts following process state detected as terminated.
<br></br>
- **Writer:** define custom `stdOut` and `stdErr` values.
<br></br>
- **Detach:** no parameters applicable. Started process detaches from the parent application and given to the current user. This setting allows the process to run even after parent termination.
<br></br>
- **EnvVar:** define environment variables. For example, us`os.Environ` for all.
<br></br>
- **Template:** requires name `run-on-startup` flag. This creates a template upon process creation. You must set the template path in the plugin.
<br></br>
- **Notify:** provide a notification channel for status changes.
<br></br>
- **autoTerm:** automatically terminate "zombie" processes. 
<br></br>
- **cpu affinity:** defines the cpu affinity for a given process.

!!! note  
    You could limit usability if you define a process with a template. Only standard `os.stdout` and `os.stderr` can be used.

---

### Templates

You define the process configuration for the process manager in a template file. You should store all templates in the path defined in the conf file.

```
./process-manager-plugin -process-manager-config=<path-to-file>
```

You can craft a template "by hand" using a proto model, or generate one with the `Template` option when creating a new process. 

Upon plugin initialization, all templates are read. Those with `run-on-startup` set to `true` in the template file are started. The [process proto](https://github.com/ligato/cn-infra/blob/master/exec/processmanager/template/model/process/process.proto) contains information you would set up in a template file.

The plugin API reads templates with `GetTemplate(<name)` or `GetAllTmplates()` methods. You can use the template object as a parameter to start a new process.

---  

## Supervisor

The supervisor plugin provides mechanisms to manage processes and process hooks. The supervisor conf file contains a simplified process definition that allows you to specify multiple program instances. You can think of the supervisor as a process manager extension with an easier way to define and manage processes.

**References:**

- [Supervisor repo folder][supervisor-repo-folder]
- [Supervisor conf file][supervisor-conf-file]
- [Superview config.go](https://github.com/ligato/cn-infra/blob/master/exec/supervisor/config.go#L62)


---

### Supervisor Conf File

The supervisor conf file defines programs and hooks. You can define multiple programs and hooks in a single conf file. 
 
The following describes the conf file definitions for a program:

- **name**: unique, but optional, program name. If omitted, the name is derived from the executable name.
<br></br>
- **executable-path**: required field containing a path to the program executable.
<br></br>
- **executable-args**: List of strings passed to the command as executable arguments.
<br></br>
- **logfile-path** defines the path to a log output file. If you do not define a logfile-path, the program log is written to stdout.
<br></br>
- **restarts**: defines the maximum auto-restarts of the given program. Note that program restarts cause termination hooks to execute, since the operating system sends events in order of: termination -> starting -> sleeping/idle.
<br></br>
- **cpu-affinity-mask** lets you bind a process to a given set of CPUs. Uses taskset to assign process to CPUs, and uses the same hexadecimal format. Invalid value prints the error message, but does not terminate the process. 
<br></br>
- **cpu-affinity-list**: bind process to a given set of CPUs. Note that you can list CPUs as a list of CPU cores.
<br></br>
- **cpu-affinity-setup-delay:** postpones CPU affinity setup for a given time period. Some processes manipulate CPU scheduling during startup. This option lets you "bypass" that, and wait until the process loads before locking it to the given value.


!!! Warning
    Do not try to outsmart OS CPU scheduling. If a program has its own config file to manage CPUs, prioritize it. Incorrect use may slow down certain applications, or that the application may contain its own CPU manager which overrides this value. Locking process to CPU does NOT keep other processes off that CPU.

---

### Hooks

Hooks are commands or scripts that execute when a specific event related to the program occurs. The conf file defines as many hooks as needed. Hooks are not bound to any specific program instance or event. Instead, every hook is called for every program event. You configure the actions to perform, given a specific event, in the hooks definition.

Hooks define two fields: 

- **cmd**: command called for the given hook.
- **cmd-args** command arguments. 

The [supervisor.conf file][supervisor-conf-file] example found in the repo includes programs and hooks.
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

The following default env variables are set before starting any hook:

- **SUPERVISOR_PROCESS_NAME** is set to the program name, and to start hooks bound to a specific process.
- **SUPERVISOR_PROCESS_STATE** is a label marking what happened with the process in a given event such as  started, idle, sleeping, and terminated.
- **SUPERVISOR_EVENT_TYPE** defines event type.

All hooks start with all environment variables set.

---

## Service Label

The service label plugin provides other plugins with the ability to support [microservice labels][microservice-label]. 

**References:**

- [Service label repo folder][service-label-repo-folder]
- [Service label conf file][service-label-conf-file]


**Configuration**

You can define the microservice label with one of the following options:

- Set the command line flag `microservice-label`
- Set the `MICROSERVICE_LABEL` environment variable.
- Use the [service label conf file][service-label-conf-file]

Example of retrieving and using the microservice label:
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
