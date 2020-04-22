# Conf Files

This section discusses plugin configuration files and flags.

---

## Conf File Location

The location of the conf files directory, or individual conf files can be set using a VPP agent CLI start flag, or env variable export.

**Conf file directory**

Conf file directory flag:
```bash
-config-dir="."
```
If the conf file directory is `/opt/vpp-agent/dev`, then start the VPP agent with this command:
```json
vpp-agent --config-dir=/opt/vpp-agent/dev
```
Same conf file directory, but using the env variable of `CONFIG_DIR`:
```json
export CONFIG_DIR=/opt/vpp-agent/dev
```

---

**Individual conf file location**

Using per-plugin flags or env variables will override the conf file directory option.

If the etcd conf file location is `/opt/vpp-agent/dev/etcd.conf`, then start the VPP agent with the etcd conf file flag of `--etcd-config=` like so:
```bash
vpp-agent -etcd-config=/opt/vpp-agent/dev/etcd.conf
```
Using the `ETCD_CONFIG` env variable:
```bash
export ETCD_CONFIG=/opt/vpp-agent/dev/etcd.conf
```

---

## Plugin Conf Files
---

<!--
### ACL plugin

```bash
-vpp-aclplugin-config=
```

Flag reserved for the ACL plugin, currently not in use.
-->

### Bolt

```bash
-bolt-config= 
```

_**Bolt Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
| **db-path** | _string_ | | Path to Bolt DB file |
| **file-mode** | _os.FileMode_ | | File mode and permission bits in decimal format |
| **lock-timeout** | _time.Duration_ | | Timeout duration for waiting to obtain file lock, set to zero to wait indefinitely. |

_**Bolt References:**_

- [Bolt concepts doc][bolt-concepts-doc] 
- [Bolt repo conf file][bolt-repo-conf-file]

---

### Cassandra

```bash
-cassandra-config= 
```


_**Cassandra Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
| **endpoints** | _string_ ||list of host IP addresses of cassandra cluster nodes |
| **port** | _int_ |9042|Cassandra port|
| **op_timeout** | _time.Duration_ |600ms|Connection Timeout|
| **dial_timeout**| _time.Duration_ |600ms|initial session timeout, used during initial dial to server |
| **redial_interval** | _time.Duration_ |60sec|Interval between gocql attempts to reconnect to known down nodes|
| **protocol_version**|_int_|4|Sets the version of the native protocol to use. This will enable features in the driver for specific protocol versions. Generally this should be set to a known version (2,3,4) for the cluster being connected to.</br></br> If it is 0 or unset (the default), then the driver will attempt to discover the highest supported protocol for the cluster. In clusters with nodes of different versions, the protocol selected is not defined (i.e., it can be any of the supported in the cluster).|
| **TLS Setup **|||Defines client cert, client private key, certificate authority, whether to skip verification of server name & certificate, disable TLS|

_**Cassandra References:**_

- [Cassandra plugin doc][cassandra-plugin-doc] 
- [Cassandra repo conf file][cassandra-repo-conf-file]


---

### Consul

```bash
-consul-config=
```

_**Consul Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**address**|_string_|0.0.0.0:8500|Consul server address|
|**resync-after-reconnect**|_bool_|false|Perform resync procedure for all registered plugins following reconnect to Consul server|

_**Consult References:**_

- [Consul plugin doc][consul-plugin-doc] 
- [Consul repo conf file][consul-repo-conf-file]

---

### etcd

```bash
-etcd-config=
```

_**etcd Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**endpoints**|_string_|172.17.0.1:2379|list of host IP addresses of ETCD database server|
|**dial-timeout**|_time.Duration_|1000000000ns|timeout for connecting to etcd|
|**operation-timeout**|_time.Duration_|3000000000ns|timeout for any request-reply etcd operation|
|**insecure-transport**|_bool_|false|TLS not used|
|**insecure-skip-tls-verify**|_bool_|false|Controls whether a client verifies the server's certificate chain and host name. If InsecureSkipVerify is true, TLS accepts any certificate presented by the server and any host name in that certificate. </br></br>In this mode, TLS is susceptible to man-in-the-middle attacks. This should be used only for testing.|
|**cert-file**|_string_||TLS Certification File|
|**key-file**|_string_||TLS certification key|
|**ca-file**|_string_||CA file used to create a set of x509 certificates|
|**auto-compact**|_time.Duration_|0|Interval between etcd auto compaction cycles. 0 means disabled|
|**resync-after-reconnect**|_bool_|false|Perform resync procedure for all registered plugins following reconnect to etcd server|
|**allow-delayed-start**|_bool_|false|Allow to start without connected ETCD database. Plugin will try to connect and if successful, overall resync will be called|
|**reconnect-interval**|_time.Duration_|2000000000ns|Interval between attempts to reconnect to the etcd server|

