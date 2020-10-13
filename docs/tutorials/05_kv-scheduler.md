# KV Scheduler

---

Tutorial code: [KV Scheduler][code-link]

In this tutorial, you will learn about the KV Scheduler. Before running through this tutorial, you should complete the [Hello World tutorial](01_hello-world.md) and the [Plugin Dependencies tutorial](02_plugin-deps.md). 

To best reinforce what you will learn from this tutorial, read about the KV Scheduler, KV Desciptors and VPP Configuration Order in the _Concepts_ section of the _User Guide_.

This tutorial does not use etcd or any other northbound (NB) KV data store for event processing. To keep it simple, this tutorial performs event processing by calling the KV Scheduler API. 

---
 
Start by defining a simple NB [proto model][1] that you will use in your HelloWorld plugin. The model defines two messages:
 
 - `Interface` 
 - `Route` 
 
 The `route` depends on the `interface`. This model demonstrates a simple dependency between two configuration items.

---

!!! Note "Important" 
    The VPP agent uses the Orchestrator component. It is responsible for collecting northbound data originating from multiple sources such as a KV data store or a gRPC client. To marshall/unmarshall proto messages defined in NB proto models, the Orchestrator requires `messages` contain `message names`.  

To generate code where message names are present in proto messages, use the following special protobuf option together with its import:
```proto
import "github.com/gogo/protobuf/gogoproto/gogo.proto";
option (gogoproto.messagename_all) = true;
```

To register your HelloWorld plugin with the KV Scheduler, and to work with your new model, you need an `Adapter` and `Descriptor` for every proto.Message.

---
 
#### Adapters
 
Let's start with adapters. An adapter handles the conversion of a proto-defined type to
a bare `proto.Message` that the KV Scheduler works with. Since this is boilerplate code, there is tooling to auto-generate
adapters. The code generator is called `descriptor-adapter` and can be found in the [KV Scheduler plugin folder][2]. 

You can install the `descriptor-adapter` manually:

```bash
go install github.com/ligato/vpp-agent/plugins/kvscheduler/descriptor-adapter
```

Alternatively, you can put the following target into your project makefile. This assumes you have a dependency on the VPP agent in your vendor directory: 

```
get-generators:
    @go install ./vendor/github.com/ligato/vpp-agent/plugins/kvscheduler/descriptor-adapter
```

---

Build the binary file from the `.go` files present inside the model folder. Then use the binary file to generate the adapters for the `Interface` and `Route` proto messages:
 
```
descriptor-adapter --descriptor-name Interface --value-type *model.Interface --import "github.com/ligato/vpp-agent/examples/tutorials/05_kv-scheduler/model" --output-dir "descriptor"
descriptor-adapter --descriptor-name Route --value-type *model.Route --import "github.com/ligato/vpp-agent/examples/tutorials/05_kv-scheduler/model" --output-dir "descriptor"
```

Include the commands, shown in the code block above, in the plugin's `main.go` file with the `//go:generate` directives. 
The `descriptor-adapter` generator will put the generated adapters into the `<plugin>/descriptor/adapter` folder.

---

#### Descriptor without dependency

The next step is to define descriptors. Let's begin by working with the interface descriptor that has no dependencies.

A descriptor can be implemented in one of two ways:

- Define the descriptor constructor that implements all required methods. This works well when the implementation uses a relatively small number of short descriptor methods.
- Define a descriptor object that implements all required methods on the object. This is the preferred technique for placing method references in the descriptor constructor.

In the interface descriptor, use the first approach. Create a new file called `descriptors.go` so that the descriptor code is outside of `main.go`.

Add the following code:

```go
func NewIfDescriptor(logger logging.PluginLogger) *api.KVDescriptor {
	typedDescriptor := &adapter.InterfaceDescriptor{
		// descriptor implementation
	}
	return adapter.NewInterfaceDescriptor(typedDescriptor)
}
```


`NewIfDescriptor` is a constructor function that returns a type-safe descriptor object. All potential descriptor 
dependencies, such as logger for example, are provided using constructor parameters.  

Examine `adapter.InterfaceDescriptor` in the `descriptors.go` file, and you will see several defined fields. The most important of these fields are function-types with CRUD definitions, and the fields resolving dependencies. 

For a complete list of all descriptor fields, see the [KV Descriptor API definition][3].

---

Next, let's implement the APIs.

Start with a `Name` that must be unique amongst all descriptors:
```go
    Name: "if-descriptor",
```

---

Define the NB key prefix for the configuration type handled by the descriptor:
```go
NBKeyPrefix: "/interface/",
```

---

Set the string representation of the type:
```go
ValueTypeName: proto.MessageName(&model.Interface{}),
```

---

Add the configuration item identifier, consisting of label, name, and index. This method returns the configuration item identifier. 
```go
KeyLabel: func(key string) string {
    return strings.TrimPrefix(key, "/interface/")
},
```

---
 
Key selector returns `true` if the descriptor describes the provided key. A descriptor can support a subset of keys, but it can only process one value type:
```go
KeySelector: func(key string) bool {
    if strings.HasPrefix(key, ifPrefix) {
        return true
    }
    return false
},
```

