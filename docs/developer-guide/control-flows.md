# Control Flows

---

This section describes the behavior of the KV Scheduler system using examples that include UML control-flow diagrams.
 
 ---
 
 Each example covers a specific scenario using configuration items supported by VPP and Linux plugins. The diagrams illustrate the interactions between the KV Scheduler, northbound (NB) control plane, and KV Descriptors for a configuration transaction.

To improve readability, the examples use shortened keys without prefixes, or more descriptive aliases as object identifiers. For example, `my-route` replaces the [VPP route key][vpp-key] for a user-defined route. 

In addition, the examples omit southbound (SB) configuration activities since they do not factor in the control flow scenarios. Note that the first resync retrieves configuration items such as routes and interfaces.


!!! Note
    The UML diagrams incorporate SVG images. They contain links to diagrams
    presenting the state of the graph with values at the end of every transaction.
    To access these links, open a separate web browser tab and click on the link.

---

### Example: AF-Packet interface

This example covers the creation of an `AF_Packet` interface. This interface type attaches to a host OS interface. It captures all incoming traffic, and permits tx packet injection through a socket interface. 

The host OS interface _must_ exist before you create an `AF_Packet` interface. This presents a challenge because the VPP agent does not control or configure the host OS interface. Typically, an external process or network administrator configures host OS interfaces during VPP run-time. In addition, you do not have a key value pair to resolve this `AF_Packet` "host interface" dependency.

The KV Scheduler solves this problem by supporting external object notifications with the `PushSBNotification(key, value, metadata)` method. Values received through notifications are marked as `OBTAINED`, and can have their own descriptors. 

!!! Note
    A resync cannot remove these values even though they are not explicitly configured by NB.  

The `Retrieve()` operation refreshes the graph. The `Create`, `Delete` and `Update` operations aren't used because of the following:

 - `OBTAINED` values are updated externally.
 - VPP agent is notified of changes only _after_ they have occurred.

The Linux plugin supports an `InterfaceWatcher` descriptor. It retrieves
and generates notifications for Linux interfaces in the VPP agent's default network namespace. 

Linux interfaces use keys containing host name with  `linux/interface/host-name/eth1` key serving as an example. The `AF_PACKET` interface then defines the dependency referencing this key with the host name of the host OS interface it attaches to. Note that you can't attach to interfaces from other namespaces.

In the control-flow diagram below, the host OS interface is created _after_ the request to configure an `AF_PACKET` interface is received. The KV Scheduler holds the `AF-Packet` in the `PENDING` state until it receives the notification from the Linux interface watcher that the host OS interface is created.  

 

![CFD][cfd-af-packet]

---

### Example: Bridge Domain 

This example describes the flows to set up a bridge domain. You will recall that a bridge domain consists of a group of interfaces that share a common L2 broadcast subnetwork. One interface floods mac-layer L2 broadcast packets to all other interfaces in the bridge domain.

An empty bridge domain has no dependencies. You can create a bridge domain without first creating the interfaces. However, to install an interface into a bridge domain, you must first create the interface and the bridge domain. 

In theory, the KV Scheduler could treat the bridge domain as a single key-value pair, with a dependency that you configure _all_ interfaces for that bridge domain. This approach introduces several challenges:

* Prevents the existence of the bridge domain even if a single interface is missing.
* Request to the KV data store to remove the interface could overtake a bridge domain configuration update to remove interface. This results in the temporary removal and re-creation of the bridge domain.

The KV Scheduler addresses these challenges through the use of derived values. This technique breaks the configuration item into multiple distinct pieces, each with their own CRUD operations and dependencies. A binding exists between the bridge domain and every bridge interface. 

The binding functions as a derived value, each coming with its own `BDInterfaceDescriptor` descriptor. A `Create()` operation puts the interface into the bridge domain; a `Delete()` operation removes the interface by breaking the binding.

You can configure a bridge domain with a request to NB. The bridge domain itself does not have any dependencies. However, the individual bindings will have a dependency on their associated interfaces, and implicitly on the bridge domain they are derived from. Even if one or more interfaces are missing, or in the process of being deleted, the bridge domain and its remaining interfaces will not be impacted and function will continue.

The control-flow diagram shows that the bridge domain is created even if you later add an interface . The binding remains in the `PENDING` state until the interface is configured.


![CFD][cfd-bridge-domain]

---

### Example: Interface Re-creation

This example covers the situation where you need to update an interface configuration. The SB doesn't support incremental updates for some items. Instead, the item is first deleted, and then re-created with the new configuration.

!!! Note
    You can view a configuration item update as just an "update operation". The incremental update or full configuration re-creation is performed "under the covers" between the VPP agent and the data plane. Incremental update is always preferred. Configuration updates require full re-creation if the specific item or attribute does not support incremental updates.   

The KV Scheduler supports re-creation scenario using the `UpdateWithRecreate()` method. It lets a descriptor inform the KV Scheduler if an item requires full re-creation before applying the configuration update. 

The control flow diagram shows interface re-creation using a VPP TAP interface, wth a NB request to modify its RX ring size. You cannot modify this configuration because the interface already exists. In addition, an L3 route is attached to the interface. The route cannot exist without the interface. 

After you delete the route, it is moved into the `PENDING` state before interface re-creation. You can then configure the route once the interface re-creation process has completed.


![CFD][cfd-interface-recreation]

---

### Example: Retry of a Failed Operation

