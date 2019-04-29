# List of supported flags

This page contains information about Ligato vpp-agent configuration flags.

---

### ACL plugin

```bash
-vpp-aclplugin-config=
```

Flag reserved for the ACL plugin, currently not in use.

### Bolt plugin

```bash
-bolt-conf= 
```

- `db-path`: Path to Bolt DB file
- `file-mode`: File's mode and permission bits in decimal format ... 432 = --rw-rw----
- `lock-timeout`: Timeout is the amount of time (in nanoseconds) to wait to obtain a file lock When setting to zero it will wait indefinitely

### Cassandra config

```bash
-cassandra-conf= 
```

- `endpoints`: A list of host IP addresses of Cassandra cluster nodes
- `port`: Cassandra client port
- `op_timeout`: Connection timeout in a nanosecond. The default value is 600ms
- `dial_timeout`: Initial session timeout in nanoseconds, used during initial dial to server. The default value is 600ms
- `redial_interval`: If set, gocql attempt to reconnect known down nodes in every ReconnectSleep. Default is 60 seconds
- `protocol_version`: ProtoVersion sets the version of the native protocol to use, this will enable features in the driver for specific protocol versions, generally this should be set to a known version (2,3,4) for the cluster being connected to. If it is 0 or unset (the default) then the driver will attempt to discover the highest supported protocol for the cluster. In clusters with nodes of different versions the protocol selected is not defined (ie, it can be any of the supported in the cluster).
- `tls`: Transport Layer Security setup

### Common configuration directory

```bash
-config-dir= 
```

Can be used to set the common location for all configuration files.

### Configurator

```bash
-configurator-config=
```

Flag reserved for configurator plugin, currently not in use.

### Consul plugin

```bash
-consul-config=
```

Provides all fields required for Consul plugin:

- `address`: IP Address of the consul server 
- `resync-after-reconnect`: this field runs resync procedure for all registered plugins in case the plugin loses connection to the database and then reconnects back 

### ETCD plugin

```bash
-etcd-config=
```

Startup-config for the ETCD plugin:

- `endpoints`: A list of IP address and port entries in format <ip-address>:<port>, where the ETCD server is reachable
- `dial-timeout`: A time window in <ns> where the connection should be established
- `operation-timeout`: Operation timeout in <ns>
- `insecure-transport`: If set to `true` the TLS is omitted
- `insecure-skip-tls-verify`: Controls whether a client verifies the server's certificate chain and hostname. If InsecureSkipVerify is true, TLS accepts any certificate presented by the server and any hostname in that certificate. In this mode, TLS is susceptible to man-in-the-middle attacks. This should be used only for testing.
- `cert-file`: Path to a TLS certification file
- `key-file`: Path to a TLS certification key
- `ca-file`: Path to a CA file used to create a set of x509 certificates
- `auto-compact`: Defines interval between ETCD auto-compaction cycles. Set to 0 to disable the feature
- `resync-after-reconnect`: If ETCD server lost connection, the flag allows to automatically run the whole resync procedure for all registered plugins if it reconnects
- `allow-delayed-start`: Allows starting without connected ETCD database. The plugin will try to connect and if successful, overall resync will be called
- `reconnect-interval`: Interval between ETCD reconnect attempts in <ns>. The default value is 2 seconds. Has no use if `delayed start` is turned off

### FileDB plugin

```bash
-filedb-config=
```

- `configuration-paths`: A set of files/directories with configuration files. If target is a directory, all .json or .yaml files are read
- `status-path`: Path where the status data will be stored. If not defined, status is not propagated. The file extension determines whether the data will be stored as .json or .yaml. Target cannot be a directory.

### GoVPPMux plugin

```bash
-govpp-config=
```

Initial configuration of the GoVPP multiplexer.

- `trace-enabled`: Enable or disable feature to measure binary API call duration. Measured time is shown directly in the log (info level). Measurement is taken also for certain procedures, like resync of plugin startup. Turned off by default
- `binapi-socket-path`: Path to a Unix-domain socket through which configuration requests are sent to VPP. Used if connect-via-shm is set to false and env. variable `GOVPPMUX_NOSOCK` is not defined. Defaults to "/run/vpp-api.sock"
- `connect-via-shm`: If enabled, GoVPP will access the VPP for configuration requests via the shared memory instead of through the socket.
- `shm-prefix`: Custom shared memory prefix for VPP. Not used by default. Relevant only when GoVPP uses shared memory to send configuration requests to VPP (connect-via-shm is enabled or env. variable `GOVPPMUX_NOSOCK` is defined)
- `stats-socket-path`: Socket path for reading VPP status, default is "/run/vpp/stats.sock"
- `resync-after-reconnect`: If VPP lost connection, this flag allows to automatically run the whole resync procedure for all registered plugins after reconnection.
- `retry-request-count`: Binary API requests failed because of the temporary VPP disconnect can be re-tried. The field defines a number of retry attempts. Default is zero, meaning the feature is disabled
- `retry-request-timeout`: Defines timeout between binary API retry attempts in case some of them fails. The default value is 500ms. If retry-request-count is set to zero, the field has no effect
- `retry-connect-count`: Defines max number of attempts GoVPPMux tries to reach the VPP (default is 3 attempts)
- `retry-connect-timeout`: Defines time in nanoseconds between VPP connection retries (default is 1 second)

