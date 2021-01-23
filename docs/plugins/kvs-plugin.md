# KV Scheduler

---

The KV Scheduler plugin supports dependency resolution, and computes the proper programming sequence for multiple interdependent configuration items. It works with VPP and Linux agents on the southbound (SB) side, and external data stores and rpc clients on the northbound (NB) side. 

To begin working with the KV Scheduler right away, start with the [tutorials](../tutorials/00_tutorial-setup.md) section and work your way to the [KV Scheduler tutorial](../plugins/kvs-plugin.md).

For an in-depth discussion, see the [KV Scheduler Section in the Developer Guide][kvs-dev-guide]

**References:** 

- [Ligato Overview](../intro/overview.md)
- [Tutorials](../tutorials/00_tutorial-setup.md)
- [Concepts](../user-guide/concepts.md)
- [Agentctl](../user-guide/agentctl.md)
- [KV Scheduler REST APIs](../api/api-kvs.md)
- [Developer Guide KV Scheduler section][kvs-dev-guide]
- [Developer Guide KVS Troubleshooting section](../developer-guide/kvs-troubleshooting.md)
- [Developer Guide KV Descriptors section](../developer-guide/kvdescriptor.md)


----
[bd-model]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto
[bd-interface]: https://github.com/ligato/vpp-agent/blob/master/api/models/vpp/l2/bridge-domain.proto#L19
[bd-derived-vals]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/l2plugin/descriptor/bridgedomain.go
[bd-iface-deps]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/l2plugin/descriptor/bd_interface.go
[kvs-dev-guide]: ../developer-guide/kvscheduler.md
[kv-scheduler-rest-api]: ../api/api-kvs.md
[kvs-txn-history-api]: ../api/api-kvs.md#transaction-history
[kvs-key-timeline-api]: ../api/api-kvs.md#key-timeline
[kvs-graph-snapshot-api]: ../api/api-kvs.md#graph-snapshot
[kvs-dump-api]: ../api/api-kvs.md#dump
[kvs-dump-parameters-example]: ../api/api-kvs.md#dump-viewkey-prefix
[kvs-downstream-resync]: ../api/api-kvs.md#downstream-resync
[kvs-status-api]: ../api/api-kvs.md#status
[kvs-flag-stats]: ../api/api-kvs.md#flag-stats
[kvs-graph-api]: ../developer-guide/kvs-troubleshooting.md#how-to-visualize-the-graph
[kvdescriptor-dev-guide]: ../developer-guide/kvdescriptor.md
[vpp-iface-idx]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/ifplugin/ifaceidx/ifaceidx.go
[vpp-iface-map]: https://github.com/ligato/vpp-agent/blob/dev/plugins/vpp/ifplugin/ifplugin_api.go#L26
[value-origin]: https://github.com/ligato/vpp-agent/blob/dev/plugins/kvscheduler/api/kv_descriptor_api.go#L53

*[ARP]: Address Resolution Protocol
*[FIB]: Forwarding Information Base
*[REST]: Representational State Transfer
*[VNF]: Virtual Network Function