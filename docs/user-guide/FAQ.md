# Frequently Asked Questions

The list of frequently asked questions (feel free to add your own).

Any issue or something is not clear? Try to find a quick solution here. 
* [What is ...](#whatis)
* [Start-up](#startup)
* [Configuration (plugins)](#config)
* [KVDB (database)](#kvdb)
* [GRPC](#grpc) 
* [REST](#rest) 
* [The issue I have is not listed here](#notlisted)

## <a name="whatis">What is ...</a>

#### ... the vpp-agent?
The vpp-agent is control/management plane for VPP-based cloud-native virtual network functions. It is built on the CN-Infra framework providing generic plugins (KVDB connection, Kafka connection, generic GRPC, HTTP, etc). The vpp-agent itself consist from plugins taken from the CN-Infra, but also defines its own plugins for the VPP management. Thus the vpp-agent can also be used as an extended framework for VPP-based solutions.

#### ... the cn-infra?
The CN-Infra is a set of plugins/libraries providing various functionality, most importantly the connection to different KVDB stores. The CN-Infra also defines the plugin infrastructure providing intuitive mechanism to load various plugins and solve dependencies between them.

#### ... the plugin?
The plugin is a structure fulfilling the CN-Infra plugin definition (implements the plugin interface). Such a structure can be used in the plugin lifecycle management and started together with other native plugins.

#### ... the resync?
The resync is a synchronization procedure which ensure the consistent initial state between the KVDB (if used), the vpp-agent and the VPP. Usually it is called at startup, but can be called on other events (like VPP restart). The resync reads the current VPP configuration and adds or removes configuration items according to the northbound data. The resync ensures consistent VPP state at any point.  

#### ... the microservice label?
The microservice label is unique vpp-agent identifier in multi-agent environment, and a part of the configuration key in the key-value scheme. Every key must contain the microservice label in format `/vnf-agent/<microservice-label>/...`. The vpp-agent has the microservice label defined at startup and uses it to identify key-value pairs for it. The label allows to use single data store for multiple agents. The default microservice label is set to `vpp1`. 

#### ... the configuration file?
The configuration file (usually `*.conf`) represents a static config passed to the agent before the startup, adjusting some parameters according to current use case. The config file is per plugin, but not all plugins support it. Sometimes it is mandatory (for example the ETCD plugin uses config file to know what is the IP of the server).

## <a name="startup">Start-up</a>

#### The vpp-agent crashed printing the stack-trace
This is also related to the case if the vpp-agent crashed during termination. Please proceed [here](#notlisted)

#### The vpp-agent failed to start with the error "VPP is incompatible: unknown message ..."
The version of the VPP API the Agent is trying to connect is different than supported. Please use the compatible VPP version (refer to the [vpp.env](https://github.com/ligato/vpp-agent/blob/master/vpp.env) in the VPP-Agent repository root) or use prepared images from the [docker-hub](https://hub.docker.com/r/ligato/vpp-agent).
If you have the local build of the VPP, you can try to re-generate binary API using the following command:
```
make generate-binapi
```
And try to run the Agent again. However, keep in mind that this procedure may cause the Agent to fail to build if there is any significant change in the VPP API against the supported version.

#### The vpp-agent failed to start with the message "Agent failed to start before timeout"
The main cause it that some plugin hanged during the initialization phase. Check which plugin was initialized last (the log should help here). In case it is some connector plugin (ETCD, GoVPP mux,..), the most often cause is that some external server is not responding. If any KVDB or Kafka server or the VPP itself is properly configured (e.g. via .conf file), make sure it is accessible and responding. Otherwise, the plugin initialization procedure waits on successful connection and exceeds the defined timeout. 

#### The vpp-agent failed to start with the message "You are trying to register same resync twice"
The agent tried to load two plugins with the same `PluginName`, mostly because two instances of any `DefaultPlugin` were used at once. If you need to specify multiple instances of the same plugin (e.g. two KVDB connections or HTTP servers), define it with `UseDeps()` option and define a custom name for each of them. //TODO link to some example

#### The vpp-agent failed to start with the message "mount --make-shared failed: permission denied"
A similar error is thrown during an attempt to manipulate a Linux namespace (create, read, ...) in non-privileged docker container, since this operation requires superuser privileges in the target host. The solution is to run the docker image with `--privileged` parameter. If it is not applicable in a given solution, the Linux plugin can be turned off ([see how to do it](Linux-Interface-plugin)).   

## <a name="config">Configuration (plugins)</a>

#### The vpp-agent obtained the configuration data from the northbound and the VPP crashed
This may happen if the Agent tries to send incomplete data to the VPP, or randomly for any configuration if the VPP version is compatible but at different commit ID as listed in the [vpp.env](https://github.com/ligato/vpp-agent/blob/master/vpp.env). Please proceed [here](#notlisted).

#### The vpp-agent obtained the configuration data from the northbound but I do not see configured item in the VPP
* The given configuration item is dependent on any other configuration item which is missing. For example the FIB entry cannot exist without defined interface and bridge domain. In such a case, the vpp-agent caches the configuration and waits for the dependencies. Follow the plugin [user guide](https://github.com/ligato/vpp-agent/wiki/user-guide#plugins-and-components) for more information. 
* The plugin expected to handle the configuration is not loaded. For example in order to configure IPSec, the IPSec plugin must be a part of the application plugins.

#### The vpp-agent obtained the configuration data from the northbound but the plugin procedure end up with error "VPPApiError: (message with numeric return value)"
Such an error indicates binary API call failure. It means data were processed successfully by the Agent, but the VPP refused it for some reason. Please make sure configured data are complete, and in the required format. Also, this kind of errors can suddenly appear while switching to different VPP version - in that case, it is the most likely bug in the Agent caused by external dependency.

## <a name="kvdb">KVDB (database)</a>

#### The data were put to the KVDB store and nothing happened
Make sure that:
* The VPP-Agent is connected to given KVDB
* The plugin configuring given data type is loaded (watcher to the key type was started)
* The microservice label is defined correctly in the key inside KVDB, and in the Agent
* The key is in the correct format (please refer to the [user guide](User-Guide.md) and find appropriate plugin documentation)
* If the Redis database is used, make sure the event notifications are allowed (they are usually disabled by default). If not, enable it with command `config SET notify-keyspace-events KA` (use for example `redis-cli`), and try again

## <a name="grpc">GRPC</a>

#### I want to create a GRPC client application
Start with the GRPC [user guide](https://github.com/ligato/vpp-agent/wiki/GRPC) and then proceed to the [tutorial](https://github.com/ligato/vpp-agent/wiki/GRPC-tutorial).

## <a name="rest">REST</a>

#### I want to use the REST API to communicate with the vpp-agent
See the list of available URLs with the REST guide [here](https://github.com/ligato/vpp-agent/wiki/REST).

## <a name="notlisted">The issue I have is not listed here</a>

If the VPP-Agent or the VPP crashed or the behavior is different as expected, please collect as much info as possible, especially:
* VPP-Agent version/commit ID
* VPP version/commit ID
* VPP-Agent logs (best with debug enabled)
* KVDB store dump if possible
* Description of what was tried to achieve

Then contact Ligato developers. We will solve the issue as fast as possible. Thank you for your contribution!















