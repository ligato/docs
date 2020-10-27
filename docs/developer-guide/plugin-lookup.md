# Plugin Lookup

---

This section describes the Plugin Lookup function. 

!!! Note
    In this section, `agent` defines a set of plugins, that start and initialize in the correct order according to their relationship between one another.

---

Ligato-supplied and custom plugins define the functions supported by your agent. In many cases, one plugin will depend on another. For example, interface plugin initialization should precede route plugin initialization. The same route plugin may have dependencies on other plugins, which in turn have their own dependencies on other plugins. All plugins and dependencies must start and initialize in the correct order. 

The plugin lookup function simplifies the process of plugin lifecycle management by supporting the following:

- Add plugins and associated dependencies to the agent's plugin list.
- Sort plugins and dependenices into the correct initialization order.         
  

### Quick Agent Setup

If you wish to skip the details, and set up an agent, follow the steps below.  

1. Define your plugin. Every plugin must implement the [infra.Plugin][infra-plugin] interface.
</br>
</br>
2. Use the `agent.Plugins(<plugin>...)` with the plugin created in the previous step. This function creates an instance of `agent.Option`, and informs the VPP agent about your plugin. You can pass multiple plugins and dependencies to this function. You must manually list all plugins and dependencies in the correct order of initialization.  

!!! Note
    Alternatively, you can use the `agent.AllPlugins(<plugin>...)` function if you wish to avoid manually listing all plugins, and their dependencies, in a specific order of initialization. The `agent.AllPlugins(<plugin>...)` function will invoke plugin lookup which *automatically* sorts plugins and dependencies into the correct initialization order. 

3. Use the agent.NewAgent(opts ...Option) function to create a new instance of the agent.
</br>
</br>
 
4. Use the `Run()` method (blocking), or the `Start()` method (non-blocking) to initiate the agent created in Step 3.
</br>
</br>

5. Stop the agent with the `Stop()` method. Alternatively, define a struct-type channel and add it to the agent using the option `agent.QuitOnClose(<channel>)`. Closing the channel stops the agent.

This code block shows the func main() for the plugin called `plugin`. 
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

