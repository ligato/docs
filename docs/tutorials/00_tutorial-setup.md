This section provides a set of simple tutorials describing code structure with code snippets and explanations.

| Tutorial  |  Description |
|---|---|
| [Hello World](01_hello-world.md)  |  create a plugin printing a "hello world" message to the log |
|  [Plugin Dependencies](02_plugin-deps.md) | add dependencies to your plugin  |
| [REST Handler](03_rest-handler.md)  |  add a REST API to your plugin |
|  [KV Data Store](04_kv-store.md) | interacting with an external data store (etcd)   |
| [KV Scheduler](05_kv-scheduler.md)  | have the KV scheduler work with your plugin  |
|  [Plugin Lookup](06_plugin-lookup.md) |  how to resolve dependencies between different plugins |
| [VPP Connection](07_vpp-connection.md)  | Using the GoVPPMux plugin with our plugin to connect to VPP  |
|  [gRPC Handler](08_grpc-tutorial.md) |  use plugins to create a gRPC client |

If you are familiar with Go and already have it set up and running on your computer, you can proceed to the tutorial of your choosing.

If you are new to Go and wish to run the Ligato tutorials, you will need to install Go on your computer.

## Install Go on your computer

Refer to [Getting Started](https://golang.org/doc/install) for download and installation procedures.

Verify it is installed by checking the version
```
go version
```

Example output
```
go version go1.13.6 darwin/amd64
```

!!! Note
    The tutorial code is contained in the examples/tutorials folders of the cn-infra or vpp-agent repositories. You will need to clone these repos and change to the tutorial folder to run the examples. A link to each folder is provided at the top if each tutorial page.