_**etcd References:**_

- [etcd plugin doc][etcd-plugin-doc] 
- [etcd repo conf file][etcd-repo-conf-file]



---

### FileDB

```bash
-filedb-config=
```

_**FileDB Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**configuration-paths**|_string_||A set of files/directories with configuration files. Examples are `/path/to/directory/` or `/path/to/file.ext`. If the target is a directory, all .json or .yaml files are read.|
|**status-path**|_string_||Path to the file where status data will be stored. `/path/to/status.txt` is an example. If it is not defined, status is not propagated. The file extension determines whether the data will be stored in .json or .yaml format.  The target cannot be a directory.|

Note: `filesystem` refers to the name of the FileDB plugin.

_**FileDB References:**_

- [FileDB plugin doc][filedb-plugin-doc] 
- [FileDB repo conf file][filedb-repo-conf-file]


---

### GoVPPMux

```bash
-govpp-config=
```

_**GoVPPMux Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**binapi-socket-path**|_string_|/run/vpp-api.sock|defines path to the binapi socket file|
|**connect-via-shm**|_bool_|false|Connect to VPP for configuration requests via the shared memory|
|**shm-prefix**|_string_||Defines a prefix prepended to the name used for shared memory (SHM) segments. </br></br>If not set, shared memory segments are created directly in the SHM directory /dev/shm.|
|**stats-socket-path**|_string_|/run/vpp/stats.sock|Defines path to the stats socket file|
|**resync-after-reconnect**|_bool_|false|Perform resync procedure for all registered plugins following reconnect to VPP|
|**retry-request-count**|_int_|0|Number of binary API request retries if VPP is suddenly disconnected|
|**retry-request-timeout**|_time.Duration_|500ms|Interval between binary API request retries|
|**retry-connect-count**|_int_|3|Number of connection request retries if VPP is not unreachable.|
|**retry-connect-timeout**|_time.Duration_|1000000000ns|Interval between connection request retries|
|**proxy-enabled**|_bool_|true|Enable VPP proxy|
|**health-check-probe-interval**|_time.Duration_||time between health check probes|
|**health-check-reply-timeout**|_time.Duration_||if this timer pops, probe is considered failed|
|**health-check-threshold**|_int_||number of consecutive failed health checks until an error is reported|



_**GoVPPMux References:**_

- [GoVPPMux plugin doc][govppmux-plugin-doc]
- [GoVPPMux repo conf file][govppmux-repo-conf-file]

---


### GRPC

```bash
-grpc-config=
```

_**GRPC Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**endpoint**|_string_|0.0.0.0:9111|address of gRPC netListener|
|**permission**|_int_|000|Three or four-digit permission setup for unix domain socket file (if used)|
|**force-socket-removal**|_bool_|false|If set and unix type network is used, the existing socket file will be always removed and re-created|
|**network**|_string_|tcp|Available socket types: tcp, tcp4, tcp6, unix and unixpacket.|
|**max-msg-size**|_int_|4096|Maximum message size in bytes for inbound messages|
|**max-concurrent-streams**|_unit32_|0|returns a ServerOption that will apply a limit on the number of concurrent streams to each ServerTransport|
**extended-logging**|_bool_|false|Enables logging additional gRPC transport messages|
|**insecure-transport**|_bool_|false|if true, TLS configuration will not be used|

The following config file options are used if `insecure-transport` is `false`:

| Option | Type | Default | Description |
|---|---|---|---|
|**cert-file**|_string_||Required for creating a secure connection. example is /path/to/cert.pem||
|**key-file**|_string_||Required for creating a secure connection. example is /path/to/key.pem|
|**ca-file**|_string_||Set custom CA to verify client's certificate. If not set, client's certificate is not required. </br></br>Examples ca-files are /path/to/ca1.pem and /path/to/ca2.pem|

