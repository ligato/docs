# Overview

_**Ligato is a Golang (Go) framework for developing software agents to control and manage cloud native network functions (CNF).**_

---

## CNF

Cloud providers are shifting towards a cloud native approach for application development, operations and management. Applications are packaged up in containers. Kubernetes automates container startup, placement and teardown based on policies, services, scale and resiliency. The cloud native lifecycle simplifies the steps beginning with development all the way to deployment and operations in modern cloud environments. 

Cloud native networks (CNF) run inside contains and inherit all behaviors and benefits afforded to cloud native applications. Indeed, you can think of CNFs as just another application container. 

By applying the Kubernetes lifecycle to CNFs, you can spin up different network configurations to meet your respective needs. Several examples follow:

- Webscale services running on multiple application pods interconnected through a CNF virtual switch network overlay.
- Cloud service bundles composed of CNFs strung together in a service function chain.
- VPN Internet Gateway solutions employing IPsec crypto/tunnel CNFs. 

CNFs require two more very important attributes:

- High performance data plane.
- Programmability using cloud native APIs including k8s CRDs, gRPCs, etcd data stores

CNF solutions must keep pace with cloud native adoption and growth. The cloud native developer community requires solutions that simplify and expedite the development and implementation of customized, scalable and versatile CNFs. 

That's where Ligato comes into play.

To learn more about CNFs, check out the following:

- [What is a CNF?](https://ligato.io/cnf/cnf-def/)
- [X-Factor CNFs](https://x.cnf.dev/config/)
- [CNFs with a Dose of Ligato and FD.io/VPP](https://ligato.io/blog/cnf-ligato-fdio/)
- 

---

## 10K foot View

_**Ligato is a Golang (Go) framework for developing software agents to control and manage cloud native network functions (CNF).**_

Notes

- CNFs with different personalities will exist in cloud native networks. Some replicate existing NFs/VNF but under cloud-native control. Others new developed to support emerging cloud-native functions 
- Ligato provides you, the developer with the tools, libraries, protobufs, plugins and docs so you can build and deploy CNF control plane and management agents
- focuses on the control plane agents, while leveraging fd.io data plane. Caters to VPP, but Linux supported
- Does not do all on its own. Embraces integration with other open source projects.


![overview][docs-overview]


## VPP Agent

_** Ligato provides a VPP agent for programming a VPP data plane**_

* Supplies VPP-specific plugins
* VPP agent and VPP packaged up in single container
* Dependency handling between related configuration items
* Transaction-based configuration processing and tracking
* Failover synchronization mechanisms, also known as resync
* Stateless configuration management based on a KV data store "watch" paradigm
* Direct access via REST or gRPC
* Component health checks
* Multi-VPP configuration support
* Multi-version configuration support

!!! Note
    The VPP agent is a control/management plane. It provides a configuration and monitoring services for the VPP data plane. It does not decide what to do with packets arriving on any of the VPP interfaces and does not change any of the configuration parameters (routes, FIBs) based on the actual VPP state.
 

---

    
## Cn-infra

_**Ligato provides the infrastructure and agents developers can use to build customized CNF solutions**_

* Supplies an assortment of reusable plugins allowing developers to "cherry pick" and implement as needed
* Plugins for VPP, Linux, etcd, Kafka, Redis, Cassandra, gRPC, REST and more
* Model-driven architecture for configuration object (i.e. interface) abstraction
* Northbound APIs based on protobufs
* Connectors for interfacing with external databases
* Examples and tutorials for building customized plugins


## Developer Resources

- VPP agent and VPP dataplane in one container
- Golang programming language
- Model-driven protobuf APIs
- Common programming pattern for agents
- Out-of-the-box VPP and Linux data plane plugins
- Automatic plugin lookup and dependency injection
- KV scheduler core plugin for handling plugin dependencies and programming sequence
- Common KV descriptor API
- agentctl
- developer guide
- Troubleshooting guide
- tutorials, examples
- Detailed logging
- Stateless configuration management based on KV data store "watch" paradigm
- Godocs

[docs-overview]: ../img/intro/docs-overview-ligato.svg









  