This example describes the flows for retrying a failed configuration operation. Transactions can run in `BestEffort` mode. The KV Scheduler will _retry_ failed operations rather than revert to the previously applied configuration operations.

In the diagram below, the `Create()` operation for the TAP interface `my-tap` fails. Before terminating the transaction, the KV Scheduler retrieves the current value of `my-value`. The KV Scheduler cannot assume this is the current value, because the `Create()` operation failed somewhere in-progress.

The KV Scheduler then schedules a *retry transaction*. This process attempts to repair the failure by re-applying the same configuration. Since the `Retrieve()` method did not find a configured `my-tap` interface, the retry transaction repeats the `Create(my-tap)` operation. 


![CFD][cfd-retry-failed]

---

### Example: Transaction Revert

This example shows what happens when a configuration update fails, and you want to revert to a previously applied configuration. Upon the first failure, the configuration update transaction terminates, and reverts to successfully applied changes. This ensures remnants of the failure do not remain in the system.  


!!! Note
    Only gRPC or localclient NB support this behavior. With a KV data store, you need to run in best-effort mode in an attempt to arrive as close as possible to the desired configuration.

In the control-flow diagram below, a transaction to create a VPP interface
`my-tap` with an attached route `my-route` is planned and executed. The interface configuration succeeds, but the route configuration fails.

The KV Scheduler then triggers the `revert procedure`. First, it retrieves the current value of `my-route`. The KV Scheduler cannot assume this is the current value, because the `Create()` operation failed somewhere in-progress. Second, it determines the route is not configured. Now it only needs to delete the interface to undo any executed changes. 

Once the interface is removed, the system returns to its pre-transaction state. Finally, the KV Scheduler returns the transaction error to the NB.


![CFD][cfd-transaction-revert]

---

### Example: Unnumbered Interface

This example discusses the use of unnumbered interfaces. You configure an unnumbered interface without an IP address. However, this interface can borrow an IP address from another interface referred to as a target interface. You must configure the target interface with at least one IP address. 

Normally, the VPP agent represents a given VPP interface using a single key-value pair. Depending on this key alone would only ensure that the target interface exists when a dependent object is being created. 

!!! Note
    The paragraph above only refers to a VPP interface. It does not refer to a VPP interface configured with an IP address.

You can restrict an object's existence based on IP addresses assigned to an interface. To accomplish this, every VPP interface value must derive a unique key-value pair for each assigned address. The KV Scheduler can then reference IP address assignments and build dependencies around them.

An unnumbered interface derives a single value from every interface. The key can indicate if the interface has at least one assigned IP address.
                                                                                                           
Here is an example:
```
vpp/interface/<interface-name>/has-ip/<true/false>
```

Dependency for an unnumbered interface can reference this key:
```
vpp/interface/<interface-name-to-borrow-IP-from>/has-ip/true
```

---

For more complex cases, you can define a dependency based on the presence and value of an assigned IP address. To achieve this, you derive an empty value for each assigned IP address with key template.

Here is an example of a key template:
```
vpp/interface/address/<interface-name>/<address>
```

This complicates the situation for unnumbered interfaces. They can't reference a key of a specific value. Instead, they would need a wildcard key to match any address-representing key. A single match satisfies the dependency. 

Here is an example of a wildcard key: 
```
vpp/interface/address/<interface-name>/*
```

The KV Scheduler offers a more generic solution than wildcards. The dependency is expressed using a callback denoted as `AnyOf`. The callback returns `true` or `false` for a given key. A callback returning `true`, for at least one of the existing configured or derived keys, satisfies the dependency.

Lastly, an unnumbered interface can 
exist even if one or more IP addresses are not available. In this case, an unnumbered interface is derived from the interface value and processed by a separate `UnnumberedIfDescriptor` descriptor. This derived value uses the `AnyOf` callback to trigger an IP address borrow action once the IP addresses become available.

The control-flow diagram below also shows that the removal of the borrowed IP address does not impact an unnumbered interface configuration. The interface first returns the address before the address itself is deleted. You end up with an interface running in L2 mode without an IP address.

![CFD][cfd-unnumbered]

---

### Example: Create VPP interface via KV Data Store

You can create a VPP interface using a KV data store as the VPP agent NB. The detailed control-flow diagram shown below includes all of the interacting components consisting of:

 * NB (KVDB): contains the configuration of a single `my-tap` interface.
 * Orchestrator: listens to the KVDB and propagates the configuration to the KV Scheduler.
 * KVScheduler: plans and executes transaction operations
 * Interface Descriptor: implements CRUD operations for VPP interfaces
 * Interface Model: builds and parses keys identifying VPP interfaces


![CFD][cfd-create-vpp-interface-kvdb]

---

### Example: Create VPP Interface via gRPC

You can create a VPP interface using gRPC as the VPP agent NB, instead of the KV data store.   
We have collapsed the control-flow diagram because there are essentially no differences between the two cases. The KV Scheduler does not care how the desired configuration is conveyed to the VPP agent.

One advantage of the gRPC approach is that the transaction error
value is propagated back to the client, which is then able to react to it
accordingly. 

With the KV data store approach, you do not require a client-to-agent connection. You can submit the configuration prior to or during a VPP agent restart. 



![CFD][cfd-create-vpp-interface-grpc]

---

### Example: Route waiting for the associated interface

This example shows how a route, received from the KV data store, is placed in a `PENDING` state until the interface is configured. In this scenario, the interface key is listed as a route dependency. 

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