This flag can be used to set the GRPC port:

```bash
-grpc-port=
```

_**GRPC References:**_

- [GRPC plugin doc][grpc-plugin-doc]
- [GRPC repo conf file][grpc-repo-conf-file]

---

### Kafka

```bash
-kafka-config=
```
_**Kafka Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**Addrs**|_string_|127.0.0.1:9092|Kafka server addresses|
|**group_id**|_string_||Name of the consumer's group|
|**TLS**|||TLS Configuration|

_**Kafka References:**_

- [Kafka plugin doc][Kafka-plugin-doc]
- [Kafka conf file][kafka-repo-conf-file]

---

### KV Scheduler

```bash
-kvscheduler-config=
```

_**KV Scheduler Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
| **record-transaction-history** | _bool_ | `true` | History of processed transactions is recorded|
| **transaction-history-age-limit** | _uint32_ (in minutes) | 24hrs | Age limit for recording transaction history with the exception of permanently recorded init period|
| **permanently-recorded-init-period**| _uint32_ (in minutes)| 60min | Duration of period from init that will be permanently recorded |
| **enable-txn-simulation** | _bool_ | `false` | Enable transaction simulation |
| **print-txn-summary** | _bool_ | `true` | Print transaction summary for each transaction |

_**KV Scheduler References:**_

- [KV Scheduler plugin doc][kvs-plugin-doc]

---

### Linux Interface Plugin

```bash
-linux-ifplugin-config=
```
_**Linux Interface Plugin Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**disabled**|_bool_|false|Used to disable linux ifplugin|
|**go-routines-count**|_int_|10|How many goroutines (at most) will split configured network namespaces to execute the Retrieve operation in parallel|

_**Linux Interface References:**_

- [Linux Interface plugin doc][linux-interface-plugin-doc]
- [Linux interface repo conf file][linux-repo-interface-conf-file]

---


### Linux IP Tables

```bash
-linux-iptables-config=
```
_**Linux IP Tables Plugin Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**disabled**|_bool_|false|Used to disable linux iptables plugin|
|**go-routines-count**|_int_|10|How many goroutines (at most) will split configured network namespaces to execute the Retrieve operation in parallel|

_**Linux IP Tables References:**_

- [Linux iptables plugin doc][linux-iptables-plugin-doc]
- [Linux iptables repo conf file][linux-repo-iptables-conf-file]

---


### Linux L3

```bash
-linux-l3plugin-config=
```
_**Linux L3 Plugin Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**disabled**|_bool_|false|Used to disable linux L3 plugin|
|**go-routines-count**|_int_|10|How many goroutines (at most) will split configured network namespaces to execute the Retrieve operation in parallel|

_**Linux L3 References:**_

- [Linux L3 plugin doc][linux-l3-plugin-doc]
- [Linux L3 repo conf file][linux-repo-l3-conf-file]

---

### Linux Namespace

```json
--linux-nsplugin-config=
```
_**Linux Namespace Plugin Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**disabled**|_bool_|false|Used to disable linux namespace plugin|

_**Linux Namespace References:**_

- [Linux namespace plugin doc][linux-nsplugin-doc]
- [Linux namespace repo conf file][linux-repo-namespace-conf-file]

---

### Log Manager

```bash
--logs-config=
```
_**Log Manager Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**default-level**|_string_|info|Set default config level for every plugin. Overwritten by environmental variable 'INITIAL_LOGLVL'|
|**loggers**|||Specifies a list of named loggers with their respective log levels. see `loggers` example below|
|**hooks**|||Specifies a list of hooks for logging to external links. Parameters for a given hook are protocol, address, port and levels. See `hooks` example below.|

Loggers example:
```
loggers:
  - name: "agentcore",
    level: debug
  - name: "status-check",
    level: info
  - name: "linux-plugin",
    level: warn
```

Hooks example:
```
hooks:
  syslog:
    levels:
    - panic
#    - fatal
#    - error
#    - warn
#    - info
#    - debug
#  fluent:
#    address: "10.20.30.41"
#    port: 4521
#    protocol: tcp
#    levels:
#     - error
#  logstash:
#    address: "10.20.30.42"
#    port: 123
#    protocol: tcp
```

