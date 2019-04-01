# Table of contents
- [What is Model?](#model)
- [Model Spec](#spec)

## <a name="model">What is Model?</a>
The model represents a northbound API data model defined together with specific protobuf message. 
It is used to allow generating keys prefix and name of model instance using its value data. 
The key (prefix + name) is used for storing model in a key-value database.

### Model components
- model spec
- protobuf message (`proto.Message`)
- name template (optional)

Single protobuf message can only be represented by one model.

## <a name="spec">Model Spec</a>
Model spec (specification) describes particular model using module, version and type fields:
  - `module` - defines module, which groups models (vpp, linux..)
  - `version` - describes version of value data (e.g. v1)
  - `type` - describes type of model for humans (unique in module)

These three parts are used for generating model prefix. The model prefix uses following format:
```
config/<module>/<version>/<type>/
```