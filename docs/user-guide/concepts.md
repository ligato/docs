# Concepts

This section will describe several key concepts for the Ligato VPP Agent and Infra.

!!! Note
    The documentation in some cases may refer to the Ligato Infra as Cloud Native - Infra or CN-infra for short.

---

## What is a Model?

The model represents an abstraction of an object (e.g. VPP interface) that can be managed through northbound APIs exposed by the VPP Agent. More specifically the model is used to generate a key associated with a value for the object that is stored in a KV store such as ETCD. 

### Model Components

- model spec
- protobuf message (`proto.Message`)
- name template (optional)

A single protobuf message can only be represented by a single model.

### Model Specification

Model spec (specification) describes the model using the module, version and type fields:

- `module` -  groups models belonging for the same configuration entity. For example, models for VPP configuration have vpp module, models for Linux configuration have linux module, etc.
- `version` - current version of the VPP Agent API. The version only changes upon release of a new version.
- `type` - keyword describing the given model (e.g. interfaces, bridge-domains, etc.)

These three parts are used to generate a model prefix. The model prefix is part of a key and uses the following format:
```
config/<module>/<version>/<type>/
```
This is an important concept so let's look at some examples.

Here is the key for a [VPP interface][vpp-keys]:

```
/vnf-agent/vpp1/config/vpp/v2/interfaces/<name>
```
Inside that key is the model prefix of `../vpp/v2/interfaces` where:
```
Module = vpp
version = v2
type = interfaces
```
An example of this key in action was shown in [section 5.1 of the Quickstart Guide][quickstart-guide-51-keys]. It was used to program a VPP loopback interface with the value of an IP address for insertion into an ETCD KV store.

```
$ docker exec etcd etcdctl put /vnf-agent/vpp1/config/vpp/v2/interfaces/loop1 \
'{"name":"loop1","type":"SOFTWARE_LOOPBACK","enabled":true,"ip_addresses":["192.168.1.1/24"]}'
``` 
Note that the value of `loop1` is the `<name>` of the interface.



## Key-value store overview

This section describes supported key-value data bases and how to use them

### Why KV store?

The vpp-agent uses external KV store to keep desired state of the VPP/Linux configuration, eventually to store and export certain VPP statistics. The Agent reacts to data events created by changes within a data-store, in case the change of the data was done under a watched key with proper microservice label.
It is the microservice label which allows using single KVDB to serve multiple Agents. Every Agent defines its own label and uses it to distinguish what data were designed for it. 

