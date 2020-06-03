# Model Registration

---

### Concept

Every top-level `message` from a proto definition is expected to have its own key. For example, `message ABF { ...` from the [abf.proto file](https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/abf/abf.proto) requires the generation and registration of a key for accessing ABF information.

A model and proto definition are prerequisites for generating a key. A description of the model, model spec, proto.Messages, and the various types of keys is provided in the [model concepts portion of the user guide](../user-guide/concepts.md#what-is-a-model).

### Registration

The model is registered to the VPP agent `spec.go`. The key, paths, and names are generated and stored in the registration map. An example registration is illustrated in the `models` package shown here.

```go
models.Register(&ProtoMessage{}, models.Spec{
    Module:  "ModuleName",
    Type:    "protomessage",
    Version: "v2",
})
```

The example above registers the `ProtoMessage` type with the given `Spec`. The first parameter is a [proto.Message](../user-guide/concepts.md#protomessage). The second parameter defines the [model specification](../user-guide/concepts.md#model-specification). The method `Register` also returns a registration object, which is stored in the registration map. This can be used for easy access to various parts of the model key definition. For example, `KeyPrefix()` returns a key prefix; `IsKeyValid(<key>)` validates the key for the given model.

Here is the registration for ABF:
```
var (
	ModelABF = models.Register(&ABF{}, models.Spec{
		Module:  "module",
		Version: "v2",
		Type:    "abf",
	}, models.WithNameTemplate("{{.Index}}"))
)
```
Generated long form key:
```
/vnf-agent/vpp1/config/vpp/abfs/v2/abf/<index>
```


### Custom Templates

The registration method defines a third optional parameter. It enables options to be passed for a given registration of type `ModelOption`. There is currently only one option available: `models.WithNameTemplate(<template>))`. This option enables keys with custom identifiers to be generated.

Keys may also be distinguished by the composition of the custom identifier as explained [here.](../user-guide/concepts.md#keys)

**Example 1:**

The proto model defines a field named `Index`, to serve as an identifier in the key. We use a custom template so that model registration will include this field in the key generation process.
```go
models.Register(&ProtoMessage{}, models.Spec{
    Module:  "module",
    Type:    "protomessage",
    Version: "v2",
}, models.WithNameTemplate("{{.Index}}"))
```

Generated `specific` long form key:
```
<microservice-label-prefix>/<config/status>/<module>/<version>/<type>/<index>
```
 
!!! note
    The module name dots are transformed to slashes in the key.  For example, `ModuleName = "vpp.abfs"` will be shown as `vpp/abfs` in the key.
 
**Example 2:**

Other models cannot be defined with a single unique field, but rather require a combination of fields. Consider a proto model with fields `Index` and `Tag`, both of which need to be included in the key.

Define the template as follows:
```go
models.Register(&ProtoMessage{}, models.Spec{
    Module:  "mp",
    Type:    "protomessage",
    Version: "v2",
}, models.WithNameTemplate("{{.Index}}/tag/{{.Tag}}"))
```

Generated `composite` long form key:
```
<microservice-label-prefix>/<config/status>/<module>/<version>/<type>/<index>/tag/<tag>
```

*[SPD]: Security Policy Database
*[VPP]: Vector Packet Processing



