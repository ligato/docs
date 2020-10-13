# REST Handler

---

Tutorial code: [REST Handler][code-link]

In this tutorial, you will learn how to add a REST API to a plugin. Before running through this tutorial, you should complete the [Hello World tutorial](01_hello-world.md) and the [Plugin Dependencies tutorial](02_plugin-deps.md).

You should also be familiar with REST APIs. To learn about REST APIs, or if you need a quick refresher, check out the [REST API Tutorial](https://www.freecodecamp.org/news/rest-api-tutorial-rest-client-rest-service-and-api-calls-explained-with-code-examples/). 

!!! Note
    MyPlugin is the name of the plugin used in this tutorial. Note that any concepts, explanations, tasks and code block contents used in this tutorial can be applied to the HelloWorld plugin used in the previous tutorials. 

---

Each plugin that requires a REST API needs to register its own custom
handler with the REST plugin. You perform this task by using the registration API:

```go
type HandlerProvider func(formatter *render.Render) http.HandlerFunc

type HTTPHandlers interface {
	RegisterHTTPHandler(path string, provider HandlerProvider, methods ...string) *mux.Route
	// ...
}
```

To use the REST plugin, first define it as a dependency in your plugin:

```go
type MyPlugin struct {
	infra.PluginDeps
	REST rest.HTTPHandlers
}
```
Note that the dependency is defined as an interface. This means that it can be
satisfied by any object that implements the interface methods. 

The `rest.HTTPHandlers`
interface is defined in [plugin_api_rest.go file](https://github.com/ligato/cn-infra/blob/master/rpc/rest/plugin_impl_rest.go).

---

Next, wire the dependency into the plugin's constructor. The code block below uses the default REST plugin of `rest.DefaultPlugin`. Most Ligato infrastructure plugins
have a default plugin instance defined as a global variable.

```go
func NewMyPlugin() *MyPlugin {
	// ...
	p.REST = &rest.DefaultPlugin
	return p
}
```

Now, you can define your handler:

```go
func (p *MyPlugin) fooHandler(formatter *render.Render) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		body, err := ioutil.ReadAll(r.Body)
		if err != nil {
			p.Log.Errorf("Error reading body: %v", err)
			http.Error(w, "can't read body", http.StatusBadRequest)
			return
		}
		formatter.Text(w, http.StatusOK, fmt.Sprintf("Hello %s", body))
	}
}
```

---

Finally, register your handler with the REST plugin using your plugin's 
`Init()` method:

```go
func (p *MyPlugin) Init() error {
	// ...
	p.REST.RegisterHTTPHandler("/greeting", p.fooHandler, "POST")
	return nil
}
```

---

**Run the REST Handler tutorial code**

1. Open a terminal session.
<br>
<br>
2. Change to the 03_rest-handler folder:
```
cn-infra git:(master) cd examples/tutorials/03_rest-handler
```
3. Run code:
```
go run main.go
```

Example output:
```
INFO[0000] Starting agent version: v0.0.0-dev            BuildDate= CommitHash= loc="agent/agent.go(134)" logger=agent
INFO[0000] Listening on http://0.0.0.0:9191              loc="rest/plugin_impl_rest.go(109)" logger=http
INFO[0000] Agent started with 2 plugins (took 1ms)       loc="agent/agent.go(179)" logger=agent
```

!!! Note
    You might see a deny/allow incoming connection warning message. Either one is okay for this example.

4. Open a new terminal session and type in this curl command:
```
curl -X POST -d 'John' -H "Content-Type: application/json" http://localhost:9191/greeting
```
Example output:
```
Hello John
```

Example output after interrupt:
```
^CINFO[0045] Signal interrupt received, stopping.          loc="agent/agent.go(196)" logger=agent
INFO[0045] Stopping agent                                loc="agent/agent.go(269)" logger=agent
INFO[0045] Agent stopped                                 loc="agent/agent.go(291)" logger=agent
```



[code-link]: https://github.com/ligato/cn-infra/tree/master/examples/tutorials/03_rest-handler

*[HTTP]: Hypertext Transfer Protocol
*[REST]: Representational State Transfer
