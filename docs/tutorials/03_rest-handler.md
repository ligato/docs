# REST Handler

---

Link to code: [Adding a REST API to your Plugin][code-link]

In this tutorial we will learn how to add a REST API to your plugin. 
The Ligato infrastructure provides an HTTP server that is used by all plugins
that wish to expose a REST API to external clients. The HTTP Server is provided
by the [REST plugin](https://github.com/ligato/cn-infra/tree/master/rpc/rest).

Requirements:

* Complete the ['Hello World Agent'](01_hello-world.md) tutorial
* Complete the ['Plugin Dependencies'](02_plugin-deps.md) tutorial

Each plugin requiring a REST API will register its own custom
handler with the REST plugin using the registration API:

```go
type HandlerProvider func(formatter *render.Render) http.HandlerFunc

type HTTPHandlers interface {
	RegisterHTTPHandler(path string, provider HandlerProvider, methods ...string) *mux.Route
	// ...
}
```

To use the REST plugin, we first define it as a dependency in our plugin:

```go
type MyPlugin struct {
	infra.PluginDeps
	REST rest.HTTPHandlers
}
```
Note that the dependency is defined as an `interface`. This means it can be
satisfied by any object that implements the interface methods. The `rest.HTTPHandlers`
interface is defined in [`cn-infra/rpc/rest/plugin_api_rest.go`](https://github.com/ligato/cn-infra/blob/master/rpc/rest/plugin_impl_rest.go).

Next, we can "wire" the dependency (i.e. set the instance) in the plugin's 
constructor. Note that we use the default REST plugin provided by the Ligato
infrastructure (`rest.DefaultPlugin`). Most Ligato infrastructure plugins
have a default plugin instance defined as a global variable that can be used.

```go
func NewMyPlugin() *MyPlugin {
	// ...
	p.REST = &rest.DefaultPlugin
	return p
}
```

Now we define our handler:

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

Finally, we register our handler with the REST plugin. This is done in our plugin's 
`Init` method:

```go
func (p *MyPlugin) Init() error {
	// ...
	p.REST.RegisterHTTPHandler("/greeting", p.fooHandler, "POST")
	return nil
}
```


__Run the REST Handler code__

```
go run main.go
```
Example output on the terminal CLI
```
INFO[0000] Starting agent version: v0.0.0-dev            BuildDate= CommitHash= loc="agent/agent.go(134)" logger=agent
INFO[0000] Listening on http://0.0.0.0:9191              loc="rest/plugin_impl_rest.go(109)" logger=http
INFO[0000] Agent started with 2 plugins (took 1ms)       loc="agent/agent.go(179)" logger=agent
```

You might see an deny/allow incoming connection warning message. Either one is okay for this example.

Open a new terminal in the `03_rest-handler chrismetz` folder and type in this curl command
```
curl -X POST -d 'John' -H "Content-Type: application/json" http://localhost:9191/greeting
```
Output
```
Hello John
```

Example output upon `ctrl-c user interrupt`
```
^CINFO[0045] Signal interrupt received, stopping.          loc="agent/agent.go(196)" logger=agent
INFO[0045] Stopping agent                                loc="agent/agent.go(269)" logger=agent
INFO[0045] Agent stopped                                 loc="agent/agent.go(291)" logger=agent
```




Complete working example can be found at [examples/tutorials/03_rest-handler](https://github.com/ligato/cn-infra/blob/master/examples/tutorials/03_rest-handler).

[code-link]: https://github.com/ligato/cn-infra/tree/master/examples/tutorials/03_rest-handler

*[HTTP]: Hypertext Transfer Protocol
*[REST]: Representational State Transfer
