# Setup

---

This page describes the tutorials provided in the tutorials section of the Ligato documentation. The tutorials are designed for developers, but can also be used as a learning aid for those interested in understanding how Ligato works. 

For more details on the topics covered in the tutorials, consult the _Developer Guide_.  

What you need before working with the tutorials:

* Familiarity with the Ligato framework. If you are working with Ligato for the first time, a great place to begin is the [QuickStart Guide][quickstart].
* Access to the Ligato [vpp-agent][vpp-agent] and [cn-infra][cn-infra] git repositories.
* Knowledge of the Go programming language. If Go is new to you, look over the [Get started with Go](https://golang.org/doc/tutorial/getting-started) tutorial. 
* Go installed on your laptop. The same _Get started with Go_ tutorial explains how to install Go on your local machine.  


The first tutorial will show you how to create a Hello World plugin. Each subsequent tutorial will build on that by applying additional Ligato functions to your Hello World plugin. 

| Tutorial  |  Description | Tutorial Code Repo |
|---|---|---|
| [Hello World](01_hello-world.md)  |  Create a plugin printing a "hello world" message to the log | cn-infra |
|  [Plugin Dependencies](02_plugin-deps.md) | Add dependencies to your plugin  | cn-infra |
| [REST Handler](03_rest-handler.md)  |  Add a REST API to your plugin | cn-infra |
|  [KV Data Store](04_kv-store.md) | Interacting with an external data store (etcd)   | cn-infra |
| [KV Scheduler](05_kv-scheduler.md)  | Have the KV scheduler work with your plugin  | vpp-agent |
|  [Plugin Lookup](06_plugin-lookup.md) |  How to resolve dependencies between different plugins |  cn-infra |
| [VPP Connection](07_vpp-connection.md)  | Using the GoVPPMux plugin with our plugin to connect to VPP  | vpp-agent |
|  [gRPC Handler](08_grpc-tutorial.md) |  Use plugins to create a gRPC client | vpp-agent |

Each tutorial includes sample code snippets that you can inspect. A link to the complete Go code is provided at the top of each tutorial page.

To execute the tutorial code on your laptop, perform the following:

1. Clone the repository containing the tutorial code.  
2. Change to the specific tutorial folder located in the __../examples/tutorials__ folder of the repository.
3. Run the go command:
```
go run main.go
```
[vpp-agent]: https://github.com/ligato/vpp-agent
[cn-infra]: https://github.com/ligato/cn-infra
[quickstart]: ../user-guide/quickstart.md
