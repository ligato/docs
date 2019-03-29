# Plugin configuration files

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

### List of supported flags

Description of all flags currently supported in the vpp-agent:

**Common configuration directory:**

```bash
-config-dir= 
```

Can be used to set the common location for all configuration files.

**Configurator:**

```bash
-configurator-config=
```

Flag reserved for configurator plugin, currently not in use.

**Consul plugin:**

```bash
-consul-config=
```

Provides all fields required for Consul plugin:
- `address`: IP Address of the consul server 
- `resync-after-reconnect`: this field runs resync procedure for all registered plugins in case the plugin losts connection to the database and then reconnects back 

**ETCD plugin:**

 // TBD
 









































