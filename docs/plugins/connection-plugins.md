# Connection Plugins

The Ligato framework defines connection plugins as those that can communicate with internal or external applications and components using REST and gRPC APIs.

---

## REST Plugin

The VPP agent REST plugin supports the retrieval of VPP configuration information. This is referred to as `dumping`. REST cannot be used to add, modify or delete configuration data. 

The plugin also provides a general purpose HTTP server that can be used by plugins to expose a REST API to external clients. 

The [REST handler tutorial][rest-handler-tutorial] illustrates how to implement a REST API for a customized plugin.

**References**

- [Ligato cn-infra REST folder][cn-infra-rest-repo-folder]
- [REST plugin folder][vpp-agent-rest-plugin]
- [REST conf file][rest-conf-file]
- [REST handler tutorial][rest-handler-tutorial]

---

### Configuration

REST plugin functionality is enabled without the need for any external configuration file. The default HTTP server endpoint is opened on port `0.0.0.0:9191`. 

There are several options for configuring a different port number:
 
- VPP agent flag:
```
 -http-port=<port>`
```

- Set the `HTTP_PORT` env variable to a desired value
- Set the endpoint field in the [REST conf file][rest-conf-file]:
```json
endpoint: 0.0.0.0:9191
```

---

### Usage

**cURL** 

Specify the VPP agent target HTTP IP address:port with a link to the desired data. All URLs are accessible via the `GET` method.

Example:
```
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces
```

---

### Supported URLs

The REST plugin enables a dump of configuration item dumps sorted by types such as interfaces, L3 routes, telemetry, and bridge domains.
 

**Index Page**

Index of supported REST API URLs:
```
curl -X GET http://localhost:9191/
```
---

**Access lists**

```
# ACL IP
/dump/vpp/v2/acl/ip

# ACL MAC IP
/dump/vpp/v2/acl/macip
```
---

**VPP Interfaces**

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

---

**Linux Interfaces**

```
/dump/linux/v2/interfaces
```
---

**L2 Plugin**

```
# Bridge domains
/dump/vpp/v2/bd

# FIB entries
/dump/vpp/v2/fib

# Cross-connects
/dump/vpp/v2/xc
```
---

**L3 Plugin**

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
---

**Linux L3 Plugin**

```
# Linux routes
/dump/linux/v2/routes

# Linux ARPs
/dump/linux/v2/arps
```

---

**NAT plugin**

```
# REST path of a NAT
/dump/vpp/v2/nat

# Global NAT config
/dump/vpp/v2/nat/global

# DNAT configurations
/dump/vpp/v2/nat/dnat
```
---

**IPSec plugin**

```
# REST path of the SPD
/dump/vpp/v2/ipsec/spds

# REST path of the SA
/dump/vpp/v2/ipsec/sas
```
---

**Punt plugins**

```
# REST path of the punt socket register
/dump/vpp/v2/punt/sockets
```
---

**Telemetry**

```
/vpp/telemetry
/vpp/telemetry/memory
/vpp/telemetry/runtime
/vpp/telemetry/nodecount
```
---

**Tracer**

```
/vpp/binapitrace
```
---

**VPP CLI**

Use the VPP CLI command via REST. Commands are defined as a map like so:
```
/vpp/command -d '{"vppclicommand":"<command>"}'
```

---

## HTTP Security

The REST plugin supports several mechanisms for HTTP security.

Configurable security functions:

- server certificate (HTTPS)
- Basic HTTP Authentication - username & password
- client certificates
- token based authorization

All are disabled by default and can be enabled by a config file:

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

If `server-cert-file` and `server-key-file` are defined, the server requires HTTPS instead of HTTP for all of its endpoints.

`client-cert-files` is the list of the root certificate authorities (CA) the server uses to validate client certificates. If the list is _NOT_ empty, only clients who provide a valid certificate are allowed to access the server.

`client-basic-auth` allows one to define user/password credentials permitting access to the server. The config option defines a static list of allowed user(s). If the list is _NOT_ empty, default staticAuthenticator is instantiated. Alternatively, you can implement custom authenticator and inject it into the plugin. This would be applicable if you wanted to read credentials from etcd.

---

**Example**

In order to generate self-signed certificates, use the following commands:

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

Once the security features are enabled, the endpoint can be accessed with the following commands:

- `HTTPS`
where `ca.pem` is a CA where server certificates should be validated, in the case of self-signed certificates

- `curl --cacert ca.crt  https://127.0.0.1:9292/log/list`


