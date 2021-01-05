# Frequently Asked Questions

---

## General

### What is the VPP agent?

A [control plane agent for a VPP data plane](framework.md#VPP agent). It comes with various plugins for programming data plane network functions. The VPP agent is separate from VPP data plane. You can use the VPP agent for developing customized or out-of-the-box VPP-based CNFs. 

Note that you can use the VPP agent with the Linux kernel data plane to develop applications. 

---

### What is cn-infra?

[Framework of libraries and plugins](framework.md#infra) for building control plane agents for CNFs and applications.

---

### Is Ligato an application?

Not really. Think of Ligato components, such as VPP agent plugins, as building blocks for your CNF and/or cloud native application implementations.

---

### What is the relationship between the VPP agent and cn-infra?

They compliment each other. Both provide plugins you can use to develop CNFs. For example, a VPP-based CNF uses VPP agent plugins for programming the data plane, and cn-infra plugins (e.g. etcd connector, REST, gRPC, etc.) for communications with external applications.

---

### What is a CNF?

[Cloud Native Network Function](https://ligato.io/cnf/cnf-def/). Containerized network functions  subscribing to cloud native architectural and operational principles.

---

### What is VPP?

[Vector Packet Processing](https://wiki.fd.io/view/VPP/What_is_VPP%3F) is a software platform for forwarding packets. [FD.io](https://fd.io/) is the open source project for VPP.

---

### Is VPP part of Ligato?

No. Ligato includes the VPP agent along with other plugins providing overall management, control and lifecycle support for VPP-based CNFs and cloud-native applications. 

However, to ensure VPP agent and VPP compatibility, and maintain its lightweight "footprint", the Ligato/vpp-agent docker images include the VPP data plane.

---

### Can I use Ligato to build non-VPP cloud native apps?

Yes. One example is an [experimental BGP agent](https://github.com/ligato/bgp-agent) that does not use the VPP data plane.  

---

### Is Ligato a CNCF project?

Ligato is a member of the [larger CNCF ecosystem](https://landscape.cncf.io/). Many Ligato plugins build on CNCF projects such as gRPC, etcd, and Prometheus.

---

### Any current solutions using Ligato?

[Contiv-VPP](https://contivpp.io/) and [Network Service Mesh (NSM)](https://networkservicemesh.io/) use Ligato.

---

## Details

### What is a plugin?

Small chunk of code performing a specific function. Ligato comes with multiple out-of-the-box [plugins](../plugins/plugin-overview.md). You can develop new plugins that perform customized functions, and you need only use the plugins required for your applications. 

---

### How does a Ligato-built CNF talk to external apps?

Ligato provides plugins, and you can craft your own, for communicating with external applications such as etcd data stores or gRPC clients.

---

### What is a Key Value (KV) Data Store?

Data store (or database) of key-value (KV) pairs. Ligato supports the [commonly used/deployed KV data store solutions](../user-guide/concepts.md#key-value-data-store-overview). You can create plugins to communicate with other external KV data stores.

---

### What programming language does Ligato use?

[Go Programming Language](https://golang.org/) developed by Google. Other cloud native projects including Kubernetes and etcd use Go. Advantages over other programming languages used in distributed systems include speed,concurrency and simplicity. 

---

### What is protobufs?

[Protocol Buffers](https://developers.google.com/protocol-buffers) is a language/platform-neutral method that defines the structure and serialized format of data associated with an object.

---

### What is a resync?

A VPP agent process that maintains consistency between configuration data provided from an external source, internal VPP agent state, and the VPP runtime.

---

### What is a model?

A model defines a configuration object. See [Model Specification](../user-guide/concepts.md#model-specification) and [proto.Message](../user-guide/concepts.md#protomessage) for more details.

---

### What is the microservice label?

A unique value assigned to a group of VPP agents watching the KV data store for configuration changes. For more details, see [keys and microservice label](../user-guide/concepts.md#keys-and-microservice-label).

---

### What does northbound (NB) mean? Same for southbound (SB)?

NB means configuration data originates from an external source such as a KV data store. SB means configuration data present in the VPP data plane.

---

## Using Ligato

### How do I get started?

Start with the [Quickstart Guide](../user-guide/quickstart.md) to quickly build and program a VPP-based CNF.

---

### Any tutorials I can follow?

See [tutorials](../tutorials/00_tutorial-setup.md) beginning with coding up a "Hello World" plugin.

---

### Any examples I can look at?

See [examples](../user-guide/examples.md). 

---

### Is there a CLI?

[Agentctl](../user-guide/agentctl.md) supplies you with numerous CLI commands for working with Ligato.

---

### How do I find the Ligato and VPP versions I am working with?

To show VPP agent info, use [agentctl status](../user-guide/agentctl.md#status) or [GET /dump/info/version REST API](../api/api-vpp-agent.md#version).

To show VPP version info, use [agentctl VPP](../user-guide/agentctl.md#vpp) or [POST /vpp/command REST API](../api/api-vpp-agent.md#vpp-cli-command).


---

### How do I configure a Ligato-based application?

Ligato employs a stateless configuration approach. VPP agents "listen" for configuration updates stored as key-value pairs in a KV data store. The [quickstart guide](../user-guide/quickstart.md#51-etcdctl) shows an example of configuration data put to an etcd data store.  

You can develop applications that use other configuration techniques including kubectl, clientv2, gRPC, REST, agentctl, and customized methods best suited for the application. 

---

### What is a conf file?

A file containing [plugin configuration values](../user-guide/config-files.md) applied when you initialize the plugin. Most, but not all plugins have their own conf file. See [Conf files](../user-guide/config-files.md) for more details and examples.

---

## KV Scheduler

### What is the KV Scheduler?

The KV Scheduler plugin supports dependency resolution, and computes the proper programming sequence for multiple interdependent configuration items. It works with VPP and Linux agents on the southbound (SB) side, and external data stores and rpc clients on the northbound (NB) side.

To learn more, see [KV Scheduler](../plugins/kvs-plugin.md).

---

### What is a Descriptor?

A construct used by the KV Scheduler that describes a particular configuration object such as a VPP route. Descriptors define dependencies and supported CRUD operations to perform against that object. To learn more, see [KV Descriptors](../developer-guide/kvdescriptor.md).

---

### What is a KV Scheduler Transaction?

A series of configuration actions planned and executed by the KV Scheduler. You can view transaction plans with the following:
 
- [KV scheduler transaction log](../developer-guide/kvs-troubleshooting.md#understanding-the-kv-scheduler-transaction-log)
- [GET /scheduler/txn-history](../api/api-kvs.md#transaction-history)
- [agentctl report](../user-guide/agentctl.md#report)

---

### KV Scheduler Troubleshooting

See [KV Scheduler Troubleshooting](../developer-guide/kvs-troubleshooting.md).

---

## APIs

### What RPCs are supported by Ligato?

gRPC and REST.

---

### What REST APIs are supported?

See [VPP Agent REST API](../api/api-vpp-agent.md) and [KV Scheduler REST API](../api/api-kvs.md).

--- 

### Can I develop my own set of customized APIs?

The [REST Handler tutorial](../tutorials/03_rest-handler.md) shows you how to create a REST API for the Hello World plugin.

---

## Where can I find support for Ligato?

Ligato is open source software. Use the [Ligato slack room](https://ligato.slack.com/) or Github issues to communicate with the greater Ligato community. 

Github Issues URLs:

* [for cn-infra](https://github.com/ligato/cn-infra/issues)
* [for vpp-agent](https://github.com/ligato/vpp-agent/issues)
* [for Ligato Docs](https://github.com/ligato/docs/issues)
* [for Ligato.io](https://github.com/ligato/site)

Before opening an issue, collect as much data as possible, including:

* VPP Agent version/commit ID
* VPP version/commit ID
* VPP Agent logs (best with debug enabled)
* KVDB store dump if possible
* Description of desired function

This will help the community to quickly pinpoint and fix the problem.

