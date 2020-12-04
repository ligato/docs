# KV Descriptors

---

This section describes KV Descriptors.

---

## Introduction

KV Descriptors implement CRUD operations, and define derived values and dependencies for an individual value type. With these "descriptions", the [KV Scheduler](kvscheduler.md) manipulates the key-value pairs , without having to understand what they actually represent. The KV Scheduler processes dependency information, reads the SB state, and performs add, delete and modify operations to synchronize NB with SB.

VPP and Linux plugins use descriptors to describe their own configuration items. For example, the Linux interface plugin use [Linux interface descriptors][linux-interface-descr]; the VPP interface plugin use [VPP interface descriptors][vpp-interface-descr]; the VPP L3 plugin use [VPP routes descriptors][vpp-route-descr] descriptors. The `/descriptor` folder for each VPP and Linux plugin contains their own descriptors.

KV Descriptors employ a [mediator pattern](https://en.wikipedia.org/wiki/Mediator_pattern), where plugins are decoupled, and do not directly communicate with each other. Instead, any interactions between plugins occur through the KV Scheduler mediator.

!!! Note
    Any system you build where key-value pairs define configuration items, and perform CRUD callback operations, can implement the notion of KV Descriptors. You do not need to confine your system to VPP or Linux operations.


---

### Descriptor API

You initialize a KV Descriptor with the correct attribute values, and CRUD operation callbacks. This approach reinforces the notion of `stateless` descriptors. The KV Scheduler maintains value state, not the KV Desciptors.  

The KV scheduler allows descriptors to append metadata. The metadata consists of extra run-time attributes for specific values. The KV Scheduler maintains a graph structure containing values and metadata. The values and metadata determine SB CRUD operations. You can inspect the complete system state and metadata using [agentctl](../user-guide/agentctl.md) and [REST APIs](../api/api-kvs.md).  

The following describes the [KV Descriptor attributes](https://github.com/ligato/vpp-agent/blob/28f384dca9eff9319e24fed28ed973b580576320/plugins/kvscheduler/api/kv_descriptor_api.go#L133) type struct you implement in your application. You do not have to initialize optional fields. 

* **Name** (string, mandatory)
    - Descriptor name unique across all registered descriptors from all initialized plugins.
</br>
</br>

* **KeySelector** (callback, mandatory)

    - Returns `true` for keys from your descriptor's key space.
</br>
</br>   

* **ValueTypeName** (string, mandatory for non-derived values)
    - proto.Message that defines your model.
<br></br>
    - proto.RegisterType() registers proto.Messages against this `ValueTypeName` in the generated code.
</br>
</br>

* **KeyLabel** (callback, optional)
    - _Optionally_ "shortens the key" and returns a value identifier. Unlike the original key, you only need to ensure uniqueness in the descriptor's key scope, not in the entire key space.
<br></br>
    - If you define the `KeyLabel`, the metadata and non-verbose logs use this as a value identifier. For example, an interface metadata request can use interface name, rather than a full key.
</br>
</br>
* **ValueComparator** (callback, optional)

    - Optional customization defining how you check two values for equivalency.
<br></br>
    - Normally, the KV Scheduler compares two values for the same key using `proto.Equal` to determine need for a `Modify` operation.
<br></br>
    - In some cases, different values for the same field may are equivalent. For example, a default MTU 0 is equivalent to an MTU 1500. Therefore, an MTU 0 change to MTU 1500 should not trigger a `Modify` operation.
</br>
</br>

* **NBKeyPrefix** (string, optional)
    - Defines key prefix the KV Scheduler should watch in NB to receive all values described by this descriptor. 
<br></br>
    - Conversely, `KeySelector` selects extra SB-only values, that the KV Scheduler will not watch from the NB.
<br></br> 
    - You do not need this prefix again, if the KV Scheduler watches these keys for another descriptor within the same plugin.
</br>
</br>
* **WithMetadata** (boolean, default is `false`)

    - `true` if configuration values carry runtime metadata.
<br></br>
    - KV Scheduler maintains map between the key and metadata.
<br></br>
    - If `false`, the `Create()` operation ignores returned metadata, and other methods receive nil metadata.  

!!! Note
    Metadata maintains extra state that could change with CRUD operations, or after agent restart. You cannot determine the extra state from just from the value itself. You will often use metadata to correlate NB configuration with dumped SB data.

* **MetadataMapFactory** (callback, optional)

    - Provides a customized map implementation for value metadata. You can extend this callback with secondary lookups.
<br></br>
    - If you do not define this callback, the KV Scheduler uses the bare [NamedMapping][named-mapping] from the idxmap package.
</br>
</br>

* **Add** (callback, mandatory)
    - Create new value operation.
<br></br>
    - Descriptor returns metadata for non-derived values.
<br></br>
    - Create, Delete, and Update functions are optional for derived values.
<br></br> 
    - You typically implement base value properties as empty derived values. SB operations are not attached and not used as dependency targets.  
</br>

* **Delete** (callback, mandatory)

    - Delete an existing value handler.
<br></br>
    - You must define delete if you define create 
</br>
</br>

* **Modify** (callback, mandatory unless full re-creation always performs update)

    - Update an existing value handler.
<br></br>
    - Value handler is optional. If you do not use the value handler, perform a full delete+creat re-creation.
<br></br>
    - New metadata can re-use old metadata.
</br>
</br>

* **ModifyWithRecreate** (callback, optional)
 
    - Use this callback if the given value change requires full re-creation.
<br></br>
    - Tell the KV Scheduler if <oldValue> to <newValue> change requires full re-creation.
<br></br>
    - If you do not define this callback, the KV Scheduler decides based on `Update()` operation availability.
<br></br>
    - In some cases, you require full recreation because SB does not support an `Update()` operation.
<br></br>
    - `default` assumes re-creation is _not_ needed.
</br>
</br>
 
* **Update** (`DEPRECATED`)
</br>
</br>

* **IsRetriableFailure** (callback, optional)

    - Tells KV Scheduler if the given error, returned by one of Add/Delete/Modify handlers, is immutable for the same value.
<br></br>
    - Tells KV Scheduler if the value can be fixed by repeating the operation.
<br></br>
    - If you do not define this callback, the KV Scheduler assumes a retriable error.
</br>
</br>

!!! Note
    Dependencies are keys that must already exist for the value to be created. Conversely, if you remove dependency, all dependent values are deleted first, and cached for a potential future re-creation.<br></br>Dependencies returned from the list are AND-ed.

* **Dependencies** (callback, optional)

    - For a value with one or more dependencies, provides keys that must exist for you to create the value.
<br></br>
    - You provide keys that must exist to create a value with one or more dependencies.
<br></br>
    - You can specify a dependency using a specific key.
<br></br>
    - You can use the `AnyOf` predicate that must return `true` for at least one of the keys of the created values. You will then satisfy the dependency.
<br></br> 
    - Your descriptor key-value pairs have no dependencies if you do not define any dependencies.
    
!!! Note
    DerivedValues returns ("derived") values inferred from the current state of the ("base") value. You cannot change derived values with NB transaction. While their state and existence is bound to the state of their base value, they can have their own descriptors.<br></br>Typically, a derived value represents the base value's properties that other key-value pairs depend on, or extra actions you perform when meeting additional dependencies . Otherwise, the base value creation is not blocked.  



* **DerivedValues** (callback, optional)

    - Decompose the value into multiple pieces managed separately by different descriptors.
<br></br>
    - Managed separately (possibly using its own dependencies) through custom CRUD operations. You can use as a target for other key-value pair dependencies.<br></br>For example, you treat every [interface assigned to a bridge domain][bd-interface] as a [separate key-value pair][bd-derived-vals], dependent on the [target interface created first][bd-iface-deps], but otherwise not blocking the remaining bridge domain programming. For more details on the operational flow, see [bridge domain control flow][bd-cfd].
</br>
</br>

* **Retrieve** (callback, optional)

      - You can read all non-derived values configured in the SB data plane. The `DerivedValues()` method automatically creates a derived value. The `DerivedValues()` method does not return a derived value if the non-derived value of the base property does not exist.
  <br></br> 
      - If you do not define this callback, the descriptor cannot read the current SB state.   
</br>
</br>

* **RetrieveDependencies** (slice of strings, optional)

      - Use this callback to list the value descriptors to read prior to performing a `Retrieve()` of the descriptor.
  <br></br> 
       - For example, you must first dump interfaces before dumping routes. The system then learns the mapping between interface names (NB ID), and their indexes (SB ID), from the interface plugin metadata map.

---

### Descriptor Adapter

You will quickly discover the inconvenience of the KV Descriptor API using a bare `proto.Message` interface for values. This means you cannot define callbacks including `Create()`, `Update()`, and `Delete()` for your model. Instead, you must use `proto.Message` for input and output parameters, and perform all re-typing inside the callbacks. 

To address this situation, the KV Scheduler ships with a [descriptor-adapter][descriptor-adapter] utility. It generates an adapter for a given value type that prepares and hides all type conversions.

Install the descriptor-adapter tool:
```
make get-desc-adapter-generator
```

To generate an adapter for your descriptor, put the `go:generate` command for `descriptor-adapter` into your plugin's main.go file:
```
//go:generate descriptor-adapter --descriptor-name <your-descriptor-name>  --value-type <your-value-type-name> [--meta-type <your-metadata-type-name>] [--import <IMPORT-PATH>...] --output-dir "descriptor"
```
For example, the [VPP interface adapter][vpp-iface-adapter] contains the `go:generate` statements for the VPP interface.

The import paths must include packages with your own value's data type definitions and any metadata you might use. The import path is relative to the file with the `go:generate` command, so use the plugin's root folder if possible.

Running `go generate <your-plugin-path>` generates your descriptor's adapter and places it in the `adapter/` sub-directory.

---

### Registering Descriptor

Once you generate the adapter and prepare CRUD callbacks, you can initialize and register your descriptor.

First, import the adapter into the Go file with the descriptor. Use the [suggested plugin directory layout][kvd-dir-layout]:
```
import "github.com/<your-organization>/<your-agent>/plugins/<your-plugin>/descriptor/adapter"
```

Next, add the constructor that returns your descriptor initialized and prepared for registration with the KV Scheduler.
The adapter presents the KV Descriptor API with value type and metadata type data casted to your own data types for all fields:
```
func New<your-descriptor-name>Descriptor(<args>) *adapter.<your-descriptor-name>Descriptor {
	return &adapter.<your-descriptor-name>Descriptor{
		Name:        <your-descriptor-name>,
		KeySelector: <your-key-selector>,
                Add:         <your-Add-operation-implementation>,
                // etc., fill all the mandatory fields or whenever the default value is not suitable
	}
}
```

Finally, inside your plugin's `Init()` method, import the package with your descriptors. Then use the  [RegisterKVDescriptor(< descriptor > )][register-kvdescriptor] method them.

```
import "github.com/<your-organization>/<your-agent>/plugins/<your-plugin>/descriptor"

func (p *YourPlugin) Init() error {
    yourDescriptor1 = descriptor.New<descriptor-name>Descriptor(<args>)
    p.Deps.KVScheduler.RegisterKVDescriptor(yourDescriptor1)
    ...
}
```

As you can see, the KV Scheduler becomes a plugin dependency that requires proper injection:
```
\\\\ plugin main go file:
import kvs "github.com/ligato/vpp-agent/plugins/kvscheduler/api"

type Deps struct {
    infra.PluginDeps
    KVScheduler kvs.KVScheduler
    ...
}

\\\\ options.go
import "github.com/ligato/vpp-agent/plugins/kvscheduler"

func NewPlugin(opts ...Option) *<your-plugin> {
    p := &<your-plugin>{}
    // ...
    p.KVScheduler = &kvscheduler.DefaultPlugin
    // ...
    return p
}
```

After you register the descriptor, call the [GetMetadataMap(< Descriptor-Name >)][get-metadata-map] method to access the metadata map. This provides you with a map reference that exposes your plugin's API to other plugins using read-only access. 

The VPP interface plugin [ifplugin.go][get-metadata-map-example] file contains an example of the interface metadata map. 

---

### Plugin Directory Layout

!!! Tip
    While not mandatory, we recommend using the following directory layout for all VPP agent plugins.
   
```
<your-plugin>/
├── model/  // + generated code
│   ├── model1.proto
│   ├── model2.proto
│   ├── ...
│   └── <modeln>.proto
├── descriptor/
│   ├── adapter/
│   │   ├── <generated-adapter-for-every-descriptor>...
│   ├── <descriptor-for-model1>.go
│   ├── <descriptor-for-model2>.go
│   ├── ...
│   └── <descriptor-for-modeln>.go
├── <metadata-map> // if custom secondary index over metadata is needed
│   └── <map-impl>.go
├── <your-plugin>.go
└── options.go
```

* **model/** folder contains your proto models and generated code.
</br>
</br>
* **descriptor/** folder contains all descriptors implemented by your plugin. This includes those optionally adapted for a specific protobuf type with generated adapters nested further in the `adapter` sub-directory. You should _NOT_ edit adapter files.
</br>
</br>
* **metadata-map/** if you define a custom metadata map. Place your implementation into a separate folder in the plugin's root folder, using `<model>idx` name.
</br>
</br>
* **< your-plugin.go >** file contains your implementation of the plugin interface, and where you register all descriptors inside the `Init()` phase.


!!! Tip
    We recommend you place the plugin constructor, default options, and default dependency injections in the `<options.go>` file. For example, the root folder of the VPP interface plugin contains the [option.go][options-example] file. .

---

## Descriptor examples

### Descriptor skeletons

To get started on descriptors, use one or both variants of the prepared plugin skeletons that implement a single descriptor: 

 * [without leveraging the metadata support][plugin-skeleton-withoutmeta]
<br></br>
 * [with metadata, including custom metadata index map][plugin-skeleton-withmeta]

!!! Note
    Use the prepared skeletons as reference only.

---

### Mock SB

The repository contains a [hands-on example][mock-plugins-example],
demonstrating the KV Scheduler framework in action. It uses replicated `vpp/ifplugin` and
`vpp/l2plugin` plugins operating in various scenarios. The example simplifies the model, and a mock SB replaces the VPP data plane. It does not execute CRUD operations, but does print them to the transaction log stdout. 

The example focuses on the scheduler and descriptors. It also illustrates that with this level of abstraction, you do not require SB implementation details.

---

### Real-world Examples

Since all VPP and Linux plugins use the KV Scheduler, the repository contains 
multiple descriptors to inspect and clone. Although interfaces lay the groundwork for network configurations, you should first examine
descriptors for simpler objects, such as [VPP ARPs][vpp-arp-descr] and 
[VPP routes][vpp-route-descr]. These use simple CRUD methods, and a single
dependency on the associated interface.

Next, learn how to break a more complex object into multiple values using [bridge domains][vpp-bd-descr] and [BD-interface bindings][vpp-bd-iface-descr] as examples. You can derive one for each
interface assigned to the domain. 

Finally, look over the
[Linux interface watcher][linux-iface-watcher]. It shows that the graph can add values obtained from SB notifications, and used as [targets for dependencies][afpacket-dep]
by other objects.

[afpacket-dep]: https://github.com/ligato/vpp-agent/blob/e8e54ef67b666e57ffef1bca555c8ce5585f215f/plugins/vpp/ifplugin/descriptor/interface.go#L421-L426
[bd-cfd]: control-flows.md#example-bridge-domain
[bd-derived-vals]: https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/l2plugin/descriptor/bridgedomain.go
[bd-iface-deps]: https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/l2plugin/descriptor/bd_interface.go
[bd-interface]: https://github.com/ligato/vpp-agent/blob/e8e54ef67b666e57ffef1bca555c8ce5585f215f/api/models/vpp/l2/bridge-domain.proto#L19-L24
[descriptor-api]: https://github.com/ligato/vpp-agent/blob/master/plugins/kvscheduler/api/kv_scheduler_api.go
[descriptor-adapter]: https://github.com/ligato/vpp-agent/tree/master/plugins/kvscheduler/descriptor-adapter
[dump-deps-example]: https://github.com/ligato/vpp-agent/blob/master/plugins/linux/l3plugin/descriptor/route.go
[existing-descriptors]: ../developer-guide/kvdescriptor.md
[get-metadata-map]: https://github.com/ligato/vpp-agent/blob/master/plugins/kvscheduler/api/kv_scheduler_api.go
[get-metadata-map-example]: https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/ifplugin/ifplugin.go
[kv-scheduler]: ../plugins/kvs-plugin.md
[kvd-dir-layout]: kvdescriptor.md#plugin-directory-layout
[linux-iface-watcher]: https://github.com/ligato/vpp-agent/blob/master/plugins/linux/ifplugin/descriptor/watcher.go
[linux-interface-descr]: https://github.com/ligato/vpp-agent/blob/master/plugins/linux/ifplugin/descriptor/interface.go
[mock-plugins-example]: https://github.com/ligato/vpp-agent/blob/master/examples/kvscheduler/mock_plugins/README.md
[named-mapping]: https://github.com/ligato/cn-infra/blob/master/idxmap/mem/inmemory_name_mapping.go
[options-example]: https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/ifplugin/options.go
[plugin-skeleton-withmeta]: https://github.com/ligato/vpp-agent/blob/master/examples/kvscheduler/plugin_skeleton/with_metadata/plugin.go
[plugin-skeleton-withoutmeta]: https://github.com/ligato/vpp-agent/blob/master/examples/kvscheduler/plugin_skeleton/without_metadata/plugin.go
[register-kvdescriptor]: https://github.com/ligato/vpp-agent/blob/ff693d160bb2f36a11800a263cbb6b1557a75a65/plugins/kvscheduler/api/kv_scheduler_api.go#L201
[value-type-name]: https://github.com/ligato/vpp-agent/blob/master/plugins/linux/ifplugin/descriptor/interface.go#L144
[vpp-arp-descr]: https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/l3plugin/descriptor/arp_entry.go
[vpp-bd-iface-descr]: https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/l2plugin/descriptor/bd_interface.go
[vpp-bd-descr]: https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/l2plugin/descriptor/bridgedomain.go
[vpp-iface-adapter]: https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/ifplugin/ifplugin.go
[vpp-interface-descr]: https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/ifplugin/descriptor/interface.go
[vpp-route-descr]: https://github.com/ligato/vpp-agent/blob/master/plugins/vpp/l3plugin/descriptor/route.go

*[MTU]: Maximum Transmission Unit
*[REST]: Representational State Transfer
*[VPP]: Vector Packet Processing