---

Enable metadata for the given type:
```go
WithMetadata: true
```

---

Add the `Create` method that configures a new interface configuration item:
```go
Create: func(key string, value *model.Interface) (metadata interface{}, err error) {
    d.log.Infof("Interface %s created", value.Name)
    return value.Name, nil
},
```

---

Here is the complete interface descriptor:
```go
func NewIfDescriptor(logger logging.PluginLogger) *api.KVDescriptor {
	typedDescriptor := &adapter.InterfaceDescriptor{
		Name: ifDescriptorName,
		NBKeyPrefix: ifPrefix,
		ValueTypeName: proto.MessageName(&model.Interface{}),
		KeyLabel: func(key string) string {
			return strings.TrimPrefix(key, ifPrefix)
		},
		KeySelector: func(key string) bool {
			if strings.HasPrefix(key, ifPrefix) {
				return true
			}
			return false
		},
		WithMetadata: true,
		Create: func(key string, value *model.Interface) (metadata interface{}, err error) {
			logger.Infof("Interface %s created", value.Name)
			return value.Name, nil
		},
	}
	return adapter.NewInterfaceDescriptor(typedDescriptor)
}
```

---

#### Descriptor with dependency

Let's continue with the route descriptor that has a dependency on an interface. This descriptor includes additional
fields since you will specify the dependency on the interface configuration item. You will also define the descriptor struct, and implement methods outside of the descriptor constructor.

---

Define the struct and constructor:

```go
type RouteDescriptor struct {
	// dependencies
	log logging.PluginLogger
}

func NewRouteDescriptor(logger logging.PluginLogger) *api.KVDescriptor {
	typedDescriptor := &adapter.RouteDescriptor{
		// descriptor implementation
	}
	return adapter.NewRouteDescriptor(typedDescriptor)
}
```

---

In this case, the descriptor fields are methods of the `RouteDescriptor` using their respective function signatures:
```go
func (d *RouteDescriptor) KeyLabel(key string) string {
	return strings.TrimPrefix(key, routePrefix)
}

func (d *RouteDescriptor) KeySelector(key string) bool {
	if strings.HasPrefix(key, routePrefix) {
		return true
	}
	return false
}

func (d *RouteDescriptor) Dependencies(key string, value *model.Route) []api.Dependency {
	return []api.Dependency{
		{
			Label: routeInterfaceDepLabel,
			Key:   ifPrefix + value.InterfaceName,
		},
	}
}
```

---

There is no requirement for the `WithMetadata` field. The `Create` method will not return any metadata:

```go
func (d *RouteDescriptor) Create(key string, value *model.Route) (metadata interface{}, err error) {
	d.log.Infof("Created route %s dependent on interface %s", value.Name, value.InterfaceName)
	return nil, nil
}
``` 

---

In addition, there are two new fields:

* Dependencies list that contains a key prefix and unique label value. The dependencies list requires a key prefix and label for each configuration item. The configuration item will not be created if the dependency key does not exist. The label is informative and should be unique:
```go
func (d *RouteDescriptor) Dependencies(key string, value *model.Route) []api.Dependency {
	return []api.Dependency{
		{
			Label: routeInterfaceDepLabel,
			Key:   ifPrefix + value.InterfaceName,
		},
	}
}
```

* Descriptors list where dependent values are processed. 

---

Return the interface descriptor since this is the one handling interfaces.
```go
RetrieveDependencies: []string{ifDescriptorName},
```

---

Define the descriptor context of type `RouteDescriptor` within `NewRouteDescriptor`:
```go
func NewRouteDescriptor(logger logging.PluginLogger) *api.KVDescriptor {
	descriptorCtx := &RouteDescriptor{
		log: logger,
	}
	typedDescriptor := &adapter.RouteDescriptor{
		// descriptor implementation
	}
	return adapter.NewRouteDescriptor(typedDescriptor)
}
```

---

Set non-function fields:
```go
func NewRouteDescriptor(logger logging.PluginLogger) *api.KVDescriptor {
	descriptorCtx := &RouteDescriptor{
		log: logger,
	}
	typedDescriptor := &adapter.RouteDescriptor{
        Name: routeDescriptorName,
        NBKeyPrefix: routePrefix,
        ValueTypeName: proto.MessageName(&model.Route{}),      
        RetrieveDependencies: []string{ifDescriptorName},
	}
	return adapter.NewRouteDescriptor(typedDescriptor)
}
```

---

Set function fields as references to the `RouteDescriptor` methods. Here is the complete route descriptor:
```go
func NewRouteDescriptor(logger logging.PluginLogger) *api.KVDescriptor {
	descriptorCtx := &RouteDescriptor{
		log: logger,
	}
	typedDescriptor := &adapter.RouteDescriptor{
		Name: routeDescriptorName,
		NBKeyPrefix: routePrefix,
		ValueTypeName: proto.MessageName(&model.Route{}),
		KeyLabel: descriptorCtx.KeyLabel,
		KeySelector: descriptorCtx.KeySelector,
		Dependencies: descriptorCtx.Dependencies,
		Create: descriptorCtx.Create,
	}
	return adapter.NewRouteDescriptor(typedDescriptor)
}
```

