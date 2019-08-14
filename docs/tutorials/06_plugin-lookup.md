# Plugin Lookup

---

Link to code: [Managing plugin dependencies with plugin lookup][code-link]

In this tutorial, we learn how to resolve dependencies between multiple plugins in various scenarios. 

Requirements:

* Complete the ['Hello World Agent'](01_hello-world.md) tutorial
* Complete the ['Plugin Dependencies'](02_plugin-deps.md) tutorial

The vpp-agent is based on plugins. A plugin is the go structure satisfying the plugin interface defined by the infrastructure. The plugin usually performs a specific task such as a connecting to the KV data store, starting HTTP handlers, VPP API support and so on. 

Often these plugins are required to work with other plugins. A good example is kvdbsync - a plugin synchronizing events from various data stores but requires one or more KV data store plugins to connect to and provide the actual data. 

The vpp-agent can consist of many plugins each with multiple dependencies. This creates a dependency structure which can sometimes be viewed as a tree although this is not always the case. 

The dependencies need to be initialized in the correct order. The rule of thumb is that the plugin listed as a dependency must be started first. The plugin interface provides two methods to work with:
 
- `Init()`
- `AfterInit()`.

The `Init()` is mandatory for every plugin and its purpose is to initialize plugin fields or other tasks such as prepare channel objects, initialize maps, start watchers). If some task cannot be performed during initialization, the plugin can call the `AfterInit()`. These scenarios will be shown in the tutorial.

The tutorial is based on the [HelloWorld](01_hello-world.md) plugin from the Hello World agent example. The original example contains a single plugin with `main()` preparing and starting the agent. 

This tutorial requires multiple files. For better clarity and understanding, we will make the following changes here at the beginning:

- Create a new go file called `plugin1.go` 
- Move hello world plugin to this file leaving only `main()` in `main.go`

Let's start with creating another plugin in a separate file called `plugin2.go`:
```go
// HelloUniverse represents another plugin.
type HelloUniverse struct{}

// String is used to identifying the plugin by giving its name.
func (p *HelloUniverse) String() string {
	return "HelloUniverse"
}

// Init is executed on agent initialization.
func (p *HelloUniverse) Init() error {
	log.Println("Hello Universe!")
	return nil
}

// Close is executed on agent shutdown.
func (p *HelloUniverse) Close() error {
	log.Println("Goodbye Universe!")
	return nil
}
```

Now we have three files: 

- `main.go` with the `main()` method 
- `plugin1.go` with the `HelloWorld` plugin 
- `plugin2.go` with the `HelloUniverse` plugin.

The next step is to start our second plugin together with the `HelloWorld`. We need to initialize a new plugin in the `main()`, add it to the `Plugins` option and rename the first plugin to `p1`):
```go
	p1 := new(HelloWorld)
	p2 := new(HelloUniverse)
	a := agent.NewAgent(agent.Plugins(p1, p2))
```

When you start/stop the program now, notice that the plugins are started in the same order as they were put to the `agent.Plugin()` method, and stopped in reversed order. 

This is very important because it ensures the dependency plugin is started before the superior plugin, and the superior plugin is stopped before its dependency.

Now we can create a relationship between plugins. In order to test this, we need some method in the `HelloUniverse` which can be called from the outside. We can create a simple flag which will be set in `Init()` and can be retrieved via `IsCreated` method. The method returns true if the `HelloUniverse` plugin was initialized, false otherwise:
````go
type HelloUniverse struct{
	created bool    // add the flag
}
````

Set the `created` field in the `Init()`:
```go
func (p *HelloUniverse) Init() error {
	log.Println("Hello Universe!")
	p.created = true
	return nil
}
```

Define method to obtain the value:
```go
// IsCreated returns true if the plugin was initialized
func (p *HelloUniverse) IsCreated() bool {
	return p.created
}
```

Now we will use this code in our `HelloWorld` plugin, so we need to set a dependency. Let's say that the world cannot exist without the universe:
```go
type HelloWorld struct{
	Universe *HelloUniverse
}
```

Our `HelloWorld` plugin is now dependent on the `HelloUniverse` plugin. The next step is to use the `IsCreated` in the `HelloWorld` method `Init()`:
```go
if p.Universe == nil || !p.Universe.IsCreated() {
    log.Panic("Our world cannot exist without the universe!")
}
log.Println("Hello World!")
return nil
``` 

The `HelloWorld` plugin verifies that the `HelloUniverse` plugin already exists, otherwise it panics. And if we try to run the agent now, this is exactly what happens. The reason is that our order of plugin initialization is not as it should be, but reversed (remember, `agent.Plugins(p1, p2)`). So what options do we have? 

!!! note
    At this point you can follow one or more of the options below. To continue with the tutorial, the third option will be required. 

