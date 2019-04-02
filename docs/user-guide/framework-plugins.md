# KV Scheduler

**Related article:** [Implement your own KV Descriptor](../developer-guide/implementing-your-own-kvdescriptor.md)

A major enhancement that led to the increase of the agent's major version number from 1.x to 2.x, is an introduction of a new framework, called **KVScheduler**, providing transaction-based configuration processing with a generic mechanism for dependency resolution between configuration items, which in effect simplifies and unifies the configurators. KVScheduler is shipped as a separate [plugin], even though it is now a core component around which all the VPP and Linux configurators have been re-build.

### Motivation

KVScheduler is a reaction to a series of drawbacks of the original design, which gradually arose with the set of supported configuration items growing:
* plugins `vpp` and `linux` became bloated and complicated, suffering with race conditions and a lack of visibility
* the `configurators` - i.e. components of `vpp` and `linux` plugins, each processing a specific configuration item type (e.g. interface, route, etc.) - had to be effectively built from scratch, solving the same set of problems again and duplicating lot's of code
* configurators would communicate with each other only through notifications, and react to changes asynchronously to ensure proper operation ordering - i.e. the problem of dependency resolution was effectively distributed across all configurators, making it very difficult to understand, predict and stabilize the system behavior from the developer's global viewpoint
* a culmination of all the issues above was an unreliable and unpredictable re-synchronization (or resync for short), also known as state reconciliation, between northbound (desired state) and southbound (actual state) - something that was meant to be marketed as the main feature of the VPP-Agent.

### Basic concepts and terminology

KVScheduler solves problems common to all configurators through generalization - moving away from specific configuration items and describing the system with abstract terms:
* **Model** is a description for a single item type, defined as Protobuf Message (for example Bridge Domain model can be found [here][bd-model-example])
* **Value** is a run-time instance of a given model
* **Key** identifies specific value (it is built from key template defined for the model, e.g. [L2 FIB key template][fib-key-template])
* **Label** can be optionally defined to provide shorter identifier unique only across values of the same type (e.g. interface name)
* **Metadata** is extra run-time information (of undefined type) assigned to a value, which may be updated after a CRUD operation or an agent restart (for example [sw_if_index][vpp-iface-idx] of every interface is kept in the metadata for its key-value pair)
* **Metadata Map**, also known as index-map, implements mapping between value label and its metadata for a given value type - typically exposed in read-only mode to allow other plugins to read and reference metadata (for example, interface plugin exposes its metadata map [here][vpp-iface-map], which is then used by ARP, route plugin etc. to read sw_if_index of target interfaces).  Metadata maps are automatically created and updated by the scheduler (he is the owner), and exposed to other plugins only in the read-only mode. 
* **[Value origin][value-origin]** defines where the value came from - whether it was received from NB to be configured or whether it was created in the SB plane automatically (e.g. default routes, loop0 interface, etc.)
* Key-value pairs are operated with through CRUD operations, where **Add** is used to denote **Create**, **Dump** is basically Read, Update is denoted by the scheduler as **Modify** and **Delete** is borrowed unchanged
* **Dependency** defined for a value, references another key-value pair that must exist (be created), otherwise the associated value cannot be Added and must remain cached in the state **Pending** - value is allowed to have defined multiple dependencies and all must be satisfied for the value to be considered ready for creation.
* **Derived value**, in future release to be renamed to **attribute** for more clarity, is typically a single field of the original value or its property, manipulated separately - i.e. with possibly its own dependencies (dependency on the source value is implicit), custom implementations for CRUD operations and potentially used as target for dependencies of other key-value pairs - for example, every [interface to be assigned to a bridge domain][bd-interface] is treated as a [separate key-value pair][bd-derived-vals], dependent on the [target interface to be created first][bd-iface-deps], but otherwise not blocking the rest of the bridge domain to be applied
* **Graph** of values is a kvscheduler-internal in-memory storage for all configured and pending key-value pairs, with edges representing inter-value relations, such as "depends-on" and "is-derived-from" - configurators no longer have to implement their own caches for pending values
* **KVDescriptor** assigns implementations of CRUD operations and defines derived values and dependencies to a single value type - this is what configurators basically boil down to - to learn more, please read how to [implement your own KVDescriptor](Implementing-your-own-KVDescriptor)

