# Control Flows

---

This section describes the behavior of the KV Scheduler system using examples that include UML control-flow diagrams.
 
 ---
 
 Each example covers a specific scenario using configuration items supported by VPP and Linux plugins. The diagrams illustrate the interactions between the KV Scheduler, northbound (NB) control plane, and KV Descriptors for a configuration transaction.

To improve readability, the examples use shortened keys without prefixes, or in some cases, more descriptive aliases as object identifiers. For example, `my-route` replaces the [VPP route key][vpp-key] for a user-defined route. In addition, the examples omit southbound (SB) configuration activities since they do not factor in the control flow scenarios. Note that the first resync retrieves configuration items such as routes and interfaces.


!!! Note
    The UML diagrams incorporate SVG images. They contain links to diagrams
    presenting the state of the graph with values at the end of every transaction.
    To access these links, open a separate web browser tab and click on the link.

---

### Example: AF-Packet interface

Let's examine the creation of an `AF_Packet` interface. This interface type attaches to a host OS interface. It captures all incoming traffic, and permits tx packet injection through a socket interface. 

The host OS interface must exist before you create an `AF_Packet` interface. This could be problematic because the VPP agent does not control or configure the host OS interface. Typically, an external process or network administrator configures host OS interfaces during VPP run-time. You do not have a key value pair to resolve this `AF_Packet` "dependency".

The KV Scheduler solves this problem by supporting external object notifications with the `PushSBNotification(key, value, metadata)` method. Values received through notifications are marked as `OBTAINED`. A resync cannot remove these values even though they are not explicitly configured by NB. `OBTAINED` values to have their own descriptors. 

The `Retrieve()` operation refreshes the graph. The `Create`, `Delete` and `Update` operations aren't  used because of the following:

 - `OBTAINED` values update externally.
 - VPP agent is notified of changes only _after_ they have occurred.

The Linux plugin supports an `InterfaceWatcher` descriptor. It retrieves
and generates notifications regarding Linux interfaces in the default network namespace of the VPP agent. Linux interfaces use keys containing the name of the host with  `linux/interface/host-name/eth1` serving as an example. The `AF_PACKET` interface then defines the dependency referencing the key with the
host name of the interface it attaches to. Note that you can't attach
to interfaces from other namespaces.

In this example, the system creates the host OS interface only after it receives the request to configure an `AF_PACKET` interface. The KV Scheduler holds the `AF-Packet` in the
`PENDING` state until it receives the notification.

 

![CFD][cfd-af-packet]

---

### Example: Bridge Domain 

It this example, let's look over the flows for creating a bridge domain. You will recall that a bridge domain consists of a group of interfaces that share a common L2 broadcast subnetwork. One interface will flood mac-layer L2 broadcast packets to all other interfaces in the bridge domain.

An empty bridge domain has no dependencies. You can create a bridge domain without first creating the interfaces. However, to install an interface into a bridge domain, you must first create the interface and bridge domain. 

The KV Scheduler could treat the bridge domain as a single key-value pair, with a dependency that you configure _all_ interfaces intended for that bridge domain. This approach introduces several challenges:

* Prevents the existence of the bridge domain even if a single interface is missing.
* Request to the KV data store to remove the interface could overtake a bridge domain configuration update to remove interface. This results in the temporary removal and recreation of the bridge domain.

The KV Scheduler addresses these challenges through the use of derived values. This technique breaks the configuration item into multiple distinct pieces, each with their own CRUD operations and dependencies. A binding exists between the bridge domain and every bridge interface. The binding functions as a derived value, each coming with its own `BDInterfaceDescriptor` descriptor. A `Create()` operation puts the interface into the bridge domain; a `Delete()` operation removes the interface by breaking the binding.

A request to NB configures a bridge domain, which itself does not have any dependencies.
However, the individual bindings will have a dependency on its associated interface, and implicitly on the bridge domain it is derived from. Even if one or more interfaces are missing, or in the process of being deleted, the bridge domain and its remaining interfaces will not be impacted and function will continue.

The control-flow diagram shows that the bridge domain is created even if an interface is configured later. The binding remains in the `PENDING` state until the interface is configured.


![CFD][cfd-bridge-domain]

---

### Example: Interface Re-creation

The SB doesn't support incremental configuration updates for some items. Instead, the given item must be deleted, and re-created with the new configuration. The KV Scheduler supports this scenario using the `UpdateWithRecreate()` method. It enables a descriptor to inform the KV Scheduler if an item requires full re-creation before applying the configuration update. 

This example demonstrates re-creation using a VPP TAP interface, and a NB request
to modify the RX ring size. Modifying this configuration item is not supported for an interface that has already been created. In addition, an L3 route is attached to the interface. The route
cannot exist without the interface. Therefore, the route must be deleted and moved into
the `PENDING` state before interface re-creation. Route configuration can proceed once the interface re-creation process has completed.


![CFD][cfd-interface-recreation]

---

### Example: Retry of a Failed Operation

Transactions can run in `BestEffort` mode. The KV Scheduler will _retry_ failed operations rather than reverting back to previously applied configuration operations.

In this example, the `Create()` operation for the TAP interface `my-tap` fails. Before terminating
the transaction, the KV Scheduler retrieves the current value of `my-value`. It cannot assume this is the current state, because the `Create()` operation failed somewhere in-progress.

