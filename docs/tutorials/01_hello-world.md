# Hello World

---

Link to code: [Getting Started: the 'Hello World' agent][code-link]

In this tutorial we will create a simple Ligato control plane agent that 
contains a single `Helloworld` plugin that prints "Hello World" to the log.

!!! Note
    In the tutorial section, the term `agent` is used to describe a Ligato software component providing life-cycle management functions for plugins.

We start with the plugin. Every plugin must implement the `Plugin` interface
defined in the `github.com/ligato/cn-infra/infra` package:

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

Let's implement the `Plugin` interface methods for our `HelloWorld` plugin:

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
Note that the `HelloWorld` struct is empty - our simple plugin does not 
have any data, so we just need an empty structure that satisfies the 
`Plugin` interface.

Some plugins may require additional initialization that can only be
performed after the base system is up. If your plugin
needs this, you can optionally define the `AfterInit` method for your
plugin. It will be executed after the `Init` method has been called for
all plugins. The `AfterInit` method comes from the `PostInit` interface
defined in the `github.com/ligato/cn-infra/infra` package as:

```go
type PostInit interface {
	// AfterInit is called once Init() of all plugins have returned without error.
	AfterInit() error
}
```

Next, in our main function we create an instance of the `HelloWorld` plugin. Then we 
create a new agent and tell it about the `HelloWorld` plugin:

```go
func main() {
    	p := new(HelloWorld)    
    	a := agent.NewAgent(agent.Plugins(p))
    	// ...
}
```

We use agent options to add the list of plugins to the agent at the agent's creation
time. In our example we use the option `agent.Plugins` to add the newly created 
`HelloWorld` instance to the agent.

Alternatively, we could use the option
`agent.AllPlugins`, which would add our `HelloWorld` plugin instance to the agent,
along with all of its dependencies (i.e. all plugins it depends on). Since our 
simple plugin has no dependencies, the simpler `agent.Plugins` option will suffice.

Finally, we can start the agent using its `Run()` method, which will initialize
all agent's plugins by calling their `Init` and `AfterInit` methods and then wait
for an interrupt from the user.



```go
if err := a.Run(); err != nil {
	log.Fatalln(err)
}
```
When the interrupt comes from the user (for example. when the user hits `ctrl-c`), 
the `Close` methods will be called on all agent's plugins and the agent will exit.

__Run the Hello World code__
```
go run main.go
```
Example output
```
INFO[0000] Starting agent version: v0.0.0-dev            BuildDate= CommitHash= loc="agent/agent.go(134)" logger=agent
2020/01/17 10:25:02 Hello World!
2020/01/17 10:25:02 All systems go!
INFO[0000] Agent started with 1 plugins (took 0s)        loc="agent/agent.go(179)" logger=agent
```

Example output upon `ctrl-c user interrupt`

```
^CINFO[0030] Signal interrupt received, stopping.          loc="agent/agent.go(196)" logger=agent
INFO[0030] Stopping agent                                loc="agent/agent.go(269)" logger=agent
2020/01/17 10:25:32 Goodbye World!
INFO[0030] Agent stopped                                 loc="agent/agent.go(291)" logger=agent
```

The complete working example can be found at [examples/tutorials/01_hello-world](https://github.com/ligato/cn-infra/blob/master/examples/tutorials/01_hello-world).

!!! Note
    In some examples, the output may differ from what was described above. Examine the working code example to locate those differences.

[code-link]: https://github.com/ligato/cn-infra/tree/master/examples/tutorials/01_hello-world