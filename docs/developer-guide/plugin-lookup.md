# Plugin Lookup

---

Under the term **Agent** is meant a set of plugins, which are started and initialized in the correct order according to relations between them.

### Quick Start Guide

1. Define your plugin. Every plugin must implement the `infra.Plugin` interface. 

2. Use the `agent.Plugins(<plugin>...)` function to create an instance of `agent.Option`, which is a configuration stanza used to tell the Agent about your plugin. Pass the plugin defined in Step 1 as a parameter into `agent.Plugins(<plugin>...)`. If you need to define multiple plugins, you can pass them into the function all at once, since its parameter is variadic. If there are relationships/dependencies between your plugins and/or if one or more of your plugins depends on other plugins which are not explicitly listed, use the `agent.AllPlugins(<plugin>...)` function to create the `agent.Options` object. `agent.AllPlugins(<plugin>...)` will automatically sort plugins into the correct initialization order.

3. Use the `agent.NewAgent(opts ...Option)` function to create a new agent instance.

4. Use the `Run()` method (blocking) or the `Start()` method (non-blocking) to start the agent created in Step 3.

5. Stop the agent with the `Stop()` method; alternatively, define a struct-type channel and add it to the agent using the option `agent.QuitOnClose(<channel>)`. Closing the channel stops the agent.


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

### Definitions

**Agent** is an object that implements the `Agent` interface. It is created using the `agent.NewAgent(<options>...)` function. Every agent instance contains configuration (basically a list of options, see below), a channel which can gracefully stop the agent and a tracer (time-measurement utility). The options are expected to be provided by the user and are used to configure the agent when it is initialized. Every agent instance also has a mechanism to prevent it from starting/stopping it multiple times.

**Options** are configuration stanzas used to customize an agent. The most important option is the *plugin list*, which tells the agent about the plugins it needs to control. If the initialization order of your plugins and their dependencies does not matter, use the `agent.Plugins(<plugin>....)` function to create the plugin list option.  If your plugins need to be initialized in certain order because there are dependencies to resolve, use the `agent.Allplugins(<plugin>)` function to create the plugin list option.

