# Plugin Lookup

---

Tutorial code: [Plugin Lookup][code-link]

In this tutorial, we learn how to resolve dependencies between multiple plugins in various scenarios. Before running through this tutorial, you should complete the [Hello World tutorial](01_hello-world.md) and the [Plugin Dependencies tutorial](02_plugin-deps.md).   

For more details on plugin lookup, see the [plugin lookup section](../developer-guide/plugin-lookup.md) of the _Developer Guide_.

---

The VPP agent is based on plugins. The plugin usually performs a specific task such as connecting to the KV data store, starting HTTP handlers, or exposing a VPP configuration API. 

Often these plugins must work with other plugins. A good example is the kvdbsync plugin which is responsible for synchronizing events from various data stores. It requires one or more KV data store plugins to connect to and provide the actual data. 

The dependencies need to be initialized in the correct order. The rule of thumb is that the plugin listed as a dependency must be started first. 

The plugin interface provides two methods to work with:
 
- `Init()`
- `AfterInit()`.

The `Init()` is mandatory for every plugin and its purpose is to initialize plugin fields or other tasks such as prepare channel objects, initialize maps, start watchers). If some task cannot be performed during initialization, the plugin can call the `AfterInit()`. These scenarios will be shown in the tutorial.

This tutorial uses the [HelloWorld](01_hello-world.md) plugin from the Hello World agent example. That original tutorial example contained a single plugin with `main()` preparing and starting the agent. 

This tutorial requires multiple files. For better clarity and understanding, make the following changes here at the beginning:

- Create a new go file called `plugin1.go`. 
- Move the HelloWorld plugin to this file leaving only `main()` in `main.go`.

Let's begin by creating another plugin in a separate file called `plugin2.go`:
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

Now you have three files: 

- `main.go` with the `main()` method 
- `plugin1.go` with the `HelloWorld` plugin 
- `plugin2.go` with the `HelloUniverse` plugin.

The next step is to start your second plugin together with the `HelloWorld`. Initialize a new plugin in the `main()` add it to the `Plugins` option. Then rename the first plugin to `p1`):
```go
	p1 := new(HelloWorld)
	p2 := new(HelloUniverse)
	a := agent.NewAgent(agent.Plugins(p1, p2))
```

If you start and stop the program now, you would notice that the code starts the plugins in the same order as they were put to the `agent.Plugin()` method. When the program terminates, the code stops the plugins in the reverse order.  
  
This is very important because this code pattern ensures the dependency plugin starts first. The primary plugin starts second. The primary plugin will stop before its dependency starts. 

Now you can create a relationship between plugins. In order to test this, you need a method in the `HelloUniverse` which is called from the outside. 

To address this requirement, create a simple flag which is set in the `Init()`, and can be retrieved via `IsCreated` method. The `IsCreated` method returns true if the code creates the `HelloUniverse` plugin. Otherwise, the method will return false:
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

Define a method to obtain the value:
```go
// IsCreated returns true if the plugin was initialized
func (p *HelloUniverse) IsCreated() bool {
	return p.created
}
```

Use this code in our `HelloWorld` plugin, and set a dependency: 
```go
type HelloWorld struct{
	Universe *HelloUniverse
}
```
Let's say that the world cannot exist without the universe. This means yur `HelloWorld` plugin is dependent on the `HelloUniverse` plugin. 

Use the `IsCreated` method in your `HelloWorld` method `Init()`:
```go
if p.Universe == nil || !p.Universe.IsCreated() {
    log.Panic("Our world cannot exist without the universe!")
}
log.Println("Hello World!")
return nil
``` 

The `HelloWorld` plugin verifies that the `HelloUniverse` plugin already exists, otherwise it panics. If you run the agent now, a panic will result. The reason is that our order of plugin initialization is not correct, but instead reversed. 
 
 Recall `agent.Plugins(p1, p2)`) statement. The order of initialization should be the HelloUniverse dependency plugin started first; The primary HelloWorld started second. 
 
 So what options do you have to ensure the correct order of initialization? 

!!! note
    At this point you can follow one or more of the options below. To run the tutorial code, you will need to work with the third option: _3: Order dependencies using plugin lookup_  

### 1: Use the second layer of initialization - AfterInit()

The first option is to use the `AfterInit()` method. If the primary plugin can "wait" for the dependency to be resolved and make use of it later, you can remove it from the `Init()`, and move it to the second initialization round. 

Let's put the logic aside for a moment. In your scenario, `HelloWorld` will be created without the universe (), and its existence verified later. This action initializes all plugins and their dependencies from the agent's perspective. This achieves the same result, but the order is incorrect.   

Move the following code from `HelloWorld` `Init()` to the `AfterInit()`:
```go
if p.Universe == nil || !p.Universe.IsCreated() {
    log.Panic("Our world cannot exist without the universe!")
}
```

Your plugin can be started without panicking. In this case the dependency plugin starts before the primary plugin, and stops in reverse order. In other words, the dependency plugin stops first. Any attempt to use it in the primary plugin `Close()`method, will cause panic. 

Despite the fact that this approach is working and correct in suitable scenarios, you should avoid this approach if possible. The `AfterInit()` can be used more effectively as you will see later in this tutorial.

### 2: Manually order plugins

!!! note
    If you followed the first approach, please move `IsCreated()` back to `Init()`.

The simplest option in our scenario is to manually switch plugins. In the `main()`, switch this code:
```go
a := agent.NewAgent(agent.Plugins(p1, p2))
```

to:

```go
a := agent.NewAgent(agent.Plugins(p2, p1))
```

