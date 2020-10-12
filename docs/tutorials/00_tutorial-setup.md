# Setup

---

This page describes the tutorials included in the Ligato documentation. Developers are the intended audience for the tutorials, but they can also be used as a learning aid for those interested in understanding how Ligato works.

For more details on the topics covered in the tutorials, consult the _User Guide_ and _Developer Guide_.  

What you need before working with the tutorials:

* Familiarity with the Ligato framework. If you are working with Ligato for the first time, a great place to start is the [QuickStart Guide][quickstart].
* Access to the Ligato [vpp-agent][vpp-agent] and [cn-infra][cn-infra] git repositories. Depending on the tutorial, its code resides in the repository as noted in the table below.   
* Knowledge of the Go programming language. If Go is new to you, look over the [Get started with Go](https://golang.org/doc/tutorial/getting-started) tutorial. 
* Go installed on your local machine. The same _Get started with Go_ tutorial explains how to install Go on your local machine.
* VPP/FD.io open source code. VPP is a high-performance software network function. To find out more about VPP, see [FD.io](https://fd.io/).
* gRPC. To learn about gRPC, see [gRPC.io](https://grpc.io).   
---

The structure of each tutorial consists of the following:

- Sequence of programming tasks supported by example code blocks. 
- Link to the complete tutorial code and, where possible, instructions on how to execute. 

!!! Note
    The purpose of the code blocks is to show you possible Go programming patterns and examples for functions executed in the tutorial program. You can also inspect the code blocks to compare them against the code contained in the complete tutorial code.

---

The first tutorial will show you how to create a Ligato agent containing a HelloWorld plugin. Subsequent tutorials will cover Ligato functions that build on the first tutorial.

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


[vpp-agent]: https://github.com/ligato/vpp-agent
[cn-infra]: https://github.com/ligato/cn-infra
[quickstart]: ../user-guide/quickstart.md
