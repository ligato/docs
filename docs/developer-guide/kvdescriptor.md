# KV Descriptors

---

This section describes KV Descriptors

---

## Introduction

KV Descriptors implement CRUD operations, and define derived values and dependencies for an individual value type. With these "descriptions", the [KV Scheduler](kvscheduler.md) is able to manipulate the key-value pairs in a generic fashion, without having to understand what they actually represent. The KV Scheduler processes the acquired dependency information, reads the SB state, and performs add, delete and modify operations as needed to synchronize NB with SB.

VPP and Linux plugins employ descriptors. Configuration items are described by their own descriptors inside the corresponding plugin. For example, there is a descriptor for [Linux interfaces][linux-interface-descr],  a descriptor for [VPP interfaces][vpp-interface-descr] and a descriptor for [VPP routes][vpp-route-descr]. Plugin-specific descriptors can be found in the `descriptor/` sub-folder of each VPP and Linux plugin.

KV Descriptors are based on the _mediator pattern_, where plugins are decoupled and no longer communicate directly with each other. Instead, interactions between plugins are handled through the KV Scheduler mediator. Any system in which configuration items are represented as key-value pairs, and operated through CRUD operations can implement the notion of KV Descriptors. It is not confined to VPP or Linux configuration operations.

!!! Terminology
    `Northbound (NB)` describes the desired or intended configuration state. `Southbound (SB)` describes the actual run-time configuration state. `Resync` is also referred to as state reconciliation. `CRUD` stands for create, read, update and delete. It standard nomenclature describing the basic actions performed by APIs. The KV Scheduler is referred to as the scheduler. KV Descriptors are also referred to as just `descriptors`.

---

### Descriptor API

A KV Descriptor is a structure that must be initialized with correct attribute values, and callbacks to CRUD operations. This approach was taken to reinforce the notion that descriptors are meant to be `stateless`. The state of values is instead maintained by the scheduler. To add and maintain extra run-time attributes alongside a value, the scheduler allows descriptors to append metadata. The state of the graph, with specific values and associated metadata determines what is executed in the SB plane. The complete system state is visible through REST APIs and formatted logs. More details can be found in the [Descriptor API][descriptor-api] definition.

The following describes the descriptor attributes. Optional fields can be left uninitialized (zero values).

* **Name** (string, mandatory)
    - descriptor names.
    - should be unique across all registered descriptors from all initialized plugins.
* **KeySelector** (callback, mandatory)
    - return `true` for keys identifying values described by a descriptor. Make sure the key matches the key template of the model.

!!! Alert
     Make sure your KeySelector function only returns `true` for keys from your descriptor's key space. Otherwise, your KeySelector function will be responding to key validation requests for keys that do not belong to your descriptor, and the behavior of the KV Scheduler will be undefined.
* **ValueTypeName** (string, mandatory for non-derived values)
    - name of the protobuf message that defines your model.
    - [example][value-type-name] showing how the proto message name can be obtained from the generated type.
* **KeyLabel** (callback, optional)
    - *optionally* will "shorten the key" and return a value identifier. Unlike the original key, this only needs to be unique in the key scope of the descriptor, and not necessarily in the entire key space.
    - if defined, the key label will be used as value identifier in the metadata map. For example, a request for interface metadata can be made using the interface name, rather than using a full key.
* **ValueComparator** (callback, optional)
    - optional customization for defining how to check two values for equivalency.
    - normally, the scheduler compares two values for the same key using `proto.Equal` to determine if a `Modify` operation is needed
    - in some cases, different values for the same field may be equivalent. For example, a default MTU 0 is equivalent MTU 1500. Therefore, an MTU 0 change to MTU 1500 should not trigger a `Modify`) operation.
* **NBKeyPrefix** (string, optional)
    - key prefix the scheduler should watch in NB to receive all values described by this descriptor. An example is etcd.
* **WithMetadata** (boolean, default is `false`)
    - `true` if configuration values carry run-time metadata.

!!! Note
    Metadata maintains extra state that may change with CRUD operations, or after agent restart, and cannot be determined just from the value itself. The sw_if_index metadata for an interface is an example. Metadata is often used to correlate NB configuration with dumped SB data.

* **MetadataMapFactory** (callback, optional)
    - Used to provide a customized map implementation for value metadata. Can be extended with secondary lookups.
    - if undefined, the scheduler will use the bare [NamedMapping][named-mapping] from the idxmap package.
* **Add** (callback, mandatory)
    - create new value operation.
* **Delete** (callback, mandatory)
    - delete an existing value operation.