The plugin order switch ensures that `HelloUniverse` is started before the `HelloWorld`.  

While this approach is useful for small agents, the disadvantage is that it becomes difficult to manage if there are several plugins with multi-level dependencies. This is especially true when a change in dependency occurs, thus making plugins difficult to update. 

Because of this, cn-Infra provides an automatic process which manages and re-orders dependencies. This is referred to as plugin lookup.

### 3: Order dependencies using plugin lookup

!!! note
    - If you followed the first approach, please move `IsCreated()` back to `Init()`.
    - If you followed the second approach, please set the plugin order back to `agent.Plugins(p1, p2)`.

The plugin lookup is an automatic process for sorting plugins according to their dependencies. 

More information about the plugin lookup process can be found in the _Plugin Lookup_ section of the _Developer Guide_. 

In your plugin, replace the `agent.Plugins()` method with the `agent.AllPlugins()` in order to use the plugin lookup feature. Note that you should only list one plugin in this method. Plugin lookup automatically finds all dependencies. The `agent.AllPlugins()` method needs only the top-level plugin to initialize the whole agent. 

The best practice if for you to specify another helper plugin which defines all other plugins listed in `agent.Plugins()` as dependencies. This top-level plugin, referred to as `Agent` in the code blocks below, will not specify any inner fields. Only external dependencies and the plugin methods `Init()` and `Close` will be empty.
 
 Let's create it in the `main.go` where the `HelloWorld` plugin was used before:
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

Define a function `New()` without a receiver. This sets inner plugin dependencies, and returns an instance of the `Agent` plugin:
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

Remove the old plugin dependency management from `main()`:
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

The agent can now be successfully started. This approach could be viewed as excess overhead since you need a new plugin. However, it can save you quite a bit of trouble when an agent is dealing with a complicated dependency tree.

**Cross dependencies**

!!! note
    Continuation of this tutorial requires the third option, _3: Order dependencies using plugin lookup_, to be completed.

Now you have two plugins: one dependent on another, and a third top-level starter plugin in `main.go`. Your `HelloWorld` plugin calls a simple method `IsCreated`. This method checks if the universe exists. 

However, you should avoid this approach if possible as noted above.

Let's turn it into something more practical.  

First, in the `HelloUniverse` plugin, remove the `IsCreated()` method together with the `created` flag and its assignment in the `Init()`. 

This is how the plugin should look:
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

Add the registry map. Since every universe contains many worlds, the plugin maintains the list of their names and sizes, and provides a method to register a new world. It is not uncommon for one plugin to registers itself to another. 

Define a map in the plugin:
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

Move to the `HelloWorld` plugin. Remove following code from the `Init()` since the `IsCreated` method does not exist anymore:
```go
if p.Universe == nil || !p.Universe.IsCreated() {
    log.Panic("Our world cannot exist without the universe!")
}
```

Instead, register the world under `world1` name:
```go
func (p *HelloWorld) Init() error {
	log.Println("Hello World!")
	p.Universe.RegisterWorld("world1")
	return nil
}
```

Note that the dependency situation is unchanged. The `HelloUniverse` plugin must still be initialized first, or an error occurs since the registration map would not be initialized. Now the code can be built and started. 

In the next step, the `HelloUniverse` works with `HelloWorld` using its methods. This requires `HelloUniverse` to treat `HelloWorld` as a dependency, creating cross (or circular) dependencies. 

This case is a bit more complicated, since the order of plugins defined in the top-level plugin matters as well.

Start with the `HelloWorld`. The world needs to be placed somewhere, and the `HelloUniverse` plugin decides where it has some free space.

Add a new method to `HelloWorld`:
```go
func (p *HelloWorld) SetPlace(place string) {
	log.Printf("world1 was placed %s", place)
}
```

Now, the question to ask is, when should this method should be called by the `HelloUniverse`? It cannot be called during the `Init()` since the `HelloWorld` does not exist at that point. You must use the `AfterInit()` method. 

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

Add the following code to the `HelloUniverse` plugin:
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

The important thing to note here is that such a cross-dependency cannot be fully resolved by the automatic plugin lookup. This is because the given plugin implementation defines the plugin order; not the dependency itself as before. 

Let's have a look at the top-level plugin:
```go
type Agent struct {
	Hw *HelloWorld
	Hu *HelloUniverse
}
```

The top-level plugin uses your two plugins as dependencies. The plugin lookup takes the provided plugin `Agent` and reads all dependencies in the order they are defined. The first plugin read is `HelloWorld`, but this plugin also has a dependency on another plugin, `HelloUniverse`. 

The `HelloUniverse` has a dependency on `HelloWorld`. But this plugin is already known to the plugin lookup process so it is skipped. The plugin places the `HelloUniverse` first, then `HelloWorld` and the `Agent` plugin is last. The plugin methods `Init()` and `AfterInit()` will be called in this order.

Note that the plugin resolution for cross dependencies The order of plugins determines the plugin resolution for cross dependencies. 

Switch the plugin order in the `Agent`:
```go
type Agent struct {
	Hu *HelloUniverse
	Hw *HelloWorld
}
``` 

The agent will end up with an error because according to the resolution key above, plugin lookup puts `HelloWorld` first which is not correct. The rule with cross dependencies is that the plugin which should be started first is placed below all dependency plugins.

[code-link]: https://github.com/ligato/cn-infra/tree/master/examples/tutorials/06_plugin_lookup

*[HTTP]: Hypertext Transfer Protocol
*[KVDB]: Key-Value Database
