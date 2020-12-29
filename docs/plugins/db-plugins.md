# Database Plugins

This section describes database plugins.

---

## Datasync

Datasync defines the interfaces for data synchronization between app plugins and different backend data sources that include data stores, message buses, and rpc clients.

Data synchronization handles the situation when multiple data sets must synchronize after a published event.

Examples of components that publish events:

- Database updates. 
- Message bus consuming messages from Kafka topics.
- RPC clients using gRPC or REST APIs.

The data synchronization APIs watch, and publish, asynchronously processed data change events. 

You might have data handled by one plugin containing references to data of another plugin. Therefore, you must maintain a proper time/order sequence of data resynchronization between plugins. 

The datasync plugin initiates a full data resync in the same order you  register your plugins in the `Init()` function. 

**References**

- [Datasync repo folder][cn-infra-datasync-repo-folder]


---

### Watch Data API

The watch data API provides following functions:

- Subscribes to channels for data changes, and "abstracts away" message sources such as an etcd server.
<br></br>
- Processes a full data resync for startup and fault recovery scenarios. You receive success or error feedback with a callback.
<br></br>
- Processes incremental data changes. This optimized resync variant propagates to plugins, the minimal set of changes to reach synchronized state. You receive success or error feedback with a callback.



![datasync][datasync-image]
<p style="text-align: center; font-weight: bold">Watch data API Functions</p>

Your plugins should support two types of events defined by this API.

- **Full resync event** triggers a full resync of the entire configuration. This event occurs after an agent start/restart, or in a fault recovery scenario, such as lost and restored connectivity to an external data source. 
<br></br>
- **Incremental data change event** triggers incremental processing of configuration changes. Each data change event contains the previous values, and the new or current values. Data synchronization switches to this optimized mode only after a successful full data resync. 

The _Watch data API Functions_ figure above depicts both event flows.
 
 - Full resync event flow shown in **2.1 - 2.4**
 - Incremental data change event flow shown in **3.1 - 3.4** 

---

### Publish Data API

Your plugins use the publish data API to asynchronously publish events with data change values. This API provides a common abstraction for your plugins that communicate with a target data store, message bus, or RPC clients.  

The _Publish data API Functions_ figure below illustrates publish event flow. 

![datasync publish][datasync-publish-image]
<p style="text-align: center; font-weight: bold">Publish data API Functions</p>




---

## Data Broker 

The Data broker abstraction is based on the broker and watcher APIs.

* **Broker** - enables your plugins to pull data from a data store, or push data to the data store. You can retrieve data for a specific key, or by running a query. You can write data for a specific key, and perform multiple writes in a single transaction.
<br></br>
* **Watcher** - enables your plugins to watch data on a specified key. Watching looks for data changes, and receives notifications upon any change occurring.
<br></br>
  
![db][db-image]
<p style="text-align: center; font-weight: bold">Broker and Watcher APIs Functions</p>

The broker and watcher APIs abstract common database operations implemented by different data stores. However, differences exist between KV-based and sql-based data stores. Therefore, each package separately defines the Go broker and watcher interfaces. They use the same method names, but with different method arguments.

---

### Keyval Package

