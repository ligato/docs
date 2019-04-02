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