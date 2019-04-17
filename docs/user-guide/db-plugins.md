# Datasync plugin

Package datasync defines the interfaces for the abstraction 
of a data synchronization between app plugins and different backend data
sources (such as data stores, message buses, or RPC-connected clients).

In this context, data synchronization is about multiple data sets 
that need to be synchronized whenever a particular event is published. 
The event can be published by:
- database (when particular data was changed); 
- by message bus (such as consuming messages from Kafka topics); 
- or by RPC clients (when GRPC or REST service call ).

The data synchronization APIs are centered around watching 
and publishing data change events. These events are processed
asynchronously.

The data handled by one plugin can have references to the data
of another plugin. Therefore, a proper time/order of data
resynchronization between plugins needs to be maintained. The datasync
plugin initiates a full data resync in the same order as the other
plugins have been registered in Init().
  
### Watch data API

Watch data API is used by app plugin to:
1. Subscribe channels for data changes using `Watch()`, while being
   abstracted from a particular message source (data store, message bus
   or RPC)
2. Process full Data RESYNC (startup & fault recovery scenarios).
   Feedback is given back to the user of this API (e.g. successful
   configuration or an error) via callback.
3. Process Incremental Data CHANGE. This is an optimized variant
   of RESYNC, where only the minimal set of changes (deltas) needed
   to get in-sync gets propagated to plugins.
   Again, feedback to the user of the API (e.g. successful configuration
   or an error) is returned via callback.

![datasync](../img/user-guide/datasync_watch.png)

This APIs define two types of events that a plugin must be able
to process:
1. Full Data RESYNC (resynchronization) event is defined to trigger
   resynchronization of the whole configuration. This event is used
   after agent start/restart, or for a fault recovery scenario
   (e.g. when agent's connectivity to an external data source is lost
   and restored).
2. Incremental Data CHANGE event is defined to trigger incremental
   processing of configuration changes. Each data change event contains
   both the previous and the new/current value. The Data synchronization
   is switched to this optimized mode only after successful Full Data
   RESYNC.

### Publish data API

Publish data API is used by app plugins to asynchronously publish events 
with particular data change values and still remain abstracted from
the target data store, message bus, local/RPC client.

![datasync publish](../img/user-guide/datasync_pub.png)

# Data Broker 

The CN-Infra Data Broker abstraction (see the diagram below) is based on
two APIs: 
* **The Broker API** - used by app plugins to PULL (i.e. retrieve) data
  from a data store or PUSH (i.e. write) data into the data store.  Data
  can be retrieved for a specific key or by running a query. Data can be 
  written for a specific key. Multiple writes can be executed in a 
  transaction.
* **The Watcher API** - used by app plugins to WATCH data on a specified 
  key. Watching means to monitor for data changes and be notified as soon
  as a change occurs.

![db](../img/user-guide/db.png)

The Broker & Watcher APIs abstract common database operations implemented 
by different databases (ETCD, Redis, Cassandra). Still, there are major 
differences between [keyval](keyval)-based & [sql](sql)-based databases.
Therefore the Broker & Watcher Go interfaces are defined in each package 
separately; while the method names for a given operation are the same, 
the method arguments are different.

### Key-value datastore

The `keyval` package defines the client API to access a key-value data 
store. It comprises two sub-APIs: the `Broker` interface supports reading 
and manipulation of key-value pairs; the `Watcher` API provides functions 
for monitoring of changes in a data store. Both interfaces are available
with arguments of type `[]bytes` (raw data) and `proto.Message` (protobuf
formatted data).

The `keyval` package also provides a skeleton for a key-value plugin.
A particular data store is selected in the `NewSkeleton` constructor
using an argument of type `CoreBrokerWatcher`. The skeleton handles
the plugin's life-cycle and provides unified access to datastore
implementing the `KvPlugin` interface.

# Etcd plugin

The Etcd plugin provides access to an etcd key-value data store.

### Configuration

- Location of the Etcd configuration file can be defined either by the 
  command line flag `etcd-config` or set via the `ETCD_CONFIG`
  environment variable.

### Status Check