_**Log Manager References:**_

- [Log manager plugin doc][log-manager-plugin-doc]
- [Logs repo conf file][logs-repo-conf-file]

---

### Process Manager

```bash
-process-manager-config=
```

_**Process Manager Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**template-path**|_string_||path where process templates will be stored|

_**Process Manager References:**_

- [Process manager plugin doc][pm-plugin-doc]
- [Process manager repo conf file][pm-repo-conf-file]

---

### REST

```bash
-http-config=
```

_**REST Plugin Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**endpoint**|_string_|0.0.0.0:9191|Address of the HTTP server|
|**read-timeout**|_time.Duration_|0|Maximum amount of time (in nanoseconds) for reading the entire request, including the body. </br></br>Read-timeout does not let handlers make per-request decisions on each request body's acceptable deadline or upload rate. Therefore most users will prefer to use read-header-timeout. It is valid to use both.|
|**read-header-timeout**|_time.Duration_|0|Maximum amount of time (in nanoseconds) to read request headers. The connection's read deadline is reset after reading the headers and the Handler can decide what is considered too slow for the body.|
|**write-timeout**|_time.Duration_|0|Maximum amount of time (in nanoseconds) before timing out writes to a response. It is reset whenever a new request's header is read. It does not let Handlers make decisions on a per-request basis.|
|**idle-timeout**|_time.Duration_|0|Maximum amount of time (in nanoseconds) to wait for the next request when keepalives are enabled. If the idle timeout is zero, the value of read-timeout is used. If both are zero, there is no timeout.|
|**max-header-bytes**|_int_|0|Maximum number of bytes the server will read parsing the request header's keys and values, including the request line. It does not limit the size of the request body.|
|**enable-token-auth**|_bool_|false|Enables or disables HTTP token authentication|
|**users**|||Registers additional users with permissions. Admin with full access to every permission group is registered automatically. Password must be in hashed form. See `users` format example below.|
|**password-hash-cost**|_int_|7|Number in range 4-31 used as a parameter for hashing passwords. Large numbers require more CPU time and memory to process.|
|**token-expiration**|_time.Duration_|60000000000ns|Token expiration time in nanoseconds. Zero means no expiration time|
|**token-signature**|_string_||string value used as key to sign a tokens|

User format example:
```
users:
   - name: <name>
     password_hash: <hash>
     permissions: [<group1>, <group2>, ...]
`
```

This flag can be used to set the HTTP port:

```bash
-http-port=
```

_**REST References:**_

- [REST plugin doc][rest-plugin-doc]
- [REST repo conf file][http-repo-conf-file]

---

### Service Label


```bash
--microservice-label=
```

_**Service Label Plugin Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**microservice-label**|_string_||Identifies a particular instance of a VPP agent. Used to form a key prefix associated with the VPP agent's config data contained in an etcd data store.|

_**Service Label References:**_

- [Service Label plugin doc][service-label-plugin-doc]

---

### Supervisor

The supervisor is an infrastructure plugin providing mechanisms to handle and manage processes and process hooks.

The conf file is split into two main categories:

 - **programs** or processes
 - **hooks**
 
 Each of these may contain multiple entries so more programs or hooks can be contained in a single file.

_**References:**_

- [Supervisor plugin doc][supervisor-plugin-doc] 
- [Supervisor repo conf File][supervisor-repo-conf-file]
 
---


### Telemetry

```json
--telemetry_config=
```
_**Telemetry Plugin Conf File Options**_


| Option | Type | Default | Description |
|---|---|---|---|
|**disabled**|_bool_|false|Used to disable telemetry plugin|
|**prometheus-disabled**|_bool_|false|export to prometheus|
|**polling-interval**|_time.Duration_|30sec|interval between VPP reads|
|**skipped**| _string_ | |skip some metrics collection such runtime, memory, buffers, nodes, interfaces|

_**Telemetry References:**_

- [Telemetry plugin doc][telemetry-plugin-doc]
- [Telemetry repo conf file][telemetry-repo-conf-file]

---


### VPP Interface

```bash
-vpp-ifplugin-config=
```

_**VPP Interface Plugin Conf File Options**_

| Option | Type | Default | Description |
|---|---|---|---|
|**MTU**|_unit32_|0|Default maximum transmission unit (MTU) size. The value is used if an interface without an MTU is created. Note that the MTU in the interface configuration is preferred.|
|**status-publishers**|_string_||Enables the VPP agent to send status data back to a KV data store. etcd, redis or both are supported.|

[_vpp-ifplugin.conf file_](https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/ifplugin/vpp-ifplugin.conf)

---

## VPP agent -h command

Use this command to display flag, conf file name, and env variable information for all conf files.

```json
vpp-agent -h
```

Output:
```json
                                      __
  _  _____  ___ _______ ____ ____ ___ / /_
 | |/ / _ \/ _ /___/ _ '/ _ '/ -_/ _ / __/  vpp-agent v3.2.0-alpha-1-g615f9fd36
 |___/ .__/ .__/   \_'_/\_' /\__/_//_\__/   Wed Mar 18 17:59:27 UTC 2020 (15 days ago)
    /_/  /_/           /___/                root@67748e05ef29 (go1.14 linux/amd64)

