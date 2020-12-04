# Model Registration

---

This section describes model registration.

Package reference: [models](https://godoc.org/github.com/ligato/vpp-agent/pkg/models) 

---

### Concept

Every top-level `message` from a proto definition must have its own key. For example, `message ABF { ...` from the [abf.proto file](https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/abf/abf.proto) requires the generation and registration of the following key to access ABF information:
```
/vnf-agent/vpp1/config/vpp/abfs/v2/abf/<index>
```


To generate a key, you must define a model and proto definition. To review a model description, model spec, proto.Messages, and key types, see [model concepts](../user-guide/concepts.md#what-is-a-model).

---

### Registration

You register a model using the `models.Register()` function. This function stores your generated keys, paths and names in the registration map. 

`models.Register()` function: 

```go
models.Register(&ProtoMessage{}, models.Spec{
    Module:  "ModuleName",
    Type:    "protomessage",
    Version: "v2",
})
```

The example above registers the `ProtoMessage` type with the given `Spec`. The details of this function consist of the following:

* [proto.Message](../user-guide/concepts.md#protomessage) parameter contains your model's top-level message.
<br></br> 
* [model.Spec](../user-guide/concepts.md#model-specification) parameter contains your model's specification
<br></br>
* `Register`method returns a registration object, and stores it in the registration map. This simplifies access to various parts of the model key definition. For example, `KeyPrefix()` returns a key prefix; `IsKeyValid(<key>)` validates the key for the given model.

Example of ABF model registration:
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

---

### Custom Templates

The registration method defines a third optional parameter. It lets you pass options for a given registration of type `ModelOption`. The `models.WithNameTemplate(<template>))` option generates keys with customer identifiers using the `<template>` parameter.

The composition of the custom identifier can distinguish between different keys. For more information about custom identifiers, see [concepts](../user-guide/concepts.md#keys).  

---

**Example 1:**

The proto model defines a an `Index` field to serve as an identifier in the key. Use a custom template so that model registration includes this field in the key generation process:
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
    Model registration transforms the dots in the module name to slashes in the key. For example, module name of `ModuleName = "vpp.abfs"` becomes `vpp/abfs` in the key.

---
 
**Example 2:**

You can define models with a combination of fields. Consider a proto model with fields `Index` and `Tag`. The key must include both.

Define the template:
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