The KV Scheduler then schedules a *retry transaction*. This will attempt to repair
the failure by re-applying the same configuration. Since the value retrieval
has not found a configured `my-tap`, the retry transaction will repeat
the `Create(my-tap)` operation. 


![CFD][cfd-retry-failed]

---

### Example: Transaction Revert

A configuration update transaction can terminate upon the first failure, and revert back to successfully applied changes. This ensures any visible effects of the failure do not remain in the system.  Note that only gRPC or localclient serving as the NB support this behavior. With a KV data sore, best-effort mode will run in an attempt to arrive as close as possible to the desired configuration.

In this example, a transaction is planned and executed to create a VPP interface
`my-tap` with an attached route `my-route`. The interface configuration succeeds, but the route configuration fails.

In this example, a transaction to create a VPP interface
`my-tap` with an attached route `my-route`. is planned and executed. The interface configuration succeeds, but the route configuration fails.

The KV Scheduler then triggers the `revert procedure`. First, it retrieves the current value of `my-route`. It cannot assume this is the current state because the `Create()` operation failed somewhere in-progress. Second, it determines that the route has not been configured. Only the interface must be deleted to undo any executed changes. 

Once the interface is removed, the system returns to its pre-transaction state. Finally, the transaction error is returned to the NB.


![CFD][cfd-transaction-revert]

---

### Example: Unnumbered Interface

An unnumbered interface is not configured with an IP address. It can borrow an IP address from another target interface. The target interface that "loans" IP addresses to unnumbered interfaces must be configured with at least one IP address. Normally, the VPP agent represents a given VPP interface using a single key-value pair. Depending on this key alone would only ensure that the target interface is configured when a dependent object is being created. 

!!! Note
    The paragraph above only refers to a VPP interface. It is not referring to a VPP interface configured with an IP address.

An object's existence can be restricted based on IP addresses assigned to an interface. To accomplish this, every VPP interface value must derive a unique key-value pair for each assigned address. This enables the KV Scheduler to reference IP address assignments and build dependencies around them.

An unnumbered interface derives a single value from every interface. The key can indicate if the interface has at least one assigned IP address.
                                                                                                           
Here is an example:
```
vpp/interface/<interface-name>/has-ip/<true/false>
```

Dependency for an unnumbered interface could reference this key:
```
vpp/interface/<interface-name-to-borrow-IP-from>/has-ip/true
```

---

For more complex cases, you can define a dependency based on the presence and value of an assigned IP address. To achieve this, you derive an empty value for each assigned IP address with key template.

Here is an example of a key template:
```
vpp/interface/address/<interface-name>/<address>
```

This complicates the situation for unnumbered interfaces. They can't reference a key of a specific value. Instead, they would need to match any address-representing key. A single match satisfies the dependency. 

Here is an example of a key with a wildcard: 
```
vpp/interface/address/<interface-name>/*
```

The KV Scheduler offers a more generic solution than wildcards. The dependency is expressed using a callback denoted as `AnyOf`. The callback returns `true` or `false` for a given key. A callback returning `true`, for at least one of the existing configured or derived keys, satisfies the dependency.

Lastly, it is possible for an unnumbered interface to exist even if the IP address(es)
to borrow are not available. In this case, an unnumbered interface is derived from the interface value and processed by a separate `UnnumberedIfDescriptor` descriptor. This derived value uses the `AnyOf` callback to trigger an IP address borrow action once the IP addresses become available.

This example demonstrates that an unnumbered interface configuration is not impacted by the removal of the borrowed IP address. It will only return the address before it is unassigned, and becomes an interface in L2 mode.


![CFD][cfd-unnumbered]

---

### Scenario: Create VPP interface via KV Data Store

This is the most basic scenario one will encounter. The control-flow diagram is detailed. It includes all of interacting components consisting of:

 * `NB (KVDB)`: contains the configuration of a single `my-tap` interface
 * `Orchestrator`: listing KVDB and propagating the configuration to the KV Scheduler
 * `KVScheduler`: planning and executing the transaction operations
 * `Interface Descriptor`: implementing CRUD operations for VPP interfaces
 * `Interface Model`: builds and parses keys identifying VPP interfaces


![CFD][cfd-create-vpp-interface-kvdb]

---

### Scenario: Create VPP Interface via gRPC

In this example, gRPC is used as the VPP agent's NB instead of the KV data store. The transaction control-flow is collapsed because there are essentially no differences between the two cases. The KV Scheduler is not concerned how the desired configuration is conveyed to the VPP agent.

The advantage of the gRPC approach is that the transaction error
value is propagated back to the client, which is then able to react to it
accordingly. On the other hand, with the KV data store approach, there is no client-to-agent connection required. The configuration can be submitted even when the VPP agent is restarting or not running at all.



![CFD][cfd-create-vpp-interface-grpc]

---

### Example: Route waiting for the associated interface

This example shows how a route, received from the KV data store, is placed in a `PENDING` state until the interface is configured. This is achieved by listing the interface key among the dependencies for the route.

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
[vpp-key]: ../user-guide/reference.md#vpp-keys

*[KVDB]: Key-Value Database
*[VPP]: Vector Packet Processing
