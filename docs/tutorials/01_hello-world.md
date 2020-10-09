# Hello World

---

Tutorial code: [Hello World][code-link]

In this tutorial, you will create a Ligato agent that 
contains a plugin called __HelloWorld__. This plugin prints "Hello World" to the log.

!!! Note
    The term `agent` is used to describe a Ligato software component providing life-cycle management functions for plugins.

Let's start with the plugin. Every plugin must implement the `Plugin` interface
defined in the [cn-infra/infra](https://github.com/ligato/cn-infra/blob/master/infra/infra.go) package:

```go
type Plugin interface {
	// Init is called in the agent`s startup phase.
	Init() error
	// Close is called in the agent`s cleanup phase.
	Close() error
	// String returns unique name of the plugin.
	String() string
}
```

Implement the `Plugin` interface methods for your `HelloWorld` plugin:

```go
type HelloWorld struct{}

func (p *HelloWorld) String() string {
	return "HelloWorld"
}

func (p *HelloWorld) Init() error {
	log.Println("Hello World!")
	return nil
}

func (p *HelloWorld) Close() error {
	log.Println("Goodbye World!")
	return nil
}
```
Note that the `HelloWorld` struct is empty. Your plugin does not 
have any data, so all that you need is an empty structure that satisfies the 
`Plugin` interface.

Some plugins require additional initialization after the base system is up. 
If required, you can optionally define the `AfterInit` method for your
plugin. This method will be executed after the `Init` method has been called for
all plugins. 

The `AfterInit` method originates from the `PostInit` interface
defined in the cn-infra/infra package:

```go
type PostInit interface {
	// AfterInit is called once Init() of all plugins have returned without error.
	AfterInit() error
}
```

Next, create an instance of the `HelloWorld` plugin. Then, create 
a new agent and tell it about the `HelloWorld` plugin:

```go
func main() {
    	p := new(HelloWorld)    
    	a := agent.NewAgent(agent.Plugins(p))
    	// ...
}
```

You can use `agent` options to add the list of plugins to the agent at the agent's creation
time. In the code snippet above, the `agent.Plugins` option is used to add the newly created 
`HelloWorld` plugin instance to the agent.

Alternatively, you could use the `agent.AllPlugins` option. This option would add your `HelloWorld` plugin instance to the agent. It will also add any dependencies that the HelloWorld plugin might have. Since your plugin has no dependencies, the simpler `agent.Plugins` option will suffice.

Finally, start the agent using its `Run()` method. This will initialize
the agent's plugins by calling their `Init` and `AfterInit` methods, and then wait
for an interrupt from the user.


```go
if err := a.Run(); err != nil {
	log.Fatalln(err)
}
```
When an interrupt, such as `ctrl-c`, arrives from the user, the `Close` methods will be called on all of the agent's plugins, and the agent will exit.

Run the Hello World code:
```
go run main.go
```
Example output:
```
INFO[0000] Starting agent version: v0.0.0-dev            BuildDate= CommitHash= loc="agent/agent.go(134)" logger=agent
2020/01/17 10:25:02 Hello World!
2020/01/17 10:25:02 All systems go!
INFO[0000] Agent started with 1 plugins (took 0s)        loc="agent/agent.go(179)" logger=agent
```

Example output after interrupt:

```
^CINFO[0030] Signal interrupt received, stopping.          loc="agent/agent.go(196)" logger=agent
INFO[0030] Stopping agent                                loc="agent/agent.go(269)" logger=agent
2020/01/17 10:25:32 Goodbye World!
INFO[0030] Agent stopped                                 loc="agent/agent.go(291)" logger=agent
```

[code-link]: https://github.com/ligato/cn-infra/tree/master/examples/tutorials/01_hello-world