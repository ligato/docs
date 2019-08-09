# Config Files

This section discusses configuration files and flags.

---

## Config Directory

```bash
-config-dir=".": Location of the config files; can also be set via 'CONFIG_DIR' env variable.
```

This flag is used to define the directory for loading plugin configuration files.
Using `-<plugin>-config` for specific plugin will override this flag.  

## Plugin Configs
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

**Config**

| Option | Type | Default | Description |
|---|---|---|---|
| **db-path** | _string_ | | Path to Bolt DB file |
| **file-mode** | _os.FileMode_ | | File mode and permission bits in decimal format |
| **lock-timeout** | _time.Duration_ | | Timeout duration for waiting to obtain file lock, set to zero to wait indefinitely. |

### Cassandra

```bash
-cassandra-config= 
```

- `endpoints`: A list of host IP addresses of Cassandra cluster nodes
- `port`: Cassandra client port
- `op_timeout`: Connection timeout in nanoseconds. Default is 600ms
- `dial_timeout`: Initial session timeout in nanoseconds, used during initial dial to server. The default value is 600ms
- `redial_interval`: If set, gocql attempt to reconnect known down nodes in every ReconnectSleep. Default is 60 seconds
- `protocol_version`: Sets the version of the native protocol to use. This will enable features in the driver for specific protocol versions. This should be set to a known version (2,3,4) for the cluster being connected to. If it is 0 or unset (the default) then the driver will attempt to discover the highest supported protocol for the cluster. In clusters with nodes of different versions, the protocol selected is not defined (i.e. it can be any of the supported in the cluster).
- `tls`: Transport Layer Security setup

Can be used to set the common location for all configuration files.

<!--
### Configurator

```bash
-configurator-config=
```

Flag reserved for configurator plugin, currently not in use.
-->

### Consul

```bash
-consul-config=
```

- `address`: IP Address of the consul server 
- `resync-after-reconnect`: this field runs a resync procedure for all registered plugins in case the plugin is disconnected and then reconnects to the database. 

### etcd

```bash
-etcd-config=
```


- `endpoints`: A list of IP address and port entries in format `<ip-address>:<port>` for etcd server reachability.
- `dial-timeout`: Timeout window in nanoseconds for connection establishment.
- `operation-timeout`: Operation timeout in nanoseconds.
- `insecure-transport`: If set to `true` the TLS is omitted
- `insecure-skip-tls-verify`: Controls whether a client verifies the server's certificate chain and hostname. If InsecureSkipVerify is true, TLS accepts any certificate presented by the server and any hostname in that certificate. In this mode, TLS is susceptible to man-in-the-middle attacks. Therefore this should be used only for testing.
- `cert-file`: Path to a TLS certification file
- `key-file`: Path to a TLS certification key
- `ca-file`: Path to a CA file used to create a set of x509 certificates
- `auto-compact`: Defines interval between etcd auto-compaction cycles. Set to 0 to disable the feature
- `resync-after-reconnect`: If the connection to the etcd server is lost, this flag set to `true` will automatically run the entire resync procedure for all registered plugins upon reconnection.
- `allow-delayed-start`: Startup is permitted without connection to the etcd data store.The plugin will attempt to connect and if successful, a resync will be called
- `reconnect-interval`: Interval between etcd reconnect attempts in nanoseconds. The default value is 2 seconds. Does not apply if `delayed start` is turned off.

### FileDB

```bash
-filedb-config=
```

- `configuration-paths`: A set of files/directories with configuration files. If the target is a directory, all .json or .yaml files are read
- `status-path`: Path where the status data will be stored. If this is not defined, status is not propagated. The file extension determines whether the data will be stored as .json or .yaml. The target cannot be a directory.

### GoVPPMux

```bash
-govpp-config=
```



- `trace-enabled`: Enable or disable feature to measure binary API call duration. Measured time is shown directly in the log (info level). Measurements are also for certain procedures, such as resync of plugin startup. Turned off by default.
- `binapi-socket-path`: Path to a Unix-domain socket through which configuration requests are sent to VPP. Used if `connect-via-shm` is set to false and env. variable `GOVPPMUX_NOSOCK` is not defined. Defaults to "/run/vpp-api.sock"
- `connect-via-shm`: If enabled, GoVPP will access VPP for configuration requests via the shared memory instead of through the socket.
- `shm-prefix`: Custom shared memory prefix for VPP. Not used by default. Relevant only when GoVPP uses shared memory to send configuration requests to VPP. This is the case when  `connect-via-shm` is enabled or the environment variable `GOVPPMUX_NOSOCK` is defined.
- `stats-socket-path`: Socket path for reading VPP status. Default is "/run/vpp/stats.sock"
- `resync-after-reconnect`: If the connection to VPP is lost, this flag set to `true` will automatically run the entire resync procedure for all registered plugins upon reconnection.
- `retry-request-count`: Binary API requests failed because the temporary VPP disconnect can be re-tried. This field defines the number of retry attempts. Default is zero, meaning the feature is disabled
- `retry-request-timeout`: Defines timeout between binary API retry attempts. The default value is 500ms. This field is not applicable if the `retry-request-count` is set to zero.
- `retry-connect-count`: Defines the maximim number of attempts GoVPPMux tries to reach VPP. The default is 3.
- `retry-connect-timeout`: Defines the VPP connection retry timeout in nanoseconds. The default is 1 second.

