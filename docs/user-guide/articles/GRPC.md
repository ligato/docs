This page describes how to use the VPP-Agent with the gRPC

# VPP-Agent GRPC

Related article: [GRPC tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial)

The base of the GRPC support in the VPP-Agent is a [CN-Infra GRPC plugin](https://github.com/ligato/cn-infra/blob/master/rpc/grpc/README.md), which is an infrastructure plugin allowing to handle GRPC requests.

The VPP-Agent GRPC can be used to:
* Send a VPP configuration
* Retrieve (dump) a VPP configuration
* Start notification watcher

Remote procedure calls defined:
**Get** is used to update existing configuration, or create it if not exists yet.
**Delete** removes desired configuration.
**Dump** (retrieve) reads existing configuration from the VPP.
**Notify** subscribes the GRPC notification service

To enable the GRPC server within the Agent, the GRPC plugin has to be added to the plugin pool and loaded (currently the GRPC plugin is a part of the Configurator plugin dependencies //TODO add link). The plugin also requires startup configuration file (see [CN-Infra GRPC plugin](https://github.com/ligato/cn-infra/blob/master/rpc/grpc/README.md)) with endpoint defined.

The communication can be done via endpoint IP address and port, or via unix domain socket file. The TCP network is set as default, but other network types are available (like TCP6 or UNIX)