- `HTTPS + client cert` where `client.crt` is a valid client certificate
  ```
  curl --cacert ca.crt  --cert client.crt --key client.key  https://127.0.0.1:9292/log/list
  ```

- `HTTPS + basic auth` where `user:pass` is a valid username password pair.
  ```
  curl --cacert ca.crt  -u user:pass  https://127.0.0.1:9292/log/list
  ```

- `HTTPS + client cert + basic auth`
  ```
  curl --cacert ca.crt  --cert client.crt --key client.key -u user:pass  https://127.0.0.1:9292/log/list
  ```

---
  
### Token-based Authorization

The REST plugin supports authorization based on tokens. To enable this feature, use the paramter contained in the [REST conf file][rest-conf-file]:

```
enable-token-auth: true
```

Authorization restricts access to all registered permission group URLs. The user receives a token after login, which grants access to all permitted sites. The token is valid until the user is logged out, or until it expires.

The expiration time is a token claim set in the [REST plugin config file][rest-conf-file]:

```
token-expiration: 600000000000  
```

Note that time is in nanoseconds. If no time is provided, the default value of 1 hour is set.

By default, token uses a pre-defined signature string as the key to sign it. This can be changed in [REST conf file][rest-conf-file].

```
token-signature: <string>
```

After login, the token is required in an authentication header in the format `Bearer <token>`, so it can be validated. If the REST interface is accessed with a browser, the token is written to a cookie file.

---

### Users and Permission Groups

Users must be pre-defined in the [REST conf file][rest-conf-file]. User definitions consists of a name, hashed password and permission groups.

User format example:
```
users:
   - name: <name>
     password_hash: <hash>
     permissions: [<group1>, <group2>, ...]
`
```

`Name` defines a username (login). Name "admin" is forbidden since the admin user is created automatically with full permissions and a password of "ligato123"

`Password` must be hashed. It is possible to use the [password-hasher utility][password-hasher] to assist with this function. Password must also be hashed with the same cost value, as defined in the [REST conf file][rest-conf-file] like so:

```
password-hash-cost: <number>
```

Minimal hash cost is 4, maximal value is 31. The higher the cost, the more CPU time/memory is required to hash the password.

`Permission Groups` define a list of permissions composed of allowed URLs and methods. Every user requires at least one permission group defined, otherwise the user will be excluded from access to any server. Permission groups described in the [access security proto][access-security-proto]. 

To add a permission group, use a rest plugin API:

```
RegisterPermissionGroup(group ...*access.PermissionGroup)
```

Every permission group has a name and a list o permissions. Permission defines a URL and a list of methods which may be performed. 

To add permission group to the user, put its name to the conf file under user field `permissions`. 

---

### Login/Logout

To log in a user, follow the URL `http://localhost:9191/login`. The site is enabled for two methods. It is possible to use a `POST` to directly provide credentials in the format:

```
{
	"username": "<name>",
	"password": "<pass>"
}
```

The site returns the access token in plain text. If the URL is accessed with `GET`, it displays the login page where the credentials can be entered. After a successful submit, the user is redirected to the index.

To log out, post the username to `http://localhost:9191/logout`.

```
{
	"username": "<name>"
}
```

---

## GRPC Plugin

gRPC support is provided by the GRPC plugin. It is a [Ligato infrastructure][ligato-cn-infra-framework] plugin that enables applications and plugins to utilize gRPC APIs to interact with other system components including the VPP agent.

!!! Note
    The documentation will use the terms: "GRPC" and "gRPC". GRPC refers to the Ligato-specific GRPC components such as the plugin, tutorials and examples. gRPC refers to the functions, flows, definitions and behaviors independent of a specific implementation, as noted in [gRPC open source project][grpc-io] documentation. 

**References** 

* [Ligato cn-infra repo][cn-infra-github]
* [GRPC plugin repo folder][grpc-cn-infra-repo-folder]
* [GRPC conf file][grpc-conf-file]

The GRPC plugin supports the following:

* Send configuration data to VPP
* Retrieve (dump) configuration from VPP
* Start a notification watcher

The following remote procedure calls (RPC) are defined:

* **Get** creates new configuration or updates an existing configuration
* **Delete** removes an existing configuration
* **Dump** reads existing configuration data from VPP
* **Notify** subscribes GRPC to the notification service

