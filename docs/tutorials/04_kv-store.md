# KV Data Store

---

Tutorial code: [KV Data Store][code-link]

In this tutorial, you will learn how use an external key-value (KV) data store. Before running through this tutorial, you should complete the [Hello World tutorial](01_hello-world.md) and the [Plugin Dependencies tutorial](02_plugin-deps.md). In addition, you should be familiar with etcd. To learn about etcd, a good place to start is this [etcd overview](https://www.ibm.com/cloud/learn/etcd). 

You will also need to install the [gogo protobuf generator](https://github.com/gogo/protobuf) on your local machine. 

[etcd][1] is used as the KV data store in this tutorial. Note that Ligato supports other [KV data stores](../user-guide/concepts.md#key-value-data-store).

!!! Note
    MyPlugin is the name of the plugin used in this tutorial. Note that the concepts, explanations, tasks and code block contents used in this tutorial apply to the HelloWorld plugin used in the previous tutorials.

---

The common interface for all KV data store implementations is `KvProtoPlugin`: 

```go
type KvProtoPlugin interface {
	NewBroker(keyPrefix string) ProtoBroker
	NewWatcher(keyPrefix string) ProtoWatcher
	Disabled() bool
	OnConnect(func() error)
	String() string
}
```

To use etcd as your KV data store plugin, define a field for the 
`KvProtoPlugin` interface in your plugin. Then initialize the interface with an etcd plugin instance in your plugin's constructor. 

This creates a dependency on the KV data store in your plugin. You have satisfied this dependency by using the default etcd plugin of `etcd.DefaultPlugin`:

```go
type MyPlugin struct {
	infra.PluginDeps
	KVStore keyval.KvProtoPlugin
}

func NewMyPlugin() *MyPlugin {
	// ...
	p.KVStore = &etcd.DefaultPlugin
	return p
}
```

---

Next, create a broker. The broker is a facade  for plugins or components that communicate with the data store. The broker conceals the complexity of interacting with the different data stores and provides a simple read/write API. 

Initialize the broker with a key prefix. The key prefix serves as the root of the
KV tree that the broker will operate on. The broker uses the key prefix for all
of its get, list, put and delete operations. 
 
The key prefix of `/myplugin/` is used in this code block:

```go
broker := p.KVStore.NewBroker("/myplugin/")
```

For more information about key prefixes, see the [key prefix section](../user-guide/concepts.md#key-prefix) of the _User Guide_.

---

The broker accepts `proto.Message` parameters in its methods. You will need to define a protobuf model for data that you wish to communicate to and from the data store.

Use the simple [.proto model][6] called  `Greetings`:
```json
message Greetings {
    string greeting = 1;
}
```

---

Next, generate Go code based on your model using the [--gogo protobuf generator](https://github.com/gogo/protobuf). You have a single `model.proto` file stored in the model folder. It is a good practice to place the generated Go code in the same folder. 

To store the proto model and Go code in the same model folder, assign the `model` directory in the `go:generate` directive's `--proto_path` and `--gogo_out` flags:
```go
//go:generate protoc --proto_path=model --gogo_out=model ./model/model.proto
```

To generate Go files using the Go compiler, use this command:
```
go generate
``` 
`Go generate` must be run explicitly. It scans Go files in the current path for
the `generate` directives and then invokes the protobuf compiler.

---

Next, employ the generated Go structures to interact with the KV data store through the broker.

Use the broker's `Put` method to update the value with a key of `/myplugin/greetings/hello`:

```go
value := &model.Greetings{
	Greeting: "Hello",
}
err := broker.Put("greetings/hello", value)
if err != nil {
	// handle error
}
```

---

Use the broker's `GetValue` method to retrieve a value from the KV data store:

```go
value := new(model.Greetings)
found, rev, err := broker.GetValue("greetings/hello", value)
if err != nil {
	// handle error
}else if !found {
	// handle not found
}
```

---

To watch for changes in the KV data store, initialize a watcher:

```go
watcher := p.KVStore.NewWatcher("/myplugin/")
```

---

Then define your callback function that will process the changes:

```go
onChange := func(resp keyval.ProtoWatchResp) {
	key := resp.GetKey()
	value := new(model.Greetings)
	if err := resp.GetValue(value); err != nil {
		// handle error
	}
	// process change
}
```

---

Start watching for key prefixes:

```go
cancelWatch := make(chan string)
err := watcher.Watch(onChange, cancelWatch, "greetings/")
if err != nil {
	// handle error
}
```

Use the `cancelWatch` channel to cancel watching.

---

**Run the KV Data Store tutorial code**

1. Open a terminal session 
<br>
<br>
2. Change to the 04_kv-store folder:
```
cn-infra git:(master) cd examples/tutorials/04_kv-store
```
3. Start etcd

!!! Note
    Make sure that the etcd.conf file contained in the 04_kv-store folder uses an endpoint of `0.0.0.0`. If not, you will receive an error when attempting to run the tutorial code.

The `etcd.conf` file should look like this: 
```
insecure-transport: true
dial-timeout: 1000000000
endpoints:
 - "0.0.0.0:2379"
```

For additional etcd troubleshooting tips, see the _Tutorial etcd troubleshooting_ section below.

To start etcd, use this command:
```
etcd
```
4. Open another terminal session.
<br>
<br>
5. Change to the 04_kv-store folder
```
cn-infra git:(master) cd examples/tutorials/04_kv-store
```
6. Run code:
```
go run main.go
```


Output:
```
INFO[0000] Starting agent version: v0.0.0-dev            CommitHash= BuildDate= logger=agent
INFO[0000] Connected to Etcd (took 1.423873ms)           endpoints="[0.0.0.0:2379]" logger=etcd
INFO[0000] Status check for etcd was started             logger=etcd
INFO[0000] Agent started with 4 plugins (took 5ms)       logger=agent
INFO[0000] No greetings found..                          logger=myplugin
INFO[0002] updating..                                    logger=myplugin
INFO[0002] Put change, Key: "greetings/hello" Value: greeting:"Hello"   logger=myplugin
INFO[0005] Agent plugin state update.                    lastErr="<nil>" logger=status-check plugin=etcd state=ok
```

# __Tutorial etcd troubleshooting__

You might encounter the following problems if the tutorial code does not run to completion properly.

!!! Error
    `etcd.conf` is not present in the tutorial folder

Output:
```
INFO[0000] Starting agent version: v0.0.0-dev            BuildDate= CommitHash= loc=“agent/agent.go(134)” logger=agent
INFO[0000] ETCD config not found, skip loading this plugin  loc="etcd/plugin_impl_etcd.go(293)" logger=etcd
ERRO[0000] KV store is disabled                          loc="04_kv-store/main.go(43)" logger=defaultLogger
```

The etcd plugin must be configured with the address of the etcd server. This is typically done through the etcd.conf file. In most cases, the etcd.conf file must be in the same folder where the agent executable (tutorial) is started.

!!! Error
    Incorrect `endpoints` value in the etcd.conf file

Output:
```
CHMETZ-M-72TZ:04_kv-store chrismetz$ go run main.go
INFO[0000] Starting agent version: v0.0.0-dev            BuildDate= CommitHash= loc=“agent/agent.go(134)" logger=agent
WARN[0001] Failed to connect to Etcd: context deadline exceeded  endpoints=“[172.17.0.1:2379]” loc=“etcd/bytes_broker_impl.go(60)” logger=etcd
ERRO[0001] error connecting to ETCD: context deadline exceeded  loc="04_kv-store/main.go(43)" logger=defaultLogger
```

The message indicates a connection problem with `172.17.0.1:2379` which presumably is the address of the etcd server.

First look for this value in the log when you started etcd
```
etcdserver: advertise client URLs = http://localhost:2379
```
This is the address the etcd plugin should use to connect to the etcd server. It is okay to use a localhost IP address for running this tutorial on our computer.

Next check the etcd.conf file
```
insecure-transport: true
dial-timeout: 1000000000
endpoints:
 - "172.17.0.1:2379"
```
It appears the etcd plugin is not configured with the address of the etcd server. The solution is to edit the etcd.conf so it looks like this:
```
insecure-transport: true
dial-timeout: 1000000000
endpoints:
 - "0.0.0.0:2379"
```



[1]: https://github.com/ligato/cn-infra/tree/master/db/keyval/etcd
[2]: https://github.com/ligato/cn-infra/tree/master/db/keyval/consul
[3]: https://github.com/ligato/cn-infra/tree/master/db/keyval/bolt
[4]: https://github.com/ligato/cn-infra/tree/master/db/keyval/filedb
[5]: https://github.com/ligato/cn-infra/tree/master/db/keyval/redis
[6]: https://github.com/ligato/cn-infra/blob/master/examples/tutorials/04_kv-store/model/model.proto
[7]: https://github.com/ligato/cn-infra/blob/master/db/keyval/plugin_api_keyval.go
[code-link]: https://github.com/ligato/cn-infra/tree/master/examples/tutorials/04_kv-store