### 1: Use the second layer of initialization - AfterInit()

The first option is to use the `AfterInit()` method. If the superior plugin can "wait" for the dependency to be resolved and make use of it later, we can remove it from the `Init()` and move it to the second initialization round. 

In our scenario, `HelloWorld` will be created without the universe (let's put the logic aside for a moment) and will verify its existence later. From the agent perspective, the result is the same - all the plugins and their dependencies are initialized.

Move the following code from `HelloWorld` `Init()` to the `AfterInit()`:
```go
if p.Universe == nil || !p.Universe.IsCreated() {
    log.Panic("Our world cannot exist without the universe!")
}
```

Our plugin can be started now without panicking. But remember, **in this case the dependent plugin is started before the superior plugin and stopped in reversed order**. This means the dependency will be stopped first. Any attempt to use it in the superior plugin `Close()` may instill panic. 

Despite the fact that this approach is working and can be absolutely correct in suitable scenarios, we do not recommend it. The `AfterInit()` can be used more effectively as we will see later.

### 2: Manually order plugins

!!! note
    If you followed the first approach, please move `IsCreated()` back to `Init()`.

The simplest option in our scenario is to just to manually switch plugins. In the `main()`, switch this code:
```go
a := agent.NewAgent(agent.Plugins(p1, p2))
```

to:

```go
a := agent.NewAgent(agent.Plugins(p2, p1))
```

This ensures that the `HelloUniverse` will be started before the `HelloWorld`. The dependency plugin will be initialized first (and closed second). 

While this approach is useful for small agents, the disadvantage is that it becomes difficult to manage if there are several plugins with multi-level dependencies. This is especially true when a change in dependency is introduced thus making the plugin difficult to update. 

Because of this, cn-Infra provides an automatic process which manages and re-orders dependencies. This is referred to as plugin lookup.

### 3: Order dependencies using plugin lookup

!!! note
    - If you followed the first approach, please move `IsCreated()` back to `Init()`.
    - If you followed the second approach, please set the plugin order back to `agent.Plugins(p1, p2)`.

The plugin lookup is an automatic process for sorting plugins according to their dependencies. More information about the plugin lookup process can be found [here](https://github.com/ligato/cn-infra/wiki/Agent-Plugin-Lookup).

In our plugin, we replace the `agent.Plugins()` method with the `agent.AllPlugins()` in order to use the plugin lookup feature. However, **only one plugin is recommended to be listed in the method**. Since all dependencies are found automatically, the method needs the top-level plugin only to initialize the whole agent. However setting more than one is not expressly forbidden.

The best practice is to specify another helper plugin which defines all other plugins otherwise listed in `agent.Plugins()` as dependencies. This top-level plugin (we will call it `Agent`) will not specify any inner fields. Only external dependencies and plugin methods `Init()` and `Close` will be empty. Let's create it in the `main.go` (where the `HelloWorld` plugin was before):
```go
type Agent struct {
	Hw *HelloWorld
	Hu *HelloUniverse
}

func (p *Agent) Init() error {
	return nil
}

func (p *Agent) Close() error {
	return nil
}

func (p *Agent) String() string {
	return "AgentPlugin"
}
```

We also define a function `New()` (without receiver) which sets inner plugin dependencies and returns an instance of the `Agent` plugin:
```go
func New() *Agent {
	hw := &HelloWorld{}
	hu := &HelloUniverse{}

	hw.Universe = hu

	return &Agent{
		Hw: hw,
		Hu: hu,
	}
}
```

Now, remove the old plugin dependency management from `main()`:
```go
// Delete following code
p1 := new(HelloWorld)
p2 := new(HelloUniverse)

p1.Universe = p2
```

The last step is to provide the `Agent` plugin to the plugin lookup:
```go
a := agent.NewAgent(agent.AllPlugins(New()))
```

The `main.go` looks like this:
```go
func main() {
	a := agent.NewAgent(agent.AllPlugins(New()))

	if err := a.Run(); err != nil {
		log.Fatalln(err)
	}
}
```

The agent can now be successfully started. This approach could be viewed as excess overhead since we need a new plugin. However it can save quite a bit of trouble when an agent with a complicated dependency tree is required.

**Cross dependencies**

!!! note
    Continuation of the tutorial requires the third option (plugin lookup) to be completed.

Now we have two plugins, one dependent on another, and third top-level starter plugin in `main.go`. Our `HelloWorld` plugin calls a simple method `IsCreated` checking whether the universe exists. However this method is not useful at all, so let's turn it into something more practical. 

The following changes are performed in the `HelloUniverse` plugin.

First remove the `IsCreated()` method together with the `created` flag and its assignment in the `Init()`. This is how the plugin should look now:
```go
type HelloUniverse struct{}

func (p *HelloUniverse) String() string {
	return "HelloUniverse"
}

func (p *HelloUniverse) Init() error {
	log.Println("Hello Universe!")
	return nil
}

func (p *HelloUniverse) Close() error {
	log.Println("Goodbye Universe!")
	return nil
}
```

Add the registry map. Since every universe contains many worlds, the plugin maintains the list of their names and sizes and, provides a method to register a new world. It is not uncommon for one plugin registers itself or some part to another. 

Define map in the plugin:
```go
type HelloUniverse struct{
	worlds map[string]int
}
```

Initialize the map in the `Init()`:
```go
func (p *HelloUniverse) Init() error {
	log.Println("Hello Universe!")
	p.worlds = make(map[string]int)
	return nil
}
```

Add the exported method which the `HelloWorld` plugin can use to register:
```go
func (p *HelloUniverse) RegisterWorld(name string, size int) {
	p.worlds[name] = size
	log.Printf("World %s (size %d) was registered", name, size)
}
```

Now move to the `HelloWorld` plugin. Remove following code from the `Init()` since the `IsCreated` method does not exist anymore:
```go
if p.Universe == nil || !p.Universe.IsCreated() {
    log.Panic("Our world cannot exist without the universe!")
}
```

Instead, register the world under some name:
```go
func (p *HelloWorld) Init() error {
	log.Println("Hello World!")
	p.Universe.RegisterWorld("world1")
	return nil
}
```

The dependency situation was not changed here. The `HelloUniverse` plugin must still be initialized first or an error occurs since the registration map would not be initialized. Now the code can be built and started. 

In the next step, the `HelloUniverse` works with `HelloWorld` using its methods. This requires `HelloUniverse` to treat `HelloWorld` as a dependency, creating cross (or circular) dependencies. Such a case is a bit more complicated, since the order of plugins defined in the top-level plugin matters as well.

Start with the `HelloWorld`. The world needs to be placed somewhere and the `HelloUniverse` plugin decides where it has some free space.

Add a new method to `HelloWorld`:
```go
func (p *HelloWorld) SetPlace(place string) {
	log.Printf("world1 was placed %s", place)
}
```

Now the question to be asked is when should this method should be called by the `HelloUniverse`. It cannot be called during the `Init()` since the `HelloWorld` does not exist at that point, so we must use the `AfterInit()`. 

The calling sequence will be:

- `HelloUniverse.Init()` - initialized required fields (map)
- `HelloWorld.Init()` - initialized the plugin and registered it to the `HelloUniverse`
- `HelloUniverse.AfterInit()` - can now manipulate the `HelloWorld` since it is initialized and registered

Set dependency for `HelloUniverse`:
```go
type HelloUniverse struct{
	worlds map[string]int
	
	World *HelloWorld
}
```

Add the following code to the `HelloUniverse`:
```go
func (p *HelloUniverse) AfterInit() error {
	for name := range p.worlds {
        p.World.SetPlace(name, "<some place>")
    }
    return nil
}
```

Then go to `main.go` function `New()` and add a dependency:
```go
hw := &HelloWorld{}
hu := &HelloUniverse{}

hw.Universe = hu
hu.World = hw       // add cross dependency
```

The code can be started now.

The important thing to note here is that such a cross-dependency cannot be fully resolved by the automatic plugin lookup. This is because the given plugin implementation defines the plugin order; not the dependency itself as before. 

Let's have a look at the top-level plugin:
```go
type Agent struct {
	Hw *HelloWorld
	Hu *HelloUniverse
}
```

The top-level plugin uses our two plugins as dependencies. The plugin lookup takes the provided plugin `Agent` and reads all dependencies in the order they are defined. The first plugin read is `HelloWorld`, but this plugin also has a dependency on another plugin, `HelloUniverse`. 

The `HelloUniverse` has dependency as well (on `HelloWorld`) but this plugin is already known to the plugin lookup so it is skipped. The plugin places the `HelloUniverse` first, then `HelloWorld` and the `Agent` is last. Plugin methods `Init()` and `AfterInit()` respectively will be called in this order.

Now we see that the plugin resolution for cross dependencies is based on the order of plugins. Quick test - switch the plugin order in the `Agent`:
```go
type Agent struct {
	Hu *HelloUniverse
	Hw *HelloWorld
}
``` 

The agent will end up with an error because according to the resolution key above, the automatic lookup puts `HelloWorld` first which is not correct. The rule with cross dependencies is that the plugin which should be started first is placed below all dependent plugins.

[code-link]: https://github.com/ligato/cn-infra/tree/master/examples/tutorials/06_plugin_lookup

*[HTTP]: Hypertext Transfer Protocol
*[KVDB]: Key-Value Database
