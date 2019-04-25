This guide aims to explain the key-value scheduling algorithm using examples
with UML control-flow diagrams, each covering a specific scenario using real
configuration items from vpp and linux plugins of the agent. The control-flow
diagrams are structured to describe all the interactions between KVScheduler, NB
plane and KVDescriptor-s during transactions. To improve readability, the examples
use shortened keys (without prefixes) or even alternative and more descriptive
aliases as object identifiers. For example, `my-route` is used as a placeholder
for user-defined route, which otherwise would be identified by key composed
of destination network, outgoing interface and next hop address. Moreover, most
of the configuration items that are automatically pre-created in the SB plane
(i.e. `OBTAINED`), normally retrieved from VPP and Linux during the first resync
(e.g. default routes, physical interfaces, etc.), are omitted from the diagrams
since they do not play any role in the presented scenarios.

The UML diagrams are plotted as SVG images, also containing links to images
presenting the state of the graph with values at the end of every transaction.
But to be able to access these links, the UML diagrams have to be clicked at
to open them as standalone inside another web browser tab. From within github
web-UI the links are not directly accessible.

### Example: AF-Packet interface

`AF-Packet` is VPP interface type attached to a host OS interface, capturing
all its incoming traffic and also allowing to inject Tx packets through a special
type of socket.

The requirement is that the host OS interface exists already before the `AF-Packet`
interface gets created. The challenge is that the host interface may not be
from the scope of items configured by the agent. Instead, it could be
a pre-existing physical device or interface created by an external process or
an administrator during the agent run-time. In such cases, however, there would
be no key-value pair to reference from within `AF-Packet` dependencies. Therefore,
KVScheduler allows to notify about external objects through 
`PushSBNotification(key, value, metadata)` method. Values received through
notifications are denoted as `OBTAINED` and will not be removed by resync even
though they are not requested to be configured by NB. Obtained values are
allowed to have their own descriptors, but from the CRUD operations only
`Retrieve()` is ever called to refresh the graph. `Create`, `Delete` and `Update`
are never used, since obtained values are updated externally and the agent is
only notified about the changes *after* they have already happened.

Linux interface plugin ships with `InterfaceWatcher` descriptor, which retrieves
and notifies about Linux interfaces in the network namespace of the agent
(so-called default network namespace). Linux interfaces are assigned unique
keys using their host names, e.g.: `linux/interface/host-name/eth1`
The `AF-Packet` interface then defines dependency referencing the key with the
host name of the interface it is supposed to attach to (cannot attach
to interfaces from other namespaces).

In this example, the host interface gets created after the request to configure
`AF-Packet` is received. Therefore, the scheduler keeps the `AF-Packet` in the
`PENDING` state until the notification is received. 

![CFD][cfd-af-packet]

### Example: Bridge Domain

Using bridge domain it is demonstrated how derived values can be used to
"break" item into multiple parts with their own CRUD operations and dependencies.

Bridge domain groups multiple interfaces to share the same flooding or broadcast
characteristics. Empty bridge domain has no dependencies and can be created
independently from interfaces. But to put an interface into a bridge domain,
both the interface and the domain must be created first. One solution for
the KVScheduler framework would be to handle bridge domain as a single key-value
pair depending on all the interfaces it is configured to contain. But this is
a rather coarse-grained approach that would prevent the existence of the bridge
domain even when only a single interface is missing. Moreover, with KVDB,
request to remove interface could overtake update of the bridge domain
configuration un-listing the interface, which would cause the bridge domain
to be temporarily removed and shortly afterwards fully re-created.

The concept of derived values allowed to specify binding between bridge
domain and every bridged interface as a separate derived value, handled
by its own `BDInterfaceDescriptor` descriptor, where `Create()` operation puts
interface into the bridge domain, `Delete()` breaks the binding, etc.
The bridge domain itself has no dependencies and will be configured as long as
it is demanded by NB.
The bindings, however, will each have a dependency on its associated interface
(and implicitly on the bridge domain it is derived from).
Even if one or more interfaces are missing or are being deleted, the remaining
of the bridge domain will remain unaffected and continuously functional.

The control-flow diagram shows that bridge domain is created even if the
interface that it is supposed to contain gets configured later. The binding
remains in the `PENDING` state until the interface is configured.


![CFD][cfd-bridge-domain]

### Example: Interface Re-creation

The example uses VPP interface with attached route to outline the control-flow
of item re-creation. A specific configuration update may not be supported by SB
to perform incrementally - instead the given item may need to be deleted and
re-created with the new configuration. Using `UpdateWithRecreate()` method,
a descriptor is able to tell the KVScheduler if the given item requires
full re-creation for the configuration update to be applied.

This example demonstrates re-creation using a VPP TAP interface and a NB request
to change the RX ring size, which is not supported for an already created
interface. Furthermore, the interface has an L3 route attached to it. The route
cannot exists without the interface, therefore it must be deleted and moved into
the `PENDING` state before interface re-creation, and configured back again once
the re-creation procedure has finalized.


![CFD][cfd-interface-recreation]

### Example: Retry of failed operation

This example demonstrates that for `best-effort` transactions (e.g. every resync),
the KVScheduler allows to enable automatic and potentially repeated *retry*
of failed operations.

In this case, a TAP interface `my-tap` fails to get created. Before terminating
the transaction, the scheduler retrieves the current value of `my-value`, the
state of which cannot be assumed since the creation failed somewhere in-progress.
Then it schedules a so-called *retry transaction*, which will attempt to fix
the failure by re-applying the same configuration. Since the value retrieval
has not found `my-tap` to be configured, the retry transaction will repeat
the `Create(my-tap)` operation and succeed in our example.


![CFD][cfd-retry-failed]

### Example: Transaction revert

An update transaction (i.e. not resync) can be configured to run in either
[best-effort mode][retry-failed], allowing partial completion,
or to terminate upon first failure and revert already applied changes so that no
visible effects are left in the system (i.e. the true transaction definition).
This behaviour is only supported with GRPC or localclient as the agent NB
interface. With KVDB, the agent will run in the best-effort mode to get as close
to the desired configuration as it is possible.

In this example, a transaction is planned and executed to create a VPP interface
`my-tap` with an attached route `my-route`. While the interface is successfully
created, the route, on the other hand, fails to get configured. The scheduler
then triggers the *revert procedure*. First the current value of `my-route` is
retrieved, the state of which cannot be assumed since the creation failed
somewhere in-progress. The route is indeed not found to be configured, therefore
only the interface must be deleted to undo already executed changes. Once
the interface is removed, the state of the system is back to where it was before
the transaction started. Finally, the transaction error is returned back to the
northbound plane.


![CFD][cfd-transaction-revert]

### Example: Unnumbered interface

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
[retry-failed]: control-flow.md#example-retry-of-failed-operation
[vpp-interface]: control-flow.md#scenario-create-vpp-interface-via-kvdb