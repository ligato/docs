# Model Registration

---

### Concept

Every top-level message from northbound proto-definition is expected to have its own key which identifies a proto message data. The key can be composite (most often the case), containing some unique parameter (e.g. an interface name or a SPD index), or it can be global, allowing only one message of the given type to be stored under a global key. 

The vpp-agent defines model specification - a set of data related to the given model, like proto name, key prefix, or model path. Very important is the model specification (or `Spec`), which defines a module, version, and type of the proto model.  

**Module** groups models belonging for the same configuration entity. For example models for VPP configuration have `vpp` module, models for Linux configuration have `linux` module, etc.

**Version** is the current version of the vpp-agent API. The version changes only between major vpp-agent versions, not after every API update.

**Type** is a keyword describing the given model (like interfaces, bridge-domains, etc.).

The constructed key has the following format:
```
<key-prefix>/<config/status>/<module-path>/<version>/<type>/<optional-identifiers>
``` 

### Registration

Model is registered to the vpp-agent `spec.go`, where the key, paths, and names are generated and stored to the registration map. The example registration is done via the `models` package:

```go
models.Register(&ProtoMessage{}, models.Spec{
    Module:  "mp",
    Type:    "protomessage",
    Version: "v2",
})
```

The example above registers the `ProtoMessage` type with the given `Spec`. The first parameter is expected to be of the `proto.Message` interface type, the second parameter defines the specification itself. The method `Register` also returns a registration object (which is stored to the registration map in the background) which can be used for easy access to various parts of the model key definition. For example `KeyPrefix()` returns key prefix, `IsKeyValid(<key>)` validates provided key for the given model, etc.

### Custom Templates

Registration method has a third optional parameter, which allows passing options for given registration of type `ModelOption`. There is currently only one option available - `models.WithNameTemplate(<template>))`. It allows generating the key with custom identifiers. 

**Example 1:**
A proto model defines a field named `Index` which should be a part of the key as an identifier. We define a custom template to let model registration know that the field should be used:
```go
models.Register(&ProtoMessage{}, models.Spec{
    Module:  "mp",
    Type:    "protomessage",
    Version: "v2",
}, models.WithNameTemplate("{{.Index}}"))
```

The result:
```
<key-prefix>/<config/status>/<module-path>/<version>/<type>/<index>
```
 
!!! note
    The module path dots are transformed to slashes (`vpp.interfaces` will be shown as `vpp/interfaces` in the key) 
 
**Example 2:**
Another model cannot be defined with a single unique field, but required a combination of them. Let's assume a proto model with fields `Index` and `Tag` which both need to be used in the key. Define the template as follows:
```go
models.Register(&ProtoMessage{}, models.Spec{
    Module:  "mp",
    Type:    "protomessage",
    Version: "v2",
}, models.WithNameTemplate("{{.Index}}/tag/{{.Tag}}"))
```

The result:
```
<key-prefix>/<config/status>/<module-path>/<version>/<type>/<index>/tag/<tag>
```

*[SPD]: Security Policy Database
*[VPP]: Vector Packet Processing