* **Modify** (callback, mandatory unless update is always performed with full re-creation)
    - update an existing value operation.
* **ModifyWithRecreate** (callback, optional, `default` assumes re-creation is `not` needed)
       - used if the given value change requires full re-creation. In some cases, a re-creation is needed because SB does not support an update operation.
* **Update** (`DEPRECATED`)
* **IsRetriableFailure** (callback, optional)
    - informs the scheduler if the given error, returned by one of Add/Delete/Modify handlers, is immutable for the same value, thus non-retriable. Or if the value can be fixed by repeating the operation.
    - if undefined, an error will be considered retriable.
* **Dependencies** (callback, optional)
    - for a value with one or more dependencies, provides keys that must exist for the value to be created.
    - dependency can be specified using a specific key. Or using predicate `AnyOf` that must return `true` for at least one of the keys of the created values, for the dependency to be considered satisfied.
    - if undefined, the descriptor key-value pairs are assumed to have no dependencies.
* **DerivedValues** (callback, optional)
    - used to decompose the value into multiple pieces managed separately by different descriptors.
    - managed separately (possibly using its own dependencies) through custom CRUD operations. It can be used as a target for dependencies of other key-value pairs. For example, every [interface to be assigned to a bridge domain][bd-interface] is treated as a [separate key-value pair][bd-derived-vals], dependent on the [target interface to be created first][bd-iface-deps], but otherwise not blocking the rest of the bridge domain to be programmed. See this [control-flow][bd-cfd] demonstrating the order of operations needed to create a bridge domain.
* **Retrieve** (callback, optional)
      - read all values configured in the SB plane.
      - if not provided, it is assumed the dump operation is not supported. The state of SB for the given value type cannot be refreshed, and will be assumed to be current.
* **RetrieveDependencies** (slice of strings, optional)
      - list of descriptors to be dumped ahead of others
      - [for example][dump-deps-example], in order to dump routes, interfaces must be dumped first, to learn the mapping between interface names (NB ID) and their indexes (SB ID) from the metadata map of the interface plugin.

---

### Descriptor Adapter

One inconvenience that you will quickly discover when using this generalized approach,  of value description, is that the KVDescriptor API uses a bare `proto.Message` interface for values. This means you cannot define callbacks including add, modify, and delete for your model. Instead you must use `proto.Message` for input and output parameters, and perform all re-typing inside the callbacks.

To address this situation, the KV Scheduler is shipped with a utility called [descriptor-adapter][descriptor-adapter]. It generates an adapter for a given value type that will prepare and hide all the type conversions.

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

Once you have the adapter generated and CRUD callbacks prepared, you can initialize and register your descriptor.
First, import the adapter into the Go file with the descriptor, [assuming the suggested plugin directory layout][kvd-dir-layout] is used:
```
import "github.com/<your-organization>/<your-agent>/plugins/<your-plugin>/descriptor/adapter"
```

Next, add the constructor that will return your descriptor initialized and prepared for registration with the scheduler.
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

In order to obtain and expose the metadata map (if used), call [KV Scheduler.GetMetadataMap(< Descriptor-Name >)][get-metadata-map], after the descriptor has been registered. This will provide a map reference that can be exposed by the plugin's own API for other plugins to access read-only. An example for VPP interface metadata map can be found [here][get-metadata-map-example].

---

### Plugin Directory Layout

While it is not mandatory, we recommend that you follow the the directory layout used for all vpp-agent plugins:
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
* **descriptor** folder contains all descriptors implemented by your plugin. This includes those optionally adapted for a specific protobuf type with generated adapters nested further in the `adapter` sub-directory. Note that you should `NOT edit` adapter files.
* **metadata-map** if you define a custom metadata map by placing the implementation into a separate folder in the plugin's root folder, using `<model>idx` name.
* **< your-plugin.go >** is where you would implement the plugin interface (`Init`, `AfterInit`, `Close` methods) and register all the descriptors inside the `Init` phase.

It is an un-written rule to place the plugin constructor, some default options, and default dependency injections into the `<options.go>` file. For example, this [option.go][options-example] file can be found in the root folder of the VPP ifplugin plugin.

---

## Descriptor examples

### Descriptor skeletons

For a quick start, you may use prepared skeleton of a plugin with a single
descriptor, available in two variants:

 * [without leveraging the support for metadata][plugin-skeleton-withoutmeta]
 * [with metadata, including custom metadata index map][plugin-skeleton-withmeta]

!!! Note
    It is strongly recommended to use the prepared skeletons as reference only.

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