Usage of vpp-agent:
  -config-dir=".": Location of the config files; can also be set via 'CONFIG_DIR' env variable.
  -configurator-config="configurator.conf": Location of the "configurator" plugin config file; can also be set via "CONFIGURATOR_CONFIG" env variable.
  -consul-config="consul.conf": Location of the "consul" plugin config file; can also be set via "CONSUL_CONFIG" env variable.
  -etcd-config="etcd.conf": Location of the "etcd" plugin config file; can also be set via "ETCD_CONFIG" env variable.
  -govpp-config="govpp.conf": Location of the "govpp" plugin config file; can also be set via "GOVPP_CONFIG" env variable.
  -grpc-config="grpc.conf": Location of the "grpc" plugin config file; can also be set via "GRPC_CONFIG" env variable.
  -grpc-port="": Configure "grpc" server port
  -http-config="http.conf": Location of the "http" plugin config file; can also be set via "HTTP_CONFIG" env variable.
  -http-port="9191": Configure "http" server port
  -kafka-config="kafka.conf": Location of the "kafka" plugin config file; can also be set via "KAFKA_CONFIG" env variable.
  -kvscheduler-config="kvscheduler.conf": Location of the "kvscheduler" plugin config file; can also be set via "KVSCHEDULER_CONFIG" env variable.
  -linux-ifplugin-config="linux-ifplugin.conf": Location of the "linux-ifplugin" plugin config file; can also be set via "LINUX-IFPLUGIN_CONFIG" env variable.
  -linux-iptablesplugin-config="linux-iptablesplugin.conf": Location of the "linux-iptablesplugin" plugin config file; can also be set via "LINUX-IPTABLESPLUGIN_CONFIG" env variable.
  -linux-l3plugin-config="linux-l3plugin.conf": Location of the "linux-l3plugin" plugin config file; can also be set via "LINUX-L3PLUGIN_CONFIG" env variable.
  -linux-nsplugin-config="linux-nsplugin.conf": Location of the "linux-nsplugin" plugin config file; can also be set via "LINUX-NSPLUGIN_CONFIG" env variable.
  -logs-config="logs.conf": Location of the "logs" plugin config file; can also be set via "LOGS_CONFIG" env variable.
  -microservice-label="vpp1": microservice label; also set via 'MICROSERVICE_LABEL' env variable.
  -msgsync-config="msgsync.conf": Location of the "msgsync" plugin config file; can also be set via "MSGSYNC_CONFIG" env variable.
  -orchestrator-config="orchestrator.conf": Location of the "orchestrator" plugin config file; can also be set via "ORCHESTRATOR_CONFIG" env variable.
  -redis-config="redis.conf": Location of the "redis" plugin config file; can also be set via "REDIS_CONFIG" env variable.
  -restpapi-config="restpapi.conf": Location of the "restpapi" plugin config file; can also be set via "RESTPAPI_CONFIG" env variable.
  -telemetry-config="telemetry.conf": Location of the "telemetry" plugin config file; can also be set via "TELEMETRY_CONFIG" env variable.
  -vpp-aclplugin-config="vpp-aclplugin.conf": Location of the "vpp-aclplugin" plugin config file; can also be set via "VPP-ACLPLUGIN_CONFIG" env variable.
  -vpp-ifplugin-config="vpp-ifplugin.conf": Location of the "vpp-ifplugin" plugin config file; can also be set via "VPP-IFPLUGIN_CONFIG" env variable.