- If injected, Etcd plugin will use StatusCheck plugin to periodically
  issue a minimalistic GET request to check for the status
  of the connection.
  The etcd connection state affects the global status of the agent.
  If agent cannot establish connection with etcd, both the readiness
  and the liveness probe from the probe plugin will return a negative result 
  (accessible only via REST API in such case).

### Compacting

You can compact Etcd using two ways.

- using API by calling `plugin.Compact()` which will compact the database
  to the current revision.
- using config file by setting `auto-compact` option to the duration of
  period that you want the Etcd to be compacted.

### Reconnect resynchronization

- If connection to the ETCD is interrupted, resync can be automatically called 
  after re-connection. This option is disabled by default and has to be allowed
  in the `etcd.conf` file.
  
  Set `resync-after-reconnect` to `true` to enable the feature.
  
Redis is the implementation of the key-value Data Broker client
API for the Redis key-value data store.
See [cn-infra/db/keyval](../../../db/keyval) for the definition
of the key-value Data Broker client API.

The entity BytesConnectionRedis provides access to CRUD as well as event
subscription API's.
```
   +-----+   (Broker)   +------------------------+ -->  CRUD      +-------+ -->
   | app |                   |  BytesConnectionRedis  |                 | Redis |
   +-----+    <-- (KeyValProtoWatcher)  +------------------------+  <--  events    +-------+
```

# Redis

The code snippets below provide examples to help you get started.
For simplicity, error handling is omitted.

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
- See sample YAML configurations [(*.yaml files)](../../../examples/redis-lib)

You can initialize any of the above configuration instances in memory,
or load the settings from file using
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
 NOTE: You must configure Redis for it to publish key space events.
```
   config SET notify-keyspace-events KA
```
See [EVENT NOTIFICATION](https://raw.githubusercontent.com/antirez/redis/3.2/redis.conf)
for more details.

You can find detailed examples in
- [simple](../../../examples/redis-lib/simple)
- [airport](../../../examples/redis-lib/airport)

### Resiliency

Connection/read/write time-outs, failover, reconnection and recovery
are validated by running the airport example against a Redis Sentinel
Cluster. Redis nodes are paused selectively to simulate server down:
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

# Consul plugin

The Consul plugin provides access to a consul key-value data store.

### Configuration

- Location of the Consul configuration file can be defined either by the
  command line flag `consul-config` or set via the `CONSUL_CONFIG`
  environment variable.

### Status Check

- If injected, Consul plugin will use StatusCheck plugin to periodically
  issue a minimalistic GET request to check for the status of the connection.
  The consul connection state affects the global status of the agent.
  If agent cannot establish connection with consul, both the readiness
  and the liveness probe from the probe plugin will return a negative result 
  (accessible only via REST API in such case).

### Reconnect resynchronization

- If connection to the Consul is interrupted, resync can be automatically called
  after re-connection. This option is disabled by default and has to be allowed
  in the etcd.conf file.

  Set `resync-after-reconnect` to `true` to enable the feature.

# FileDB

The fileDB plugin allows to use the file system of a operating system as a key-value data store. The filesystem
plugin watches for pre-defined files or directories, reads a configuration and sends response events according
to changes.

All the configuration is resynced in the beginning (as for standard key-value data store). Configuration files
then can be added, updated, moved, renamed or removed, plugin makes all the necessary changes.

Important note: fileDB as datastore is read-only from the plugin perspective, changes from within the plugin
are not allowed.

### Configuration

All files/directories used as a data store must be defined in configuration file. Location of the file
can be defined either by the command line flag `filedb-config` or set via the `FILEDB_CONFIG`
environment variable.

### Supported formats

* JSON `(*.json)`
* YAML `(*.yaml)`

### Data structure

Plugin currently supports only JSON and YAML-formatted data. The format of the file is as follows for JSON:

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

Key has to contain also instance prefix with micro service label, so plugin knows which parts of the configuration 
are intended for it. All configuration is stored internally in local database. It allows to compare events and 
respond with correct 'previous' value for a given key. 

### Data state propagation

Data types supporting status propagation (like interfaces or bridge domains) can store the state in filesystem.
There is a field in configuration file called `status-path` which has to be set in order to store the status.
Status data will be stored in the same format as for configuration, type is defined by the file extension 
(JSON or YAML).

Data will not be propagated if target path directory.