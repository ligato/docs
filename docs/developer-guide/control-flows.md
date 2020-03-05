# Control Flows

---

This section describes the behavior of the KV Scheduler system using examples accompanied by UML control-flow diagrams. Each example covers a specific scenario using
configuration items supported by the VPP and Linux plugins. The diagrams illustrate the interactions between the KV Scheduler, NB plane and KV Descriptors for an executed configuration transaction.

To improve readability, the examples use shortened keys without prefixes, or in some cases, more descriptive aliases, as object identifiers. For example, `my-route` is used as a placeholder for a user-defined route, which otherwise would be identified by a [key](../user-guide/reference.md#vpp-keys) composed of a destination network, outgoing interface and next hop address. In addition, most of the configuration items that are automatically created in the SB plane are omitted from the diagrams since they do not play any role in the scenarios described below. These items are retrieved from the VPP and Linux plugins during the first resync. Examples include routes and physical interfaces.


!!! Note
    The UML diagrams are plotted as SVG images. They also contain links to diagrams
    presenting the state of the graph with values at the end of every transaction.
    To access these links, the UML diagrams must be opened as standalone inside a separate web browser tab.

### Example: AF-Packet interface

`AF-Packet` is a VPP interface type attached to a host OS interface. It captures all incoming traffic, and permits Tx packet injection through a socket interface.

The host OS interface must exist before the `AF-Packet` interface is created. This could be problematic because the vpp-agent does not control or configure the host OS interface. Instead, this task could be handled by an external process or
an administrator during vpp-agent run-time. In this situation, there is no key-value pair to resolve `AF-Packet` dependencies.

The KV Scheduler solves this problem by supporting external object notifications through the use of the `PushSBNotification(key, value, metadata)` method. Values received through notifications are denoted as `OBTAINED`. They cannot be removed by a resync even
though they are not explicitly configured by NB. Obtained values are allowed to have their own descriptors. The `Retrieve()` operation is called to refresh the graph.
 `Create`, `Delete` and `Update` operations are never used because:

 - obtained values are updated externally.
 - vpp-agent is notified about any changes _after_ they have occurred.

The Linux plugin support an `InterfaceWatcher` descriptor. It retrieves
and generates notifications regarding Linux interfaces in the default network namespace of the vpp-agent. Linux interfaces are assigned unique
keys using their host names with `linux/interface/host-name/eth1` serving as an example.
The `AF-Packet` interface then defines the dependency referencing the key with the
host name of the interface it is supposed to attach to. Note that it cannot attach
to interfaces from other namespaces.

In this example, the host OS interface is created after the request to configure
`AF-Packet` is received. Therefore, the KV Scheduler holds the `AF-Packet` in the
`PENDING` state until the notification is received. 

![CFD][cfd-af-packet]

### Example: Bridge Domain

A bridge domain consists of a group of interfaces that share a common L2 broadcast subnetwork. Mac-layer L2 broadcast packets originating from one interface will be flooded to all other interfaces in the bridge domain.

An empty bridge domain has no dependencies, and can be created
independently from interfaces. However, to install an interface into a bridge domain,
both the interface and the bridge domain must be created first. The KV Scheduler could treat the bridge domain as a single key-value pair, with a dependency that  _all_ interfaces intended for that bridge domain are configured.

This approach introduces several challenges:

* prevents the existence of the bridge domain even if a single interface is missing.
* request to the KV data store to remove the interface could overtake the bridge domain  configuration update to delist the interface. This results in the bridge domain being temporarily removed and then re-created.

The KV Scheduler addresses these challenges through the use of derived values. The idea is to break apart the configuration item into multiple distinct pieces, each with their own CRUD operations and dependencies. In this scenario, a binding is established between the bridge domain and every bridge domain interface. This is treated as derived value, each coming with their own `BDInterfaceDescriptor` descriptor. A `Create()` operation puts the interface into the bridge domain; a `Delete()` operation removes the interface by breaking the binding.

The bridge domain itself has no dependencies, and will be configured as requested by the NB. However, the individual bindings will have a dependency on its associated interface and implicitly on the bridge domain it is derived from. Even if one or more interfaces are missing or in the process of being deleted, the bridge domain and its remaining interfaces will not be impacted and function will continue.

The control-flow diagram shows that the bridge domain is created even if an interface is configured later. The binding remains in the `PENDING` state until the interface is configured.


![CFD][cfd-bridge-domain]

### Example: Interface Re-creation

Incremental configuration updates for some items are not supported by the SB. Instead, the given item may need to be deleted and re-created with the new configuration. The KV Scheduler supports this scenario using the `UpdateWithRecreate()` method. It enables a descriptor to inform the KV Scheduler if an item requires
   full re-creation for the configuration update to be applied.

This example demonstrates re-creation using a VPP TAP interface and a NB request
to modify the RX ring size. Modifying this configuration item is not supported for an interface that has already been created. In addition, an L3 route is attached to the interface. The route
cannot exist without the interface. Therefore, the route must be deleted and moved into
the `PENDING` state before interface re-creation. The route configuration can then be performed once the interface re-creation process has completed.


![CFD][cfd-interface-recreation]

### Example: Retry of Failed Operation

Transactions can run in `BestEffort` mode meaning that the KV Scheduler will _retry_ failed operations rather than reverting back to previously applied configuration operations.

In this example, the create() operation for the TAP interface `my-tap` fails. Before terminating
the transaction, the KV Scheduler retrieves the current value of `my-value`. It cannot assume this is the current state because the create() operation failed somewhere in-progress.

The KV Scheduler then schedules a *retry transaction*. This will attempt to repair
the failure by re-applying the same configuration. Since the value retrieval
has not found `my-tap` to be configured, the retry transaction will repeat
the `Create(my-tap)` operation and ultimately succeed as shown.


![CFD][cfd-retry-failed]

### Example: Transaction Revert

A update transaction can be configured to run in either best-effort mode supporting partial completion,
or to terminate upon first failure and revert back to successfully applied changes. The latter ensures there are no visible effects of the failure remaining in the system.  This behaviour is only supported with gRPC or localclient serving as the NB. With a KV data sore, best-effort mode will run to attempt to arrive as close as possible to the desired configuration.

In this example, a transaction is planned and executed to create a VPP interface
`my-tap` with an attached route `my-route`. The interface configuration succeeds but the route configuration fails.

The KV Scheduler then triggers the `revert procedure`. First, the current value of `my-route` is
retrieved. It cannot assume this is the current state because the create() operation failed somewhere in-progress. Second, it is determined that the route has not been configured. Therefore,
only the interface must be deleted to undo any executed changes. Once
the interface is removed, the system is returned to its pre-transaction state. Finally, the transaction error is returned back to NB.


![CFD][cfd-transaction-revert]

### Example: Unnumbered Interface

Turning interface into unnumbered allows to enable IP processing without assigning
it an explicit IP address. An unnumbered interface can "borrow" an IP address
of another interface, already configured on VPP, which conserves network and
address space.

The requirement is that the interface from which the IP address is supposed
to be borrowed has to be already configured with at least one IP address assigned.
Normally, the agent represents a given VPP interface using a single key-value
pair. Depending on this key alone would only ensure that the target interface is
already configured when a dependent object is being created. In order to be able
to restrict an object existence based on the set of assigned IP addresses to
an interface, every VPP interface value must `derive` single (and unique)
key-value pair for each assigned IP address. This will enable to reference IP
address assignments and build dependencies around them.

For unnumbered interfaces alone it would be sufficient to derive single value
from every interface, with key that would allow to determine if the interface
has at least one IP address assigned,
something like: `vpp/interface/<interface-name>/has-ip/<true/false>`.
Dependency for an unnumbered interface could then reference key of this value:
`vpp/interface/<interface-name-to-borrow-IP-from>/has-ip/true`

For more complex cases, which are outside of the scope of this example, it may
be desirable to define dependency based not only on the presence but also on the
value of an assigned IP address. Therefore we derive (empty) value for each
assigned IP address with key template:
`"vpp/interface/address/<interface-name>/<address>"`.
This complicates situation for unnumbered interfaces a bit, since they are not
able to reference key of a specific value. Instead, what they would need is to
match any address-representing key, so that the dependency gets satisfied when
at least one of them exists for a given interface. With wildcards this could
be expressed as: `"vpp/interface/address/<interface-name>/*`.
The KVScheduler offers even more generic (i.e. expressive) solution than
wildcards: dependency expressed using callback denoted as `AnyOf`. The callback
is a predicate, returning `true` or `false` for a given key. The semantics is
similar to that of the wildcards. The dependency is considered satisfied, when
for at least one of the existing (configured/derived) keys, the callback returns
`true`.

Lastly, to allow a (to-be-)unnumbered interface to exist even if the IP address(es)
to borrow are not available yet, the call to turn interface into unnumbered is
derived from the interface value and processed by a separate descriptor:
`UnnumberedIfDescriptor`. It is this derived value that uses `AnyOf` callback
to trigger IP address borrowing only once the IP addresses become available.
For the time being, the interface is available at least in the L2 mode.

The example also demonstrates that when the borrowed IP address is being removed,
the unnumbered interface will not get un-configured, instead it will only return
the address before it is unassigned and turn back into the L2 mode.  


![CFD][cfd-unnumbered]

### Scenario: Create VPP interface via KVDB

This is the most basic scenario covered in the guide. On the other hand,
the attached control-flow diagram is the most verbose - it includes all the
interacting components (listed from the top to the bottom layer):

 * `NB (KVDB)`: contains configuration of a single `my-tap` interface
 * `Orchestrator`: listing KVDB and propagating the configuration to the KVScheduler
 * `KVScheduler`: planning and executing the transaction operations
 * `Interface Descriptor`: implementing CRUD operations for VPP interfaces
 * `Interface Model`: builds and parses keys identifying VPP interfaces


![CFD][cfd-create-vpp-interface-kvdb]

### Scenario: Create VPP interface via GRPC

Variant of [this][vpp-interface] example, with GRPC used as the NB interface
for the agent instead of KVDB. The transaction control-flow is collapsed, however,
since there are actually no differences between the two cases. For KVScheduler
it is completely irrelevant how the desired configuration gets conveyed
into the agent. The advantage of GRPC over KVDB is that the transaction error
value gets propagated back to the client, which is then able to react to it
accordingly. On the other hand, with KVDB the client does not have to maintain
a connection with the agent and the configuration can be submitted even when
the agent is restarting or not running at all.



![CFD][cfd-create-vpp-interface-grpc]

### Example: Route waiting for the associated interface

This example shows how route, received from KVDB before the associated
interface, gets delayed and stays in the `PENDING` state until the interface
configuration is received and applied. This is achieved simply by listing the
key representing the interface among the dependencies for the route.

![CFD][cfd-route-waiting]

[cfd-af-packet]: ../img/control-flow-diagram/add_af_packet_interface.svg?sanitize=true
[cfd-bridge-domain]: ../img/control-flow-diagram/add_bd_before_interface.svg?sanitize=true
[cfd-create-vpp-interface-grpc]: ../img/control-flow-diagram/add_interface_grpc_txn_collapsed.svg?sanitize=true
[cfd-create-vpp-interface-kvdb]: ../img/control-flow-diagram/add_interface.svg?sanitize=true
[cfd-interface-recreation]: ../img/control-flow-diagram/recreate_interface_with_route.svg?sanitize=true
[cfd-retry-failed]: ../img/control-flow-diagram/fails_to_add_interface.svg?sanitize=true
[cfd-route-waiting]: ../img/control-flow-diagram/add_route_before_interface.svg?sanitize=true
[cfd-transaction-revert]: ../img/control-flow-diagram/interface_and_route_with_revert.svg?sanitize=true
[cfd-unnumbered]: ../img/control-flow-diagram/unnumbered_interface.svg?sanitize=true
[retry-failed]: control-flows.md#example-retry-of-failed-operation
[vpp-interface]: control-flows.md#scenario-create-vpp-interface-via-kvdb

*[KVDB]: Key-Value Database
*[VPP]: Vector Packet Processing