### GRPC handler

```bash
-grpc-config=
```

- `endpoint`: GRPC endpoint defines IP address and port (if TCP type) or unix domain socket file (if Unix type)
- `permission`: If Unix domain socket file is used for GRPC communication, permissions to the file can be set here. The permission value uses standard three-or-four number Linux binary reference.
- `force-socket-removal`: If socket file exists in a defined path, it is not removed by default, GRPC plugin tries to use it. Set the force removal flag to 'true' ensures that the socket file will be always re-created
- `network`: Available socket types: tcp, tcp4, tcp6, unix, unixpacket. If not set, defaults to TCP
- `max-msg-size`: Maximum message size in bytes for inbound messages. If not set, GRPC uses the default 4MB
- `max-concurrent-streams`: Limit of server streams to each server transport

GRPC also allows to use of simple flag to set GRPC port:
```bash
-grpc-port=
```

### Kafka

```bash
-kafka-config=
```
- `addrs`: Kafka server addresses
- `group_id`: Name of the consumer's group
- `tls`: Crypto/TLS configuration

### KV Scheduler

```bash
-kvscheduler-config=
```

Flag reserved for the KVScheduler plugin, currently not in use.

### Linux Interface plugin

```bash
-linux-ifplugin-config=
```

- `disabled`: Used to disable Linux ifplugin. Turned off by default
- `go-routines-count`: How many goroutines (at most) will split configured network namespaces to execute the Retrieve operation in parallel

### Linux IP tables plugin

```bash
-linux-iptables-config=
```

- `disabled`: Used to disable Linux iptables plugin. Turned off by default
- `go-routines-count`: How many goroutines (at most) will split configured network namespaces to execute the Retrieve operation in parallel.

### Linux L3 plugin

```bash
-linux-l3plugin-config=
```

- `disabled`: Used to disable Linux l3plugin. Turned off by default
- `go-routines-count`: How many goroutines (at most) will split configured network namespaces to execute the Retrieve operation in parallel

### Log manager

```bash
--logs-config=
```

- `default-level`: Set default config level for every plugin. Overwritten by environmental variable `INITIAL_LOGLVL`
- `loggers`: Specifies a list of named loggers with respective log level
Example:
```
loggers:
  - name: "agentcore",
    level: debug
  - name: "status-check",
    level: info
  - name: "linux-plugin",
    level: warn
```
- `hooks`: Specifies a list of hook for logging to external links with respective parameters (protocol, address, port and levels) for given hook
Example:
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

### Namespace plugin

```bash
-linux-nsplugin-config=
```

- `disabled`: Used to disable Linux nsplugin. Turned off by default

### Process Manager

```bash
-process-manager-config=
```

- `template-path`: Template path is a path where the templates will be stored in the filesystem

### REST handler

```bash
-http-config=
```

- `endpoint`: Address of the HTTP server
- `read-timeout`: Maximum duration for reading the entire request, including the body. Because read timeout does not let handlers make per-request decisions on each request body's acceptable deadline or upload rate, most users will prefer to use read-header-timeout. It is valid to use them both. Read timeout is set in nanoseconds.
- `read-header-timeout`: Header timeout is the amount of time allowed to read request headers. The connection's read deadline is reset after reading the headers and the Handler can decide what is considered too slow for the body. Read header timeout is set in nanoseconds.
- `write-timeout`: WriteTimeout is the maximum duration before timing out writes of the response. It is reset whenever a new request's header is read. Like ReadTimeout, it does not let Handlers make decisions on a per-request basis. Write timeout is set in nanoseconds.
- `idle-timeout`: Maximum amount of time to wait for the next request when keepalives are enabled. If the idle timeout is zero,  the value of ReadTimeout is used. If both are zero, there is no timeout. Idle timeout is set in nanoseconds.
- `max-header-bytes`: Field controls the maximum number of bytes the server will read parsing the request header's keys and values, including the request line. It does not limit the size of the request body.
- `enable-token-auth`: Enables/disabled HTTP token authentication
- `users`: Registers additional users with permissions. Admin with full access to every permission group is registered automatically. Password has to be in hashed form.
Format:
```
users:
   - name: <name>
     password_hash: <hash>
     permissions: [<group1>, <group2>, ...]
```
- `token-expiration`: Token expiration time in nanoseconds. Zero means no expiration time
- `password-hash-cost`: Number in range 4-31 used as a parameter for hashing passwords. Large numbers require a lot of CPU time and memory to process.
- `token-signature`: A string value used as a key to sign a tokens

REST also allows to use of the simple flag to set HTTP port:
```bash
-http-port=
```

### Service Label

Flag to set the microservice label for a given agent.
```bash
--microservice-label=
```

### VPP Interface plugin

```bash
-vpp-ifplugin-config=
```

- `mtu`: Default maximum transmission unit. The value is used if an interface without MTU is created (i.e. MTU in interface configuration is preferred)
- `status-publishers`: VPP agent allows to send status data back to ETCD. To allow it, add desired status publishers. Currently supported for **etcd** and **redis** (both options can be chosen together)