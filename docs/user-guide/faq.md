# FAQ


### What is ...

**... the vpp-agent?**<br>
The vpp-agent is a control/management plane for VPP-based cloud-native virtual network functions. It is composed of a various plugins (e.g. route, ACL) plugins accessible via APIs to northbound applications. There are other plugins (i.e. GoVPPmux) for programming the VPP data plane.

**... the cn-infra?**<br>
The cn-infra is a set of plugins/libraries providing various functions such as KV data store communications, dependency resolution, health checks and telemetry. It furnishes a lifecycle management infrastructure for existing and customized cloud-native applications. cn-infra code is stored in its own [repository](https://github.com/ligato/cn-infra) and usually (but necessarily) used with VPP-based CNFs.

**... the plugin?**<br>
The plugin is a structure fulfilling the CN-Infra plugin definition (implements the plugin interface). Such a structure can be used in the plugin lifecycle management and started together with other native plugins.

**... the resync?**<br>
The resync is a synchronization procedure which ensure the consistent initial state between the KVDB (if used), the vpp-agent and the VPP. Usually it is called at startup, but can be called on other events (like VPP restart). The resync reads the current VPP configuration and adds or removes configuration items according to the northbound data. The resync ensures consistent VPP state at any point.  

**... the microservice label?**<br>
The microservice label is unique vpp-agent identifier in multi-agent environment, and a part of the configuration key in the key-value scheme. Every key must contain the microservice label in format `/vnf-agent/<microservice-label>/...`. The vpp-agent has the microservice label defined at startup and uses it to identify key-value pairs for it. The label allows to use single data store for multiple agents. The default microservice label is set to `vpp1`. 

**... the configuration file?**<br>
The configuration file (usually `*.conf`) represents a static config passed to the agent before the startup, adjusting some parameters according to current use case. The config file is per plugin, but not all plugins support it. Sometimes it is mandatory (for example the ETCD plugin uses config file to know what is the IP of the server).

### Start-up issues

**The vpp-agent crashed printing the stack-trace**<br>
This is also related to the case if the vpp-agent crashed during termination. Please proceed [here][not-listed-issue].

**The vpp-agent failed to start with the error "VPP is incompatible: unknown message ..."**<br>
The version of the VPP API the Agent is trying to connect is different than supported. Please use the compatible VPP version (refer to the [vpp.env][vpp-env] in the VPP-Agent repository root) or use prepared images from the [docker-hub][docker-hub] which always contain compatible versions of the VPP-Agent and the VPP.
If you have the local build of the VPP, you can try to re-generate binary API using the following command:
```
make generate-binapi
```
And try to run the Agent again. However, keep in mind that this procedure may cause the Agent to fail to build if there is any significant change in the VPP API against the supported version.

**The vpp-agent failed to start with the message "Agent failed to start before timeout"**<br>
The main cause it that some plugin was loading longer than the maximum time provided during the initialization phase. Check which plugin was initialized the last (the log should help here). In case it is some connector plugin (ETCD, GoVPPMux,..), the most often cause is that some external server is not responding. If any KVDB or Kafka server or the VPP itself is properly configured (e.g. via .conf file), make sure it is accessible and responding. Otherwise, the plugin initialization procedure waits on successful connection and exceeds the defined timeout. 

**The vpp-agent failed to start with the message "You are trying to register same resync twice"**<br>
The agent tried to load two plugins with the same `PluginName`, mostly because two instances of any `DefaultPlugin` were used at once. If you need to specify multiple instances of the same plugin (e.g. two KVDB connections or HTTP servers), define it with `UseDeps()` option and define a custom name for each of them. //TODO link to some example

**The vpp-agent failed to start with the message "mount --make-shared failed: permission denied"**<br>
A similar error is thrown during an attempt to manipulate a Linux namespace (create, read, ...) in non-privileged docker container, since this operation requires superuser privileges in the target host. The solution is to run the docker image with `--privileged` parameter. If it is not applicable in a given solution, the Linux plugin can be turned off ([see how to do it][linux-interface-plugin]).   

### Configuration (plugin issues)

**The vpp-agent obtained the configuration data from the northbound and the VPP crashed**<br>
This may happen if the Agent tries to send incomplete data to the VPP, or randomly for any configuration if the VPP version is compatible but at different commit ID as listed in the [vpp.env][vpp-env]. Please proceed [here][not-listed-issue].

**The vpp-agent obtained the configuration data from the northbound but I do not see configured item in the VPP**<br>

Two possibilities:

* The given configuration item is dependent on any other configuration item which is missing. For example the FIB entry cannot exist without defined interface and bridge domain. In such a case, the vpp-agent caches the configuration and waits for the dependencies. Follow the plugin user guide for more information. 
* The plugin expected to handle the configuration is not loaded. For example in order to configure IPSec, the IPSec plugin must be a part of the application plugins.

**The vpp-agent obtained the configuration data from the northbound but the plugin procedure end up with error "VPPApiError: (message with numeric return value)"**<br>
Such an error indicates binary API call failure. It means data were processed successfully by the Agent, but the VPP refused it for some reason. Please make sure configured data are complete, and in the required format. Also, this kind of errors can suddenly appear while switching to different VPP version - in that case, it is the most likely bug in the Agent caused by external dependency.

**The vpp-agent obtained the configuration data from the northbound and caused VPP to crash**<br>
This could happen when the configuration data were incomplete or corrupted, the VPP-Agent does not recognize it and the VPP does not handle them properly. Make sure the data are correct and also please [contact Ligato team][not-listed-issue].

**The vpp-agent obtained the configuration data from the northbound printed error "No reply received within timeout ..."**<br>
Similar problem as above, the VPP got incomplete or incorrect data which caused it to stuck, so no reply was sent back to the VPP-Agent. Check the data correctness, and we also recommend to [contact Ligato team][not-listed-issue].  

**A have a problem which is most likely related specifically to the KV Scheduler**<br>
KV Scheduler has its own troubleshooting guide with detailed description of most common issues, please refer [here][kvs-faq]

### KVDB (database)

**The data was put to the KV data store and nothing happened**<br>
Make sure that:

* The VPP-Agent is connected to given KVDB
* The plugin configuring given data type is loaded (watcher to the key type was started)
* The microservice label is defined correctly in the key inside KVDB, and in the Agent
* The key is in the correct format (please refer to the [supported keys][key-reference] and find appropriate plugin documentation)
* If the Redis database is used, make sure the event notifications are allowed (they are usually disabled by default). If not, enable it with command `config SET notify-keyspace-events KA` (use for example `redis-cli`), and try again

### GRPC

**I want to create a GRPC client application**

Start with the [GRPC user guide][grpc-plugin] and then proceed to the [tutorial][grpc-tutorial].

### REST

**I want to use the REST API to communicate with the vpp-agent**

See the list of available URLs with the REST guide [here][rest-api]. Note that the REST plugin must be loaded, otherwise REST handlers are not started.


### Other issues

**The issue I have is not listed here or cannot be solved by myself**

If the VPP-Agent or the VPP crashed or the behavior is different than expected, please collect as much info as possible, including:

* VPP-Agent version/commit ID
* VPP version/commit ID
* VPP-Agent logs (best with debug enabled)
* KVDB store dump if possible
* Description of what was tried to achieve

Then open an issue in the respective [vpp-agent](https://github.com/ligato/vpp-agent/issues)  or [cn-infra](https://github.com/ligato/cn-infra/issues) github repositories.

Further discussions with the Ligato community can take place in the [Ligato Slack Room](https://ligato.slack.com/)

[docker-hub]: https://hub.docker.com/r/ligato/vpp-agent
[grpc-plugin]: ../plugins/connection-plugins.md#grpc-plugin
[grpc-tutorial]: ../tutorials/08_grpc-tutorial.md
[key-reference]: ../user-guide/reference.md
[kvs-faq]: ../developer-guide/kvs-troubleshooting.md
[linux-interface-plugin]: ../plugins/linux-plugins.md#interface-plugin
[not-listed-issue]: faq.md#other-issues
[rest-api]: ../plugins/connection-plugins/#supported-urls
[vpp-env]: https://github.com/ligato/vpp-agent/blob/master/vpp.env


*[FIB]: Forwarding Information Base
*[HTTP]: Hypertext Transfer Protocol
*[KVDB]: Key-Value Database
*[REST]: Representational State Transfer











