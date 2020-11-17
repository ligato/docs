# KVS Troubleshooting

This section contains troubleshooting information for the KV Scheduler.

---

### NB value not configured in SB

[Look over the transaction logs][understand-transaction-logs], printed
by the agent into `stdout`, and locate the transaction triggered
to configure the value:

1. **Transaction is not triggered** (not found in the logs), or the **value is
   missing** in the transaction input
    - Make sure you register the model
   
!!! danger "Important"
    Not using a model is considered a programming error
    
- [Check if the plugin implementing the value is loaded][debug-plugin-lookup]
- [check if the descriptor associated with the value is registered][how-to-descriptors]
- [check if the key prefix is being watched][how-to-descriptors]
- for NB KV data store, verify the key used to put the value is correct
- check if an incorrect key prefix is used. Note that an incorrect `suffix` renders the value `UNIMPLEMENTED`, but is still included in the transaction input.
- [check the debug logs][debug-logs] of the Orchestrator (NB of KV Scheduler). These can also be used to learn the set of key-value pairs received with each event from NB.

Here is a RESYNC example:
```
DEBU[0005] => received RESYNC event (1 prefixes)         loc="orchestrator/orchestrator.go(150)" logger=orchestrator.dispatcher
DEBU[0005]  -- key: config/mock/v1/interfaces/tap1       loc="orchestrator/orchestrator.go(168)" logger=orchestrator.dispatcher
DEBU[0005]  -- key: config/mock/v1/interfaces/loopback1  loc="orchestrator/orchestrator.go(168)" logger=orchestrator.dispatcher
DEBU[0005] - "config/mock/v1/interfaces/" (2 items)      loc="orchestrator/orchestrator.go(173)" logger=orchestrator.dispatcher
DEBU[0005] 	 - "config/mock/v1/interfaces/tap1": (rev: 0)  loc="orchestrator/orchestrator.go(178)" logger=orchestrator.dispatcher
DEBU[0005] 	 - "config/mock/v1/interfaces/loopback1": (rev: 0)  loc="orchestrator/orchestrator.go(178)" logger=orchestrator.dispatcher
DEBU[0005] Resync with 2 items                           loc="orchestrator/orchestrator.go(181)" logger=orchestrator.dispatcher
DEBU[0005] Pushing data with 2 
KV pairs (source: watcher)  loc="orchestrator/dispatcher.go(67)" logger=orchestrator.dispatcher
DEBU[0005]  - PUT: "config/mock/v1/interfaces/tap1"      loc="orchestrator/dispatcher.go(78)" logger=orchestrator.dispatcher
DEBU[0005]  - PUT: "config/mock/v1/interfaces/loopback1"   loc="orchestrator/dispatcher.go(78)" logger=orchestrator.dispatcher
```
Here is a CHANGE example:
```
DEBU[0012] => received CHANGE event (1 changes)          loc="orchestrator/orchestrator.go(121)" logger=orchestrator.dispatcher
DEBU[0012] Pushing data with 1 KV pairs (source: watcher)  loc="orchestrator/dispatcher.go(67)" logger=orchestrator.dispatcher
DEBU[0012]  - UPDATE: "config/mock/v1/interfaces/tap2"   loc="orchestrator/dispatcher.go(93)" logger=orchestrator.dispatcher
```

2. **Transaction containing the value was triggered**, yet the value is not configured in SB. This could be the result of one of the following:

* Value is `PENDING`

    - [display the graph][how-to-graph] and check the state of dependencies
    - dependency is missing so state is `NONEXISTENT`), or in a failed state indicated by `INVALID`/`FAILED`/`RETRYING`
    - plugin implementing the dependency [is not loaded][debug-plugin-lookup], thus the dependency state is `UNIMPLEMENTED`
    - unintended dependency was added. Verify the implementation of the dependencies method of the value descriptor

---

* Value in the `UNIMPLEMENTED` state
    - for NB KV data store, verify the key suffix used to put the value is correct. The prefix is valid and watched by the VPP agent, but the suffix, normally composed of value primary fields, is malformed. There is a mismatch with the descriptor's `KeySelector`
    - `KeySelector` or `NBKeyPrefix` of the descriptor do not use the model, or does use it, but incorrectly. `NBKeyPrefix` of this or another descriptor selects the value, but `KeySelector` does not

---

* Value *failed* to be applied
    - [display the graph after txn][how-to-graph] and check the state of the value. As long as it is `FAILED`, `RETRYING` or `INVALID`, it is not assumed to be applied properly to SB
    - common error `value has invalid type for key` appears.  This is usually caused by a mismatch between the descriptor and the model
    - set of value dependencies as listed by the descriptor is not complete. Look over the descriptor/ folders of the respective VPP or Linux plugins for additional dependencies needed for the value to be applied properly to SB