[![KVDB_microservice_label](../img/user-guide/kvdb-microservice-label.png)](https://www.draw.io/?state=%7B%22ids%22:%5B%221ShslDzO9eDHuiWrbWkxKNRu-1-8w8vFZ%22%5D,%22action%22:%22open%22,%22userId%22:%22109151548687004401479%22%7D#Hligato%2Fdocs%2Fmaster%2Fdocs%2Fimg%2Fuser-guide%2Fkvdb-microservice-label.xml)

Underlying CN-Infra plugin validates key prefix (always in format `/vnf-agent/<microservice-label>`) and if the label matches, KV-pair is passed to vpp-agent configuration watchers. If the prefix of the rest of the key is registered, KV-pair is sent via channel to particular watcher for processing.

The Agent can also connect to multiple different KVDB stores - the plugin called Orchestrator safely collects the data from multiple sources and provides it to underlying configuration plugins.

However, it is not inevitable to use the Agent with any KVDB store - it can perfectly work without it, obtaining configuration via GRPC (REST in the future) or via local or remote [Clientv2][client-v2] API (if the Agent is used as library). The KVDB offers very convenient way how to store and manage configuration data.

### What KV store can I use?

Technically any. The CN-Infra the vpp-agent is based on provides connectors to several types of KVDB (see the list below). All of them are built on the common abstraction (called KVDB sync). It comes with significant advantages - the most important is that the change of the data store can be done very easily.

Let's have a look at the example:
```go
import (
	"github.com/ligato/cn-infra/db/keyval/etcd"
	// ...
)

func New() *VPPAgent {
	// Prepare KVDB sync plugin with ETCD as a connector
	etcdDataSync := kvdbsync.NewPlugin(kvdbsync.UseKV(&etcd.DefaultPlugin))
	
	// Put the KVDB sync to a list of proto watchers 
	watchers := datasync.KVProtoWatchers{
		etcdDataSync,
	}
	// Provide connection to the orchestrator (or any other plugin)
	orchestrator.DefaultPlugin.Watcher = watchers
	
	// Other plugins
	....
}
```

The code above prepares KVDB sync plugin (KVDB abstraction) with ETCD plugin as connector. The KVDB sync can be then provided as watcher to other plugins (or writer if passed as `KVProtoWriters` object).

Switch to Redis:
```go
import (
	"github.com/ligato/cn-infra/db/keyval/redis"
	...
)

func New() *VPPAgent {
	// Change KVDB sync plugin to use Redis connector
	redisDataSync := kvdbsync.NewPlugin(kvdbsync.UseKV(&redis.DefaultPlugin))
	
	watchers := datasync.KVProtoWatchers{
		redisDataSync,
	}
	orchestrator.DefaultPlugin.Watcher = watchers
	
	// Other plugins
	....
}
```
 
The orchestrator now connects to Redis data base.

Another advantage is that to add support for new KVDB basically means to write a plugin able to establish connection to the desired database and wire it with the KVDB sync, which is significantly easier and faster.

### ETCD

More information: [ETCD documentation][etcd-plugin]

The ETCD is a distributed KV store that provides data read-write. The ETCD can be started on local machine in its own container with following command: 
```bash
sudo docker run -p 2379:2379 --name etcd --rm \
    quay.io/coreos/etcd:v3.1.0 /usr/local/bin/etcd \
    -advertise-client-urls http://0.0.0.0:2379 \
    -listen-client-urls http://0.0.0.0:2379
``` 

Vpp-agent must start with the KVDB sync plugin using ETCD as a connector (to know how to do this, see code above or code inside [this example][datasync-example]). Information where the ETCD is running (IP address, port) has to be also provided on the Agent startup via the configuration file. 

The minimal content of the file may look as follows:
```
endpoints:
  - "172.17.0.1:2379"
```
The config file is passed to the Agent via flag `--etcd-confg=<path>`. The file contains more fields where ETCD-specific parameters can be set (dial timeout, certification, compaction, etc.). See [guide to config files][list-of-supported] for more details.

Note that if the config file is not provided, the connector plugin will not be started and no connection will be established. If the ETCD is not reachable at the address provided, the Agent may not start at all.

Recommended tool to manage ETCD database is the official `etcdctl`.

### Redis

More information: [Redis documentation][redis-plugin]

Redis is another type of in-memory data structure store, used as a database, cache or message broker. 

Follow [this guide][redis-quickstart] to learn how to install `redis-server` on any machine.

The Agent must start with the KVDB sync plugin using Redis as a connector (to know how to do this, see code above or code inside [this example][datasync-example]). Information where the Redis is running (IP address, port) has to be also provided on the Agent startup via the configuration file. 

The content of the .yaml configuration file:
```yaml
endpoint: localhost:6379
```

The config file is provided to the Agent via the flag `--redis-config=<path>`. See [guide to config files][list-of-supported] to learn more about Redis configuration.

Note that if the Redis config file is not provided, the particular connector plugin will not be started and no connection will be established. If the Redis is not reachable at the provided address, the Agent may not start at all.

Recommended tool to manage Redis database is the `redis-cli` tool.

!!! note "Important" 
    The Redis server is by default started with keyspace event notifications disabled. It means, no data change events are forwarded to the Agent watcher. To enable it, use `config SET notify-keyspace-events KA` command directly in the `redis-cli`, or change the settings in the Redis server startup config.

### Consul

More information: [consul documentation][consul-plugin]

Consul is a service mesh solution providing a full featured control plane with service discovery, configuration, and segmentation functionality. The Consul plugin provides access to a consul key-value data store. Location of the Consul configuration file can be defined either by the command line flag `consul-config` or set via the `CONSUL_CONFIG` environment variable.

### Bolt
  
Bolt is low-level, simple and fast key-value data store. The Bolt plugin provides an API to access the Bolt server.

### FileDB

More information: [fileDB documentation][filedb-plugin]

The fileDB is a special case of database, which uses host OS filesystem as a database. The key-value configuration is stored in text files in defined path. The fileDB connector works as any other KVDB connector, reacts on data change events (file edits) in real time and supports all KVDB features (resync, versioning, microservice labels, ...).

The fileDB is not a process, so it does not need to be started - the Agent only needs correct permission to access files with configuration (and write access if the status is published).

The file with configuration has requisite format and can be kept as `.json` or `.yaml`.

As any other KVDB plugin, fileDB also requires config file to load, provided via flag `--filedb-config=<path>`. 

Example config file:

```
configuration-paths: [/path/to/directory/or/file.txt]
```

In this case, absence of the config file does not prevent agent from starting, since the configuration files can be created at later point.

The file with configuration can be edited with any text editor, like `vim`.

### How to use it in a plugin

In the plugin which intends to use KVDB connection for publishing or watching, define similar fields of the type:

```go 
import (
    "github.com/ligato/cn-infra/datasync"
    ...
}

type Plugin struct {

    ...

    Watcher    datasync.KeyValProtoWatcher 
    Publisher datasync.KeyProtoValWriter    
}        

```

Prepare the connector in the application plugin definition and set it to the example plugin:

```go
func New() *VPPAgent {
	// Prepare KVDB connector writer and watcher
	etcdDataSync := kvdbsync.NewPlugin(kvdbsync.UseKV(&etcd.DefaultPlugin))
	watchers := datasync.KVProtoWatchers{
		etcdDataSync,
	}
	writers := datasync.KVProtoWriters{
        etcdDataSync,
    }
	
	// Pass connector to the plugin:
	plugin.Defaultplugin.Watcher = etcdDataSync
	plugin.Defaultplugin.Publisher = etcdDataSync
	
	...
}
```

**Watcher**

Back in the Plugin, start watcher. The watcher requires two channels of type `datasync.ChangeEvent` and `datasync.ResyncEvent` (the resync one is optional) and a set of key prefixes to watch on:

```go
p.resyncEventChannel := make(chan datasync.ResyncEvent)
p.changeEventChannel := make(chan datasync.ChangeEvent)
keyPrefixes := []string{<prefixes>...}

watchRegistration, err = p.Watcher.Watch("plugin-resync-name", p.resyncEventChannel, p.changeEventChannel, keyPrefixes)
```

Data change and resync events arrive via the particular channel, so the next step is to start the watcher in new go routine:

```go
func (p *Plugin) watchEvents() {
	for {
		select {
		case e := <-p.resyncEventChannel:
			// process resync event
        case e := <-p.changeEventChannel:
        	// process data change event
		case <-p.ctx.Done():
			//exit
		}
	}
}
```

It is a good practice to start event watcher before the watcher registration. 

**Publisher**

Nothing special is required for the publisher. The `KeyProtoValWriter` object defines method `Put(<key>, <value>, <variadic-options>)` which allows storing key-value pair to the data store. No 'Delete()' method is defined, objects are removed sending nil data under the key intended to remove.

## VPP configuration order

One of the VPP biggest drawbacks is that the dependency relations of various configuration items are very strict, and ability of the VPP to manage it is limited, or not present at all. There are two problems to be solved:

1. A configuration item dependent on any other configuration item cannot be created "in advance", the dependency must be fulfilled first
2. If the dependency item is removed, the VPP behavior is not consistent here. Sometimes, also the dependent item is removed as expected but there are cases where it becomes unmanageable or in any other kind of invalid state where it cannot be directly handled.

### Configuration order using the VPP CLI

The base kind of the VPP configuration are interfaces. After the interface creation, the VPP generates its index which is serves as a reference for other configuration items which plan to use it (bridge domains, routes, ARP entries, ...). Other items, like FIBs may have even more complicated dependencies on several configuration types.

Example:

Start with the empty VPP and configure an interface:
```bash
vpp# create loopback interface 
loop0
vpp# 
```

The interface `loop0` was added and interface index (and also name) was generated for it. The index can be shown by `show interfaces`. 

Set the interface up:
```bash
vpp# set interface state loop0 up
vpp#
``` 

The CLI command requires interface name to know which interface should be set to UP state. 

Now let's create the bridge domain:
```bash
vpp# create bridge-domain 1 learn 1 forward 1 uu-flood 1 flood 1 arp-term 1   
bridge-domain 1
vpp# 
```

The bridge domain is currently empty, so let's assign our interface to it:
```bash
vpp# set interface l2 bridge loop0 1 
vpp#
``` 

We set `loop0` to bridge domain with index 1. Again, we cannot use non-existing interface (since the names are generated) but we also cannot use non-existing bridge domain.

At last, let's configure the FIB table entry:
```bash
vpp# l2fib add 52:54:00:53:18:57 1 loop0
vpp#
```

The call contains dependencies on the `loop0` interface and on our bridge domain with ID=1. From the example steps above, we see that the order of CLI commands must follow in this way (with some exceptions like the bridge domain creation (third step) which can be called earlier).

Now remove the interface:
```bash
vpp# delete loopback interface intfc loop0
vpp#
```

If we check the FIB table now with `show l2fib details`, we can see that the interface name was changed from `loop0` to `Stale`.

Remove bridge domain. The command is:
```bash
vpp# create bridge-domain 1 del
vpp#
```

Here we get into trouble, since the output of the `show l2fib verbose` is still the same:
```bash
vpp# sh l2fib verbose
    Mac-Address     BD-Idx If-Idx BSN-ISN Age(min) static filter bvi         Interface-Name        
 52:54:00:53:18:57    1      1      0/0      no      *      -     -               Stale                     
L2FIB total/learned entries: 2/0  Last scan time: 6.6309e-4sec  Learn limit: 4194304 
vpp#
``` 
But attempt to remove the FIB entry will not pass:
```bash
vpp# l2fib del 52:54:00:53:18:57 1 
l2fib del: bridge domain ID 1 invalid
vpp#
```

So we ended up with the configuration which cannot be removed (until the bridge domain is re-created). To avoid similar scenarios and provide more freedom in configuration order, the vpp-agent uses automatic configuration order mechanism which handles this one and similar cases.

### Configuration order using the Ligato vpp-agent

The vpp-agent uses northbound API definition of every supported configuration item, while the single proto-modelled dataset may call multiple binary API calls to the VPP. For example NB interface configuration creates the interface itself, sets its state, MAC address, IP addresses and other. However, the vpp-agent goes even further - it allows to "configure" VPP items with not-yet-existing references. Such an item is not really configured (since it cannot be), but the agent "remembers" it and puts to the VPP when possible, without any other intervention, thus removing strict VPP ordering.

!!! note
    If you want to follow, this part expects to have the basic setup prepared (vpp-agent + VPP + ETCD). If you need help with the setup, [here is the guide][quickstart-guide].

Let's start from the end and put configuration for the FIB entry:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/fib/bd1/mac/62:89:C6:A3:6D:5C '{"phys_address":"62:89:C6:A3:6D:5C","bridge_domain":"bd1","outgoing_interface":"if1","action":"FORWARD"}'
```

Have a look at the vpp-agent output of `planned operations`:
```bash
1. CREATE [NOOP IS-PENDING]:
  - key: config/vpp/l2/v2/fib/bd1/mac/62:89:C6:A3:6D:5C
  - value: { phys_address:"62:89:C6:A3:6D:5C" bridge_domain:"bd1" outgoing_interface:"if1" } 
```

There is one `CREATE` transaction planned - our FIB entry with interface `if1` and bridge domain `bd1`. None of these exists, so the transaction was postponed, highlighting it as `[NOOP IS-PENDING]`.

Configure the bridge domain:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1 '{"name":"bd1","interfaces":[{"name":"if1"}]}'
```

We have created a simple bridge domain with interface. Let's break down the output:
```bash
1. CREATE:
  - key: config/vpp/l2/v2/bridge-domain/bd1
  - value: { name:"bd1" interfaces:<name:"if1" >  } 
2. CREATE [DERIVED NOOP IS-PENDING]:
  - key: vpp/bd/bd1/interface/tap1
  - value: { name:"if1" }
```

Now there are two planned operations - the first is the bridge domain `bd1` which can be created since there are no restrictions. The second operation contains following highlight: `[DERIVED NOOP IS-PENDING]`. 'DERIVED' means that this item was processed as separate key within the vpp-agent (it is internal functionality so let's not to bother with it now). The rest of the flag is the same as before, meaning that the value `if1` is not present yet.

Next step is to add incriminated interface:
```bash
etcdctl put /vnf-agent/vpp1/config/vpp/v2/interfaces/tap1 '{"name":"tap1","type":"TAP","enabled":true}
```

The output:
```bash
1. CREATE:
  - key: config/vpp/v2/interfaces/tap1
  - value: { name:"if1" type:TAP enabled:true } 
2. CREATE [DERIVED WAS-PENDING]:
  - key: vpp/bd/bd1/interface/tap1
  - value: { name:"if1" } 
3. CREATE [WAS-PENDING]:
  - key: config/vpp/l2/v2/fib/bd1/mac/62:89:C6:A3:6D:5C
  - value: { phys_address:"62:89:C6:A3:6D:5C" bridge_domain:"bd1" outgoing_interface:"if1" }
```

There are three `planned operations` present. The first one is the interface itself, which was created (and also enabled). 
The second operation is marked as `DERIVED` (the same meaning, not important here) and `WAS-PENDING` which means the cached value could be finally resolved. This operation means that the interface was added to the bridge domain, as defined in the bridge domain configuration.
The last statement is marked `WAS-PENDING` as well and represents our FIB entry, cached until now. Since all dependencies were fulfilled (the bridge domain and the interface as a part of it), the FIB was configured.

Now let's try to remove the bridge domain as before:
```bash
etcdctl del /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1
```

The output:
```bash
1. DELETE [IS-PENDING]:
  - key: config/vpp/l2/v2/fib/bd1/mac/62:89:C6:A3:6D:5C
  - value: { phys_address:"62:89:C6:A3:6D:5C" bridge_domain:"bd1" outgoing_interface:"if1" } 
2. DELETE [DERIVED]:
  - key: vpp/bd/bd1/interface/tap1
  - value: { name:"if1" } 
3. DELETE:
  - key: config/vpp/l2/v2/bridge-domain/bd1
  - value: { name:"bd1" interfaces:<name:"if1" > } 
```

Notice that the first performed operation was removal of the FIB entry, but the value was not discarded by the vpp-agent since it still exists in the ETCD. The vpp-agent put the value back to the cache. This is important step since the FIB entry is removed before the bridge domain, so it will not get mis-configured and stuck in the VPP. The value no longer exists on the VPP (since logically it cannot without all the dependencies met), but if the bridge domain reappears, the FIB will be added back without any action required from outside.
The second step means that the interface `if1` was removed from the bridge domain, before its removal in the last step of the transaction.

The ordering and caching of the configuration is performed by the vpp-agent KVScheduler component. For more information how it works, please refer [here][telemetry-plugin].

## VPP multi-version support

The vpp-agent is highly dependent on the version of the VPP binary API used to send and receive various types of configuration messages. The VPP API is changing over time, adding new binary calls or modifying or removing existing ones. The latter is the most crucial from the vpp-agent perspective since it introduces incompatibilities between vpp-agent and the VPP.
For that reason the vpp-agent does a compatibility check in the GoVPP multiplex plugin (an adapter of the GoVPP - the GoVPP component provides API for communication with the VPP). The compatibility check attempts to read an ID of the provided message. If just one message ID is not found (validation if the cyclic redundancy code), the VPP is considered incompatible and the vpp-agent will not connect. The message validation is essential for successful connection. Now it is clear that the vpp-agent is tightly bound to the version of the VPP (for that reason the vpp-agent image is shipped together with compatible VPP version to make it easier for users).

This concept may cause some inconveniences during manual setup, and especially in scenarios where the vpp-agent needs to be quickly switched to different VPP which is not compatible.

### VPP compatibility

The vpp-agent bindings are generated from the VPP JSON API definitions. Those can be found in the path `/usr/share/vpp/api`. Full VPP installation is required if definitions should be generated. The json API definition is then transformed to the `*.ba.go` file using `binapi-generater` which is a part of the GoVPP project. All generated structures implement the GoVPP `Message` interface which allows to get message name, CRC or message type, and represents generic type for all messages which could be sent via the VPP channel.

Example:
```go
type CreateLoopback struct {
	MacAddress []byte `struc:"[6]byte"`
}

func (*CreateLoopback) GetMessageName() string {
	return "create_loopback"
}
func (*CreateLoopback) GetCrcString() string {
	return "3b54129c"
}
func (*CreateLoopback) GetMessageType() api.MessageType {
	return api.RequestMessage
}
```

The code above is generated from `create_loopback` within the `interface.api.json`. The structure represents the binary API request call, and usually contains a set of fields which can be set to required values (like MAC address of the loopback interface in this example). Every VPP request call requires a response:

```go
type CreateLoopbackReply struct {
	Retval    int32
	SwIfIndex uint32
}

func (*CreateLoopbackReply) GetMessageName() string {
	return "create_loopback_reply"
}
func (*CreateLoopbackReply) GetCrcString() string {
	return "fda5941f"
}
func (*CreateLoopbackReply) GetMessageType() api.MessageType {
	return api.ReplyMessage
}
``` 

The response has a `Retval` field, which is `0` if the API call was successful and without errors. In case of any error, the field contains numerical index of VPP-defined error message. Other fields can be present (usually some information generated within the VPP, as the `SwIfIndex` of the created interface in this case). Notice that the response message has the same name but with `Reply` suffix. Other types of messages may serve to obtain various information from the VPP - those API calls have suffix `Dump` (and for them, the reply message is with `Details` suffix).

If the json API was changed, it must be re-generated in the vpp-agent in order to be compatible again (and all changes caused by the modified binary api structures must be resolved, like added, removed or renamed fields). This process can be time-consuming sometimes (depends on the size of the difference). In order to minimize updates for various VPP versions, the vpp-agent introduced multi-version support. 

### Multi-version

The vpp-agent multi-version support allows to switch to the different VPP version (with different API) without any changes to the vpp-agent itself and without any need to rebuild the vpp-agent binary. Plugins can now obtain the version of the VPP and the vpp-agent is trying to connect and initialize correct set of `vppcalls` - base methods preparing API requests for the VPP. 

Every `vppcalls` handler registers itself with the VPP version intended to support (like `vpp1810`, `vpp1901`, etc.). During initialization, the vpp-agent does compatibility check with with all available handlers, until it founds some which is compatible with required messages. The chosen handled must be in line with all messages, it is not possible (and reasonable) to use multiple handlers for single VPP. When the compatibility check found any handler, it is returned to the main plugin for use. 

The little drawback of this solution is a lot of duplicated code across `vppcalls`, since there is not much of significant API changes between versions.

## Client v2

Client v2 (i.e. the second version) defines an API that allows to manage
configuration of VPP and Linux plugins.
How the configuration is transported between APIs and the plugins
is fully abstracted from the user.

The API calls can be split into two groups:

- **resync** applies a given (full) configuration. An existing configuration, if present, is replaced. The name is an abbreviation of *resynchronization*. It is used initially and after any system event that may leave the configuration out-of-sync while the set of outdated configuration options is impossible to determine locally (e.g. temporarily lost connection to data store).
- **data change** allows to deliver incremental changes of a configuration.

There are two implementations:

- **local client** runs inside the same process as the agent and delivers configuration through go channels directly to the plugins.
- **remote client** stores the configuration using the given `keyval.broker`.
   
## Plugin configuration files

Some plugins may require an external information in order to set proper behavior. The good example are database connector plugins like ETCD. By default, the plugin tries default IP address and port to connect to the running database instance. If we want to connect to a different IP address or some custom port, we need to pass this information to the plugin. For that purpose, the vpp-agent plugins support configuration files. 
The plugin configuration file (or just config-file) is a short file with fields (every plugin can define its own set of fields), which can statically modify the initial plugin setup and change some aspects of its behavior. 

### Plugin definition of the config-file

The configuration file is passed to the plugin via the vpp-agent flags. Using the command `vpp-agent -h` can be shown the list of all plugins supporting any static config and can be set simply via the flag command:

```bash
vpp-agent -etcd-config=/opt/vpp-agent/dev/etcd.conf
```

Other option is to set related environment variable:
```bash
export ETCD_CONFIG=/opt/vpp-agent/dev/etcd.conf
```

The provided config follows YAML syntax and is un-marshaled to defined `Config` go structure. All fields are then processed, usually in the plugin `Init()`. It is a good practice to always use default values in case the configuration file or any of its field is not provided, so the plugin can be successfully started without it. The config-file (yaml) representation:
```bash
db-path: /tmp/bolt.db
file-mode: 0654
```

Every plugin supporting configuration file defines its content as a separate structure. For example, here is the definition of the BoltDB config-file:
```go
type Config struct {
	DbPath      string        `json:"db-path"`
	FileMode    os.FileMode   `json:"file-mode"`
	LockTimeout time.Duration `json:"lock-timeout"`
}
```

As you can see, the config file can contain multiple fields of various types together with the json struct tag. Lists and maps are also allowed. Notice that not all fields must be defined in the file itself - empty fields are set to default values or handled as not set. 

List of supported configuration files can be found [here][config-files]

[client-v2]: ../user-guide/concepts.md#client-v2
[config-files]: config-files.md
[consul-plugin]: ../plugins/db-plugins.md#consul-plugin
[datasync-example]: https://github.com/ligato/cn-infra/tree/master/examples/datasync-plugin
[etcd-plugin]: ../plugins/db-plugins.md#etcd-plugin
[filedb-plugin]: ../plugins/db-plugins.md#filedb
[list-of-supported]: config-files.md
[redis-plugin]: ../plugins/db-plugins.md#redis
[redis-quickstart]: https://redis.io/topics/quickstart
[telemetry-plugin]: ../plugins/vpp-plugins.md#telemetry
[quickstart-guide]: ../user-guide/quickstart.md
[vpp-keys]: ../user-guide/reference.md#vpp-keys
[quickstart-guide-51-keys]: ../user-guide/quickstart.md#51-configure-the-vpp-dataplane-using-the-vpp-agent

*[ARP]: Address Resolution Protocol
*[CLI]: Command-Line Interface
*[KVDB]: Key-Value Database
*[FIB]: Forwarding Information Base
*[REST]: Representational State Transfer
