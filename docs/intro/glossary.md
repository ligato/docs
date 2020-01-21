# Glossary

## CNF

[Cloud Native Network Function](https://github.com/lf-edge/glossary/blob/master/edge-glossary.md#cloud-native-network-function-cnf). Containerized network function (e.g. IPsec) subscribing to cloud native architecture and operational principles (e.g. Kubernetes lifecycle).

## VNF

General term describing a network function formally running on a dedicated hardware device that runs now as a virtual network or service appliance in virtualized network. Some documentation will implicitly or explicitly define a VNF as a VM-based network function.

## Memif

Shared memory interface between two instances of VPP

## Tap

Linux kernel interface allowing user space networking apps to send/receive raw ethernet or IP packets.

## Kubernetes

[Orchestration and scheduling platform](https://kubernetes.io/) for cloud native environments

## vSwitch

Virtual networking component for switching packets between virtual machines or containers.

## VPP

[Vector Packet Processing](https://wiki.fd.io/view/VPP/What_is_VPP%3F), a software platform for forwarding packets

## KV Data Store

Data store (or database) of key-value (KV) pairs or more generally structured data objects

## etcd

[etcd](https://etcd.io/) is an open source distributed KV data store

## Redis

[Redis](https://redis.io/) is an open source data store and database

## Ligato

[Ligato](https://ligato.io/) is an open source framework consisting of a vpp agent and suite of plugins for building and developing CNFs

## FD.io

[Open source project for VPP](https://fd.io/)

## Contiv - VPP

[Kubernetes CNI plugin using FD.io/VPP as the dataplane](https://contivpp.io/)

## Cloud Native

Architecture pattern based on microservices, containers and Kubernetes for building cloud-based applications

## Protobufs

[Protocol Buffers](https://developers.google.com/protocol-buffers) is a language/platform-nuetral method that defines the structure and serialized format of the data associated with the object. Each object supported in Ligato (e.g. interfaces) has a `.proto` definition.

## Go

[Programming Language](https://golang.org/) developed by Google. Used by Ligato and other cloud-native projects including Kubernetes and etcd. Advantages over other languages (i.e. python, java) employed in distributed systems are speed (it is a compiled languagee), concurrency and simple.

## Plugin

Small chunk of code performing a specific function. Ligato (vpp-agent and cn-infra) come with multiple out-of-the-box [plugins](../plugins/plugin-overview.md). New plugins can be developed performing additional functions. Developers need only use the plugins required for their application

## KVScheduler

Handles configuration VPP/Linux plugin dependency resolution thus achieving a working sequence of configuration transactions.

## Connectors

Another term for plugins (e.g. etcd plugin) that communicate with an external application (e.g. etcd server)

## Descriptors

Describes object data (dependencies, metadata, etc.) enabling the KVscheduler to perform CRUD callbacks against such objects

## Model

Each object is defined by a model consisting of a [module, version and type fields](../user-guide/concepts.md). Models for specific objects are defined in the `models.go` files located in the /proto folder




*[CLI]: Command-Line Interface
*[NFV]: Network Functions Virtualization
*[REST]: Representational State Transfer
*[SDN]: Software-Defined Networking
*[SNMP]: Simple Network Management Protocol
*[VNF]: Virtual Network Function
*[VPP]: Vector Packet Processing