### GRPC

```bash
-grpc-config=
```

- `endpoint`: GRPC endpoint defines IP address and port (if TCP type) or unix domain socket file (if Unix type)
- `permission`: If Unix domain socket file is used for GRPC communication, permissions to the file can be set here. The permission value uses the standard three-or-four number Linux binary reference.
- `force-socket-removal`: If socket file exists in a defined path, it is not removed by default and the GRPC plugin attempts to use it. Set the force removal flag to `true` ensures that the socket file will always be recreated.
- `network`: Available socket types are tcp, tcp4, tcp6, unix and unixpacket. If not set, defaults to TCP.
- `max-msg-size`: Maximum message size in bytes for inbound messages. If not set, GRPC uses the default is 4096.
- `max-concurrent-streams`: Limit of server streams to each server transport

This flag can be used to set the GRPC port:

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

| Option | Type | Default | Description |
|---|---|---|---|
| **record-transaction-history** | _bool_ | `true` | Enable recording history of processed transactions |
| **transaction-history-age-limit** | _uint32_ (in minutes) | `24 * 60` | Age limit for recording transaction history |
| **permanently-recorded-init-period** | _uint32_ (in minutes) | `60` | Duration of period from init that will be permanently recorded |
| **enable-txn-simulation** | _bool_ | `false` | Enable transaction simulation |
| **print-txn-summary** | _bool_ | `true` | Print transaction summary for each transaction |

### Linux Interface plugin

```bash
-linux-ifplugin-config=
```

- `disabled`: Used to disable Linux ifplugin. Turned off by default
- `go-routines-count`: How many goroutines (at most) will split configured network namespaces to execute the Retrieve operation in parallel

### Linux IP Tables

```bash
-linux-iptables-config=
```

- `disabled`: Used to disable Linux iptables plugin. Turned off by default
- `go-routines-count`: How many goroutines (at most) will split configured network namespaces to execute the Retrieve operation in parallel.

### Linux L3

```bash
-linux-l3plugin-config=
```

- `disabled`: Used to disable Linux l3plugin. Turned off by default
- `go-routines-count`: How many goroutines (at most) will split configured network namespaces to execute the Retrieve operation in parallel

### Log Manager

```bash
--logs-config=
```

- `default-level`: Sets default config level for every plugin. Overwritten by environmental variable `INITIAL_LOGLVL`
- `loggers`: Specifies a list of named loggers with their respective log level
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
- `hooks`: Specifies a list of hooks for logging to external links. This includes parameters such as protocol, address, port and levels for a specific hook.
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

### Namespace

```bash
-linux-nsplugin-config=
```

- `disabled`: Used to disable Linux nsplugin. Turned off by default

### Process Manager

```bash
-process-manager-config=
```

- `template-path`: Path where the templates will be stored in the filesystem

### REST

```bash
-http-config=
```

- `endpoint`: Address of the HTTP server
- `read-timeout`: Maximum amount of time for reading the entire request, including the body. Because read timeout does not let handlers make per-request decisions on each request body's acceptable deadline or upload rate, most users will prefer to use read-header-timeout. It is valid to use both. `read-timeout` is set in nanoseconds.
- `read-header-timeout`: Maximum amount of time to read request headers. The connection's read deadline is reset after reading the headers and the Handler can decide what is considered too slow for the body. `read-header-timeout` is set in nanoseconds.
- `write-timeout`: Maximum amount of time before timing out writes to a response. It is reset whenever a new request's header is read. It does not let Handlers make decisions on a per-request basis. Write timeout is set in nanoseconds.
- `idle-timeout`: Maximum amount of time to wait for the next request when keepalives are enabled. If the idle timeout is zero,  the value of ReadTimeout is used. If both are zero, there is no timeout. Idle timeout is set in nanoseconds.
- `max-header-bytes`: Maximum number of bytes the server will read parsing the request header's keys and values, including the request line. It does not limit the size of the request body.
- `enable-token-auth`: Enables or disables HTTP token authentication
- `users`: Registers additional users with permissions. Admin with full access to every permission group is registered automatically. Password must be in hashed form.

Format:
```
users:
   - name: <name>
     password_hash: <hash>
     permissions: [<group1>, <group2>, ...]
```

- `token-expiration`: Token expiration time in nanoseconds. Zero means no expiration time
- `password-hash-cost`: Number in range between 4 and 31 used as a parameter for hashing passwords. Large numbers require more CPU time and memory to process.
- `token-signature`: A string value used as a key to sign tokens

This flag can be used to set the HTTP port:

```bash
-http-port=
```

### Service Label

Flag to set the microservice label for a given vpp-agent.

```bash
--microservice-label=
```

### VPP Interface

```bash
-vpp-ifplugin-config=
```

- `mtu`: Default maximum transmission unit (MTU) size. The value is used if an interface without an MTU is created. Note that the MTU in the interface configuration is preferred.
- `status-publishers`: enables the vpp-agent to send status data back to etcd. To allow it, add the desired status publishers. Currently supported for `etcd` and `redis` and both options can be chosen together.


*[ACL]: Access Control List
*[HTTP]: Hypertext Transfer Protocol
*[REST]: Representational State Transfer