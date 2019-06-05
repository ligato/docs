# Connection plugins

---

## REST plugin

The "builtin" REST support (plugin) in the VPP-Agent is currently limited to retrieving existing VPP configuration (called dumping) for core plugins. The Agent also provides simple html template (usable in browser) and optional support for https security, authentication and authorization.

This article will often refer to two HTTP plugins which must be distinguished to understand all concepts:

- **The CN-Infra REST (HTTP) plugin** which enables general HTTP functionality and security
- **The VPP-Agent REST plugin** which is the Agent-specific implementation of the CN-Infra REST plugin  

### Basics

The VPP-Agent contains the REST API plugin, which is based on CN-Infra HTTP plugin (HTTPMux). The basic functionality is allowed by default without need of any external configuration file, just add the VPP-Agent REST plugin to the Agent plugin pool. The default HTTP endpoint is opened on socket `0.0.0.0:9191`. There are several ways how to setup different port:

**1. Using VPP-Agent flag:** the port can be set via flag `-http-port=<port>`
**2. Using environment variable:** set the variable `HTTP_PORT` to desired value
**3. Using the CN-Infra HTTP plugin config file:** this option allows to change the whole endpoint and also enable other features described in the part HTTP config file

### Supported URLs

There is a list of all supported URLs sorted by VPP-Agent plugins. If the retrieve URL is used (currently the only supported), the output is based on proto model structure for given data type together with VPP-specific data which are not a part of the model (like indexes for interfaces or ACLs, various internal names, etc.). Those data are in separate section labeled as `<type>_meta`.

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

The REST plugin exposes configured VPP interfaces, which can be shown all together, or interfaces of specific type only:
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

The REST plugin exposes configured Linux interfaces. All configured interfaces are retrieved all together with interfaces in the default namespace: 
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

ARPs, proxy ARP interfaces/ranges, static routes and IP scan neighbor config exposed via REST:
```
# Routes
/dump/vpp/v2/routes

# ARPs
/dump/vpp/v2/arps

# Proxy ARP interfaces
dump/vpp/v2/proxyarp/interfaces

# Proxy ARP ranges
/dump/vpp/v2/proxyarp/ranges

# IP scan neighbor
/dump/vpp/v2/ipscanneigh
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

**IPSec plugin**

The REST plugin allows to dump IPSec security policy databases and security associations:
```
# REST path of the SPD
/dump/vpp/v2/ipsec/spds

# REST path of the SA
/dump/vpp/v2/ipsec/sas
```

**Punt plugins**

The REST plugin allows to dump registered punt sockets:
```
# REST path of the punt socket register
/dump/vpp/v2/punt/sockets
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

## Security

REST plugin allows to optionally configure following security features:
- server certificate (HTTPS)
- Basic HTTP Authentication - username & password
- client certificates
- token based authorization

All of them are disabled by default and can be enabled by config file:

```yaml
endpoint: 127.0.0.1:9292
server-cert-file: server.crt
server-key-file: server.key
client-cert-files:
  - "ca.crt"
client-basic-auth:
  - "user:pass"
  - "foo:bar"
```

If `server-cert-file` and `server-key-file` are defined the server requires HTTPS instead of HTTP for all its endpoints.

`client-cert-files` the list of the root certificate authorities that server uses to validate client certificates. If the list is not empty only client who provide a valid certificate is allowed to access the server.

`client-basic-auth` allows to define user password pairs that are allowed to access the server. The config option defines a static list of allowed user. If the list is not empty default staticAuthenticator is instantiated. Alternatively, you can implement custom authenticator and inject it into the plugin (e.g.: if you want to read credentials from ETCD).


***Example***

In order to generated self-signed certificates you can use the following commands:

```bash
#generate key for "Our Certificate Authority"
openssl genrsa -out ca.key 2048

#generate certificate for CA
openssl req -new -nodes -x509 -key ca.key -out ca.crt  -subj '/CN=CA'

#generate certificate for the server assume that server will be accessed by 127.0.0.1
openssl genrsa -out server.key 2048
openssl req -new -key server.key -out server.csr -subj '/CN=127.0.0.1'
openssl x509 -req -extensions client_server_ssl -extfile openssl_ext.conf -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt

#generate client certificate
openssl genrsa -out client.key 2048
openssl req -new -key client.key -out client.csr -subj '/CN=client'
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 360

```

Once the security features are enabled, the endpoint can be accessed by the following commands:

- **HTTPS**
where `ca.pem` is a certificate authority where server certificate should be validated (in case of self-signed certificates)
  ```
  curl --cacert ca.crt  https://127.0.0.1:9292/log/list
  ```

- **HTTPS + client cert** where `client.crt` is a valid client certificate.
  ```
  curl --cacert ca.crt  --cert client.crt --key client.key  https://127.0.0.1:9292/log/list
  ```

- **HTTPS + basic auth** where `user:pass` is a valid username password pair.
  ```
  curl --cacert ca.crt  -u user:pass  https://127.0.0.1:9292/log/list
  ```

- **HTTPS + client cert + basic auth**
  ```
  curl --cacert ca.crt  --cert client.crt --key client.key -u user:pass  https://127.0.0.1:9292/log/list
  ```
  
### Token-based authorization

REST plugin supports authorization based on tokens. To enable the feature, use `http.conf` file:

```
enable-token-auth: true
```

Authorization restricts access to every registered permission group URLs. The user receives token after login, which grants him access to all permitted sites. The token is valid until the user is logged in, or until it expires.

The expiration time is a token claim, set in the config file:

```
token-expiration: 600000000000  
```

Note that time is in nanoseconds. If no time is provided, the default value of 1 hour is set.

Token uses by default pre-defined signature string as the key to sign it. This can be also changed via config file:

