# Startup Troubleshooting

---

**The VPP agent crashed printing the stack-trace**

This is also related to the case if the VPP agent crashed during termination. [Reference Ligato Support](../intro/faq.md#where-can-i-find-support-for-ligato).

---

**The VPP agent failed to start with the error "VPP is incompatible: unknown message ..."**


The agent's VPP API version is different from the version supported by the VPP it is attempting to connect to. Please use the compatible VPP version (refer to the [vpp.env](https://github.com/ligato/vpp-agent/blob/master/vpp.env) in the vpp-agent repository) or use prepared images from the [docker-hub](https://hub.docker.com/r/ligato/vpp-agent) which always contain compatible versions of the vpp-gent and VPP.
If you have the local build of the VPP, you can try to re-generate the binary API using the following command:
```
make generate-binapi
```
Try to run the agent again. However, keep in mind that this procedure may cause the agent to fail to build if there is any significant change in the VPP API for the supported version.

---

**The VPP agent failed to start with the message "Agent failed to start before timeout"**


The main cause it that some plugin was loading longer than the maximum time provided during the initialization phase. Check which plugin was initialized the last (the log should help here). In case it is a connector plugin (etcd, GoVPPMux,..), the most often cause is an unresponsive external server. If the KV data store, Kafka server or VPP itself is properly configured (e.g. via .conf file), make sure it is accessible and responding. Otherwise, the plugin initialization procedure will exceed the defined timeout.

---

**The VPP agent failed to start with the message "You are trying to register same resync twice"**

The agent tried to load two plugins with the same `PluginName`, mostly because two instances of any `DefaultPlugin` were used at once. If you need to specify multiple instances of the same plugin (e.g. two KVDB connections or HTTP servers), define it with `UseDeps()` option and use a custom name for each.

---

**The VPP agent failed to start with the message "mount --make-shared failed: permission denied"**

A similar error is thrown during an attempt to manipulate a Linux namespace (create, read, ...) in a non-privileged docker container. This operation requires superuser privileges in the target host. The solution is to run the docker image with `--privileged` parameter. If it is not applicable in a given solution, the Linux plugin can be [turned off.](../plugins/linux-plugins.md#disable-the-linux-interface-plugin)

