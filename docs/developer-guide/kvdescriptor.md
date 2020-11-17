# KV Descriptors

---

This section describes KV Descriptors.

---

## Introduction

KV Descriptors implement CRUD operations, and define derived values and dependencies for an individual value type. With these "descriptions", the [KV Scheduler](kvscheduler.md) manipulates the key-value pairs , without having to understand what they actually represent. The KV Scheduler processes dependency information, reads the SB state, and performs add, delete and modify operations to synchronize NB with SB.

VPP and Linux plugins employ descriptors. Plugins use descriptors to describe their own configuration items. For example, the Linux interface plugin defines [Linux interface descriptors][linux-interface-descr]; the VPP interface plugin uses[VPP interface descriptors][vpp-interface-descr]; VPP L3 plugin uses [VPP routes][vpp-route-descr] descriptors. The `/descriptor` folder for each VPP and Linux plugin contains their own descriptors.

KV Descriptors employ a [mediator pattern](https://en.wikipedia.org/wiki/Mediator_pattern), where plugins are decoupled and no longer communicate directly with each other. Instead, any interactions between plugins occur through the KV Scheduler mediator.

!!! Note
    Any system you build where key-value pairs define configuration items, and CRUD callback operations, can implement the notion of KV Descriptors. Your system is not confined to VPP or Linux operations.



---

### Descriptor API

You must initialize a KV Descriptor with the correct attribute values, and CRUD operation callbacks. This approach reinforces the notion of `stateless` descriptors. The KV Scheduler maintains value state, not the KV Desciptors.  

The KV scheduler allows descriptors to append metadata. The metadata consists of extra run-time attributes for specific values. The KV Scheduler maintains a graph structure containing values and metadata. The values and meta determine SB CRUD operations. You can inspect the complete system state and metadata using agentctl, REST APIs or formatted logs.  

The following describes the [KV Descriptor attributes](https://github.com/ligato/vpp-agent/blob/28f384dca9eff9319e24fed28ed973b580576320/plugins/kvscheduler/api/kv_descriptor_api.go#L133) type struct you can implement in your application. You do not have to initialize optional fields. 

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
    - proto.RegisterType() registers proto.Messages against this `ValueTypeName` in the generated code.
</br>
</br>

* **KeyLabel** (callback, optional)
    - _Optionally_ "shortens the key" and returns a value identifier. Unlike the original key, you only need to ensure uniqueness in the descriptor's key scope, not in the entire key space.
    - If you define the `KeyLabel`, the metadata and non-verbose logs use this as a value identifier. For example, an interface metadata request can use interface name, rather than a full key.
</br>
</br>
* **ValueComparator** (callback, optional)
    - Optional customization defining how you check two values for equivalency.
    - Normally, the KV Scheduler compares two values for the same key using `proto.Equal` to determine need for a `Modify` operation.
    - In some cases, different values for the same field may are equivalent. For example, a default MTU 0 is equivalent to an MTU 1500. Therefore, an MTU 0 change to MTU 1500 should not trigger a `Modify` operation.
</br>
</br>
* **NBKeyPrefix** (string, optional)
    - Defines key prefix the KV Scheduler should watch in NB to receive all values described by this descriptor. 
    - Conversely, `KeySelector` can select extra SB-only values, which the KV Scheduler will not watch from the NB. 
    - You do not need this prefix again, if the KV Scheduler is watching these keys for another descriptor within the same plugin.may select some extra
</br>
</br>
* **WithMetadata** (boolean, default is `false`)
    - `true` if configuration values carry runtime metadata.
    - KV Scheduler maintains map between the key and metadata.
    - If `false`, `Create()` ignores returned metadata, and other methods receive nil metadata.  

!!! Note
    Metadata maintains extra state that could change with CRUD operations, or after agent restart. It cannot cannot be determined just from the value itself. You will often use metadata to correlate NB configuration with dumped SB data.

* **MetadataMapFactory** (callback, optional)
    - Provides a customized map implementation for value metadata. You can extend this callback with secondary lookups.
    - If you do not define this callback, the KV Scheduler uses the bare [NamedMapping][named-mapping] from the idxmap package.
</br>
</br>
* **Add** (callback, mandatory)
    - Create new value operation.
    - Descriptor returns metadata for non-derived values.
    - Create, Delete, and Update functions are optional for derived values. 
    - You typically implement base value properties as empty derived values. SB operations are not attached and not used as dependency targets.  
</br>
</br>
* **Delete** (callback, mandatory)
    - Delete an existing value handler.
    - You must define delete if you define create 


* **Modify** (callback, mandatory unless full re-creation always performs update)
    - Update an existing value handler.
    - Value handler is optional. If you do not the value handler, perform a full delete+creat re-creation.
    - New metadata can re-use old metadata.


* **ModifyWithRecreate** (callback, optional), 
    - Use this callback if the given value change requires full re-creation.
    - Tell the KV Scheduler if <oldValue> to <newValue> change requires full re-creation.
    - If you do not define this callback, the KV Scheduler decides based on availability of the `Update()` operation.
    - In some cases, you require full recreation because SB does not support an `Update()` operation.
    - `default` assumes re-creation is _not_ needed.
 
* **Update** (`DEPRECATED`)
</br>
</br>

* **IsRetriableFailure** (callback, optional)
    - Tells the KV Scheduler if the given error, returned by one of Add/Delete/Modify handlers, is immutable for the same value.
    - Also tells the KV Scheduler if the value can be fixed by repeating the operation.
    - If you do not define this callback, the KV Scheduler assumes an error is retriable.
</br>
</br>

!!! Note
    Dependencies are keys that must already exist for the value to be created. Conversely, if a dependency is removed, all dependent values are deleted first, and cached for a potential future re-creation. Dependencies returned in the list are AND-ed.

* **Dependencies** (callback, optional)
    - for a value with one or more dependencies, provides keys that must exist for the value to be created.
    - You provide keys that must exist to create a value with one or more dependencies.
    - You can specify a dependency using a specific key.
    - You can also use the `AnyOf` predicate that must return `true` for at least one of the keys of the created values. You will then satisfy the dependency. 
    - Your descriptor key-value pairs have no dependencies if you do not define any dependencies.
    
!!! Note
    DerivedValues returns ("derived") values solely inferred from the current state of this ("base") value. Derived values cannot be changed by NB transaction. While their state and existence is bound to the state of their base value, they are allowed to have their own descriptors.</br>Typically, derived value represents the base value's properties that other key-value pairs depend on, or extra actions taken when additional dependencies are met. Otherwise, the base value creation is not blocked.  



* **DerivedValues** (callback, optional)
    - used to decompose the value into multiple pieces managed separately by different descriptors.
    - managed separately (possibly using its own dependencies) through custom CRUD operations. It can be used as a target for dependencies of other key-value pairs. For example, every [interface to be assigned to a bridge domain][bd-interface] is treated as a [separate key-value pair][bd-derived-vals], dependent on the [target interface to be created first][bd-iface-deps], but otherwise not blocking the rest of the bridge domain to be programmed. See this [control-flow][bd-cfd] demonstrating the order of operations needed to create a bridge domain.
</br>
</br>
* **Retrieve** (callback, optional)
      - You can read all non-derived values _actually configured_ in the SB plane. The `DerivedValues()` method automatically creates a derived value. The `DerivedValues()` method does not return a derived value if the non-derived value of the base property does not actually exist. 
      - If you do not define this callback, the descriptor cannot read the current SB state. Refresh cannot be performed if its key-value pairs.  

* **RetrieveDependencies** (slice of strings, optional)
      - Use this callback to list the descriptors of values to read prior to performing a `Retrieve()` of the descriptor. 
       - [For example][dump-deps-example], you must dump interfaces first before dumping routes. The system can then learn the mapping between interface names (NB ID), and their indexes (SB ID), from the metadata map of the interface plugin.

---

### Descriptor Adapter

You will quickly discover the inconvenience of the KVDescriptor API using a bare `proto.Message` interface for values. This means you cannot define callbacks including `Add()`, `Modify()`, and `Delete` for your model. Instead, you must use `proto.Message` for input and output parameters, and perform all re-typing inside the callbacks. 

<remove>
One inconvenience that you will quickly discover when using this generalized approach,  of value description, is that the KVDescriptor API uses a bare `proto.Message` interface for values. This means you cannot define callbacks including `Add()`, `Modify()`, and `Delete` for your model. Instead you must use `proto.Message` for input and output parameters, and perform all re-typing inside the callbacks.
</remove>

To address this situation, the KV Scheduler ships with a [descriptor-adapter][descriptor-adapter] utility. It generates an adapter for a given value type that prepares and hides all type conversions.

The tool can be installed with:
```
make get-desc-adapter-generator
```

To generate an adapter for your descriptor, put the `go:generate` command for `descriptor-adapter` into your plugin's main.go file:
```
//go:generate descriptor-adapter --descriptor-name <your-descriptor-name>  --value-type <your-value-type-name> [--meta-type <your-metadata-type-name>] [--import <IMPORT-PATH>...] --output-dir "descriptor"
```
For example, `go:generate` for the VPP interface can be found [here][vpp-iface-adapter].
The import paths must include packages with your own value's data type definitions and any metadata, if used. The import path can be relative to the file with the `go:generate` command, hence the plugin's root folder is prefered.
Running `go generate <your-plugin-path>` will generate the adapter for your descriptor and place it into the `adapter` sub-directory.

---

### Registering Descriptor

Once you generate the adapter and prepare CRUS callbacks, you can initialize and register your descriptor.
First, import the adapter into the Go file with the descriptor, [assuming the suggested plugin directory layout][kvd-dir-layout] is used:
```
import "github.com/<your-organization>/<your-agent>/plugins/<your-plugin>/descriptor/adapter"
```

Next, add the constructor that returns your initialized descriptor initialized and prepared for registration with the scheduler.
The adapter will present the KV Descriptor API with value type and metadata type data already casted to your own data types for all fields:
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

Next, inside the `Init` method of your plugin, import the package with all of your descriptors and register them using the [KVScheduler.RegisterKVDescriptor(< descriptor > )][register-kvdescriptor] method:

```
import "github.com/<your-organization>/<your-agent>/plugins/<your-plugin>/descriptor"

func (p *YourPlugin) Init() error {
    yourDescriptor1 = descriptor.New<descriptor-name>Descriptor(<args>)
    p.Deps.KVScheduler.RegisterKVDescriptor(yourDescriptor1)
    ...
}
```

As you can see, the KV Scheduler becomes a plugin dependency, which needs to be properly injected:
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

In order to obtain and expose the metadata map (if used), call [KV Scheduler.GetMetadataMap(< Descriptor-Name >)][get-metadata-map], after you register the descriptor. This will provide a map reference that can be exposed by the plugin's own API for other plugins to access read-only. An example for VPP interface metadata map can be found [here][get-metadata-map-example].

---

### Plugin Directory Layout

While not mandatory, we recommend using the following directory layout for all vpp-agent plugins:
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

* **model** folder contains your proto models and generated code.
</br>
</br>
* **descriptor** folder contains all descriptors implemented by your plugin. This includes those optionally adapted for a specific protobuf type with generated adapters nested further in the `adapter` sub-directory. Note that you should `NOT edit` adapter files.
</br>
</br>
* **metadata-map** if you define a custom metadata map by placing the implementation into a separate folder in the plugin's root folder, using `<model>idx` name.
</br>
</br>
* **< your-plugin.go >** is where you implement the plugin interface (`Init`, `AfterInit`, `Close` methods) and register all the descriptors inside the `Init` phase.

We also recommend you place the plugin constructor, default options, and default dependency injections in the `<options.go>` file. For example, this [option.go][options-example] file can be found in the root folder of the VPP ifplugin plugin.

---

## Descriptor examples

### Descriptor skeletons

For a quick start, you may use prepared skeleton of a plugin with a single
descriptor, available in two variants:

 * [without leveraging the support for metadata][plugin-skeleton-withoutmeta]
 * [with metadata, including custom metadata index map][plugin-skeleton-withmeta]

!!! Note
    Use the prepared skeletons as reference only.

---

### Mock SB

We have prepared an [interactive hands-on example][mock-plugins-example],
demonstrating the KV Scheduler framework in action. It uses replicated `vpp/ifplugin` and
`vpp/l2plugin` plugins under various scenarios. The models are simplified and VPP
is replaced with a mock SB plane. The triggered CRUD operations are printed to
the stdout and not executed. The example is focused on the scheduler and descriptors. It also illustrates that with this level of abstraction, the makeup of the actual SB plane is not required.

---

### Real-world Examples

Since all VPP and Linux plugins employ the KV Scheduler framework, there are
multiple descriptors in the repository to inspect and clone. Although
interfaces are the basis of network configuration, it is suggested that you first examine
descriptors for simpler objects, such as [VPP ARPs][vpp-arp-descr] and 
[VPP routes][vpp-route-descr]. These use simple CRUD methods, and a single
dependency on the associated interface.

Then, you can learn how to break a more complex
object into multiple values using [bridge domains][vpp-bd-descr] and 
[BD-interface bindings][vpp-bd-iface-descr] as examples. One can be derived for each
interface assigned into the domain.

Finally, check out the
[Linux interface watcher][linux-iface-watcher], which shows that values may enter
the graph even from below as SB notifications, and used as [targets for dependencies][afpacket-dep]
by other objects.

These descriptors cover most of the features and should help you get started
implementing your own.

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
