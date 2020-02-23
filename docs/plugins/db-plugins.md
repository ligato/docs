# DB plugins

## Datasync

Datasync defines the interfaces for the abstraction of data synchronization between app plugins and different backend data sources such as data stores, message buses, or RPC-connected clients.

Data synchronization addresses the situation when multiple data sets must by synchronized as a result of a published event.

Examples of components that publish events:

- database upon updates to data
- message bus consuming messages from Kafka topics
- RPC clients using GRPC or REST API calls

The data synchronization APIs are centered around watching, and publishing data change events. These events are processed asynchronously.

The data handled by one plugin can have references to the data of another plugin. Therefore, a proper time/order sequence of data resynchronization between plugins must be maintained. The datasync plugin initiates a full data resync in the same order as the other plugins have been registered in Init().

!!! Note
    In the discussions that follow, the term `server agent` is used to describe a server. A KV data store holds data/configuration information for multiple server agents. Mechanisms are introduced to propagate  data/configurations changes from the KV data store to the different server agents. These changes may require a resynchronization.

---

### Watch Data API

The watch data API is used by plugins to:

- Subscribe to channels for data changes using `Watch()`, while being "abstracted away" from the particular message source such as an etcd server.
- Process a full Data RESYNC (startup & fault recovery scenarios). Feedback is provided to the user of this API (e.g. success or error) via callback.
- Process Incremental Data CHANGE. This is an optimized variant of RESYNC, where the minimal set of changes (deltas) to reach synchronized state are propagated to plugins. Feedback to the user of the API (e.g. successful configuration or error) is returned via callback.

![datasync][datasync-image]
<p style="text-align: center; font-weight: bold">Watch data API Functions</p>

This API defines two types of events that a plugin should support:

- Full Data RESYNC (resynchronization) event triggers a resync of the entire configuration. This event is used after an agent start/restart, or for a fault recovery scenario (e.g. when the agent's connectivity to an external data source is lost and restored).
- Incremental Data CHANGE event triggers incremental processing of configuration changes. Each data change event contains both the previous and the new/current value. The Data synchronization is switched to this optimized mode only after a successful Full Data RESYNC.

---

### Publish Data API

The publish data API is used plugins to asynchronously publish events with data change values and still remain abstracted away from the target data store, message bus or RPC client(s).

![datasync publish][datasync-publish-image]
<p style="text-align: center; font-weight: bold">Publish data API Functions</p>
## Data Broker 

The Data Broker abstraction is based on two APIs:

* **Broker** - used by plugins to `pull` (i.e. read) data from a data store or `push` (i.e. write) data into the data store. Data can be retrieved for a specific key or by running a query. Data can be written for a specific key. Multiple writes can be executed in a transaction.
* **Watcher** - used by plugins to `watch` data on a specified key. Watching means to monitor for data changes, and receive a notifications upon any change occurring.
  
![db][db-image]
<p style="text-align: center; font-weight: bold">Broker and Watcher APIs Functions</p>

The broker amd watcher APIs abstract common database operations implemented by different data stores such as etcd, Redis and Cassandra. Still, there are major differences between KV-based and sql-based data stores. Therefore the broker and watcher Go interfaces are defined in each package separately; The method names for a given operation are the same, the method arguments are different.

---

### Keyval Package

The `keyval` package defines the client API for accessing a KV data store. It is comprised of two sub-APIs:

- `Broker` supports reading and manipulation of key-value pairs. 
- `Watcher` provides functions for monitoring of changes in a data store. 

Both interfaces are available with arguments of type `bytes` (raw data) and `proto.Message` (protobuf-formatted data).

The `keyval` package also provides a skeleton for a KV data store plugin. A particular data store is selected in the `NewSkeleton` constructor using an argument of type `CoreBrokerWatcher`. The skeleton handles the plugin's life-cycle and provides unified access to data stores implementing the `KvPlugin` interface.

---

## etcd

The etcd plugin provides access to an etcd KV data store. The host or server where the etcd data store is running is also known as an etcd server.

### Configuration

- Location of the etcd configuration file can be defined either by the command line flag `etcd-config` or by setting the `ETCD_CONFIG` environment variable. Examples:

```bash
vpp-agent -etcd-config=/opt/vpp-agent/dev/etcd.conf
```

```bash
export ETCD_CONFIG=/opt/vpp-agent/dev/etcd.conf
```

etcd config file options are located [here](../user-guide/config-files.md#etcd)

### Status Check

The etcd plugin will uses Status Check plugin to periodically issue a GET request to verify connection status. The etcd connection state affects the global status of the agent. If the agent cannot establish a connection with the etcd server, both the readiness and the liveness probe from the probe plugin will return a negative result.

### Compacting

You can compact etcd using two ways.

- using an API by calling `plugin.Compact()` which will compact the database to the current revision.
- using a config file by setting `auto-compact` option to the duration of period that wish the etcd to be compacted.

### Reconnect resynchronization

If connection to the etcd server is interrupted, resync can be automatically called following reconnection. This option is disabled by default but can be enabled in the `etcd.conf` file.
  
Set `resync-after-reconnect` to `true` to enable the feature.
  

## Redis

The code snippets below provide examples to help you get started. For simplicity, error handling is omitted.

### Import Dependencies

```
import "github.com/ligato/cn-infra/db/keyval/kvproto"
import "github.com/ligato/cn-infra/db/keyval/redis"
import "github.com/ligato/cn-infra/utils/config"
import "github.com/ligato/cn-infra/logging/logrus"
```

### Define Client Configuration

- Single Node
var cfg redis.NodeConfig
- Sentinel Enabled Cluster
var cfg redis.SentinelConfig
- Redis Cluster
var cfg redis.ClusterConfig

You can initialize any of the above configuration instances in memory, or load the settings from a file using 
```
err = config.ParseConfigFromYamlFile(configFile, &cfg)
```
You can also load any of the three configuration files using
```
var cfg interface{}
cfg, err := redis.LoadConfig(configFile)
```

### Create Connection

```
client, err := redis.CreateClient(cfg)
db, err := redis.NewBytesConnection(client, logrus.DefaultLogger())
```

### Create Brokers and Watchers

```
//create broker/watcher that share the same connection pools.
bytesBroker := db.NewBroker("some-prefix")
bytesWatcher := db.NewWatcher("some-prefix")

// create broker/watcher that share the same connection pools,
// capable of processing protocol-buffer generated data.
wrapper := kvproto.NewProtoWrapper(db)
protoBroker := wrapper.NewBroker("some-prefix")
protoWatcher := wrapper.NewWatcher("some-prefix")
```

### Perform CRUD Operations

```
// put
err = db.Put("some-key", []byte("some-value"))
err = db.Put("some-temp-key", []byte("valid for 20 seconds"),
             datasync.WithTTL(20*time.Second))

// get
value, found, revision, err := db.GetValue("some-key")
if found {
   ...
}

// Note: flight.Info implements proto.Message.
f := flight.Info{
       Airline:  "UA",
       Number:   1573,
       Priority: 1,
    }
err = protoBroker.Put("some-key-prefix", &f)
f2 := flight.Info{}
found, revision, err = protoBroker.GetValue("some-key-prefix", &f2)

// list
keyPrefix := "some"
kv, err := db.ListValues(keyPrefix)
for {
   kv, done := kv.GetNext()
   if done {
       break
   }
    key := kv.GetKey()
   value := kv.GetValue()
}

// delete
found, err := db.Delete("some-key")
// or, delete all keys matching the prefix "some-key".
found, err := db.Delete("some-key", datasync.WithPrefix())

// transaction
var txn keyval.BytesTxn = db.NewTxn()
txn.Put("key101", []byte("val 101")).Put("key102", []byte("val 102"))
txn.Put("key103", []byte("val 103")).Put("key104", []byte("val 104"))
err := txn.Commit()
```

### Key Space Event Subscription

```
watchChan := make(chan keyval.BytesWatchResp, 10)
err = db.Watch(watchChan, "some-key")
for {
   select {
    case r := <-watchChan:
       switch r.GetChangeType() {
       case datasync.Put:
           log.Infof("KeyValProtoWatcher received %v: %s=%s", r.GetChangeType(),
                     r.GetKey(), string(r.GetValue()))
       case datasync.Delete:
           ...
       }
   ...
   }
}
```
!!! note
    You must configure Redis to publish key space events.
```
config SET notify-keyspace-events KA
```

### Resiliency

Connection/read/write time-outs, failover, reconnection and recovery are validated by running the [airport example][redis-airport-example] against a Redis Sentinel Cluster. Redis nodes are paused selectively to simulate server down:
```
$ docker-compose ps
```
|Name   |Command  |State|Ports|
|-------|---------|-----|-----|
|dockerredissentinel_master_1 | docker-entrypoint.sh redis ... | Paused | 6379/tcp |
|dockerredissentinel_slave_1 | docker-entrypoint.sh redis ... | Up | 6379/tcp |
|dockerredissentinel_slave_2 | docker-entrypoint.sh redis ... | Up | 6379/tcp |
|dockerredissentinel_sentinel_1 | sentinel-entrypoint.sh | Up | 26379/tcp, 6379/tcp |
|dockerredissentinel_sentinel_2 | sentinel-entrypoint.sh | Up | 26379/tcp, 6379/tcp |
|dockerredissentinel_sentinel_3 | sentinel-entrypoint.sh | Up | 26379/tcp, 6379/tcp |

Redis is the implementation of the KV data broker client API for the Redis KV data store. The entity `BytesConnectionRedis` provides access to CRUD as well as event subscription APIs.
```
   +-----+   (Broker)   +------------------------+ -->  CRUD      +-------+ -->
   | app |                   |  BytesConnectionRedis  |                 | Redis |
   +-----+    <-- (KeyValProtoWatcher)  +------------------------+  <--  events    +-------+
```


## Consul

The Consul plugin provides access to a Consul KV data store.

### Configuration

The location of the Consul configuration file can be defined by the command line flag `consul-config`, or set via the `CONSUL_CONFIG` environment variable.

### Status Check

If injected, the Consul plugin will use the Status Check plugin to periodically issue a GET request to verify connection status. The Consul connection state affects the global status of the agent. If the agent cannot establish a connection with Consul, both the readiness and the liveness probe from the probe plugin will return a negative result (accessible only via a REST API in such cases).

### Reconnect Resync

If connection to the Consul data store is interrupted, resync can be automatically called upon reconnection. This option is disabled by default but can be enabled in the consul.conf file. Set `resync-after-reconnect` to `true` to enable the feature.

## FileDB

The fileDB plugin uses the operating system's file system as a KV data store. This plugin watches for pre-defined files or directories, reads a configuration file, and generates events according to configuration changes.

All configuration data is resynced in the beginning just as it is for KV data stores. Configuration files can then be added, updated, moved, renamed or deleted. The plugin performs all of the necessary changes.

### Configuration

All files or directories used as a data store must be defined in the configuration file. The location of the file can be defined either by the command line flag `filedb-config` or set using the `FILEDB_CONFIG` environment variable.

### Supported Formats

JSON and YAML-formatted data are supported.

* JSON `(*.json)`
* YAML `(*.yaml)`

### Data Structure

The structure of the configuration files conform to standard .json and.yaml syntax.

JSON format example:
```
{
    "data": [
        {
            "key": "<key>",
            "value": {
                <proto-modelled data>
            }
        },
        {
            "key": "<key>",
            "value": {
                <proto-modelled data>
            }
        },
        ...
    ]
}

``` 

YAML format example:

```

---
data:
    -
        key: '<key>'
        value: '<modelled data>'

```

The `key` must contain a prefix with a microservice label. The fileDB plugin uses the prefix to identify the data in the configuration file it will apply to configure the system. All configuration data is stored internally in a local database. This allows a user or an application to compare events and respond with the correct `previous` value for a given key.

The `value` identifies a configuration item such as an interface.

### Data State Propagation

Data types supporting status propagation (e.g. interfaces or bridge domains) can store their state in the filesystem. There is a field in the configuration file called `status-path` which must be set in order to store status. Status data will be stored in JSON or YAML formats.


[datasync-image]: ../img/user-guide/datasync_watch.png
[datasync-publish-image]: ../img/user-guide/datasync_pub.png
[db-image]: ../img/user-guide/db.png
[redis-airport-example]: https://github.com/ligato/cn-infra/tree/master/examples/redis-lib/airport

*[REST]: Representational State Transfer