```

[bolt-concepts-doc]: ../user-guide/concepts.md#bolt 
[bolt-repo-conf-file]: https://github.com/ligato/cn-infra/blob/master/db/keyval/bolt/bolt.conf
[cassandra-plugin-doc]: ../plugins/plugin-overview.md#cassandra 
[cassandra-repo-conf-file]: https://github.com/ligato/cn-infra/blob/master/db/sql/cassandra/cassandra.conf
[consul-plugin-doc]: ../plugins/db-plugins.md#consul
[consul-repo-conf-file]: https://github.com/ligato/cn-infra/blob/master/db/keyval/consul/consul.conf
[etcd-plugin-doc]: ../plugins/db-plugins.md#etcd
[etcd-repo-conf-file]: https://github.com/ligato/cn-infra/blob/master/db/keyval/etcd/etcd.conf
[filedb-plugin-doc]: ../plugins/db-plugins.md#filedb
[filedb-repo-conf-file]: https://github.com/ligato/cn-infra/blob/master/db/keyval/filedb/filesystem.conf
[govppmux-plugin-doc]: ../plugins/vpp-plugins.md#govppmux-plugin
[govppmux-repo-conf-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/govppmux/govpp.conf
[grpc-plugin-doc]: ../plugins/connection-plugins.md#grpc-plugin
[grpc-repo-conf-file]: https://github.com/ligato/cn-infra/blob/master/rpc/grpc/grpc.conf
[rest-plugin-doc]: ../plugins/connection-plugins.md#rest-plugin
[kafka-plugin-doc]: ../plugins/infra-plugins.md#messagingkafka
[http-repo-conf-file]: https://github.com/ligato/cn-infra/blob/master/rpc/rest/http.conf
[kafka-repo-conf-file]: https://github.com/ligato/cn-infra/blob/master/messaging/kafka/kafka.conf
[kvs-plugin-doc]: ../plugins/kvs-plugin.md
[linux-interface-plugin-doc]: ../plugins/linux-plugins.md#linux-interface-plugin
[linux-repo-interface-conf-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/linux/ifplugin/linux-ifplugin.conf
[linux-repo-iptables-conf-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/linux/iptablesplugin/linux-iptablesplugin.conf
[linux-nsplugin-doc]: ../plugins/linux-plugins.md#namespace-plugin
[linux-repo-namespace-conf-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/linux/nsplugin/linux-nsplugin.conf
[linux-interface-proto]: https://github.com/ligato/vpp-agent/blob/master/proto/ligato/linux/interfaces/interface.proto
[linux-iptables-plugin-doc]: ../plugins/linux-plugins.md#ip-tables-plugin
[linux-l3-plugin-doc]: ../plugins/linux-plugins.md#l3-plugin
[linux-repo-l3-conf-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/linux/l3plugin/linux-l3plugin.conf
[linux-repo-iptables-conf-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/linux/iptablesplugin/linux-iptablesplugin.conf
[log-manager-plugin-doc]: ../plugins/infra-plugins.md#log-manager
[logs-repo-conf-file]: https://github.com/ligato/cn-infra/blob/master/logging/logmanager/logs.conf
[pm-plugin-doc]: ../plugins/infra-plugins.md#process-manager
[pm-repo-conf-file]: https://github.com/ligato/cn-infra/blob/master/exec/processmanager/pm.conf
[service-label-plugin-doc]: ../plugins/infra-plugins.md#service-label
[service-label-repo-conf-file]: 
[supervisor-plugin-doc]: ../plugins/infra-plugins.md#supervisor
[supervisor-repo-conf-file]: https://github.com/ligato/cn-infra/blob/master/exec/supervisor/supervisor.conf
[telemetry-plugin-doc]: ../plugins/vpp-plugins.md#telemetry-plugin 
[telemetry-repo-conf-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/telemetry/telemetry.conf

*[ACL]: Access Control List
*[HTTP]: Hypertext Transfer Protocol
*[REST]: Representational State Transfer