---

* Derived Value is treated as `PROPERTY` when it should have CRUD operations assigned
    - Tools to define models for derived values are not available. Developers must implement their own key building/parsing methods which, unless diligently covered by UTs, are prone to error. In corner cases, a mismatch in the assignment of derived values to descriptors could occur.

---

### Resync triggers operations with SB in-sync with NB

- [Run KV Scheduler in verification mode][crud-verification] to check for CRUD inconsistencies
- descriptors of values unnecessarily updated by every resync forget to consider equivalency between attribute values inside [ValueComparator](kvdescriptor.md#descriptor-api). For example, NB defined interface with MTU 0 should be configured in SB with default MTU. For most interface types, this means MTU 0 is equivalent to MTU 1500. This avoids an `update` operation trigger to go from 0 to 1500 or vice-versa.
- if `ValueComparator` is implemented as a separate method, and not as a function literal inside the descriptor structure, do not forget to plug it in via reference

---

### Resync tries to create objects which already exist

- verify `Retrieve` method is implemented for the descriptor of the created objects duplicates, or the method has returned an error:
```
 ERRO[0005] failed to retrieve values, refresh for the descriptor will be skipped  descriptor=mock-interface loc="kvscheduler/refresh.go(104)" logger=kvscheduler
```
- avoid implementing an empty Retrieve method when the operation is not supported by SB

---

### Resync removes item not configured by NB

- use Retrieved with Origin `FromSB` for objects automatically created in SB such as default routes
- `UnknownOrigin` can be used and the KV Scheduler will search the history of transactions to determine if the given value has been configured by the VPP agent
- defaults to `FromSB` when the first resync and transactions history is empty

---

### Retrieve cannot find dependency metadata

- for example, when VPP routes are retrieved, the route descriptor must read the interfaces metadata to translate `sw_if_index` from the routes dump into logical interface names used in NB models. This means interfaces must be dumped first so current metadata is present for the routes retrieval
- `Dependencies` descriptor method is used for the ordering of `Create`, `Update` and `Delete` operations between values. The `RetrieveDependencies` method determines the order of the `Retrieve` operations between descriptors
- if the implementation of the `Retrieve` method reads the metadata of another descriptor, it must be mentioned inside `RetrieveDependencies`

---

### Value re-created when Update should be called

- verify implementation of the `Update` method is plugged into the descriptor structure. Without `Update`, re-creation becomes the only way to apply changes
- verify implementation of `UpdateWithRecreate`. This will result in an unintentional re-creation for the given update

---

### Metadata passed to `Update` or `Delete` is nil

- descriptor attribute `WithMetadata` is not set to `true`. It is insufficient to define the factory only with `MetadataMapFactory`
- metadata for derived values is not supported
- return new metadata in `Update` even if unchanged

---

### Unexpected transaction plan (wrong ordering, missing operations)

* [display the graph visualization][how-to-graph] and check:
     - derived values and dependencies (relations, i.e. graph edges) are correct
     - value states before and after the transaction are correct
  - as a last resort, [follow the KV Scheduler as it walks through the graph][graph-walk] during the transaction processing. Attempt to locate the point where it diverges from the expected path. The descriptor of the value where this occurs is likely to contain bug(s)

---

### Commonly Returned Errors

 * <a name="retrieve-failed"></a>
   **`value (...) has invalid type for key: ...`; Transaction Error**
      - mismatch between the proto message registered with the model and the value type name defined for the [descriptor adapter][descriptor-adapter]

---

 * <a name="retrieve-failed"></a>
   **`failed to retrieve values, refresh for the descriptor will be skipped`; Logs**
    - `Retrieve` of the given descriptor has failed and returned an error
    - KV Scheduler treats failed retrieval as non-fatal. The error is printed to the log as a warning and graph refresh is skipped for the values of the descriptor
    - if this occurs often for a given descriptor, verify implementation of the `Retrieve` operation, ensure  `RetrieveDependencies` mentions all of the dependencies

---

 * <a name="unimplemented-create"></a>
   **`Create operation is not implemented`; Transaction Error**
    - descriptor of the value for which this error was returned is missing the `Create` method. Or it is not plugged into the descriptor structure

---

 * <a name="unimplemented-delete"></a>
   **`Delete operation is not implemented`; Transaction error**
    - descriptor of the value for which this error was returned is missing the `Delete` method. Or it is not plugged into the descriptor structure

---

 * <a name="descriptor-exists"></a>
   **`descriptor already exists` returned by `KVScheduler.RegisterKVDescriptor()`**
    - same descriptor is being registered more than once
    - verify the `Name` attribute of the descriptor is unique across all descriptors for all initialized plugins

---

### Common Programming Mistakes / Bad Practices

 * <a name="changing-value"></a>
   **changing value content inside descriptor methods**
    - values should be treated as if they were mutable, otherwise it could confuse the scheduling algorithm and lead to incorrect transaction plans
    - for example, this is a bug:
``` golang
    func (d *InterfaceDescriptor) Create(key string, iface *interfaces.Interface) (metadata *ifaceidx.IfaceMetadata, err error) {
	    if iface.Mtu == 0 {
    		// MTU not set - work with the default MTU 1500
	    	iface.Mtu = 1500 // BUG - do not change content of "iface" (the value)
    	}
        //...
    }
```
   - only exception in which an input argument can be edited in place is metadata passed to the `Update` method, and are allowed to be re-used. Here is an example:
```
    func (d *InterfaceDescriptor) Update(key string, oldIntf, newIntf *interfaces.Interface, oldMetadata *ifaceidx.IfaceMetadata) (newMetadata *ifaceidx.IfaceMetadata, err error) {
        // ...

        // update metadata
    	oldMetadata.IPAddresses = newIntf.IpAddresses
    	newMetadata = oldMetadata
    	return newMetadata, nil
    }
```

---

 * <a name="stateful-descriptor"></a>
   **implementing stateful descriptors**
    - descriptors are meant to be stateless. Inside callbacks, they should operate only with method input arguments, as received from the KV Scheduler (i.e. key, value, metadata)
    - it is still permitted to implement CRUD operations as methods of a structure. But the structure should only act as a "static context" for the descriptor. Storing references to the logger and SB handler(s) are examples. These are items that do not change once the descriptor is constructed, and typically received as input arguments for the descriptor constructor
    - all key-value pairs and the associated metadata are stored inside the graph, and exposed through [transaction logs](../developer-guide/kvs-troubleshootingmd#understanding-the-kv-scheduler-transaction-log) and [REST APIs][rest-kv-system]. If descriptors do not hide any state internally, the system state will be visible from the outside, and issues will be easier to reproduce
    - use metadata to maintain extra run-time data alongside values. Do `NOT` use context
    - this is considered bad practice:

``` golang
func (d *InterfaceDescriptor) Create(key string, intf *interfaces.Interface) (metadata *ifaceidx.IfaceMetadata, err error) {

	// BAD PRACTISE - do not use your own cache, instead use metadata to store
	// additional run-time data and metadata maps for lookups
	d.myCache[key] = intf
	anotherIface, anotherIfaceExists := d.myCache[<related-interace-key>]
	if anotherIfaceExists {
	    // ...
	}
    //...
}
```

---

* <a name="metadata-with-derived"></a>
  **using metadata with unsupported derived values**
    - associating metadata with a derived value is not supported. Use the parent value instead
    - limitation exists because derived values cannot be retrieved directly. They are derived from from parent values that have already been retrieved. Parent values may have metadata for additional run-time state. Derived values cannot carry additional state-data, beyond what is already included in the metadata of their parent values

---

 * <a name="derived-key-collision"></a>
   **deriving the same key from different values**
    - include the parent value ID inside a derived key, so that derived values do not collide with key-value pairs across the entire system

---

 * <a name="models-bad-usage"></a>
   **not using models for non-derived values; mixing with custom key building/parsing methods**
      - models are `NOT` supported with derived values at this time
      - for non-derived values, the models are mandatory and should be used to define these four descriptor fields: `NBKeyPrefix`, `ValueTypeName`, `KeySelector`, `KeySelector`
      - eventually these fields will all be replaced with a single `Model` reference. It is recommended to prepare and use the models for an easy transition to a new release
      - it is bad practice to use a model, only partially, like so:
```
    descriptor := &adapter.InterfaceDescriptor{
		Name:               InterfaceDescriptorName,
		NBKeyPrefix:        "config/vpp/v1/interfaces/", // BAD PRACTISE, use the model instead
		ValueTypeName:      interfaces.ModelInterface.ProtoName(),
		KeySelector:        interfaces.ModelInterface.IsKeyValid,
		KeyLabel:           interfaces.ModelInterface.StripKeyPrefix,
		// ...
	}
```

---

  * <a name="empty-descriptor-methods"></a>
    **defining unused descriptor methods**
      - over-relying on copy-pasting from [prepared skeletons][descriptor-skeleton] can lead to unused callback skeleton leftovers
      - for example, if the `Update` method is not required, then do not define the method. There is no value in retaining the method, even if empty, in the code
      - descriptors are defined as structures and not as interfaces. This allows unused methods to remain undefined, rather than defined and empty

---

  * <a name="empty-retrieve"></a>
    **using unsupported `Retrieve` to return empty set of values**
      - if the `Retrieve` operation is not supported by SB for a particular value type, then leave the callback undefined in the descriptor
      - KV Scheduler skips refresh for values which cannot be retrieved (undefined `Retrieve`). It assumes that what has been set through previous transactions corresponds with current SB state
      - if instead, `Retrieve` always returns an empty set of values, then the KV Scheduler will re-create every value defined by NB with each resync under the belief the values are missing. This could result in duplicate-value errors

---

  * <a name="manipulating-with-derived"></a>
    **manipulating (derived) value attributes**
      - value attribute derived into a separate key-value pair and handled by CRUD operations of another descriptor, can be imagined as a slice of the original value that was "split off". An implicit dependency on its original value remains, but should no longer be considered as a part of it
      - for example, [if we define a separate derived value for every IP address to be assigned to an interface][derived-if-img], and a separate descriptor which implements these assignments (i.e. `Create` = add IP, `Delete` = unassign IP, etc.), then the descriptor for interfaces should no longer:
         - consider IP addresses when comparing interfaces in `ValueComparator`
         - (un)assign IP addresses in `Create`/`Update`/`Delete`
         - consider IP addresses for interface dependencies

---

  * <a name="retrieve-derived"></a>
    **implementing Retrieve method for descriptor with only derived values in the scope**
      - derived values should never be Retrieved directly (returned by `Retrieve`), but only returned by `DerivedValues()` of the descriptor with retrieved parent values

---

  * <a name="obtained-without-retrieve"></a>
    **not implementing Retrieve() method for values announced to the KV Scheduler as `OBTAINED` via notifications**
      - note that  `OBTAINED` values need to be refreshed, even though they are not touched by resync
      - this is because NB-defined values may depend on `OBTAINED` values. The KV Scheduler needs to know their state to determine if the dependencies are satisfied and plan the resync accordingly

---

  * <a name="blocking-crud"></a>
    **sleeping/blocking inside descriptor methods**
     - transaction processing is synchronous. Sleeping inside a CRUD method would not only delay the remaining operations of the transactions, but other queued transactions as well
     - if a CRUD operation needs to wait for "something", then express that "something" as a separate key-value pair and add it into the list of dependencies. When it becomes available, send notifications using `KVScheduler.PushSBNotification()` and the KV Scheduler will automatically apply pending operations ready for execution
     - in other words, do not hide dependencies inside CRUD operations. Instead, use the framework to express them in a way that is visible to the scheduling algorithm

---

  * <a name="metadata-map-with-write"></a>
    **exposing metadata map with write access**
      - KV Scheduler is the owner of metadata maps, making sure they are current. This is why custom metadata maps are not created by descriptors, but instead given to the KV Scheduler in the form of factories (`KVDescriptor.MetadataMapFactory`)
      - maps retrieved from the KV Scheduler using `KVScheduler.GetMetadataMap()` should remain read-only and exposed to other plugins

---

## Debugging

You can change the agent's log level globally, or individually per logger

 - via the [logs.conf][logs-conf-file] configuration file
 - environment variable `INITIAL_LOGLVL=<level>`
 - during run-time through the agent's REST API: `POST /log/<logger-name>/<log-level>`.

 Detailed info about setting log levels in the agent can be found in the [log manager plugin documentation][logmanager-readme].

The KV Scheduler prints [transaction logs][understand-transaction-logs] or [graph walk logs][graph-walk] directly to `stdout`. The output is intended to provide sufficient information and visibility to debug and resolve KV Scheduler issues.

KV Scheduler-internal debug messages, which require some knowledge of the underlying implementation, are logged to `kvscheduler` logger.

---

### How-to debug agent plugin lookup

The easiest way to determine if your plugin has been found and properly initialized by the [plugin lookup](plugin-lookup.md) procedure is to enable verbose lookup logs. Before the agent
is started, set the `DEBUG_INFRA` environment variable as follows:
``` 
export DEBUG_INFRA=lookup
```

Then search for `FOUND PLUGIN: <your-plugin-name>` in the logs. If you do not find a log entry for your plugin, it means that it is either not listed among 
the agent dependencies or it does not implement the [plugin interface][plugin-interface].

---

### How-to list registered descriptors and watched key prefixes

To determine what descriptors are registered with the KV Scheduler, use the REST API `GET /scheduler/dump` without any arguments.

For example:
```
$ curl localhost:9191/scheduler/dump
{
  "Descriptors": [
    "vpp-bd-interface",
    "vpp-interface",
    "vpp-bridge-domain",
    "vpp-dhcp",
    "vpp-l2-fib",
    "vpp-unnumbered-interface",
    "vpp-xconnect"
  ],
  "KeyPrefixes": [
    "config/vpp/v2/interfaces/",
    "config/vpp/l2/v2/bridge-domain/",
    "config/vpp/l2/v2/fib/",
    "config/vpp/l2/v2/xconnect/"
  ],
  "Views": [
    "SB",
    "NB",
    "cached"
  ]
}
```

With this API, you can also find out which key prefixes are being watched for in the agent NB. This is useful when a value requested by NB is not being applied to SB. If the value key prefix or the associated descriptor are not registered, the value will not be delivered to the KV Scheduler.

---

### Understanding the KV Scheduler transaction log

The KV Scheduler prints a summary of every executed transaction to `stdout`. The output describes

- transaction type
- assigned sequence number
- values to be changed
- transaction plan prepared by the scheduling algorithm
- actual sequence of executed operations, which may differ from the plan if there were any errors.
 
An example of transaction log output with explanations:

![NB transaction][txn-update-img]

---

resync transaction that failed to apply one value:

![Full Resync with error][resync-with-error-img]

---

Retry transaction automatically triggered for the failed operation from the resync transaction shown above

![Retry of failed operations][retry-txn-img]

---

In addition, before a [Full or Downstream Resync][resync] (but not for Upstream Resync), or after a transaction error, the KV Scheduler dumps the state of the graph to `stdout` *after* it was refreshed:

![Graph dump][graph-dump-img]

---

### CRUD verification mode

The KV Scheduler supports a verification mode to check the correctness of CRUD operations provided by descriptors. If enabled, the KV Scheduler will trigger verification inside the post-processing stage of every transaction. The values changed by the transaction (i.e the created / updated / deleted values) are re-read using the `Retrieve` methods from the descriptors, and compared to the intended values to verify they have been applied correctly.

A failed check may mean that the affected values have been changed by some external entity, or that some of the CRUD operations of the corresponding descriptor(s) are not implemented correctly. Note that since the SB values are re-read  immediately after the changes have been applied, it is unlikely that they have been modified by an external entity.

The verification mode is costly. `Retrieve` operations are run after every transaction for descriptors with changed values. Because of this overhead, verification mode is disabled by default and not recommended for use in production environments.

However, for development and testing purposes, the feature is very handy. CRUD implementation bugs can be quickly discovered. We recommend testing newly implemented descriptors in verification mode before they are released. Also, consider the use of the feature with regression test suites.

The verification mode is enabled using the environment variable before the agent is started: `export KVSCHED_VERIFY_MODE=1`

Values with read-write inconsistencies are reported in the transaction output with the [verification error information][verification-error] attached.

---

### How to visualize the graph

A graph-based representation of the system state, used internally by the KV Scheduler, can be displayed using any modern web browser supporting SVG.

Here is the URL:
```
http://<host>:9191/scheduler/graph
```
!!! note  
    9191 is the default port number for the REST API, but it can be changed in the conf file of the [REST plugin][rest-plugin].

The requirement is to have the `dot` renderer from graphviz installed on the host which is running the agent. The renderer is shipped with the [graphviz package](http://graphviz.org/).

Ubuntu version install command:
```
root$ apt-get install graphviz
```

An example of a rendered graph can be seen below. Graph vertices, drawn as rectangles, are used to represent key-value pairs. Derived values have rounded corners. Different fill-colors represent different value states. If you hover over a graph node, a tooltip will pop up, describing the state and content of the corresponding value.

The edges are used to show the relationships between values:

 * black arrows point to dependencies of values they originate from
 * gold arrows connect derived values with their parent values, with cursors
   oriented backwards, pointing to the parents

![graph example][graph-visualization-img]

Without any GET arguments, the API returns the rendering of the graph in its current state. Alternatively, it is possible to pass argument `txn=<seq-num>`, to display the graph state as it was when the given transaction had just finalized, highlighting the vertices updated by the transaction with a yellow border. For example, to display the state of the graph after the first transaction, access URL:
```
http://<host>:9191/scheduler/graph?txn=0
```

See the [KV Scheduler REST API](kvscheduler.md#rest-api) section for more information.

The graph visualization tool is quite helpful for debugging. It provides an instantaneous global view of the system state. The source of potential problems can be easily pinpointed. The result is reduced time and effort in development and troubleshooting.

---

### Understanding the graph walk (advanced)

To observe and understand how the KV Scheduler walks through the graph to process transactions, define the environment variable `KVSCHED_LOG_GRAPH_WALK` before the agent is started. This generates verbose logs showing how the graph nodes are visited by the scheduling algorithm.

The KV Scheduler may visit a graph node in one of the transaction processing stages:

1. graph refresh
2. transaction simulation
3. transaction execution

---

### Graph refresh

During the graph refresh, some or all the registered descriptors are asked to `Retrieve` the values currently created in the SB. Nodes corresponding to the retrieved values are refreshed by the method `refreshValue()`. This method propagates the call to `refreshAvailNode()` for the node itself, and for every value which is derived from the node, and therefore subject to refresh. The method updates the value state and its content to reflect the retrieved data.

Obsolete derived values (those previously derived, but no longer relevant given the the latest retrieved revision of the value), are visited with `refreshUnavailNode()`, marking them as unavailable in the SB. Finally, the graph refresh procedure visits all nodes for which the values were `not` retrieved and marks them as unavailable through method `refreshUnavailNode()`.

The control-flow is depicted in the following diagram:

![Graph refresh diagram][graph-refresh]

Example verbose log of graph refresh as printed by the KV Scheduler to stdout:
```
[BEGIN] refreshGrap (keys=<ALL>)
  [BEGIN] refreshValue (key=config/vpp/v2/interfaces/loopback1)
    [BEGIN] refreshAvailNode (key=config/vpp/v2/interfaces/loopback1)
      -> change value state from NONEXISTENT to DISCOVERED
    [END] refreshAvailNode (key=config/vpp/v2/interfaces/loopback1)
  [END] refreshValue (key=config/vpp/v2/interfaces/loopback1)
  [BEGIN] refreshValue (key=config/vpp/v2/interfaces/tap1)
    [BEGIN] refreshAvailNode (key=config/vpp/v2/interfaces/tap1)
      -> change value state from NONEXISTENT to DISCOVERED
    [END] refreshAvailNode (key=config/vpp/v2/interfaces/tap1)
  [END] refreshValue (key=config/vpp/v2/interfaces/tap1)
  [BEGIN] refreshValue (key=config/vpp/v2/interfaces/UNTAGGED-local0)
    [BEGIN] refreshAvailNode (key=config/vpp/v2/interfaces/UNTAGGED-local0)
      -> change value state from NONEXISTENT to OBTAINED
    [END] refreshAvailNode (key=config/vpp/v2/interfaces/UNTAGGED-local0)
  [END] refreshValue (key=config/vpp/v2/interfaces/UNTAGGED-local0)
  [BEGIN] refreshValue (key=config/vpp/l2/v2/bridge-domain/bd1)
    [BEGIN] refreshAvailNode (key=config/vpp/l2/v2/bridge-domain/bd1)
      -> change value state from NONEXISTENT to DISCOVERED
    [END] refreshAvailNode (key=config/vpp/l2/v2/bridge-domain/bd1)
    [BEGIN] refreshAvailNode (key=vpp/bd/bd1/interface/loopback1, is-derived)
      -> change value state from NONEXISTENT to DISCOVERED
    [END] refreshAvailNode (key=vpp/bd/bd1/interface/loopback1, is-derived)
  [END] refreshValue (key=config/vpp/l2/v2/bridge-domain/bd1)
  [BEGIN] refreshValue (key=config/vpp/l2/v2/fib/bd1/mac/02:fe:d9:9f:a2:cf)
    [BEGIN] refreshAvailNode (key=config/vpp/l2/v2/fib/bd1/mac/02:fe:d9:9f:a2:cf)
      -> change value state from NONEXISTENT to OBTAINED
    [END] refreshAvailNode (key=config/vpp/l2/v2/fib/bd1/mac/02:fe:d9:9f:a2:cf)
  [END] refreshValue (key=config/vpp/l2/v2/fib/bd1/mac/02:fe:d9:9f:a2:cf)
[END] refreshGrap (keys=<ALL>)
```

---

### Transaction simulation / execution

Both transaction simulation and execution follow the same algorithm. The only difference is that during the simulation, the CRUD operations provided by descriptors are `not executed`. Instead, the calls are simulated with a nil error value returned. In addition, all graph updates performed during the simulation are discarded at the end. If a transaction executes without any errors, however, the path taken through the graph by the scheduling algorithm will be the same for both simulation and execution.

The main for-cycle of the transaction processing engine visits every value to be modified by the transaction using the method `applyValue()`. This method determines which of the `applyCreate()` / `applyUpdate()` / `applyDelete()` methods to execute, based on the current and the new value data to be applied.

Update of a value often requires some related values to be updated as well. This is handled through *recursion*. For example, `applyCreate()` will use `applyDerived()` method to call `applyValue()` for every derived value to be created. Additionally, once the value is created, `applyCreate()` will call `runDepUpdates()` to recursively call `applyValue()` for values which are dependent on the created value.  These are currently in a `PENDING` state from previous transaction, but now with the dependency satisfied, are ready to be created. Similarly, `applyUpdate()` and `applyDelete()` may also cause the KV Scheduler to recursively continue and *walk* through the edges of the graph to update related values.

The control-flow of transaction processing is depicted in the following diagram:
 
![KVScheduler diagram][graph-wal-img]

Example verbose log of transaction processing as printed by the KV Scheduler to stdout:
```
[BEGIN] simulate transaction (seqNum=1)
  [BEGIN] applyValue (key = config/vpp/v2/interfaces/tap2)
    [BEGIN] applyCreate (key = config/vpp/v2/interfaces/tap2)
      [BEGIN] applyDerived (key = config/vpp/v2/interfaces/tap2)
      [END] applyDerived (key = config/vpp/v2/interfaces/tap2)
      -> change value state from NONEXISTENT to CONFIGURED
      [BEGIN] runDepUpdates (key = config/vpp/v2/interfaces/tap2)
        [BEGIN] applyValue (key = vpp/bd/bd1/interface/tap2)
          [BEGIN] applyCreate (key = vpp/bd/bd1/interface/tap2)
            -> change value state from PENDING to CONFIGURED
            [BEGIN] runDepUpdates (key = vpp/bd/bd1/interface/tap2)
              [BEGIN] applyValue (key = config/vpp/l2/v2/fib/bd1/mac/aa:aa:aa:bb:bb:bb)
                [BEGIN] applyCreate (key = config/vpp/l2/v2/fib/bd1/mac/aa:aa:aa:bb:bb:bb)
                  [BEGIN] applyDerived (key = config/vpp/l2/v2/fib/bd1/mac/aa:aa:aa:bb:bb:bb)
                  [END] applyDerived (key = config/vpp/l2/v2/fib/bd1/mac/aa:aa:aa:bb:bb:bb)
                  -> change value state from PENDING to CONFIGURED
                  [BEGIN] runDepUpdates (key = config/vpp/l2/v2/fib/bd1/mac/aa:aa:aa:bb:bb:bb)
                  [END] runDepUpdates (key = config/vpp/l2/v2/fib/bd1/mac/aa:aa:aa:bb:bb:bb)
                  [BEGIN] applyDerived (key = config/vpp/l2/v2/fib/bd1/mac/aa:aa:aa:bb:bb:bb)
                  [END] applyDerived (key = config/vpp/l2/v2/fib/bd1/mac/aa:aa:aa:bb:bb:bb)
                [END] applyCreate (key = config/vpp/l2/v2/fib/bd1/mac/aa:aa:aa:bb:bb:bb)
              [END] applyValue (key = config/vpp/l2/v2/fib/bd1/mac/aa:aa:aa:bb:bb:bb)
            [END] runDepUpdates (key = vpp/bd/bd1/interface/tap2)
          [END] applyCreate (key = vpp/bd/bd1/interface/tap2)
        [END] applyValue (key = vpp/bd/bd1/interface/tap2)
      [END] runDepUpdates (key = config/vpp/v2/interfaces/tap2)
      [BEGIN] applyDerived (key = config/vpp/v2/interfaces/tap2)
      [END] applyDerived (key = config/vpp/v2/interfaces/tap2)
    [END] applyCreate (key = config/vpp/v2/interfaces/tap2)
  [END] applyValue (key = config/vpp/v2/interfaces/tap2)
  [BEGIN] applyValue (key = config/vpp/l2/v2/bridge-domain/bd1)
    [BEGIN] applyUpdate (key = config/vpp/l2/v2/bridge-domain/bd1)
      [BEGIN] applyDerived (key = config/vpp/l2/v2/bridge-domain/bd1)
      [END] applyDerived (key = config/vpp/l2/v2/bridge-domain/bd1)
      [BEGIN] applyDerived (key = config/vpp/l2/v2/bridge-domain/bd1)
        [BEGIN] applyValue (key = vpp/bd/bd1/interface/loopback1)
          [BEGIN] applyUpdate (key = vpp/bd/bd1/interface/loopback1)
          [END] applyUpdate (key = vpp/bd/bd1/interface/loopback1)
        [END] applyValue (key = vpp/bd/bd1/interface/loopback1)
        [BEGIN] applyValue (key = vpp/bd/bd1/interface/tap1)
          [BEGIN] applyCreate (key = vpp/bd/bd1/interface/tap1)
            -> change value state from NONEXISTENT to CONFIGURED
            [BEGIN] runDepUpdates (key = vpp/bd/bd1/interface/tap1)
              [BEGIN] applyValue (key = config/vpp/l2/v2/fib/bd1/mac/cc:cc:cc:dd:dd:dd)
                [BEGIN] applyCreate (key = config/vpp/l2/v2/fib/bd1/mac/cc:cc:cc:dd:dd:dd)
                  [BEGIN] applyDerived (key = config/vpp/l2/v2/fib/bd1/mac/cc:cc:cc:dd:dd:dd)
                  [END] applyDerived (key = config/vpp/l2/v2/fib/bd1/mac/cc:cc:cc:dd:dd:dd)
                  -> change value state from PENDING to CONFIGURED
                  [BEGIN] runDepUpdates (key = config/vpp/l2/v2/fib/bd1/mac/cc:cc:cc:dd:dd:dd)
                  [END] runDepUpdates (key = config/vpp/l2/v2/fib/bd1/mac/cc:cc:cc:dd:dd:dd)
                  [BEGIN] applyDerived (key = config/vpp/l2/v2/fib/bd1/mac/cc:cc:cc:dd:dd:dd)
                  [END] applyDerived (key = config/vpp/l2/v2/fib/bd1/mac/cc:cc:cc:dd:dd:dd)
                [END] applyCreate (key = config/vpp/l2/v2/fib/bd1/mac/cc:cc:cc:dd:dd:dd)
              [END] applyValue (key = config/vpp/l2/v2/fib/bd1/mac/cc:cc:cc:dd:dd:dd)
            [END] runDepUpdates (key = vpp/bd/bd1/interface/tap1)
          [END] applyCreate (key = vpp/bd/bd1/interface/tap1)
        [END] applyValue (key = vpp/bd/bd1/interface/tap1)
        [BEGIN] applyValue (key = vpp/bd/bd1/interface/tap2)
          [BEGIN] applyUpdate (key = vpp/bd/bd1/interface/tap2)
          [END] applyUpdate (key = vpp/bd/bd1/interface/tap2)
        [END] applyValue (key = vpp/bd/bd1/interface/tap2)
      [END] applyDerived (key = config/vpp/l2/v2/bridge-domain/bd1)
    [END] applyUpdate (key = config/vpp/l2/v2/bridge-domain/bd1)
  [END] applyValue (key = config/vpp/l2/v2/bridge-domain/bd1)
[END] simulate transaction (seqNum=1)
```

[crud-verification]: kvs-troubleshooting.md#crud-verification-mode
[debug-logs]: kvs-troubleshooting.md#debugging
[debug-plugin-lookup]: kvs-troubleshooting.md#how-to-debug-agent-plugin-lookup
[derived-if-img]: ../img/developer-guide/derived-interface-ip.svg
[descriptor-adapter]: ../developer-guide/kvdescriptor.md#descriptor-adapter
[descriptor-skeleton]: ../developer-guide/kvdescriptor.md#descriptor-skeletons
[graph-dump-img]: ../img/developer-guide/graph-dump.png
[graph-refresh]: ../img/developer-guide/graph-refresh.svg
[graph-visualization-img]: ../img/developer-guide/graph-visualization.svg
[graph-walk]: kvs-troubleshooting.md#understanding-the-graph-walk-advanced
[graph-wal-img]: ../img/developer-guide/graph-walk.svg
[how-to-descriptors]: kvs-troubleshooting.md#how-to-list-registered-descriptors-and-watched-key-prefixes
[how-to-graph]: kvs-troubleshooting.md#how-to-visualize-the-graph
[logs-conf-file]: https://github.com/ligato/cn-infra/blob/master/logging/logmanager/logs.conf
[logmanager-readme]: ../plugins/infra-plugins.md#log-manager
[plugin-interface]: https://github.com/ligato/cn-infra/blob/425b8dd352626b88fb36713d7589ac9fc678bdb7/infra/infra.go#L8-L16
[rest-plugin]: ../plugins/connection-plugins.md#rest-plugin
[rest-kv-system]: ../developer-guide/kvscheduler.md#rest-api
[resync]: ../developer-guide/kvscheduler.md#resync
[resync-with-error-img]: ../img/developer-guide/resync-with-error.png
[retry-txn-img]: ../img/developer-guide/retry-txn.png
[txn-update-img]: ../img/developer-guide/txn-update.png
[understand-transaction-logs]: kvs-troubleshooting.md#understanding-the-kv-scheduler-transaction-log
[verification-error]: https://github.com/ligato/vpp-agent/blob/de1a2254298d61c5712b8e4d6a4b24648b229f04/plugins/kvscheduler/api/errors.go#L162-L213

*[KVDB]: Key-Value Database
*[REST]: Representational State Transfer
*[URL]: Uniform Resource Locator
*[VPP]: Vector Packet Processing
