# KV Data Store

---

Link to code: [Working with KV Data Stores][code-link]

In this tutorial we will learn how to use an external key-value (KV) data store.
The tutorial shows you how to read and write data to/from the data store and how to 
watch for changes. 

Requirements:

* Complete the ['Hello World Agent'](01_hello-world.md) tutorial
* Complete the ['Plugin Dependencies'](02_plugin-deps.md) tutorial

We will be using [etcd][1] as the KV data store. Note that is not the only choice as the Ligato infrastructure supports several other key-value data stores: [Consul][2], [BoltDB][3], [FileDB][4], 
[Redis][5].

The common interface for all KV data store implementations is `KvProtoPlugin`, 
defined in [`cn-infra/db/keyval/plugin_api_keyval.go`][7]:

```go
type KvProtoPlugin interface {
	NewBroker(keyPrefix string) ProtoBroker
	NewWatcher(keyPrefix string) ProtoWatcher
	Disabled() bool
	OnConnect(func() error)
	String() string
}
```

To use etcd as our KV data store plugin, we simply define a field for the 
`KvProtoPlugin` interface in our plugin and initialize it with an etcd plugin 
instance in our plugin's constructor. Note that we use the default etcd plugin
(`etcd.DefaultPlugin`). 

Essentially we create a dependency on the KV data store in our plugin and satisfy it with the default etcd KV data
store implementation:

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

Once we have the appropriate KV data store plugin set up, we can create a broker. The broker is a facade (mediator) through which we will communicate with the data store. The broker conceals the complexity of interacting with the different data stores and provides a simple read/write API. 

The broker must be initialized with a key prefix that becomes the root for the
kv tree that the broker will operate on. The broker uses the key prefix for all
of  its operations (Get, List, Put, Delete). In this example we will use `/myplugin/`.

```go
broker := p.KVStore.NewBroker("/myplugin/")
```

!!! note
    The etcd plugin must be configured with the address of the etcd server. This is typically done through the etcd confige file. In most cases, the etcd config file must be in the same folder where the agent executable is started. 

If the etcd config file is not found, the etcd plugin will be disabled, and you will see an error that looks something like this:
```
level=error msg="KV store is disabled" loc="04_kv-store/main.go(41)" logger=defaultLogger
```

The broker accepts `proto.Message` parameters in its methods. Therefore we need to define a protobuf model for data that we want to put in and read from the data store respectively.

For this tutorial we define a very simple model - `Greetings`. You can find it in
the [`model.proto`][6] file.

```proto
message Greetings {
    string greeting = 1;
}
```
!!! note
    It is a good practice to put all protobuf definitions for a plugin in a `model` directory.

Next, we need to generate Go code from our model. We will use the generated Go 
structures as parameters in calls to the broker. The code generation is controlled
from the `go:generate` directive. Since we only have one go file in this tutorial,
we put the directive `model` directory like so:

```go
//go:generate protoc --proto_path=model --gogo_out=model ./model/model.proto
```
The above directive assumes that we use the gogo protobuf generator.
The source protobuf files can be found in the model directory and the
generated files will also be put into the model directory. Note also that to
use the gogo protobuf generator, you must install it on your machine as 
described [here](https://github.com/gogo/protobuf).

We use the go compiler to generate Go files from the model with this command:
```
go generate
``` 
Go generate must be run explicitly. It scans go files in the current path for
the `generate` directives and then invokes the protobuf compiler. In our 
tutorial, `go generate` will create the file `model.pb.go`.

Now we can finally use the generated Go structures to update a value in the 
KV data store. We will use the broker's `Put` method:

```go
value := &model.Greetings{
	Greeting: "Hello",
}
err := broker.Put("greetings/hello", value)
if err != nil {
	// handle error
}
```

The value above will be updated for key `/myplugin/greetings/hello`.

To retrieve a value from the KV data store we will use broker's `GetValue` method:

```go
value := new(model.Greetings)
found, rev, err := broker.GetValue("greetings/hello", value)
if err != nil {
	// handle error
}else if !found {
	// handle not found
}
```

To watch for changes in the KV data store we need to initialize a watcher:

```go
watcher := p.KVStore.NewWatcher("/myplugin/")
```

Then we need to define our callback function that will process the changes:

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

Now we can start watching for a key prefix(es):

```go
cancelWatch := make(chan string)
err := watcher.Watch(onChange, cancelWatch, "greetings/")
if err != nil {
	// handle error
}
```

The channel `cancelWatch` can be used to cancel watching.

Complete working example can be found at [examples/tutorials/04_kv-store](https://github.com/ligato/cn-infra/blob/master/examples/tutorials/04_kv-store).

[1]: https://github.com/ligato/cn-infra/tree/master/db/keyval/etcd
[2]: https://github.com/ligato/cn-infra/tree/master/db/keyval/consul
[3]: https://github.com/ligato/cn-infra/tree/master/db/keyval/bolt
[4]: https://github.com/ligato/cn-infra/tree/master/db/keyval/filedb
[5]: https://github.com/ligato/cn-infra/tree/master/db/keyval/redis
[6]: https://github.com/ligato/cn-infra/blob/master/examples/tutorials/04_kv-store/model/model.proto
[7]: https://github.com/ligato/cn-infra/blob/master/db/keyval/plugin_api_keyval.go
[code-link]: https://github.com/ligato/cn-infra/tree/master/examples/tutorials/04_kv-store