### Dependencies

The idea behind scheduler is based on the Mediator pattern - configurators do not communicate directly, but instead interact through the mediator. This reduces the dependencies between communicating objects, thereby reducing coupling.

The values are described for scheduler by registered KVDescriptor-s.
The scheduler learns two kinds of relations between values that have to be respected by the scheduling algorithm:
1. `A` **depends on** `B`:
   - `A` cannot exist without `B`
   - request to add `A` without `B` existing must be postponed by marking `A` as `pending` (value with unmet dependencies) in the in-memory graph 
   - if `B` is to be removed and `A` exists, `A` must be removed first and set to `pending` state in case `B` is restored in the future
   - Note: values pushed from SB are not checked for dependencies
2. `B` **is derived from** `A`:
   - value `B` is not added directly (by NB or SB) but gets derived from base value `A` (using the DerivedValues() method of the base value's descriptor)
   - a derived value exists only as long as its base does and gets removed (immediately, not pending) once the base value goes away
   - a derived value may be described by a different descriptor than the base and usually represents the property of the base value (that other values may depend on) or an extra action to be taken when additional dependencies are met.

### Resync 

Configurators no longer have to implement resync on their own. As they "teach" KVScheduler how to operate with configuration items by providing callbacks to CRUD operations through KVDescriptors, the scheduler has all it needs to determine and execute the set of operation needed to get the state in-sync after a transaction or restart.

Furthermore, KVScheduler enhances the concept of state reconciliation, and defines three types of the resync:
* **Full resync**: the desired configuration is re-read from NB, the view of SB is refreshed via Dump operations and inconsistencies are resolved via Add/Delete/Modify operations
* **Upstream resync**: partial resync, same as Full resync except for the view of SB is assumed to be up-to-date and will not get refreshed - can be used by NB when it is easier to re-calculate the desired state than to determine the (minimal) difference
* **Downstream resync**: partial resync, same as Full resync except the desired configuration is assumed to be up-to-date and will not be re-read from NB - can be used periodically to resync, even without interacting with NB

### Transactions

The scheduler allows to group related changes and applies them as transactions. This is not supported, however, by all agent NB interfaces - for example, changes from `etcd` datastore are always received one a time. To leverage the transaction support, localclient (the same process) or GRPC API (remote access) have to be used instead.

Inside the scheduler, transactions are queued and executed synchronously to simplify the algorithm and avoid concurrency issues. The processing of a transaction is split into two stages:
* **Simulation**: the set of operations to execute and their order is determined (so-called *transaction plan*), without actually calling any CRUD callback from descriptors - assuming no failures.
* **Execution**: executing the operations in the right order. If any operation fails, the already applied changes are reverted, unless the so called `BestEffort` mode is enabled, in which case the scheduler tries to apply the maximum possible set of required changes. `BestEffort` is the default for resync.  

Right after simulation, transaction metadata (sequence number printed as `#xxx`, description, values to apply, etc.) are printed, together with transaction plan. This is done before execution, to ensure that the user is informed about the operations that were going to be executed even if any of the operations cause the agent to crash. After the transaction has executed, the set of actually executed operations and potentially some errors are printed to finalize the output for the transaction.

An example transaction output printed to logs (in this case there were no errors, therefore the plan matches the executed operations):
```
+======================================================================================================================+
| Transaction #5                                                                                        NB transaction |
+======================================================================================================================+
  * transaction arguments:
      - seq-num: 5
      - type: NB transaction
      - Description: example transaction
      - values:
          - key: vpp/config/v2/nat44/dnat/default/kubernetes
            value: { label:"default/kubernetes" st_mappings:<external_ip:"10.96.0.1" external_port:443 local_ips:<local_ip:"10.3.1.10" local_port:6443 > twice_nat:SELF >  }
          - key: vpp/config/v2/ipneigh
            value: { mode:IPv4 scan_interval:1 stale_threshold:4  }
  * planned operations:
      1. ADD:
          - key: vpp/config/v2/nat44/dnat/default/kubernetes
          - value: { label:"default/kubernetes" st_mappings:<external_ip:"10.96.0.1" external_port:443 local_ips:<local_ip:"10.3.1.10" local_port:6443 > twice_nat:SELF >  }
      2. MODIFY:
          - key: vpp/config/v2/ipneigh
          - prev-value: { scan_interval:1 max_proc_time:20 max_update:10 scan_int_delay:1 stale_threshold:4  }
          - new-value: { mode:IPv4 scan_interval:1 stale_threshold:4  }
o----------------------------------------------------------------------------------------------------------------------o
  * executed operations (2019-01-21 11:29:27.794325984 +0000 UTC m=+8.270999232 - 2019-01-21 11:29:27.797588466 +0000 UTC m=+8.274261700, duration = 3.262468ms):
     1. ADD:
         - key: vpp/config/v2/nat44/dnat/default/kubernetes
         - value: { label:"default/kubernetes" st_mappings:<external_ip:"10.96.0.1" external_port:443 local_ips:<local_ip:"10.3.1.10" local_port:6443 > twice_nat:SELF >  }
      2. MODIFY:
          - key: vpp/config/v2/ipneigh
          - prev-value: { scan_interval:1 max_proc_time:20 max_update:10 scan_int_delay:1 stale_threshold:4  }
          - new-value: { mode:IPv4 scan_interval:1 stale_threshold:4  }
x----------------------------------------------------------------------------------------------------------------------x
x #5                                                                                                          took 3ms x
x----------------------------------------------------------------------------------------------------------------------x
```

### REST API

KVScheduler exposes the state of the system and the history of operations not only via formatted logs but also through a set of REST APIs:
* **transaction history**: `GET /scheduler/txn-history`
    - returns the full history of executed transactions or only for a given time window
    - args:
        - `format=<json/text>`
        - `seq-num=<txn-seq-num>`: transaction sequence number
        - `since=<unix-timestamp>`: if undefined, the output starts with the oldest kept record
        - `until=<unix-timestamp>`: if undefined, the output end with the last executed transaction
* **key timeline**: `GET /scheduler/key-timeline`
     - args:
        - `key=<key-without-agent-prefix>`: key of the value to show changes over time for
* **graph snapshot**: `GET /scheduler/graph-snaphost`
    - args:
        - `time=<unix-timestamp>`: if undefined, current state is returned
* **dump values**: `GET /scheduler/dump`
    - args (without args prints Index page):
        - `descriptor=<descriptor-name>`
        - `state=<NB, SB, internal>` (in the next release will be renamed to `view`): whether to dump desired, actual or the configuration state as known to kvscheduler
* **request downstream resync**: `POST /scheduler/downstream-resync`
    - args:
        - `retry=< 1/true | 0/false >`: allow to retry operations that failed during resync
        - `verbose=< 1/true | 0/false >`: print graph after refresh (Dump)

[plugin]: https://github.com/ligato/vpp-agent/tree/dev/plugins/kvscheduler
[bd-model-example]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/model/l2/bd.proto
[fib-key-template]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/model/l2/keys.go#L38
[vpp-iface-idx]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/ifplugin/ifaceidx/ifaceidx.go#L62
[vpp-iface-map]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/ifplugin/ifplugin_api.go#L26
[value-origin]: https://github.com/ligato/vpp-agent/blob/dev/plugins/kvscheduler/api/kv_descriptor_api.go#L53
[bd-interface]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/model/l2/bd.proto#L14
[bd-derived-vals]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/l2plugin/descriptor/bridgedomain.go#L225
[bd-iface-deps]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vppv2/l2plugin/descriptor/bd_interface.go#L128

This page describes how to use the VPP-Agent with the gRPC

# GoVPP Mux

The `govppmux` is a core Agent plugin which allows other plugins access the VPP
independently at each other by means of connection multiplexing.

Any plugin (core or external) that interacts with the VPP can ask `govppmux`
to get its own, potentially customized, communication channel to access running VPP instance.
Behind the scene, all channels share the same connection created during the plugin
initialization using `govpp` core.

### Connection

By default, the GoVPP connects to that instance of VPP which uses the default shared memory segment prefix. 
The default behaviour assumes that there is only a single VPP running in a sand-boxed environment together with the agent (e.g. through containerization). In case the VPP runs with customized SHM prefix, or there are several VPP instances running side-by-side, the GoVPP needs to know the prefix in order to connect to the desired VPP instance - the prefix has to be put into the govppmux configuration file (govpp.conf) with key `shm-prefix` and value matching the VPP shared memory prefix name.

### Multiplexing

The `NewAPIChannel` call returns a new API channel for communication with VPP via the `govpp` core. It uses default buffer sizes for the request and reply Go channels (by default both are 100 messages long).

If it is expected that the VPP may get overloaded at peak loads, for example if the user plugin sends configuration requests in bulks, then it is recommended to use `NewAPIChannelBuffered` and increase the buffer size for requests appropriately. Similarly, `NewAPIChannelBuffered` allows to configure the size of the buffer for responses. This is also useful since the buffer for responses is also used to carry VPP notifications and statistics which may temporarily rapidly grow in size and frequency. By increasing the reply channel size, the probability of dropping messages from VPP decreases at the cost of increased memory footprint.

### Trace
 
Duration of the VPP binary api call can be measured using trace feature. These data are logged after every event(any resync, interfaces, bridge domains, fib entries etc.). Enable trace in the govpp.conf: 
 
`trace-enabled: true` or  `trace-enabled: false`
  
The trace deature is disabled by default (if there is no config available). 

**Configuration**

The plugin allows to configure parameters of vpp health-check probe.
The items that can be configured are:
- *health check probe interval* - time between health check probes
- *health check reply timeout* - if the reply doesn't arrive until timeout
elapses probe is considered failed
- *health check threshold* - number of consequent failed health checks
until an error is reported
- *reply timeout* - if the reply from channel doesn't arrive until timeout
elapses, the request fails
- *shm-prefix* - used for connection to a VPP instance which is not using 
default shared memory prefix
- *resync-after-reconnect* - allows to run resync after recoonection

**Example**

The following example shows how to dump VPP interfaces using a multi-response request:
```
// Create a new VPP channel with the default configuration.
plugin.vppCh, err = govppmux.NewAPIChannel()
if err != nil {
    // Handle error condition...
}
// Close VPP channel.
defer safeclose.Close(plugin.vppCh)

req := &interfaces.SwInterfaceDump{}
reqCtx := plugin.vppCh.SendMultiRequest(req)

for {
    msg := &interfaces.SwInterfaceDetails{}
    stop, err := reqCtx.ReceiveReply(msg)
    if err != nil {
        // Handle error condition...
    }

    // Break out of the loop in case there are no more messages.
    if stop {
        break
    }

    log.Info("Found interface with sw_if_index=", msg.SwIfIndex)
    // Process the message...
}

```

This page describes how to use the VPP-Agent with representational state transfer

If you look for the tutorial how to create custom HTTP plugin, refer to CN-Infra REST wiki and tutorial //TODO add links

# VPP-Agent REST

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

### Logging mechanism

The REST API request is logged to stdout. The log contains VPP CLI command and VPP CLI response. It is searchable in elastic search using "VPPCLI".

### Security

The CN-Infra REST plugin provides option to secure the HTTP communication. The plugin supports HTTPS client/server certificates, HTTP credentials authentication (username and password) and authorization based on tokens.

This feature is disabled by default and if required, must be enabled by the CN-Infra HTTP plugin config file. 

More information about security setup and usage, see [security](https://github.com/ligato/cn-infra/blob/master/rpc/rest/README.md#security) for certificates and [tokens](https://github.com/ligato/cn-infra/blob/master/rpc/rest/README.md#token-based-authorization) for token-based authorization.

**Basic usage**

**1. cURL** 

Specify the VPP-Agent target HTTP IP address and port with link to desired data. All URLs are accessible via the `GET` method.

Example:
```
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces
```

**2. Postman**

Choose the `GET` method, provide desired URL and send the request.

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

# Telemetry Plugin

The `telemetry` plugin is a core Agent Plugin for exporting telemetry statistics from the VPP to the Prometheus.
Statistics are published via registry path `/vpp` on port `9191` and updated every 30 seconds.

### Exported data

- VPP runtime (`show runtime`)

```bash
                 Name                 State         Calls          Vectors        Suspends         Clocks       Vectors/Call
    acl-plugin-fa-cleaner-process  event wait                0               0               1          4.24e4            0.00
    api-rx-from-ring                any wait                 0               0              18          2.92e5            0.00
    avf-process                    event wait                0               0               1          1.18e4            0.00
    bfd-process                    event wait                0               0               1          1.21e4            0.00
    cdp-process                     any wait                 0               0               1          1.34e5            0.00
    dhcp-client-process             any wait                 0               0               1          4.88e3            0.00
    dns-resolver-process            any wait                 0               0               1          5.88e3            0.00
    fib-walk                        any wait                 0               0               4          1.67e4            0.00
    flow-report-process             any wait                 0               0               1          3.19e3            0.00
    flowprobe-timer-process         any wait                 0               0               1          1.40e4            0.00
    igmp-timer-process             event wait                0               0               1          1.29e4            0.00
    ikev2-manager-process           any wait                 0               0               7          4.58e3            0.00
    ioam-export-process             any wait                 0               0               1          3.49e3            0.00
    ip-route-resolver-process       any wait                 0               0               1          7.07e3            0.00
    ip4-reassembly-expire-walk      any wait                 0               0               1          3.92e3            0.00
    ip6-icmp-neighbor-discovery-ev  any wait                 0               0               7          4.78e3            0.00
    ip6-reassembly-expire-walk      any wait                 0               0               1          5.16e3            0.00
    l2fib-mac-age-scanner-process  event wait                0               0               1          4.57e3            0.00
    lacp-process                   event wait                0               0               1          2.46e5            0.00
    lisp-retry-service              any wait                 0               0               4          1.05e4            0.00
    lldp-process                   event wait                0               0               1          6.79e4            0.00
    memif-process                  event wait                0               0               1          1.94e4            0.00
    nat-det-expire-walk               done                   1               0               0          5.68e3            0.00
    nat64-expire-walk              event wait                0               0               1          5.01e7            0.00
    rd-cp-process                   any wait                 0               0          174857          4.04e2            0.00
    send-rs-process                 any wait                 0               0               1          3.22e3            0.00
    startup-config-process            done                   1               0               1          4.99e3            0.00
    udp-ping-process                any wait                 0               0               1          2.65e4            0.00
    unix-cli-127.0.0.1:38288         active                  0               0               9          4.62e8            0.00
    unix-epoll-input                 polling          12735239               0               0          1.14e3            0.00
    vhost-user-process              any wait                 0               0               1          5.66e3            0.00
    vhost-user-send-interrupt-proc  any wait                 0               0               1          1.95e3            0.00
    vpe-link-state-process         event wait                0               0               1          2.27e3            0.00
    vpe-oam-process                 any wait                 0               0               4          1.11e4            0.00
    vxlan-gpe-ioam-export-process   any wait                 0               0               1          4.04e3            0.00
    wildcard-ip4-arp-publisher-pro event wait                0               0               1          5.49e4            0.00

```

Example:

```bash
# HELP vpp_runtime_calls Number of calls
# TYPE vpp_runtime_calls gauge
...
vpp_runtime_calls{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 7.65806939e+08
...
# HELP vpp_runtime_clocks Number of clocks
# TYPE vpp_runtime_clocks gauge
...
vpp_runtime_clocks{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 1150
...
# HELP vpp_runtime_suspends Number of suspends
# TYPE vpp_runtime_suspends gauge
...
vpp_runtime_suspends{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 0
...
# HELP vpp_runtime_vectors Number of vectors
# TYPE vpp_runtime_vectors gauge
...
vpp_runtime_vectors{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 0
...
# HELP vpp_runtime_vectors_per_call Number of vectors per call
# TYPE vpp_runtime_vectors_per_call gauge
...
vpp_runtime_vectors_per_call{agent="agent1",item="unix-epoll-input",thread="",threadID="0"} 0
...
```

- VPP buffers (`show buffers`)

```bash
 Thread             Name                 Index       Size        Alloc       Free       #Alloc       #Free
      0                       default           0        2048      0           0           0           0
      0                 lacp-ethernet           1         256      0           0           0           0
      0               marker-ethernet           2         256      0           0           0           0
      0                       ip4 arp           3         256      0           0           0           0
      0        ip6 neighbor discovery           4         256      0           0           0           0
      0                  cdp-ethernet           5         256      0           0           0           0
      0                 lldp-ethernet           6         256      0           0           0           0
      0           replication-recycle           7        1024      0           0           0           0
      0                       default           8        2048      0           0           0           0
```

Example:

```bash
# HELP vpp_buffers_alloc Allocated
# TYPE vpp_buffers_alloc gauge
vpp_buffers_alloc{agent="agent1",index="0",item="default",threadID="0"} 0
vpp_buffers_alloc{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 0
...
# HELP vpp_buffers_free Free
# TYPE vpp_buffers_free gauge
vpp_buffers_free{agent="agent1",index="0",item="default",threadID="0"} 0
vpp_buffers_free{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 0
...
# HELP vpp_buffers_num_alloc Number of allocated
# TYPE vpp_buffers_num_alloc gauge
vpp_buffers_num_alloc{agent="agent1",index="0",item="default",threadID="0"} 0
vpp_buffers_num_alloc{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 0
...
# HELP vpp_buffers_num_free Number of free
# TYPE vpp_buffers_num_free gauge
vpp_buffers_num_free{agent="agent1",index="0",item="default",threadID="0"} 0
vpp_buffers_num_free{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 0
...
# HELP vpp_buffers_size Size of buffer
# TYPE vpp_buffers_size gauge
vpp_buffers_size{agent="agent1",index="0",item="default",threadID="0"} 2048
vpp_buffers_size{agent="agent1",index="1",item="lacp-ethernet",threadID="0"} 256
...
...
```

- VPP memory (`show memory`)

```bash
Thread 0 vpp_main
20071 objects, 14276k of 14771k used, 21k free, 12k reclaimed, 315k overhead, 1048572k capacity
```

Example:

```bash
# HELP vpp_memory_capacity Capacity
# TYPE vpp_memory_capacity gauge
vpp_memory_capacity{agent="agent1",thread="vpp_main",threadID="0"} 1.048572e+09
# HELP vpp_memory_free Free memory
# TYPE vpp_memory_free gauge
vpp_memory_free{agent="agent1",thread="vpp_main",threadID="0"} 4000
# HELP vpp_memory_objects Number of objects
# TYPE vpp_memory_objects gauge
vpp_memory_objects{agent="agent1",thread="vpp_main",threadID="0"} 20331
# HELP vpp_memory_overhead Overhead
# TYPE vpp_memory_overhead gauge
vpp_memory_overhead{agent="agent1",thread="vpp_main",threadID="0"} 319000
# HELP vpp_memory_reclaimed Reclaimed memory
# TYPE vpp_memory_reclaimed gauge
vpp_memory_reclaimed{agent="agent1",thread="vpp_main",threadID="0"} 0
# HELP vpp_memory_total Total memory
# TYPE vpp_memory_total gauge
vpp_memory_total{agent="agent1",thread="vpp_main",threadID="0"} 1.471e+07
# HELP vpp_memory_used Used memory
# TYPE vpp_memory_used gauge
vpp_memory_used{agent="agent1",thread="vpp_main",threadID="0"} 1.4227e+07
```

- VPP node counters (`show node counters`)

```bash
Count                    Node                  Reason
120406            ipsec-output-ip4            IPSec policy protect
120406               esp-encrypt              ESP pkts received
123692             ipsec-input-ip4            IPSEC pkts received
  3286             ip4-icmp-input             unknown type
120406             ip4-icmp-input             echo replies sent
    14             ethernet-input             l3 mac mismatch
   102                arp-input               ARP replies sent
```

Example:

```bash
# HELP vpp_node_counter_count Count
# TYPE vpp_node_counter_count gauge
vpp_node_counter_count{agent="agent1",item="arp-input",reason="ARP replies sent"} 103
vpp_node_counter_count{agent="agent1",item="esp-encrypt",reason="ESP pkts received"} 124669
vpp_node_counter_count{agent="agent1",item="ethernet-input",reason="l3 mac mismatch"} 14
vpp_node_counter_count{agent="agent1",item="ip4-icmp-input",reason="echo replies sent"} 124669
vpp_node_counter_count{agent="agent1",item="ip4-icmp-input",reason="unknown type"} 3358
vpp_node_counter_count{agent="agent1",item="ipsec-input-ip4",reason="IPSEC pkts received"} 128027
vpp_node_counter_count{agent="agent1",item="ipsec-output-ip4",reason="IPSec policy protect"} 124669
```
    
### Configuration file

The telemetry plugin configuration file allows to change polling interval, or turn the polling off. The `polling-interval` is time in nanoseconds between reads from the VPP. Parameter `disabled` can be set to `true` in order to disable the telemetry plugin.    