The [keyval package](https://godoc.org/github.com/ligato/cn-infra/db/keyval) defines the client API for accessing a KV data store. The broker API supports reading and manipulation of key-value pairs. The watcher API monitors changes in a data store. You can use arguments of type bytes, and type proto.Message, for both interfaces. 

The `keyval` package also provides a skeleton for a KV data store plugin. You select a data store in the `NewSkeleton` constructor using an argument of type `CoreBrokerWatcher`. The skeleton handles the plugin's life-cycle, and offers unified access to data stores implementing the `KvPlugin` interface.

**References**

- [Keyval repo folder][cn-infra-keyval-repo-folder]
- [keyval GoDocs](https://godoc.org/github.com/ligato/cn-infra/db/keyval)

---

## etcd

The etcd plugin communicates with an etcd KV data store.

**References**

- [KV data store concepts][kv-data-store-concepts]
- [etcd concepts][etcd-concepts]
- [etcd plugin folder][etcd-plugin-folder]
- [etcd conf file][etcd-conf-file]

### Configuration

You define the location of the etcd conf file using the command line flag `etcd-config`, or by setting the `ETCD_CONFIG` env variable. 

For a list of etcd configuration options, see [etcd conf file][etcd-conf-file].  


### Status Check

The etcd plugin uses the [status check plugin](infra-plugins.md#status-check) to verify connection status. It performs this task by sending periodic probes. The etcd connection state impacts the global status of your agent. If your agent cannot establish a connection with the etcd server, the readiness and liveness probes issued by the probe plugin return a negative result.

### Compacting

You compact etcd using one of two ways:

- Create an API by calling `plugin.Compact()` that compacts the database to the current revision.
<br></br>
- Set the `auto-compact` option in the conf file to the interval duration between auto compaction cycles.

### Reconnect Resync

If the connection to the etcd server breaks, you can automatically call a resync following reconnection. This option is disabled by default. You enable this feature with the `resync-after-reconnect` flag contained in the conf file. 
```
resync-after-reconnect: true
```
  

---

## Redis

The redis plugin communicates with a Redis data store.

**References**

- [KV data store concepts][kv-data-store-concepts]
- [Redis concepts][redis-concepts]
- [Redis plugin folder][redis-plugin-folder]

---

The following list describes the steps to integrate a Redis data store:

- Import dependencies.
- Define client configuration.
- Create connection.
- Create brokers and watchers.
- Perform CRUD operations.
- Key space event subscription.
 

!!! Note
    The code examples that follow omit error handling. 

### Import Dependencies

```
import "github.com/ligato/cn-infra/db/keyval/kvproto"
import "github.com/ligato/cn-infra/db/keyval/redis"
import "github.com/ligato/cn-infra/utils/config"
import "github.com/ligato/cn-infra/logging/logrus"
```

### Define Client Configuration

You can configure different types of clients:

- Single Node
```
var cfg redis.NodeConfig
```
- Sentinel Enabled Cluster
```
var cfg redis.SentinelConfig
```
- Redis Cluster
```
var cfg redis.ClusterConfig
```
You initialize any of the above configuration instances in memory, or load the settings from a file using the following: 
```
err = config.ParseConfigFromYamlFile(configFile, &cfg)
```
You can load any of the three configuration files using the following:
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

You must configure Redis to publish key space events.
```
config SET notify-keyspace-events KA
```

---

### Resiliency

To see how the Redis plugin handles time-outs, failover, reconnection and recovery, run the [airport example][redis-airport-example] against a Redis sentinel cluster. The example selectively pauses Redis nodes to simulate server down events.

The following figure illustrates the KV data broker client API for the Redis data store. The `BytesConnectionRedis` function provides access to CRUD callbacks and event subscription APIs.
```
+-----+         (Broker)          +------------------------+ --->   CRUD     +-------+ 
| app |                           |  BytesConnectionRedis  |                 | Redis |
+-----+ <-- (KeyValProtoWatcher)  +------------------------+ <---  events    +-------+
```
<p style="text-align: center; font-weight: bold">Data broker Client API for Redis</p>

---

## Consul

The Consul plugin communicates with a Consul KV data store.

**References**

- [KV data store concepts][kv-data-store-concepts]
- [Consul concepts][consul-concepts]
- [Consul plugin folder][consul-plugin-folder]
- [Consul conf file][consul-conf-file]

### Configuration

You define the location of the Consul conf file using the command line flag `consul-config`, or by setting the `CONSUL_CONFIG` environment variable.

For a list of Consul configuration options, see [Consul conf file][consul-conf-file]. 

### Status Check

The Consul plugin uses the [status check plugin](infra-plugins.md#status-check) to verify connection status. It performs this task by sending periodic probes. The Consul connection state impacts the global status of your agent. If your agent cannot establish a connection with the Consul server, the readiness and liveness probes issued by the probe plugin return a negative result.
 

### Reconnect Resync

If the connection to the Consul server breaks, you can automatically call a resync following reconnection. This option is disabled by default. You enable this feature with the `resync-after-reconnect` flag contained in the conf file. 
```
resync-after-reconnect: true
```
 

---

## FileDB

The fileDB plugin uses the operating system's file system as a KV data store. This plugin watches for pre-defined files or directories, reads a configuration file, and generates events according to configuration changes.

As with other KV data stores, the KV Scheduler resyncs all configuration data at startup. You can then add, update, move, rename, or delete configuration files. The plugin handles all necessary changes.

**References**

- [KV data store concepts][kv-data-store-concepts]
- [FileDB concepts][filedb-concepts]
- [FileDB plugin folder][filedb-plugin-folder]
- [FileDB conf file][filedb-conf-file]

### Configuration

You must define all data store files or directories in the conf file. You define the location of the filedb conf file using the command line flag `filedb-config`, or by setting the `FILEDB_CONFIG` environment variable.


### Data Structure

The structure of the configuration file conforms to standard JSON and YAML syntax.

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

The `key` contains a prefix with a [microservice label][microsoft-label-prefix]. The fileDB plugin uses the prefix to identify configuration data. You store all configuration data internally in a local database. This lets you compare events and respond with the correct `previous` value for a given key.

The `value` identifies a configuration item such as an interface.

### Data State Propagation

Data types, such as interfaces and bridge domains, that support status propagation, store their state in the filesystem. To store state, you must set the `status-path` field in the conf file.
```
status-path: "/path/to/status.ext"
```
 

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
