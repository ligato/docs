# DB plugins

## Datasync plugin

Datasync defines the interfaces for the abstraction of data synchronization between app plugins and different backend data sources such as data stores, message buses, or RPC-connected clients.

Data synchronization is about multiple data sets that need to be synchronized whenever a particular event is published. The event can be published by:

- database (when particular data was changed); 
- message bus (such as consuming messages from Kafka topics); 
- or by RPC clients (using GRPC or REST calls ).

The data synchronization APIs are centered around watching and publishing data change events. These events are processed asynchronously.

The data handled by one plugin can have references to the data of another plugin. Therefore, a proper time/order sequence of data resynchronization between plugins needs to be maintained. The datasync plugin initiates a full data resync in the same order as the other plugins have been registered in Init().

!!! Note
    In the discussions that follow, the term `agent` is used to describe a server. A KV data store holds data/configuration information for multiple agents (servers). Mechanisms are introduced to propagate  data/configurations changes from the KV data store to the different agents. In addition these changes may require a resynchronization. 
  
### Watch data API

Watch data API (see `Watch data API Functions` figure below) is used by app plugins to:

- Subscribe to channels for data changes using `Watch()`, while being "abstracted away" from the particular message source such as etcd.
- Process a full Data RESYNC (startup & fault recovery scenarios). Feedback is provided to the user of this API (e.g. success or error) via callback.
- Process Incremental Data CHANGE. This is an optimized variant of RESYNC, where only the minimal set of changes (deltas) needed to required to reach synchronized state is propagated to plugins. Again, feedback to the user of the API (e.g. successful configuration or an error) is returned via callback.

![datasync][datasync-image]
<p style="text-align: center; font-weight: bold">Watch data API Functions</p>

This API define two types of events that a plugin be able to process:

- Full Data RESYNC (resynchronization) event triggers a resync of the entire configuration. This event is used after an agent start/restart, or for a fault recovery scenario (e.g. when the agent's (i.e. server) connectivity to an external data source is lost and restored).
- Incremental Data CHANGE event triggers incremental processing of configuration changes. Each data change event contains both the previous and the new/current value. The Data synchronization is switched to this optimized mode only after a successful Full Data RESYNC.

### Publish data API

Publish data API (see `Publish data API Functions` figure below) is used by app plugins to asynchronously publish events with data change values and still remain abstracted away from the target data store, message bus or RPC client(s).

![datasync publish][datasync-publish-image]
<p style="text-align: center; font-weight: bold">Publish data API Functions</p>
## Data Broker 

The Data Broker abstraction (see `Broker and Watcher APIs Functions` figure below) is based on two APIs: 

* **Broker** - used by app plugins to `pull` (i.e. read) data from a data store or `push` (i.e. write) data into the data store. Data can be retrieved for a specific key or by running a query. Data can be written for a specific key. Multiple writes can be executed in a transaction.
* **Watcher** - used by app plugins to WATCH data on a specified key. Watching means to monitor for data changes and receive a notification as soon as a change occurs.
  
![db][db-image]
<p style="text-align: center; font-weight: bold">Broker and Watcher APIs Functions</p>

The Broker & Watcher APIs abstract common database operations implemented by different data stores such as etcd, Redis and Cassandra. Still, there are major differences between key-value-based & sql-based data stores. Therefore the Broker & Watcher Go interfaces are defined in each package separately; while the method names for a given operation are the same, the method arguments are different.

### Key-value data store

The `keyval` package defines the client API to access a key-value data store. It is comprised of two sub-APIs: 

- `Broker` supports reading and manipulation of key-value pairs. 
- `Watcher` provides functions for monitoring of changes in a data store. 

Both interfaces are available with arguments of type `[]bytes` (raw data) and `proto.Message` (protobuf formatted data).

The `keyval` package also provides a skeleton for a key-value plugin. A particular data store is selected in the `NewSkeleton` constructor using an argument of type `CoreBrokerWatcher`. The skeleton handles the plugin's life-cycle and provides unified access to data stores implementing the `KvPlugin` interface.

## etcd plugin

The etcd plugin provides access to an etcd key-value data store. The host or server where the etcd data store is running is also known as an etcd server.

### Configuration

- Location of the etcd configuration file can be defined either by the command line flag `etcd-config` or by setting the `ETCD_CONFIG` environment variable. Examples:

```bash
vpp-agent -etcd-config=/opt/vpp-agent/dev/etcd.conf
```

```bash
export ETCD_CONFIG=/opt/vpp-agent/dev/etcd.conf
```

### Status Check

- If injected, the etcd plugin will use the Status Check plugin to periodically issue a GET request to check connection status. The etcd connection state affects the global status of the agent. If the agent cannot establish a connection with etcd, both the readiness and the liveness probe from the probe plugin will return a negative result (accessible only via a REST API in such cases).

### Compacting

You can compact etcd using two ways.

- using an API by calling `plugin.Compact()` which will compact the database to the current revision.
- using a config file by setting `auto-compact` option to the duration of period that wish the etcd to be compacted.

### Reconnect resynchronization

- If connection to the etcd is interrupted, resync can be automatically called after re-connection. This option is disabled by default but can be enabled in the `etcd.conf` file.
  
Set `resync-after-reconnect` to `true` to enable the feature.
  

## Redis

The code snippets below provide examples to help you get started. For simplicity, error handling is omitted.

### Need to import following dependencies

```
import "github.com/ligato/cn-infra/db/keyval/kvproto"
import "github.com/ligato/cn-infra/db/keyval/redis"
import "github.com/ligato/cn-infra/utils/config"
import "github.com/ligato/cn-infra/logging/logrus"
```

### Define client configuration based on your Redis installation.

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

### Create connection from configuration

```
client, err := redis.CreateClient(cfg)
db, err := redis.NewBytesConnection(client, logrus.DefaultLogger())
```

### Create Brokers / Watchers from connection

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

### Perform CRUD operations

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

### Subscribe to key space events

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
    You must configure Redis so it may publish key space events.
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

Redis is the implementation of the key-value Data Broker client API for the Redis key-value data store. The entity `BytesConnectionRedis` provides access to CRUD as well as event subscription API's.
```
   +-----+   (Broker)   +------------------------+ -->  CRUD      +-------+ -->
   | app |                   |  BytesConnectionRedis  |                 | Redis |
   +-----+    <-- (KeyValProtoWatcher)  +------------------------+  <--  events    +-------+
```


## Consul plugin

The Consul plugin provides access to a consul key-value data store.

### Configuration

- Location of the Consul configuration file can be defined either by the command line flag `consul-config` or set via the `CONSUL_CONFIG` environment variable.

### Status Check

- If injected, the Consul plugin will use the Status Check plugin to periodically issue a GET request to check connection status. The Consul connection state affects the global status of the agent. If the agent cannot establish a connection with Consul, both the readiness and the liveness probe from the probe plugin will return a negative result (accessible only via a REST API in such cases).

### Reconnect resynchronization

- If connection to the Consul data store is interrupted, resync can be automatically called after re-connection. This option is disabled by default but can be enabled in the etcd.conf file. Set `resync-after-reconnect` to `true` to enable the feature.

## FileDB

The fileDB plugin uses the file system of an operating system as a key-value data store. The filesystem plugin watches for pre-defined files or directories, reads a configuration and sends response events according to changes.

All configuration is resynced in the beginning (as for standard key-value data store). Configuration files then can be added, updated, moved, renamed or removed, plugin makes all of the necessary changes.

!!! danger "Important"
    FileDB as datastore is read-only from the plugin perspective. Changes from within the plugin are not allowed.

### Configuration

All files/directories used as a data store must be defined in the configuration file. Location of the file can be defined either by the command line flag `filedb-config` or set via the `FILEDB_CONFIG` environment variable.

### Supported formats

* JSON `(*.json)`
* YAML `(*.yaml)`

### Data structure

Currently only JSON and YAML-formatted data is supported. JSON format follows:

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

For YAML:

```

---
data:
    -
        key: '<key>'
        value: '<modelled data>'

```

Key must contain instance prefix with a microservice label. This is so the plugin knows which portions of the configuration are applicable to it. All configuration data is stored internally in a local database. This allows one to compare events and respond with the correct `previous` value for a given key. 

### Data state propagation

Data types supporting status propagation (e.g. interfaces or bridge domains) can store their state in the filesystem. There is a field in the configuration file called `status-path` which must be set in order to store status. Status data will be stored in JSON or YAML formats.


[datasync-image]: ../img/user-guide/datasync_watch.png
[datasync-publish-image]: ../img/user-guide/datasync_pub.png
[db-image]: ../img/user-guide/db.png
[redis-airport-example]: https://github.com/ligato/cn-infra/tree/master/examples/redis-lib/airport

*[REST]: Representational State Transfer