The `agent` object defines a set of plugins, that start and initialize in the correct order according to their relationship between one another. The agent object implements the [agent interface](https://github.com/ligato/cn-infra/blob/master/agent/agent.go), and the `agent.NewAgent(<options>...)` function creates an agent. 

Every instance of the agent contains configuration data, a channel which can gracefully stop the agent, and a tracer time-measurement utility. You provide options to configure the agent, and every agent instance possesses a mechanism to prevent it from starting/stopping multiple times.

---

**Options**

You can configure and customize an agent with options. The *plugin list* option contains the plugins that you control as part of your agent. If a specific initialization order of your plugins, and their dependencies, isn't required, use the `agent.Plugins(<plugin>....)` function to create the plugin list option.  If your plugins require a specific initialization order because of dependencies to resolve, use the `agent.Allplugins(<plugin>)` function to create the plugin list option.

The available [options](https://godoc.org/github.com/ligato/cn-infra/agent#Option) to customize your agent consist of the following:

* `Plugins(...)` adds one or more plugins in the order of initialization.
* `AllPlugins(...)` adds one or more plugins and sorts them into the correct startup order. This option will automatically add all plugins listed as dependencies in other plugins.
* `Version(<version>, <date>, <id>)` sets the version of the program.
* `QuitOnClose(<channel>)` sets the channel to terminate the running agent when it is closed.
* `QuitSignals(<signals>)` sets the OS signals to quit the running agent. SIGINT and SIGTERM serve as defaults.
* `StartTimeout(duration)/StopTimeout(duration)` sets the start/stop timeout (defaults: 15s/5s)
  
---

**Plugin**

The plugin object implements the [plugin interface](https://github.com/ligato/cn-infra/blob/master/infra/infra.go), which defines methods required for plugin lifecycle management. The plugin list option contains the list of plugins to read at agent startup.

At this point, manual or plugin lookup have performed plugin initialization sorting. The plugin interface defines agent initialization consisting of two steps:

* Startup procedure calls the `Init()` method for every plugin, one-by-one in a single thread, in their initialization order.
* Startup procedure calls the `AfterInit()` method for each plugin one-by-one, in a single thread, in their initialization order.

This two-step procedure ensures the execution of certain initialization tasks only occur following the pre-initialization of your agent's plugins. Note that you may leave the `AfterInit()` method empty if you do not need the second initialization phase.

---

### Plugin Lifecycle

You initialize a plugin using the `Init()`method. This allocates all resources needed by your plugin.
For example, this method can generate channels or maps, or initialize other fields used by your plugin.

The plugins in your agent likely include dependencies. This occurs when the initialization of one plugin depends on the successful initialization of another *dependency* plugin. Therefore, the dependency plugin must initialize before the plugin that depends on it.    

Consider the following example:
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

The code block above includes two plugins, P1 and P2. P1 uses P2 as a dependency. Keep your code clean by moving all plugin-type dependencies into a separate structure usually called `Deps`. To execute the startup sequence properly, P2 initializes first. Otherwise, P1 may work with an empty reference, which will result in unexpected behavior during agent startup. You might see`panics` using a nil pointer as one indication of unexpected behavior. 

Every plugin you define must satisfy two methods:

- `Close()` which releases all plugin resources, such as close channels, or close connections.
- `String()` which should return a text representation of the plugin.

---

The responsibility for the correct initialization order depends on the function used to create the *plugin list* option. You have two choices: `agent.Plugins()` and `agent.AllPlugins()`.

Items to note if you choose the `agent.Plugins()` for plugin initialization ordering: 

* Parameter field lists the plugins and start order.  
* Parameter field must list all plugins, including any dependencies. 
* Plugins and dependencies `NOT` listed will `NOT` initialize. 

In general, use the `agent.Plugins()` function if your agent contains one plugin, or you have multiple plugins with no dependencies. However, if your agent includes plugin dependencies, then you must define the correct plugin initialization sequence in the parameters field of the `agent.Plugins()` function. 

`agent.AllPlugins(<plugin>...)` as the plugin list option:

* Automatically resolves the plugin dependency tree itself using [plugin lookup](plugin-lookup.md#plugin-lookup-procedure). 
* you do not need to list all plugin dependencies. 

You should use the `agent.AllPlugins()` function if your agent includes multiple plugins with dependencies. Plugin lookup automatically figures out dependency relationships and sorts them into the correct  initialization sequence. 

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

The example above shows that **P1** depends on **P2** and **P3**. Therefore, **P2** and **P3** must initialize before **P1**. However, no dependency exists between **P2** and **P3**, so they can initialize in any order.  The example yields two correct answers for the initialization sequence: **[P2, P3, P1]** or **[P3, P2, P1]**. 

If you use the `agent.Plugins()` function in your agent, you must list all plugins in one of the correct initialization sequences as noted.

If you use the `agent.AllPlugins()` function in your agent, you only need to list **P1**. Plugin lookup locates the **P2** and **P3** dependencies, and then sequences all plugins in the correct initialization order. 

### Plugin Options

Dependency management can introduce complexity into your agent development efforts. To simplify this task, Ligato cn-infra includes an `option.go` helper file for every plugin. `option.go` represents the standard naming convention used by cn-infra. However, you can use any name of your choosing. 

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

Almost all cn-infra plugins use the `DefaultPlugin` global variable. It constructs a new instance of the plugin with pre-defined dependency fields. Keep your code clean by ensuring the default plugin instance can call the `init()` method without any customization.

Cn-infra defines `NewPlugin()` as a base constructor. This constructor returns a default instance of the plugin if you use without any options. Note that some dependencies in the example above rely on the default instances of other plugins. You should use this method only if you need to customize plugin dependencies.

The example uses three options:

* `UseDeps(cb func(*Deps))` is useful if some or all dependencies should be customized. It replaces the whole `Dep` with all fields.
* `UseDep2(d2 dep2.SomePlugin)` replaces a specific dependency with a desired value
* `UseDep3(d3 dep3.SomePlugin)` sets a dependency which is an empty value by default. This is useful for dependencies which are not optional.

This scheme makes it possible to define multiple instances of the same plugin with different dependencies. A good example is the kvdbsync plugin, which can be instantiated multiple times with different key-value plugins such as etcd and Redis.

### Plugin Lookup Procedure

Plugin lookup handles the ordering process in various scenarios where multiple plugins possess complicated dependencies. It accomplishes this by building a tree-like structure to determine which plugins should be loaded first. The advantage of this approach is that only the top level plugin needs to be defined. Plugin lookup can then locate dependent plugins, even in a multi-layer scenario.

For example, if P1 depends on P2, which depends on P3, defining P1 is sufficient. The other plugins will be found. Therefore it is good practice to define a top-level plugin; Plugin lookup will locate all other plugins required for the application.

!!! Note
    Plugin lookup is called only in `AllPlugins()` method.


### Cross dependencies

What if two plugins are dependent on each other? Plugin lookup can handle this scenario. Let's show how this works with the following two files: `P1.go` and `P2.go`.

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

`P1.go` defines a simple plugin with `Init()`, `Close()` and `String()`. It has two "indexes" fields; one its own which is initialized during init, and one obtained from the dependent plugin. This scenario expects `P2` will be initialized first, so during `P1` init, the instance will be available.

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

The `P2.go` looks the same, but it uses `P1` as a dependency. This infers that an agent consisting of `P1` and `P2` cannot be initialized correctly. The agent lookup would have taken `P1` and found `P2` as a dependent plugin.

Inside `P2`, there is another dependency which is `P1` again. Since this plugin is known, the order would be `[P2, P1]`. The agent would panic on `P1.GetIndexes()` since `P1` is nil. The result will be the same if the order is switched. Panicking on `P2.GetIndexes()` will occur.

To address this problem, it is possible to use `AfterInit()`. It is called after the init procedure succeeded for every plugin, and it honors the same order.

```go
func (p *P1) Init() error {
	p.indexes = NewIndexes()
	return nil
}

func (p P1*) AfterInit() error {
	p.remoteIndexes = P2.GetIndexes()
}
``` 

This change applied to both plugins is sufficient to build the agent. This is because:

- during the init() phase, both plugins initialize their own indexes.
- in the afterinit() phase, both plugins obtain the respective references which have already been made available.

### Building an Agent

Several [VPP agent plugins](../plugins/vpp-plugins.md) are used in this example for building an agent.

1. Create the top-level plugin, which contains all plugins required for the application. Define a method which returns an instance of the `MyApp` plugin called `NewApp()`. Include all methods required to implement the plugin interface:

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

3 Add the connection to the etcd database. Datasync is usually wrapped as `KVProtoWatcher` or `KVProtoWriter` and then injected into the plugin:

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

4. Add the connection to the VPP (`GoVPPMux` plugin):

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

6. Optionally, other plugins such as logger or REST API can be added:

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

7. If REST API needs to work with `MyPlugin`, pass the instance:

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

Only `MyApp` (top-level plugin) needs to be set to `agent.AllPlugins()`. Plugin lookup determines which plugins should be started and in what order.

[hw-tutorial]: ../tutorials/01_hello-world.md

*[REST]: Representational State Transfer
*[VPP]: Vector Packet Processing
[infra-plugin]: https://github.com/ligato/cn-infra/blob/master/infra/infra.go