```
token-signature: <string>
```

After login, the token is required in authentication header in the format `Bearer <token>`, so it can be validated. If REST interface is accessed via a browser, the token is written to cookie file.

### Users and permission groups

Users have to be pre-defined in `http.conf` file. User definition consists of name, hashed password and permission groups.

**Name** defines username (login). Name "admin" is forbidden since the admin user is created automatically with full permissions and password "ligato123"

**Password** has to be hashed. It is possible to use [password-hasher utility][password-hasher] to help with it. Password also has to be hashed with the same cost value, as defined in the configuration file:

```
password-hash-cost: <number>
```

Minimal hash cost is 4, maximal value is 31. The higher the cost, the more CPU time memory is required to hash the password.

**Permission Groups** define a list of permissions; allowed URLs and methods. Every user needs at least one permission group defined, otherwise it will not have access to anything. Permission group is described in [proto model][access-security-model]. 

To add permission group, use rest plugin API:

```
RegisterPermissionGroup(group ...*access.PermissionGroup)
```

Every permission group has a name and a list o permissions. Permission defines URL and a list of methods which may be performed. 

To add permission group to the user, put its name to the config file under user's field `permissions`. 

### Login and logout

To log in a user, follow the URL `http://localhost:9191/login`. The site is enabled for two methods. It is possible to use a `POST` to directly provide credentials in the format:

```
{
	"username": "<name>",
	"password": "<pass>"
}
```

The site returns access token as plain text. If URL is accessed with `GET`, it shows the login page where the credentials are supposed to be put. After successful submit, it redirects to the index.

To log out, post username to `http://localhost:9191/logout`.

```
{
	"username": "<name>"
}
```

### Basic usage

**1. cURL** 

Specify the VPP-Agent target HTTP IP address and port with link to desired data. All URLs are accessible via the `GET` method.

Example:
```
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces
```

**2. Postman**

Choose the `GET` method, provide desired URL and send the request.

## VPP-Agent GRPC

Related articles: 

* [GRPC client tutorial][grpc-client-tutorial] shows how to create a client for the off-the-shelf VPP Agent's GRPC Server
* [GRPC server tutorial][grpc-server-tutorial] shows how to create your own GRPC Server using the [CN-Infra GRPC Plugin][grpc-plugin].

GRPC support in the VPP-Agent is provided by the [CN-Infra GRPC plugin][grpc-plugin] that implements handling of GRPC requests.

The VPP-Agent GRPC plugin can be used to:

* Send configuration to VPP
* Retrieve (dump) configuration from VPP
* Start a notification watcher

The following remote procedure calls are defined:

* **Get** creates new configuration or updates existing configuration.
* **Delete** removes specified (existing) configuration.
* **Dump** (Retrieve) reads existing configuration from the VPP.
* **Notify** subscribes GRPC to the notification service

To enable the GRPC server within the Agent, the GRPC plugin has to be added to the plugin pool and loaded (currently the GRPC plugin is a [part of the Configurator plugin dependencies][configurator-grpc]). The plugin also requires a startup configuration file (see [CN-Infra GRPC plugin][grpc-plugin]), where the endpoint is defined.

Clients with the GRPC Server via an endpoint IP address and port or via a unix domain socket file. The TCP network is set as default, but other network types are also available (like TCP6 or UNIX).

### GRPC Plugin

The `GRPC Plugin` is a infrastructure Plugin which allows app plugins to handle GRPC requests (see the diagram below) as follows:

1. The GRPC Plugin starts the GRPC server + net listener in its own goroutine
2. Plugins register their handlers with the `GRPC Plugin`. To service GRPC requests, a plugin must first implement a handler function and register it at a given URL path using the `RegisterService` method. `GRPC Plugin` uses an GRPC request multiplexer from `grpc/Server`.
3. The GRPC Server routes GRPC requests to their respective registered handlers using the `grpc/Server`.

![grpc][grpc-image]

**Configuration**

- The GRPC Server's port can be defined using the commandline flag `grpc-port` or via the environment variable GRPC_PORT.

**Example**

The [grpc-server greeter example]*(../../examples/grpc-plugin/grpc-server) demonstrates the usage of the `GRPC Plugin` plugin API GetServer():
```
// Register our GRPC request handler/service using generated RegisterGreeterServer:
RegisterGreeterServer(plugin.GRPC.Server(), &GreeterService{})
```
Once the handler is registered with the `GRPC Plugin` and the agent is running, you can use a grpc client to call the service .

[access-security-model]: https://github.com/ligato/cn-infra/blob/master/rpc/rest/security/model/access-security/accesssecurity.proto
[configurator-grpc]: https://github.com/ligato/vpp-agent/blob/master/plugins/configurator/plugin.go#L54
[grpc-client-tutorial]: https://github.com/ligato/cn-infra/tree/master/examples/grpc-plugin/grpc-client
[grpc-server-tutorial]: https://github.com/ligato/cn-infra/tree/master/examples/grpc-plugin/grpc-server
[grpc-image]: ../img/user-guide/grpc.png
[grpc-plugin]: https://github.com/ligato/cn-infra/tree/master/rpc/grpc
[grpc-tutorial]: ../tutorials/08_grpc-tutorial.md
[password-hasher]: https://github.com/ligato/cn-infra/blob/master/rpc/rest/security/password-hasher/README.md

*[ACL]: Access Control List
*[ARP]: Address Resolution Protocol
*[CLI]: Command-Line Interface
*[FIB]: Forwarding Information Base
*[HTTP]: Hypertext Transfer Protocol
*[HTTPS]: Hypertext Transfer Protocol Secure
*[NAT]: Network Address Translation
*[REST]: Representational State Transfer
*[URL]: Uniform Resource Locator
