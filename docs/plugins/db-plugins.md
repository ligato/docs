# Database Plugins

This section discusses  database plugins.

---

## Datasync

Datasync defines the interfaces for data synchronization between app plugins and different backend data sources that include s data stores, message buses, or rpc-connected clients.

Data synchronization addresses the situation when multiple data sets must  synchronize following a published event.

Examples of components that publish events:

- Database updates 
- Message bus consuming messages from Kafka topics
- RPC clients using gRPC or REST APIs

The data synchronization APIs watch, and publish, asynchronously processed data change events. 

You might have data handled by one plugin containing references to data of another plugin. Therefore, you must maintain a proper time/order sequence of data resynchronization between plugins. 

The datasync plugin initiates a full data resync in the same order you  register your plugins in the `Init()` function. 

**References**

- [Ligato cn-infra repo][cn-infra-github]
- [Datasync repo folder][cn-infra-datasync-repo-folder]


---

### Watch Data API

The watch data API provides following functions:

- Subscribes to channels for data changes, and "abstracts away" message sources such as an etcd server.
<br></br>
- Processes a full data resync for startup and fault recovery scenarios. You will receive success or error feedback via callback.
<br></br>
- Process incremental data change. This optimized resync variant propagates to plugins, the minimal set of changes to reach synchronized state. You will receive success or error feedback via callback.



![datasync][datasync-image]
<p style="text-align: center; font-weight: bold">Watch data API Functions</p>

This API defines two types of events your plugins should support:

- Full resync event triggers a full resync of the entire configuration. This event occurs after an agent start/restart, or in a fault recovery scenario, such as lost and restored connectivity to an external data source.
<br></br>
- Incremental data change event triggers incremental processing of configuration changes. Each data change event contains the previous values, and the new or current values. Data synchronization switches to this optimized mode only after a successful full data resync.

The **Watch data API Functions** figure above:
 
 - Full resync event flow shown in **2.1 - 2.3**
 - Incremental data change event flow shown in **3.1 - 3.4** 

---

### Publish Data API

Your plugins use the publish data API to asynchronously publish events with data change values, and still remain abstracted away from the target data store, message bus or RPC client(s). The figure below illustrates the publish event flow. 

![datasync publish][datasync-publish-image]
<p style="text-align: center; font-weight: bold">Publish data API Functions</p>




---

## Data Broker 

The Data broker abstraction is based on the broker and watcher APIs:

* **Broker** - enables your plugins to pull data from a data store, or push data to the data store. You can retrieve data for a specific key, or by running a query. You can write data for a specific key, and multiple writes can execute in a transaction.
<br></br>
* **Watcher** - enables your plugins to watch data on a specified key. Watching looks for data changes, and receives notifications upon any change occurring.
<br></br>
  
![db][db-image]
<p style="text-align: center; font-weight: bold">Broker and Watcher APIs Functions</p>

The broker and watcher APIs abstract common database operations implemented by different data stores. However, differences exist between KV-based and sql-based data stores. Therefore, each package separately defines the Go broker and watcher interfaces. They use the same method names, but with different method arguments.

---

### Keyval Package