To enable the GRPC server, the GRPC plugin must be added to the plugin pool and loaded. Currently, the GRPC plugin is a [part of the Configurator plugin dependencies][configurator-grpc]. The plugin also requires a [conf file][grpc-conf-file] where the endpoint is defined. 

---

### Configuration

Clients can reach the GRPC Server with an endpoint IP:Port address, or by a unix domain socket file.  

The GRPC endpoint is the address of the gRPC netListener. There are two way to modify the endpoint address: using the port flag, or modifying the GRPC conf file.

GRPC port flag:
```
-grpc-port=<port>
```
Endpoint field in the [GRPC conf file][grpc-conf-file]:
```json
endpoint: 0.0.0.0:9191
```

If a unix domain socket file is used, the socket type can be modified using the network field of the [GRPC conf file][grpc-conf-file]:
```json
network: tcp
```
TCP is set as the default, but other types are possible, including TCP6, unix, and unixpacket.

---

### Plugin API

Application plugins can handle gRPC requests as follows:

- The GRPC Plugin starts the GRPC server + net listener in its own goroutine
- Plugins register their handlers with the `GRPC Plugin`. To service gRPC requests, a plugin must implement a handler function and register it at a given URL path using the `RegisterService` method. The GRPC Plugin uses an GRPC request multiplexer from `grpc/Server`.
- GRPC Server routes gRPC requests to their respective registered handlers using the `grpc/Server`.

![grpc][grpc-image]
<p style="text-align: center; font-weight: bold">GRPC Plugin API</p>
<br/>

The [GRPC server example][grpc-server-example] demonstrates the usage of the plugin API GetServer():
```
// Register our GRPC request handler/service using generated RegisterGreeterServer:
RegisterGreeterServer(plugin.GRPC.Server(), &GreeterService{})
```
Once the handler is registered with the GRPC plugin, and the agent is running, you can use a gRPC client to call the service.

---

**Examples & Tutorials**

* [GRPC client example][grpc-client-example] shows how to create a client for the GRPC Server
* [GRPC server example][grpc-server-example] shows how to create a GRPC Server
* [GRPC handler tutorial][grpc-handler-tutorial]
* [GRPC VPP notifications example][example-grpc-vpp-notifications]
* [GRPC VPP configuration example][example-grpc-vpp-remote]


[access-security-proto]: https://github.com/ligato/cn-infra/blob/master/rpc/rest/security/model/access-security/accesssecurity.proto
[cn-infra-github]: https://github.com/ligato/cn-infra
[cn-infra-rest-repo-folder]: https://github.com/ligato/cn-infra/tree/master/rpc/rest
[configurator-grpc]: https://github.com/ligato/vpp-agent/blob/master/plugins/configurator/plugin.go#L54
[example-grpc-vpp-notifications]: https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications
[example-grpc-vpp-remote]: https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client
[grpc-client-example]: https://github.com/ligato/cn-infra/tree/master/examples/grpc-plugin/grpc-client
[grpc-server-example]: https://github.com/ligato/cn-infra/tree/master/examples/grpc-plugin/grpc-server
[grpc-conf-file]: ../user-guide/config-files.md#grpc
[grpc-image]: ../img/user-guide/grpc.png
[grpc-io]: https://grpc.io/
[grpc-plugin]: https://github.com/ligato/cn-infra/tree/master/rpc/grpc
[grpc-cn-infra-repo-folder]: https://github.com/ligato/cn-infra/tree/master/rpc/grpc
[grpc-handler-tutorial]: ../tutorials/08_grpc-tutorial.md
[rest-conf-file]: ../user-guide/config-files.md#rest
[ligato-cn-infra-framework]: ../intro/framework.md#cn-infra
[password-hasher]: https://github.com/ligato/cn-infra/blob/master/rpc/rest/security/password-hasher/README.md
[rest-handler-tutorial]: ../tutorials/03_rest-handler.md
[vpp-agent-rest-plugin]: https://github.com/ligato/vpp-agent/tree/master/plugins/restapi

*[ACL]: Access Control List
*[ARP]: Address Resolution Protocol
*[CLI]: Command-Line Interface
*[FIB]: Forwarding Information Base
*[HTTP]: Hypertext Transfer Protocol
*[HTTPS]: Hypertext Transfer Protocol Secure
*[NAT]: Network Address Translation
*[REST]: Representational State Transfer
*[URL]: Uniform Resource Locator