The descriptor API provides additional methods such as `Update()`, `Delete()`, `Retrieve()` and `Validate()`. 

For more information about the descriptor API, see the [KV Descriptor API definition][3].

#### Wire your plugin into the KV Scheduler

Let's start by registering the completed descriptors in the `main.go` file. The first step is to add the `KV Scheduler` to your HelloWorld plugin as a plugin dependency:
```go
type HelloWorld struct {
	infra.PluginDeps
	KVScheduler api.KVScheduler
}
```

Second, register the descriptors with the KV Scheduler in the HelloWorld plugin `Init()`:
```go
func (p *HelloWorld) Init() error {
	p.Log.Println("Hello World!")

	err := p.KVScheduler.RegisterKVDescriptor(adapter.NewInterfaceDescriptor(NewIfDescriptor(p.Log).GetDescriptor()))
	if err != nil {
		// handle error
	}

	err = p.KVScheduler.RegisterKVDescriptor(adapter.NewRouteDescriptor(NewRouteDescriptor(p.Log).GetDescriptor()))
	if err != nil {
		// handle error
	}

	return nil
}
```

The last step is to replace the plugin initialization method with `AllPlugins()` in `main()`. This will ensure that the 
KV Scheduler loads and initializes from your HelloWorld plugin.
```go
a := agent.NewAgent(agent.AllPlugins(p))
```

Starting the agent will load the KV Scheduler plugin together with your HelloWorld plugin. The KV Scheduler will
receive all northbound data and pass it to the HelloWorld descriptor in the correct order. If dependencies for a 
configuration item aren't met, the item will be cached. 

An example is programming a route before resolving the interface dependency. The KV Scheduler will cache the route if it has not resolved the interface dependency. 

#### Run KV Scheduler tutorial code

The tutorial code contains `main.go`, `descriptors.go`, a model, and the generated adapters. The code includes the `AfterInit()` method. This method starts a new Go routine with a testing procedure.

The tutorial code executes three test cases. All can be built and started without any conf files. The KV Scheduler `StartNBTransaction()` method simulates NB transactions.

Steps to run the tutorial code:

1. Open a terminal session.
<br>
<br>
2. Change to the kv scheduler tutorial folder:
```
vpp-agent git:(master) ✗ cd examples/tutorials/05_kv-scheduler
```
3. Run code
```
go run main.go descriptors.go
```

!!! Note
    By the running the code, you will print the transaction log for all three cases. The discussion below includes a subset of the transaction log pertaining to the specific test case.
    
     
**1. Configure the interface and the route in a single transaction**

Transaction log output:
```bash
1. CREATE:
  - key: /interface/if1
  - value: { name:"if1"  } 
2. CREATE:
  - key: /route/route1
  - value: { name:"route1" interface_name:"if1"  } 
``` 

As expected, the interface creation is first; route creation is second. This follows the configuration order performed in this test case.

---

**2. Configure the route first, and the interface second, in a single transaction.** 
This reverses the configuration order performed in the first test case.

Transaction log output:
```bash
1. CREATE:
  - key: /interface/if2
  - value: { name:"if2"  } 
2. CREATE:
  - key: /route/route2
  - value: { name:"route2" interface_name:"if2"  } 
```

The `Create` sequence is exactly the same. This is despite the fact that the code reversed the configuration order. The KV Scheduler re-ordered the configuration items in the correct sequence before executing the transaction.


---

**3. Configure the route and interface in separate transactions**

In this case, you have two outputs since there are two transactions. 

The route comes first, but it is cached. The dependent interface does not exist, and the KV Scheduler does not know when it will appear. The route is marked as `[NOOP IS-PENDING]`:

Transaction log output:
```bash
1. CREATE [NOOP IS-PENDING]:
  - key: /route/route3
  - value: { name:"route3" interface_name:"if3"  } 
```

---

The second transaction introduces the expected interface. The KV Scheduler:
 
- recognizes the interface as a dependency for the cached route.
- sorts the items into the correct order.
- calls the appropriate configuration method. 

Transaction log output:
```bash
1. CREATE:
  - key: /interface/if3
  - value: { name:"if3"  } 
2. CREATE [WAS-PENDING]:
  - key: /route/route3
  - value: { name:"route3" interface_name:"if3"  } 
```

The KV Scheduler marked the cached route as `[WAS-PENDING]`. This indicates the item had been cached previously.

 [1]: https://github.com/ligato/vpp-agent/blob/master/examples/tutorials/05_kv-scheduler/model/model.proto
 [2]: https://github.com/ligato/vpp-agent/tree/master/plugins/kvscheduler/descriptor-adapter
 [3]: ../developer-guide/kvdescriptor.md#descriptor-api
 [4]: https://github.com/ligato/vpp-agent/tree/master/examples/tutorials/05_kv-scheduler
 [5]: ../plugins/kvs-plugin.md
 [code-link]: https://github.com/ligato/vpp-agent/tree/master/examples/tutorials/05_kv-scheduler
 
 
 