The available [options](https://godoc.org/github.com/ligato/cn-infra/agent#Option) to customize an agent are:
* `Plugins(...)` adds one or more plugins that will be started in the same order they were added
* `AllPlugins(...)` adds one or more plugins and sorts them into the correct startup order; this option will automatically add all plugins listed as dependencies in other plugins
* `Version(<version>, <date>, <id>)` sets the version of the program
* `QuitOnClose(<channel>)` sets the channel which can be used to terminate the running agent when closed
* `QuitSignals(<signals>)` sets the OS signals which can be used to quit the running agent (default: SIGINT, SIGTERM)
* `StartTimeout(duration)/StopTimeout(duration)` sets the start/stop timeout (defaults: 15s/5s)
  
**Plugin** is an object that implements the `Plugin` interface, which defines methods required for plugin lifecycle management. At program startup, a list of plugins is read from options - at this point plugins are already sorted for initialization (either manually or via plugin lookup). Agent initialization is then performed in two steps:
* The startup procedure calls the `Init()` method for every plugin, one-by-one in a single thread in the order in which they are sorted for initialization.
* The startup procedure calls the `AfterInit()` method for each plugin one-by-one, in a single thread in the order in which they are sorted for initialization. This two-step procedure ensures that certain initialization tasks are only performed after all plugins in the agent were pre-initialized (See [this tutorial][hw-tutorial] for more details). Note that you may leave the `AfterInit()` method empty if you do not need the second initialization phase.

### Plugin Lifecycle

The plugin is initialized in `Init()` method, which should allocate all resources required by the plugin, for example make channels or maps, or initialize any other fields used within the plugin. If a plugin needs another plugin in order to be initialized properly, such a relation is called *dependency*. If there are dependencies, plugins must be initialized in proper order. i.e. the "dependency" plugin must be initialized before the plugin that depends on it.

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

In the above code we have two plugins, P1 and P2. P1 uses P2 as a dependency. It is good practice to move all plugin-type dependencies into a separate structure (usually called `Dep`). To execute the startup sequence properly, P2 must be initialized first. Otherwise, P1 may work with an empty reference, which will result in unexpected behavior during agent startup (mostly panics with nil pointer reference).

Every plugin also must satisfy the `Close()` method, in which you should release all plugin resources (for example, close channels or connections), and the `String()` method, which should return a text representation of the plugin.

The responsibility for the correct initialization order depends on the function used to create the *plugin list* option:
*  `agent.Plugins(<plugin>...)` starts only plugins set as a parameter and also in the same order as they were set. All plugins (including dependencies) have to be listed. If not, they won't be initialized at all. This method is best for scenarios with one plugin, or for more plugins which are not dependent. Otherwise, it is the programmer responsibility to set them in the correct order.
* `agent.AllPlugins(<plugin>...)` resolves the dependency tree itself using [plugin lookup](plugin-lookup.md#plugin-lookup-procedure). Also, not all dependencies have to be listed; plugin lookup can find them by itself.

Let's try a simple example:

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

The correct initialization order for the example above is **[P2, P3, P1]** or **[P3 P2, P1]** (since P2 and P3 are independent of each other, the order in which they are initialized does not matter). If you use `agent.Plugins()`, you must list all plugins in the desired initialization order. If you use `agent.AllPlugins()`, only P1 needs to be listed. Since P2 and P3 are dependencies, the plugin lookup mechanism will find them and then put all plugins into the right initialization order.

### Plugin Options

Since the dependency management is complicated, and it's not an easy task to make it reliable, easy to use and not overwhelming, every plugin comes with a simple helper file called `options.go` (this name is the convention used in the cn-infra, the name of the struct file can by anything).

Let's start with a template:

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

Almost all of the cn-infra plugins use the global variable `DefaultPlugin`. It constructs a new instance of the plugin with pre-defined dependency fields. A good practice is to keep default plugin instance perfectly usable and able to init without any customization.

The `NewPlugin()` is a base constructor. If used without any options (parameters), it returns the default instance. Notice that some dependencies are set to default instances of other plugins. This method should be used only if we want to somehow customize plugin dependencies. The example uses three options:
* `UseDeps(cb func(*Deps))` is a convenient way if many or all dependencies should be customized. It replaces the whole `Dep` with all fields. 
* `UseDep2(d2 dep2.SomePlugin)` replaces specific dependency with desired value
* `UseDep3(d3 dep3.SomePlugin)` sets dependency which is an empty value by default, good for dependencies which are optional

Such a scheme allows to easily define multiple instances of the same plugin with different dependencies. A good example is the kvdbsync plugin, which can be instantiated multiple times with different key-value proto plugins (i.e. for ETCD, Redis, ...).

### Plugin Lookup Procedure

The plugin lookup resolves plugin ordering in difficult scenarios where multiple plugins have complicated dependencies. It tries to build a tree-like structure and determine, which plugins should be loaded first. The big advantage is that only the top level plugin must be defined. The plugin lookup can find dependent plugins itself, even in a multi-layer scenario. So if P1 depends on P2, which depends on P3, defining P1 is just enough, other plugins will be found. Because of this, it is good practice to define a simple top-level plugin which contains all the other plugins needed for the application.

However, there are still particular scenarios where extra care needs to be taken, discussed in part [cross dependencies](plugin-lookup.md#cross-dependencies).

The plugin lookup is called only in `AllPlugins()` method.  

### Cross dependencies

What if two plugins are dependent on each other? The plugin lookup can handle such a scenario, but it may require some extra steps to ensure that it will be done correctly. Let's demonstrate the following scenario with two files

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

The `p1.go` defines simple plugin with `Init()`, `Close()` and `String()` (implements plugin interface) and has two "indexes" fields; one its own which is initialized anew during init, and one got from the dependent plugin. This scenario expects that the `P2` will be initialized first, so during `P1` init the instance will be available.

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

The `p2.go` looks the same, but it uses `P1` as a dependency. The brief look tells, that an agent consisting of `P1` and `P2` cannot be initialized correctly. The agent lookup would have taken `P1` and found a `P2` as a dependent plugin. Inside it, there is another dependency which is `P1` again - but since this plugin is known, the result order would be `[P2, P1]`. The Agent would panic on `P1.GetIndexes()` since the `P1` is nil.

However, if the order is switched, the result will be the same (panicking on `P2.GetIndexes()`). A way how to solve this may depend on application's needs. In this case, it is possible to use `AfterInit()`. It is called after the init procedure was successful for every plugin, and it respects the same order.

```go
func (p *P1) Init() error {
	p.indexes = NewIndexes()
	return nil
}

func (p P1*) AfterInit() error {
	p.remoteIndexes = P2.GetIndexes()
}
``` 

Such a change in both plugins is enough to have the agent built since during the init phase both plugins initialize their own indexes and in after init phase, they get respective references which are already available at this point.

### Building an Agent

In this example are used some plugins from the [vpp-agent](https://github.com/ligato/vpp-agent) project.

1. Create the top-level plugin, which will contain all plugins required for the application. Also define some method which returns an instance of the `MyApp` plugin (let's call it `NewApp()`) and all methods required to implement the plugin interface:

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

2. Add a user-defined plugin (with name `MyPlugin`):

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

3 Add the connection to ETCD database. Datasync is usually wrapped as `KVProtoWatcher` or `KVProtoWriter` and then injected into the plugin:

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

6. Optionally, other plugins like logger or REST API can be added:

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

7. If REST API is expected to work with `MyPlugin`, pass the instance:

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

Only `MyApp` (top-level plugin) needs to be set to `agent.AllPlugins()`. The plugin lookup figures out which plugins should be started and also their order. 

[hw-tutorial]: ../tutorials/01_hello-world.md