# Plugin Lookup

---

This section describes the plugin Lookup function. 

---

Package references: [cninfra](https://godoc.org/github.com/ligato/cn-infra), [agent](https://godoc.org/github.com/ligato/cn-infra/agent) 

!!! Note
    In this section, `agent` defines a set of plugins, that start and initialize in the correct order according to their relationship to one another.

---

Ligato-supplied and custom plugins define the functions supported by your agent. In many cases, one plugin depends on another. For example, interface plugin initialization should precede route plugin initialization. The same route plugin may have dependencies on other plugins, that have their own dependencies on other plugins. All plugins and dependencies must start and initialize in the correct order. 

The plugin lookup function simplifies plugin lifecycle management by automating the following functions:

- Adds plugins and associated dependencies to the agent's plugin list.
<br></br>
- Sorts plugins and dependencies in the correct initialization order.         
 
--- 

### Quick Agent Setup

If you wish to skip the details, and quickly set up an agent, follow the steps below.  

1. Define your plugin. Every plugin must implement the [infra.Plugin][infra-plugin] interface.
</br>
</br>
2. Use the `agent.Plugins(<plugin>...)` function with the plugin you created in the previous step. This function creates an instance of `agent.Option`, and informs the VPP agent about your plugin. 

You can pass multiple plugins and dependencies to this function. You must manually list all plugins and dependencies in the correct order of initialization.  

!!! Note
    Alternatively, you can use the `agent.AllPlugins(<plugin>...)` function to avoid manually listing all plugins, and their dependencies, in a specific order of initialization. This function invokes plugin lookup that *automatically* locates dependency plugins, and sorts all plugins and dependencies in the correct initialization order. 

3. Use the `agent.NewAgent(opts ...Option)` function to create a new instance of the agent.
</br>
</br>
 
4. Use the `Run()` method (blocking), or the `Start()` method (non-blocking) to initiate the agent created in Step 3.
</br>
</br>

5. Stop the agent with the `Stop()` method. Alternatively, define a struct-type channel and add it to the agent using the option `agent.QuitOnClose(<channel>)`. Closing the channel stops the agent.

Code block shows the `func main()` for a plugin called `plugin`. 
```go
func main() {
	plugin := myplugin.NewPlugin()
	
	a := agent.NewAgent(
		agent.Plugins(plugin),
	)
	if err := a.Run(); err != nil {
		log.Fatal(err)
	}
}
```

---

### Definitions

**Agent** 

The agent object defines a set of plugins, that start and initialize in the correct order according to their relationship between one another. The agent object implements the [agent interface](https://github.com/ligato/cn-infra/blob/master/agent/agent.go). The `agent.NewAgent(<options>...)` function creates an agent. 

Every instance of the agent contains configuration data, a channel to stop the agent, and a tracer time-measurement utility. You provide options to configure the agent, and every agent instance possesses a mechanism to prevent it from starting/stopping multiple times.

---

**Options**

You configure and customize an agent with options. The *plugin list* option contains the plugins you control as part of your agent. If your plugins and their dependencies do not require a specific initialization order, use the `agent.Plugins(<plugin>....)` function to create the plugin list option. 

If your plugins require a specific initialization order because of dependencies to resolve, use the `agent.Allplugins(<plugin>)` function to create the plugin list option.

You can customize your agent with the following [options](https://godoc.org/github.com/ligato/cn-infra/agent#Option):

* `Plugins(...)` adds one or more plugins in the order of initialization.
<br></br>
* `AllPlugins(...)` adds one or more plugins and sorts them in the correct initialization order. This option recursively adds all plugins listed as dependencies by other plugins.
<br></br>
* `Version(<version>, <date>, <id>)` sets the program version.
<br></br>
* `QuitOnClose(<channel>)` sets the channel to terminate the running agent when it closes.
<br></br>
* `QuitSignals(<signals>)` sets the OS signals to quit the running agent. SIGINT and SIGTERM serve as defaults.
<br></br>
* `StartTimeout(duration)` sets the timeout duration for agent start. The default is 15s.
<br></br>
* `StopTimeout(duration)` sets the timeout duration for agent. The default is 5s.

  
---

**Plugin**

The plugin object implements the [plugin interface](https://github.com/ligato/cn-infra/blob/master/infra/infra.go), that defines methods required for plugin lifecycle management. The plugin list option contains the list of plugins to read at agent startup.

At this point, manual or plugin lookup has performed plugin initialization sorting. The plugin interface defines agent initialization the startup procedure consisting of two steps:

* Calls the `Init()` method for every plugin, one-by-one in a single thread, in their initialization order.
<br></br>
* Calls the `AfterInit()` method for specific plugins, one-by-one, in a single thread, in their initialization order.

This two-step procedure ensures the execution of certain initialization tasks only occur following the pre-initialization of your agent's plugins. Note that you may leave the `AfterInit()` method empty if you do not need the second initialization phase.

To look over code implementing this function, see the [Hello World](../tutorials/01_hello-world.md) tutorial.

---

### Plugin Lifecycle

You initialize a plugin using the `Init()`method. This allocates all resources required by your plugin.
For example, this method can generate channels or maps, or initialize specific fields used by your plugin.

Your agent plugins likely include dependencies. This situation arises when the initialization of one plugin depends on the successful initialization of another *dependency* plugin. Therefore, the dependency plugin must initialize _before_ the plugin that depends on it.    

Code block includes two plugins, **P1** and **P2**. **P1** depends on **P2**.
```go
// First plugin
type P1 struct {
	Deps    
}

// P1 dependent plugins
type Deps struct {
	P2
}

// Second plugin
type P2 struct {}
```

To execute the proper startup sequence, **P2** initializes first. Otherwise, **P1** starts with an empty reference, that results in unexpected behavior during agent startup. You might see`panics` using a nil pointer as one indication of unexpected behavior. 

Every plugin you define must satisfy two methods:

- `Close()` releases all plugin resources, such as close channels, or close connections.
<br></br>
- `String()` returns a text representation of the plugin.

!!! Tip
    Keep your code clean by moving all plugin dependencies into a separate structure usually called `Dep` or `Deps`. 


---

The responsibility for the correct initialization order depends on the function used to create the *plugin list* option. You have two choices: `agent.Plugins()` and `agent.AllPlugins()`.

---

**agent.Plugins()**

If you choose the `agent.Plugins()` function for plugin initialization ordering, note the following: 

* Parameter field lists the plugins and in their initialization order.
<br></br>  
* Parameter field must list all plugins, including any dependencies.
<br></br> 
* Unlisted plugins and dependencies will _NOT_ initialize.  

In general, use the `agent.Plugins()` function if your agent contains one plugin, or you have multiple plugins with no dependencies.

If your agent includes plugin dependencies, then you must define the correct plugin initialization order in the parameters field of the `agent.Plugins()` function. 

---

**agent.AllPlugins()**

If you choose the `agent.AllPlugins(<plugin>...)` function for plugin initialization ordering, note the following:

* Automatically resolves the plugin dependency "tree" using [plugin lookup](plugin-lookup.md#plugin-lookup-procedure).
<br></br> 
* You do not need to list all plugin dependencies. 

Use the `agent.AllPlugins()` function if your agent includes multiple plugins with dependencies. Plugin lookup automatically locates dependency plugins, figures out dependency relationships, and sorts them in the correct initialization order. 

Let's look at an example:

```go
// Top level plugin
type P1 struct {
	Deps
}

type Deps struct {
	P2
	P3
}

// Dependency plugins
type P2 struct {}
type P3 struct {}
```

It shows that **P1** depends on **P2** _and_ **P3**. Therefore, **P2** and **P3** must initialize before **P1**. However, no dependency exists between **P2** and **P3**, so they can initialize in any order.  The example yields two correct answers for the initialization sequence: **[P2, P3, P1]** or **[P3, P2, P1]**. 

If you implement the `agent.Plugins()` function in your agent, you must list all plugins in one of the correct initialization sequences as noted.

If you implement the `agent.AllPlugins()` function in your agent, you only need to list **P1**. Plugin lookup locates the **P2** and **P3** dependencies, and then sequences all plugins and dependencies in the correct initialization order. 

!!! Note
    The `agent.AllPlugins()` method invokes plugin lookup.

---

### Plugin Options

Dependency management can introduce complexity into your agent implementation. To simplify this task, Ligato cn-infra includes an `option.go` helper file for every plugin. This file describes the standard naming convention used by cn-infra. However, you can use any name of your choosing. 

Let's stay on the topic of dependencies and start with a template:

```go
// DefaultPlugin is helper global variable - default plugin instance
var DefaultPlugin = *NewPlugin()

// NewPlugin returns an instance of plugin with default dependencies
func NewPlugin(opts ...Option) *Plugin {
	// Prepare empty plugin instance
	p := &Plugin{}

    // Define plugin names and default dependency instances
	p.PluginName = "myplugin"
	p.Dep1 = &dep1.DefaultPlugin
	p.Dep2 = &dep2.DefaultPlugin

    // Apply options (see below)
	for _, o := range opts {
		o(p)
	}

	return p
}

// Option is a function that can be used to customize the Plugin.
type Option func(*Plugin)

// UseDeps returns Option that can inject custom dependencies to the plugin.
func UseDeps(cb func(*Deps)) Option {
	return func(p *Plugin) {
		cb(&p.Deps)
	}
}

// Replace instance of the Dep2
func UseDep2(d2 dep2.SomePlugin) Option {
	return func(p *Plugin) {
		dep2.Dep2 = d2
	}
}

// Set instance of the Dep3
func UseDep3(d3 dep3.SomePlugin) Option {
	return func(p *Plugin) {
		dep3.Dep3 = d3
	}
}
```

Almost all cn-infra plugins use the `DefaultPlugin` global variable. It constructs a new instance of the plugin with pre-defined dependency fields. 

Cn-infra defines `NewPlugin()` as a base constructor. This constructor returns a default instance of the plugin if you don't include any options. Note that some dependencies in the example above rely on the default instances of other plugins. Use this technique only if you need to customize plugin dependencies.

The example uses three options:

* `UseDeps(cb func(*Deps))` replaces the whole `Dep` structure with all fields. If you require customization of some or all of your dependencies, use this function.
<br></br>  
* `UseDep2(d2 dep2.SomePlugin)` replaces a specific dependency with a desired value.
<br></br>
* `UseDep3(d3 dep3.SomePlugin)` sets a dependency, which by default, is an empty value. Use this function for non-optional dependencies.  

This scheme lets you define multiple instances of the same plugin with different dependencies. For example, you can instantiate the kvdbsync plugin multiple times with different key-value plugins, one for etcd, and one for Redis.

!!! Tip
    Keep your code clean by ensuring the default plugin instance can call the `init()` method without any customization.

---

### Plugin Lookup Procedure

Plugin lookup handles the ordering process in scenarios where multiple plugins have complicated dependencies. It builds a tree-like structure to determine which plugins to load first. You only need to define a top-level plugin. Plugin lookup locates dependent plugins for you, even in a multi-layer scenario.

For example, if **P1** depends on **P2**, which depends on **P3**, then you only need to define **P1**. Plugin lookup locates the other plugins. 

!!! Tip
    Keep your code clean by defining a top-level plugin. Plugin lookup will locate all other plugins and dependencies required for the application.



---

### Cross Dependencies

What happens if you have two plugins, both dependent on the other?  Plugin lookup can handle this scenario. 

Let's take an example. Start with the following two files: `P1.go` and `P2.go`.

```go
// P1.go

type P1 struct {
	Deps 
	indexes       P1Indexes
	remoteIndexes P2Indexes
} 

type Deps struct {
	P2
}

func (p *P1) Init() error {
	p.indexes = NewIndexes()
	p.remoteIndexes = P2.GetIndexes()
	return nil
}

func (p *P1) Close() error {
	return nil
}

func (p *P1) String() string {
	return ""
}

func (p *P1) GetIndexes() P1Indexes {
	return p.indexes
}
```

`P1.go` defines a simple plugin with `Init()`, `Close()` and `String()`. It contains two "indexes" fields: One for **P1** that you initialize with the `Init()` method, and one obtained from **P2**. This scenario expects **P2** will initialize first:

```go
// P2.go

type P2 struct {
	Deps 
	indexes       P2Indexes
	remoteIndexes P1Indexes
} 

type Deps struct {
	P1
}

func (p *P2) Init() error {
	p.indexes = NewIndexes()
	p.remoteIndexes = P1.GetIndexes()
	return nil
}

func (p *P2) Close() error {
	return nil
}

func (p *P2) String() string {
	return ""
}

func (p *P2) GetIndexes() P1Indexes {
	return p.indexes
}
```

The `P2.go` file looks the same, but uses **P1** as a dependency. This suggests that an agent consisting of **P1** and **P2** cannot initialize correctly. The agent would take **P1**, and found **P2** as a dependent plugin.

Inside **P2**, you can see **P1** defined as a dependency. The initialization order becomes **[P2, P1]** since the agent knows about **P1**. However, the agent panics on `P1.GetIndexes()`, since **P1** is nil. Switch the order, and you will observe the same result: panic on `P2.GetIndexes()`.

To address this problem, you can use the `AfterInit()` method. This method lets you call functions for plugins after the `Init()` methods succeed for all plugins. 

Code block with `AfterInit()`:
```go
func (p *P1) Init() error {
	p.indexes = NewIndexes()
	return nil
}

func (p P1*) AfterInit() error {
	p.remoteIndexes = P2.GetIndexes()
}
``` 

You can apply this code to both plugins. The solution works for two reasons:

- During the `Init()` phase, both plugins initialize their own indexes.
<br></br>
- In the `AfterInit()` phase, both plugins obtain the respective references. The previous `Init()` phase made both available. 

---

### Building an Agent

The following example uses several [VPP agent plugins](../plugins/vpp-plugins.md) to build an agent. 

1. Create the `MyApp` top-level plugin, that contains all plugins required for the application. Define a method called `NewApp()`that returns an instance of the `MyApp` plugin. Include all methods required to implement the plugin interface:

```go
type MyApp struct {}

func NewApp() *MyApp {
	return &MyApp{}
}

func (p *MyApp) Init() error {
    return nil	
}

func (p *MyApp) Close() error {
	return nil
}

func (p *MyApp) String() string {
	return "MyApp"
}
``` 

---

2. Add a user-defined plugin called `MyPlugin`:

```go
type MyPlugin struct {
	Publish datasync.KeyProtoValWriter
    Watcher datasync.KeyValProtoWatcher
    GoVPP   govppmux.API
    GRPC    *rpc.Plugin
}

...

type MyApp struct {
	MyPlugin mp.MyPlugin
}

func NewApp() *MyApp {
	myPlugin := mp.NewPlugin()
	
	return &MyApp{
		MyPlugin    myplugin,
	}
} 
```

---

3. Add the connection to the etcd database. Datasync is usually wrapped as `KVProtoWatcher` or `KVProtoWriter`, and injected into the plugin:

```go
type MyApp struct {
	MyPlugin        mp.MyPlugin
	ETCDDataSync    *kvdbsync.Plugin
}

func NewApp() *MyApp {
	etcdDataSync := kvdbsync.NewPlugin(kvdbsync.UseKV(&etcd.DefaultPlugin))
	
	watchers := datasync.KVProtoWatchers{ etcdDataSync }
    writers := datasync.KVProtoWriters{ etcdDataSync }
	
	myPlugin := mp.NewPlugin(mp.UseDeps(func(deps *mp.Deps) {
		deps.Publish = writers
        deps.Watcher = watchers
	})
	
	return &MyApp{
		MyPlugin    myplugin,
	}
} 
```

---

4. Add the connection to the VPP GoVPPMux plugin:

```go
type MyApp struct {
	MyPlugin        mp.MyPlugin
}

func NewApp() *MyApp {
	etcdDataSync := kvdbsync.NewPlugin(kvdbsync.UseKV(&etcd.DefaultPlugin))
	
	watchers := datasync.KVProtoWatchers{ etcdDataSync }
    writers := datasync.KVProtoWriters{ etcdDataSync }
    
	myPlugin := mp.NewPlugin(mp.UseDeps(func(deps *mp.Deps) {
		deps.Publish = writers
        deps.Watcher = watchers
        deps.GoVPP = &govppmux.DefaultPlugin
	})
	
	return &MyApp{
		MyPlugin    myplugin,
	}
} 
```

---
   
5. Add support for gRPC:

```go
type MyApp struct {
	MyPlugin        mp.MyPlugin
}

func NewApp() *MyApp {
	etcdDataSync := kvdbsync.NewPlugin(kvdbsync.UseKV(&etcd.DefaultPlugin))
	
	watchers := datasync.KVProtoWatchers{ etcdDataSync }
    writers := datasync.KVProtoWriters{ etcdDataSync }
    
	myPlugin := mp.NewPlugin(mp.UseDeps(func(deps *mp.Deps) {
		deps.Publish = writers
        deps.Watcher = watchers
        deps.GoVPP = &govppmux.DefaultPlugin
        deps.GRPC = &grpc.DefaultPlugin
	})
	
	return &MyApp{
		MyPlugin    myplugin,
	}
} 
```

---

6. Optionally, you can add other plugins such as logger or REST:

```go
type MyApp struct {
	MyPlugin        *mp.MyPlugin
	LogManager      *logmanager.Plugin
	RESTAPI         *rest.Plugin
}

func NewApp() *MyApp {
	etcdDataSync := kvdbsync.NewPlugin(kvdbsync.UseKV(&etcd.DefaultPlugin))
	
	watchers := datasync.KVProtoWatchers{ etcdDataSync }
    writers := datasync.KVProtoWriters{ etcdDataSync }
    
	myPlugin := mp.NewPlugin(mp.UseDeps(func(deps *mp.Deps) {
		deps.Publish = writers
        deps.Watcher = watchers
        deps.GoVPP = &govppmux.DefaultPlugin
        deps.GRPC = &grpc.DefaultPlugin
	})
	
	return &MyApp{
		MyPlugin    myplugin,
		LogManager: &logmanager.DefaultPlugin,
		RESTAPI:    &rest.DefaultPlugin
	}
} 
```

---

7. If you need REST to work with `MyPlugin`, pass an instance of the REST plugin:

```go
func NewApp() *MyApp {
	etcdDataSync := kvdbsync.NewPlugin(kvdbsync.UseKV(&etcd.DefaultPlugin))
	
	watchers := datasync.KVProtoWatchers{ etcdDataSync }
    writers := datasync.KVProtoWriters{ etcdDataSync }
    
	myPlugin := mp.NewPlugin(mp.UseDeps(func(deps *mp.Deps) {
		deps.Publish = writers
        deps.Watcher = watchers
        deps.GoVPP = &govppmux.DefaultPlugin
        deps.GRPC = &grpc.DefaultPlugin
	})
	
	restPlugin := rest.NewPlugin(rest.UseDeps(func(deps *rest.Deps) {
        deps.MyPlugin = myPlugin
        
    }))
	
	return &MyApp{
		MyPlugin    myplugin,
		LogManager: &logmanager.DefaultPlugin,
		RESTAPI:    restPlugin
	}
} 
```

You only need to include the `MyApp` top-level plugin in the `agent.AllPlugins()` function. Again, plugin lookup automatically locates dependency plugins, figures out dependency relationships, and sorts them in the correct initialization order. 

[hw-tutorial]: ../tutorials/01_hello-world.md

*[REST]: Representational State Transfer
*[VPP]: Vector Packet Processing
[infra-plugin]: https://github.com/ligato/cn-infra/blob/master/infra/infra.go