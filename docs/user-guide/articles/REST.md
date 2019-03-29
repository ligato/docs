This page describes how to use the VPP-Agent with representational state transfer

If you look for the tutorial how to create custom HTTP plugin, refer to CN-Infra REST wiki and tutorial //TODO add links

# VPP-Agent REST

The "builtin" REST support (plugin) in the VPP-Agent is currently limited to retrieving existing VPP configuration (called dumping) for core plugins. The Agent also provides simple html template (usable in browser) and optional support for https security, authentication and authorization.

This article will often refer to two HTTP plugins which must be distinguished to understand all concepts:
- **The CN-Infra REST (HTTP) plugin** which enables general HTTP functionality and security
- **The VPP-Agent REST plugin** which is the Agent-specific implementation of the CN-Infra REST plugin  

Content:

- [Basics](#basics)
  - [Supported URLs](#urls)
  - [Logging mechanism](#logging)
- [Security](#security)  
- [Basic usage](#usage)

# <a name="basics">Basics</a>

The VPP-Agent contains the REST API plugin, which is based on CN-Infra HTTP plugin (HTTPMux). The basic functionality is allowed by default without need of any external configuration file, just add the VPP-Agent REST plugin to the Agent plugin pool. The default HTTP endpoint is opened on socket `0.0.0.0:9191`. There are several ways how to setup different port:

**1. Using VPP-Agent flag:** the port can be set via flag `-http-port=<port>`
**2. Using environment variable:** set the variable `HTTP_PORT` to desired value
**3. Using the CN-Infra HTTP plugin config file:** this option allows to change the whole endpoint and also enable other features described in the part HTTP config file

#### <a name="urls">Supported URLs</a>

There is a list of all supported URLs sorted by VPP-Agent plugins. If the retrieve URL is used (currently the only supported), the output is based on proto model structure for given data type together with VPP-specific data which are not a part of the model (like indexes for
interfaces or ACLs, various internal names, etc.). Those data are in separate section labeled as `<type>_meta`.

**Index page**

The REST to get the index page. Configuration items are sorted by type (interface plugin, telemetry, etc.). The index is a root directory.
```
/
```

**Access lists**

URLs to obtain ACL IP/MACIP configuration:
```
# ACL IP
/dump/vpp/v2/acl/ip

# ACL MAC IP
/dump/vpp/v2/acl/macip
```

**VPP Interfaces**

The REST plugin exposes configured VPP interfaces, which can be shown all together, or interfaces
of specific type only:
```
# All interfaces
/dump/vpp/v2/interfaces

# Loopback
/dump/vpp/v2/interfaces/loopback

# Ethernet
/dump/vpp/v2/interfaces/ethernet

# Memory interface
/dump/vpp/v2/interfaces/memif

# Tap
/dump/vpp/v2/interfaces/tap

# VxLAN tunnel
/dump/vpp/v2/interfaces/vxlan

# Af-Packet interface
/dump/vpp/v2/interfaces/afpacket
```

**Linux Interfaces**

The REST plugin exposes configured Linux interfaces. All configured interfaces are retrieved all together
with interfaces in the default namespace: 
```
/dump/linux/v2/interfaces
```

**L2 plugin**

The support for bridge domains, FIB entries and cross connects:
```
# Bridge domains
/dump/vpp/v2/bd

# FIB entries
/dump/vpp/v2/fib

# Cross-connects
/dump/vpp/v2/xc
```

**L3 plugin**

ARPs, proxy ARP interfaces/ranges and static routes exposed via REST:
```
# Routes
/dump/vpp/v2/routes

# ARPs
/dump/vpp/v2/arps

# Proxy ARP interfaces
dump/vpp/v2/proxyarp/interfaces

# Proxy ARP ranges
/dump/vpp/v2/proxyarp/ranges
```

**Linux L3 plugin**

The Linux ARPs and routes exposed via REST:
```
# Linux routes
/dump/linux/v2/routes

# Linux ARPs
/dump/linux/v2/arps
```

**NAT plugin**

The REST plugin allows to dump NAT44 global configuration, DNAT configuration or both of them together:
```
# REST path of a NAT
/dump/vpp/v2/nat

# Global NAT config
/dump/vpp/v2/nat/global

# DNAT configurations
/dump/vpp/v2/nat/dnat
```

**CLI command**

Allows to use VPP CLI command via REST. Commands are defined as a map as following:
```
/vpp/command -d '{"vppclicommand":"<command>"}'
```


**Telemetry**

The REST allows to get various types of telemetry metrics data, or selective using specific key:
```
/vpp/telemetry
/vpp/telemetry/memory
/vpp/telemetry/runtime
/vpp/telemetry/nodecount
```

**Tracer**

The Tracer plugin exposes data via the REST as follows:
```
/vpp/binapitrace
```

#### <a name="logging">Logging mechanism</a>

The REST API request is logged to stdout. The log contains VPP CLI command and VPP CLI response. It is searchable in elastic search using "VPPCLI".

# <a name="httpcf">Security</a>

The CN-Infra REST plugin provides option to secure the HTTP communication. The plugin supports HTTPS client/server certificates, HTTP credentials authentication (username and password) and authorization based on tokens.

This feature is disabled by default and if required, must be enabled by the CN-Infra HTTP plugin config file. 

More information about security setup and usage, see [security](https://github.com/ligato/cn-infra/blob/master/rpc/rest/README.md#security) for certificates and [tokens](https://github.com/ligato/cn-infra/blob/master/rpc/rest/README.md#token-based-authorization) for token-based authorization.

# <a name="usage">Basic usage</a>

**1. cURL** 

Specify the VPP-Agent target HTTP IP address and port with link to desired data. All URLs are accessible via the `GET` method.

Example:
```
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces
```

**2. Postman**

Choose the `GET` method, provide desired URL and send the request.
