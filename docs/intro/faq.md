# Frequently Asked Questions

---

## General

### What is the VPP agent?

A [control plane agent for a VPP data plane](framework.md#VPP agent) that resides inside a single container. It comes with various plugins (e.g. ACL) for programming specific data plane functions. It is an entity completely separate from VPP. Customized or out-of-the-box VPP-based CNFs can incorporate the VPP agent.

Note that the VPP agent can be used with the Linux data plane. It is not explicitly limited to the VPP data plane. 

### What is cn-infra?

[Framework of libraries and plugins](framework.md#infra) for building control plane agents for CNFs and applications.


### Is Ligato an application?

No. Select Ligato components (i.e. VPP agent, prometheus plugin, etcd connectors, etc.) are used as building blocks for CNF and/or cloud native application implementations.


### How are the VPP agent and cn-infra related?

They compliment each other. Both provide plugins for developers to use in implementing CNFs. For example, a VPP-based CNF will use VPP agent plugins for data plane programmability and cn-infra plugins (e.g. etcd connector, REST, gRPC, etc.) for communications with external applications.

### What is a CNF?

[Cloud Native Network Function](https://ligato.io/cnf/cnf-def/). Containerized network functions (e.g. IPsec) subscribing to cloud native architectural and operational principles (e.g. Kubernetes lifecycle).

### What is VPP?

[Vector Packet Processing](https://wiki.fd.io/view/VPP/What_is_VPP%3F) is a software platform for forwarding packets. [FD.io](https://fd.io/) is the open source project for VPP.

### Is VPP part of Ligato?

No. Ligato includes the VPP agent along with other plugins providing overall management, control and lifecycle support for VPP-based CNFs and cloud-native applications. 

However, to ensure VPP agent and VPP compatibility, and maintain its lightweight "footprint", Ligato/vpp-agent docker images include the VPP data plane.

### Can Ligato be used to build non-VPP cloud native apps?

Yes. One example is an [experimental BGP agent](https://github.com/ligato/bgp-agent) that does not use VPP.  


### Is Ligato a CNCF project?

No. However it is a member of the [larger CNCF ecosystem](https://landscape.cncf.io/). Many of its plugins are based on CNCF projects (e.g. gRPC, etcd, Prometheus).

### Are their solutions using Ligato?

Yes. [Contiv-VPP](https://contivpp.io/) and [Network Service Mesh (NSM)](https://networkservicemesh.io/) use Ligato components.

---

## Details

### What is a plugin?

Small chunk of code performing a specific function. Ligato (VPP agent and cn-infra) come with multiple out-of-the-box [plugins](../plugins/plugin-overview.md). New plugins can be developed performing additional functions. Developers need only use the plugins required for their application.

### How does a Ligato-built CNF talk to external apps?

Ligato provides plugins, and developers can craft their own for communicating with external applications.

### What is a Key Value (KV) Data Store?

Data store (or database) of key-value (KV) pairs. Ligato supports the [commonly used/deployed KV data store solutions](../user-guide/concepts.md#key-value-data-store-overview). Plugins can be created to interface to other external KV data stores.

### What programming language does Ligato use?

[Go Programming Language](https://golang.org/) developed by Google. Other cloud native projects including Kubernetes and etcd use Go. Advantages over other languages (i.e. python, java) used in distributed systems include speed (it is a compiled language), concurrency and simplicity. External applications interfacing to a Ligato application can use any programming language.

### What is protobufs?

[Protocol Buffers](https://developers.google.com/protocol-buffers) is a language/platform-neutral method that defines the structure and serialized format of the data associated with an object. [Here](https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/l3/route.proto) is an example of a VPP route in protobuf (.proto) format.

### What is a resync?

A VPP agent process that maintains consistency between configuration data provided from an external source, internal VPP agent state and the actual VPP data plane state.

### What is a model?

Each object is defined by a model consisting of a [module, version and type fields](../user-guide/concepts.md).

### What is the microservice label?

A unique value assigned to a group of VPP agents watching the KV data store for config changes with a common key prefix. This is useful if you have CNFs with different config data contained in a KV data store. More details and an example are shown [here](../user-guide/concepts.md#keys-and-microservice-label).

### What does northbound (NB) mean? Same for southbound (SB)?
NB means configuration data originates from an external source such as a KV data store. SB means configuration data present in the VPP data plane.

---

## Using Ligato

### How do I get started?

Start with the [Quickstart Guide](../user-guide/quickstart.md) to quickly build and program a VPP-based CNF.

### Are there any tutorials I could follow?

Yes. See [tutorials](../tutorials/00_tutorial-setup.md) beginning with coding up a "Hello World" plugin.

### Are there any example apps I could look at?

Yes. See the [examples](../user-guide/examples.md) section

### Is there a CLI?

Yes. [Agentctl](../user-guide/agentctl.md) supplies the user with numerous CLI commands for working with Ligato.

### Can I use other CLIs with Agentctl?

Absolutely. Kubernetes [Kubectl](https://kubernetes.io/docs/reference/kubectl/overview/) and Contiv-VPP [contiv-netctl](https://contivpp.io/blog/using-conti-vpp-netctl-blog/) are examples associated with complete Kubnetes cluster environments. They compliment agentctl.

Interacting with an etcd data and VPP data plane can be accomplished using etcdctl and vppctl respectively. The quickstart guide contains samples for [etcdctl](../user-guide/quickstart.md#32-etcdctl) and [vppctl](../user-guide/quickstart.md#53-vpp-cli). The VPP CLI is supported by agentctl, REST or by attaching to VPP is explained [here](../user-guide/vpp-cli.md).

### How do I determine what Ligato version I am working with? Same for VPP.

[AgentCtl](../user-guide/agentctl.md) provides commands to display VPP agent and VPP version information. [Agentctl status](../user-guide/agentctl.md#status) shows VPP agent info and [agentctl VPP](../user-guide/agentctl.md#vpp) shows VPP info. 


### How is a Ligato-based application configured?

Ligato employs a stateless configuration approach. VPP agents "listen" for config updates stored as key-value pairs in a KV data store. There is an example from the [quickstart guide](../user-guide/quickstart.md#51-etcdctl) showing configuration data in etcd pushed to the VPP agent. 

Ligato applications can be developed to use a myriad of other configuration techniques including kubectl, clientv2, gRPC, REST, agentctl, and customized methods best suited for the application. [Here](../plugins/connection-plugins.md#vpp-agent-grpc) is an example using gRPC to convey configuration data to a VPP agent.

### What is a .conf file

Contain [plugin configuration values](../user-guide/config-files.md). Most but not all plugins have their own .conf file.

---

## KVScheduler

### What is the KV Scheduler?

Provides transaction-based configuration processing employing a generic (abstraction) mechanism for dependency resolution between different configuration items. 

For example, the KV scheduler sorts out and resolves dependencies where an interface must be configured first before proceeding with a bridge-domain. More details on the KV scheduler can be found [here](../plugins/kvs-plugin.md).

### What is a Descriptor?

A construct used by the KV Scheduler that describes a particular configuration object (e.g. VPP route). Among other values, descriptors define dependencies and supported create, read, update and delete (CRUD) operations to be performed against that object. More on descriptors [here](../developer-guide/kvdescriptor.md).

### What is a KV Scheduler Transaction?

This is the series of configuration actions as determine by the KV scheduler. There is a [KV scheduler transaction log](../developer-guide/kvs-troubleshooting.md#understanding-the-kv-scheduler-transaction-log) function and [REST APIs](../plugins/kvs-plugin.md#rest-api) illustrating the details of a transaction.


### KV Scheduler Troubleshooting

Troubleshooting KV scheduler operations including several tools to assist in this process can be found [here](../developer-guide/kvs-troubleshooting.md).

---

## APIs

### What RPCs are supported by Ligato?

gRPC and REST.

### What REST APIs are supported?

VPP REST APIs are documented [here](../plugins/../plugins/connection-plugins.md#supported-urls). There are also REST APIs for determining the [descriptors registered with the KVS](../developer-guide/kvs-troubleshooting.md#how-to-list-registered-descriptors-and-watched-key-prefixes) and for [retrieving data for visualizing the KVS graph](../developer-guide/kvs-troubleshooting.md#how-to-visualize-the-graph)

### Can I develop my own set of customized APIs?

Yes. The [REST Handler tutorial](../tutorials/03_rest-handler.md) shows you how to create a REST API for the Hello World plugin.

---

## Where can I find support for Ligato?

Ligato is open source software so support, issues, questions, bugs, fixes, commits, PRs and so on should be communicated and shared with the [community.](https://ligato.io/community/page/)

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

Further discussions with the Ligato community can take place in the [Ligato Slack Room](https://ligato.slack.com/)