The [keyval package](https://godoc.org/github.com/ligato/cn-infra/db/keyval) defines the client API for accessing a KV data store. The broker API supports reading and manipulation of key-value pairs. The watcher API provides monitors changes in a data store. You can use arguments of type bytes and proto.Message for both interfaces. 

The `keyval` package also provides a skeleton for a KV data store plugin. A particular data store is selected in the `NewSkeleton` constructor using an argument of type `CoreBrokerWatcher`. The skeleton handles the plugin's life-cycle and provides unified access to data stores implementing the `KvPlugin` interface.

**References**

- [Ligato cn-infra repo][cn-infra-github]
- [Keyval repo folder][cn-infra-keyval-repo-folder]
- [keyval package](https://godoc.org/github.com/ligato/cn-infra/db/keyval)

---

## etcd

The etcd plugin provides access to an etcd KV data store.

**References**

- [KV data store concepts][kv-data-store-concepts]
- [etcd concepts][etcd-concepts]
- [etcd plugin folder][etcd-plugin-folder]
- [etcd conf file][etcd-conf-file]

### Configuration

You can define the location of the etcd conf file using the command line flag `etcd-config`, or by setting the `ETCD_CONFIG` env variable. 

For a list of etcd configuration options, see [etcd conf file][etcd-conf-file].  


### Status Check

The etcd plugin uses status check plugin to periodically issue a GET request to verify connection status. The etcd connection state can impact the global status of your agent. If your agent cannot establish a connection with the etcd server, the readiness and liveness probes issued by the probe plugin will return a negative result.

### Compacting

You can compact etcd using in one of two ways:

- Create an API by calling `plugin.Compact()` which will compact the database to the current revision.
- Set the `auto-compact` option in the conf file to the interval duration between auto compaction cycles.

### Reconnect Resynchronization



If connection to the etcd server breaks, you can automatically call a resync following reconnection. This option is disabled by default. Set `resync-after-reconnect` to `true` in the conf file to enable the feature.
  

---

## Redis

**References**

- [KV data store concepts][kv-data-store-concepts]
- [Redis concepts][redis-concepts]
- [Redis plugin folder][redis-plugin-folder]

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

You can initialize any of the above configuration instances in memory, or load the settings from a file using: 
```
err = config.ParseConfigFromYamlFile(configFile, &cfg)
```
You can also load any of the three configuration files using:
```
var cfg interface{}
cfg, err := redis.LoadConfig(configFile)
```

---

### Create Connection

```
client, err := redis.CreateClient(cfg)
db, err := redis.NewBytesConnection(client, logrus.DefaultLogger())
```

---

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

---

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

---

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

---

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

---

## Consul

The Consul plugin provides access to a Consul KV data store.

**References**

- [KV data store concepts][kv-data-store-concepts]
- [Consul concepts][consul-concepts]
- [Consul plugin folder][consul-plugin-folder]
- [Consul conf file][consul-conf-file]

### Configuration

The location of the Consul configuration file can be defined by the command line flag `consul-config`, or set via the `CONSUL_CONFIG` env variable.

The configuration options are described in the [Consul conf file][consul-conf-file] section of the user guide. 

### Status Check

The Consul plugin will use the Status Check plugin to periodically issue a GET request to verify connection status. The Consul connection state affects the global status of the agent. If the agent cannot establish a connection with Consul, both the readiness and the liveness probes from the probe plugin will return a negative result. 

### Reconnect Resync

If connection to the Consul data store is interrupted, resync can be automatically called upon reconnection. This option is disabled by default, but can be enabled in the Consul conf file. 

Set `resync-after-reconnect` to `true` to enable the feature.

---

## FileDB

The fileDB plugin uses the operating system's file system as a KV data store. This plugin watches for pre-defined files or directories, reads a configuration file, and generates events according to configuration changes.

All configuration data is resynced in the beginning just as it is for KV data stores. Configuration files can then be added, updated, moved, renamed or deleted. The plugin performs all of the necessary changes.

**References**

- [KV data store concepts][kv-data-store-concepts]
- [FileDB concepts][filedb-concepts]
- [FileDB plugin folder][filedb-plugin-folder]
- [FileDB conf file][filedb-conf-file]

### Configuration

All files or directories used as a data store must be defined in the conf file. The location of the file can be defined either by the command line flag `filedb-config`, or set using the `FILEDB_CONFIG` environment variable.

### Supported Formats

JSON and YAML-formatted data are supported.

* JSON `(*.json)`
* YAML `(*.yaml)`

The configuration options are described in the [FileDB conf file][filedb-conf-file] section of the user guide. 

### Data Structure

The structure of the configuration file conforms to standard .json and.yaml syntax.

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

The `key` must contain a prefix with a [microservice label][microsoft-label-prefix]. The fileDB plugin uses the prefix to identify the data in the conf file it will use to configure the system. All configuration data is stored internally in a local database. This allows a user or an application to compare events and respond with the correct `previous` value for a given key.

The `value` identifies a configuration item such as an interface.

### Data State Propagation

Data types such as interfaces and bridge domains that support status propagation can store their state in the filesystem. Set the `status-path` field in the conf file in order to store status. Status data will be stored in JSON or YAML formats.

[cn-infra-datasync-repo-folder]: https://github.com/ligato/cn-infra/tree/master/datasync
[cn-infra-db-repo-folder]: https://github.com/ligato/cn-infra/tree/master/db
[cn-infra-github]: https://github.com/ligato/cn-infra
[cn-infra-keyval-repo-folder]: https://github.com/ligato/cn-infra/tree/master/db/keyval
[consul-concepts]: ../user-guide/concepts.md#consul
[consul-conf-file]: ../user-guide/config-files.md#consul
[consul-plugin-folder]: https://github.com/ligato/cn-infra/tree/master/db/keyval/consul
[etcd-concepts]:  ../user-guide/concepts.md#etcd
[etcd-conf-file]: ../user-guide/config-files.md#etcd
[etcd-plugin-folder]: https://github.com/ligato/cn-infra/tree/master/db/keyval/etcd 
[filedb-concepts]: ../user-guide/concepts.md#filedb
[filedb-conf-file]: ../user-guide/config-files.md#filedb
[filedb-plugin-folder]: https://github.com/ligato/cn-infra/tree/master/db/keyval/filedb
[datasync-image]: ../img/user-guide/datasync_watch.png
[datasync-publish-image]: ../img/user-guide/datasync_pub.png
[db-image]: ../img/user-guide/db.png
[kv-data-store-concepts]: ../user-guide/concepts.md#key-value-data-store
[microsoft-label-prefix]: ../user-guide/concepts.md#keys-and-microservice-label
[redis-airport-example]: https://github.com/ligato/cn-infra/tree/master/examples/redis-lib/airport
[redis-concepts]: ../user-guide/concepts.md#redis
[redis-plugin-folder]: https://github.com/ligato/cn-infra/tree/master/db/keyval/redis

*[REST]: Representational State Transfer
