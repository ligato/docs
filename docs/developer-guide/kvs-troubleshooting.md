# KVS Troubleshooting

This page contains troubleshooting information for KV Scheduler.

---

### Value entered via NB was not configured in SB

[Look over the transaction logs][understand-transaction-logs], printed
by the agent into `stdout`, and locate the transaction triggered
to configure the value:

1. **Transaction is not triggered** (not found in the logs) or the **value is
   missing** in the transaction input
   - make sure the model is registered
   
!!! danger "Important"
    Not using a model is considered a programming error
    
- [check if the plugin implementing the value is loaded][debug-plugin-lookup]
- [check if the descriptor associated with the value is registered][how-to-descriptors]
- [check if the key prefix is being watched][how-to-descriptors]
- for NB KV data store, verify the key used to put the value is correct
- check if an incorrect key prefix is used. Note that an incorrect suffix renders the value `UNIMPLEMENTED`, but it is included in the transaction input.
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
DEBU[0005] Pushing data with 2 KV pairs (source: watcher)  loc="orchestrator/dispatcher.go(67)" logger=orchestrator.dispatcher
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

    - [display the graph][how-to-graph] and check the state of dependencies, by following the black arrows originating from the value
    - dependency is missing so state is `NONEXISTENT`), or in a failed state indicated by `INVALID`/`FAILED`/`RETRYING`)
    - plugin implementing the dependency [is not loaded][debug-plugin-lookup], thus the dependency state is `UNIMPLEMENTED`)
    - unintended dependency was added. Verify the implementation of the dependencies method of the value descriptor

---

* Value in the `UNIMPLEMENTED` state
    - for NB KV data store, verify the key suffix used to put the value is correct. The prefix is valid and watched by the vpp-agent, but the suffix, normally composed of value primary fields, is malformed. There is a mismatch with the descriptor's `KeySelector`
    - `KeySelector` or `NBKeyPrefix` of the descriptor do not use the model, or does use it, but incorrectly. `NBKeyPrefix` of this or another descriptor selects the value, but `KeySelector` does not

---

* Value *failed* to be applied
    - [display the graph after txn][how-to-graph] and check the state of the value. As long as it is `FAILED`, `RETRYING` or `INVALID`, it is not assumed to be applied properly to SB
    - common error `value has invalid type for key` appears.  This is usually caused by a mismatch between the descriptor and the model
    - set of value dependencies as listed by the descriptor is not complete. Look over the descriptor/ folders of the respective VPP or Linux plugins for any additional dependencies needed for the value to be applied properly to SB

---

* Derived Value is treated as `PROPERTY` when it should have CRUD operations assigned
    - we do not yet provide tools to define models for derived values. Developers must implement their own key building/parsing methods which, unless diligently covered by UTs, are prone to error. In corner cases, a mismatch in the assignment of derived values to descriptors could occur

### Resync triggers some operations even if the SB is in fact in-sync with NB

- [consider running the KVScheduler in the verification mode][crud-verification] to check for CRUD inconsistencies
- descriptors of values unnecessarily updated by every resync often forget to consider equivalency between some attribute values inside `ValueComparator` \- for example, interface defined by NB with MTU 0 is supposed to be configured in SB with the default MTU, which, for most of the interface types, means that MTU 0 should be considered as equivalent to 1500 (i.e. not needing to trigger the `Update` operation to go from 0 to 1500 or vice-versa)
- also, if `ValueComparator` is implemented as a separate method and not as a function literal inside the descriptor structure, make sure to not forget to plug it in via reference then

### Resync tries to create objects which already exist

- most likely you forgot to implement the `Retrieve` method for the descriptor of duplicately created objects, or the method has returned an error:
```
 ERRO[0005] failed to retrieve values, refresh for the descriptor will be skipped  descriptor=mock-interface loc="kvscheduler/refresh.go(104)" logger=kvscheduler
```
- make sure to avoid a common mistake of implementing empty Retrieve method when the operation is not supported by SB

### Resync removes item not configured by NB

- objects not configured by the agent, but instead created in SB automatically (e.g. default routes) should be Retrieved with Origin `FromSB`
- `UnknownOrigin` can also be used and the scheduler will search through the history of transactions to see if the given value has been configured by the agent
- defaults to `FromSB` when history is empty (first resync)

### Retrieve fails to find metadata for a dependency

- for example, when VPP routes are being retrieved, the Route descriptor must also read metadata of interfaces to translate `sw_if_index` from the dump of routes into the logical interface names as used in NB models, meaning that interfaces must be dumped first to have their metadata up-to-date for the retrieval of routes
- while the descriptor method `Dependencies` is used to restrict ordering of `Create`, `Update` and `Delete` operations between values,`RetrieveDependencies` is used to determine the ordering for the `Retrieve` operations between descriptors
- if your implementation of the `Retrieve` method reads metadata of another descriptor, it must be mentioned inside `RetrieveDependencies`

### Value re-created when just Update should be called instead

- check that the implementation of the `Update` method is indeed plugged into the descriptor structure (without `Update`, the re-creation becomes the only way to apply changes)
- double-check your implementation of `UpdateWithRecreate` - perhaps you unintentionally requested the re-creation for the given update

### Metadata passed to `Update` or `Delete` are unexpectedly nil

- could be that descriptor attribute `WithMetadata` is not set to `true` (it is not enough to just define the factory with `MetadataMapFactory`)
- metadata for derived values are not supported (so don't expect to receive anything else than nil)
- perhaps you forgot to return the new metadata in `Update` even if they have not changed

### Unexpected transaction plan (wrong ordering, missing operations)

- [display the graph visualization][how-to-graph] and check:
  - if derived values and dependencies (relations, i.e. graph edges) are as expected
    - if the value states before and after the transaction are as expected
  - as a last resort, [follow the scheduler as it walks through the graph][graph-walk] during the transaction processing and try to localize the point where it diverges from the expected path - descriptor of the value where it happens is likely to have some bug(s)

### Commonly returned errors

 * <a name="value-invalid-type"></a>
   `value (...) has invalid type for key: ...` (transaction error)
    - mismatch between the proto message registered with the model and the value type name defined for the [descriptor adapter][descriptor-adapter]

 * <a name="retrieve-failed"></a>
   `failed to retrieve values, refresh for the descriptor will be skipped` (logs)
    - `Retrieve` of the given descriptor has failed and returned an error
    - the scheduler treats failed retrieval as non-fatal - the error is printed to the log as a warning and the graph refresh is skipped for the values of that particular descriptor
    - if this happens often for a given descriptor, double-check its implementation of the `Retrieve` operation and also make sure that `RetrieveDependencies` properly mentions all the dependencies

 * <a name="unimplemented-create"></a>
   `operation Create is not implemented` (transaction error)
    - descriptor of the value for which this error was returned is missing implementation of the `Create` method - perhaps the method is implemented, but it is not plugged into the descriptor structure?

 * <a name="unimplemented-delete"></a>
   `operation Delete is not implemented` (transaction error)
    - descriptor of the value for which this error was returned is missing implementation of the `Delete` method - perhaps the method is implemented, but it is not plugged into the descriptor structure?

 * <a name="descriptor-exists"></a>
  `descriptor already exist` (returned by `KVScheduler.RegisterKVDescriptor()`)
    - returned when the same descriptor is being registered more than once
    - make sure the `Name` attribute of the descriptor is unique across all descriptors of all initialized plugins

### Common programming mistakes / bad practises

 * <a name="changing-value"></a>
   **changing the value content inside descriptor methods**
    - values should be manipulated as if they were mutable, otherwise it could confuse the scheduling algorithm and lead to incorrect transaction plans
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
   - the only exception when input argument can be edited in-place are metadata passed to the `Update` method, which are allowed to be re-used, e.g.:
```
    func (d *InterfaceDescriptor) Update(key string, oldIntf, newIntf *interfaces.Interface, oldMetadata *ifaceidx.IfaceMetadata) (newMetadata *ifaceidx.IfaceMetadata, err error) {
        // ...

        // update metadata
    	oldMetadata.IPAddresses = newIntf.IpAddresses
    	newMetadata = oldMetadata
    	return newMetadata, nil
    }
```

 * <a name="stateful-descriptor"></a>
   **implementing stateful descriptors**
    - descriptors are meant to be stateless and inside callbacks should operate only with method input arguments, as received from the scheduler (i.e. key, value, metadata)
    - it is still allowed (and very common) to implement CRUD operations as methods of a structure, but the structure should only act as a "static
      context" for the descriptor, storing for example references to the logger, SB handler(s), etc. - things that do not change once the descriptor is constructed (typically received as input arguments for the descriptor constructor)
    - to maintain an extra run-time data alongside values, use metadata and not context
    - all key-value pairs and the associated metadata are already stored inside the graph and fully exposed through transaction logs and REST APIs, therefore if descriptors do not hide any state internally, the system state will be fully visible from the outside and issues will be easier to reproduce
    - this would be considered bad practice:
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

 * <a name="metadata-with-derived"></a>
   **trying to use metadata with derived values**
    - it is not supported to associate metadata with a derived value - use the parent value instead
    - the limitation is due to the fact that derived values cannot be retrieved directly (and have metadata received from the `Receive` callback) - instead, they are derived from already retrieved parent values, which effectively means that they cannot carry additional state-data, beyond what is already included in the metadata of their parent values

 * <a name="derived-key-collision"></a>
   **deriving the same key from different values**
    - make sure to include the parent value ID inside a derived key, to ensure that derived values do not collide across all key-value pairs

 * <a name="models-bad-usage"></a>
   **not using models for non-derived values or mixing them with custom key building/parsing methods**
      - it is true that models are work-in-progress and not yet supported with derived values
      - for non-derived values, however, the models are already mandatory and should be used to define these four descriptor fields: `NBKeyPrefix`, `ValueTypeName`, `KeySelector`, `KeySelector` - eventually these fields will all be replaced with a single `Model` reference, hence it is recommended to have the models prepared and already in-use for an easy transition
      - it is also a very bad practise to use a model only partially, e.g.:
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

  * <a name="empty-descriptor-methods"></a>
    **leaving descriptor methods which are not needed defined**
      - relying too much on copy-pasting from [prepared skeletons][descriptor-skeleton] can lead to having unused callback skeleton leftovers - for example, if the `Update` method is not needed (update is always handled via full-recreation), then simply do not define the method instead of leaving it empty
      - descriptors are defined as structures and not as interfaces exactly for this reason - to allow the unused methods to remain undefined instead of being just empty, avoiding what would be otherwise nothing but a boiler-plate code

  * <a name="empty-retrieve"></a>
    **unsupported `Retrieve` defined to always return empty set of values**
      - if the `Retrieve` operation is not supported by SB for a particular value type, then simply leave the callback undefined in the descriptor
      - the scheduler skips refresh for values which cannot be retrieved (undefined `Retrieve`) and will assume that what has been set through previous transactions exactly corresponds with the current state of SB
      - if instead, `Retrieve` always return empty set of values, then the scheduler will re-Create every value defined by NB with each resync, thinking that they are all missing, which is likely to end with duplicate-value kind of errors

  * <a name="manipulating-with-derived"></a>
    **manipulating with value attributes which were derived out**
      - value attribute derived into a separate key-value pair and handled by CRUD operations of another descriptor, can be imagined as a slice of the original value that was split away - it still has an implicit dependency on its original value, but should no longer be considered as a part of it
      - for example, [if we would define a separate derived value for every IP address to be assigned to an interface][derived-if-img], and there would be another descriptor which implements these assignments (i.e. `Create` = add IP, `Delete` = unassign IP, etc.), then the descriptor for interfaces should no longer:
         - consider IP addresses when comparing interfaces in `ValueComparator`
         - (un)assign IP addresses in `Create`/`Update`/`Delete`
         - consider IP addresses for interface dependencies
         - etc.

  * <a name="retrieve-derived"></a>
    **implementing Retrieve method for descriptor with only derived values in the scope**
      - derived values should never be Retrieved directly (returned by `Retrieve`), but always only returned by `DerivedValues()` of the descriptor with retrieved parent values

  * <a name="obtained-without-retrieve"></a>
    **not implementing Retrieve method for values announced to KVScheduler as `OBTAINED` via notifications**
      - it is a common mistake to forget that `OBTAINED` values also need to be refreshed, even though they are not touched by the resync
      - it is because NB-defined values may depend on `OBTAINED` values, and the scheduler therefore needs to know their state to determine if the dependencies are satisfied and plan the resync accordingly

  * <a name="blocking-crud"></a>
    **sleeping/blocking inside descriptor methods**
     - transaction processing is synchronous and sleeping inside a CRUD method would not only delay the remaining operations of the transactions, but other queued transactions as well
     - if a CRUD operation needs to wait for something, then express that "something" as a separate key-value pair and add it into the list of dependencies - then, when it becomes available, send notification using `KVScheduler.PushSBNotification()` and the scheduler will automatically apply pending operations which became ready for execution
     - in other words - do not hide any dependencies inside CRUD operations, instead use the framework to express them in a way that is visible to the scheduling algorithm

  * <a name="metadata-map-with-write"></a>
    **exposing metadata map with write access**
      - the KVScheduler is the owner of metadata maps, making sure they are always up-to-date - this is why custom metadata maps are not created by descriptors, but instead given to the scheduler in the form of factories (`KVDescriptor.MetadataMapFactory`)
      - maps retrieved from the scheduler using `KVScheduler.GetMetadataMap()` should remain read-only and exposed to other plugins as such

## Debugging

You can change the agent's log level globally or individually per logger via the configuration file `logging.conf`, the environment variable `INITIAL_LOGLVL=<level>`
or during run-time through the Agent's REST API: `POST /log/<logger-name>/<log-level>`. Detailed info about setting log levels in the Agent can be found in the [documentation for the logmanager plugin][logmanager-readme].

The KVScheduler prints most of its interesting data, such as [transaction logs][understand-transaction-logs] or [graph walk logs][graph-walk] directly to `stdout`. This output is concise and easy to read, providing enough information and visibility to debug and resolve most of the issues that are in some way related to the KVScheduler framework. These transaction logs are not dependent on any KVScheduler implementation details, and therefore not expected to change much between releases.

KVScheduler-internal debug messages, which require some knowledge of the underlying implementation, are logged using the logger named `kvscheduler`.

### How-to debug agent plugin lookup

The easiest way to determine if your plugin has been found and properly initialized by the Agent's plugin lookup procedure is to enable verbose lookup logs. Before the agent
is started, set the `DEBUG_INFRA` environment variable as follows:
``` 
export DEBUG_INFRA=lookup
```

Then search for `FOUND PLUGIN: <your-plugin-name>` in the logs. If you do not find a log entry for your plugin, it means that it is either not listed among 
the agent dependencies or it does not implement the [plugin interface][plugin-interface].

### How-to list registered descriptors and watched key prefixes

The easiest way to determine what descriptors are registered with the KVScheduler is to use the REST API `GET /scheduler/dump` without any arguments.
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

Moreover, with this API you can also find out which key prefixes are being watched for in the agent NB. This is particularly useful when some value requested by
NB is not being applied into SB. If the value key prefix or the associated descriptor are not registered, the value will not be even delivered into the KVScheduler. 

### Understanding the KVScheduler transaction log

The KVScheduler prints well-formatted and easy-to-read summary of every executed transaction to `stdout`. The output describes the transaction type, the
assigned sequence number, the values to be changed, the transaction plan prepared by the scheduling algorithm and finally the actual sequence of executed operations,
which may differ from the plan if there were any errors.
 
Screenshot of an actual transaction output with explanation:

![NB transaction][txn-update-img]

Screenshot of a resync transaction that failed to apply one value:

![Full Resync with error][resync-with-error-img]

Retry transaction automatically triggered for the failed operation from the resync transaction shown above:

![Retry of failed operations][retry-txn-img]

Furthermore, before a [Full or Downstream Resync][resync] (not for Upstream Resync), or after a transaction error, the KVScheduler dumps the state of the graph to `stdout` *after* it was refreshed:

![Graph dump][graph-dump-img]

### CRUD verification mode

The KVScheduler allows to verify the correctness of CRUD operations provided by descriptors. If enabled, the KVScheduler will trigger verification inside the post-processing stage of every transaction. The values changed by the transaction (i.e the created / updated / deleted values) are re-read (using the `Retrieve` methods from descriptors) and compared to the intended values to verify that they have been applied correctly. A failed check may mean that the affected values have been changed by some external entity or, more likely, that some of the CRUD operations of the corresponding descriptor(s) are not implemented correctly. Note that since the SB values are re-read  practically immediately after the changes have been applied, it is very unlikely that an external entity has changed them.

The verification mode is costly - `Retrieve` operations are run after every transaction for descriptors with changed values - therefore it is disabled by default and not recommended for use in production environments.

However, for development and testing purposes, the feature is very handy and allows to quickly discover bugs ins the CRUD operation implementations. We recommend to test newly implemented descriptors in the verification mode before they are released. Also, consider the use of the feature with regression test suites.      

The verification mode is enabled using the environment variable (before the agent is started): `export KVSCHED_VERIFY_MODE=1`

Values with read-write inconsistencies are reported in the transaction output having the [verification error][verification-error] attached. 

### How-to visualize the graph

The graph-based representation of the system state, as used internally by the KVScheduler, can be displayed using any modern web browser (supporting SVG) at the URL:
```
http://<host>:9191/scheduler/graph
```
!!! note  
    9191 is the default port number for the REST API, but it can be changed in the configuration file for the [REST plugin][rest-plugin-readme].

The requirement is to have the `dot` renderer from graphviz installed on the host which is running the agent. The renderer is shipped with the `graphviz` package, which for Ubuntu can be installed with:
```
root$ apt-get install graphviz
```

An example of a rendered graph can be seen below. Graph vertices, drawn as rectangles, are used to represent key-value pairs. Derived values have rounded corners. Different fill-colors represent different value states. If you hover with the mouse cursor over a graph node, a tooltip will pop up, describing the state and the content of the corresponding value. The edges are used to show relations between values:
 * black arrows point to dependencies of values they originate from
 * gold arrows connect derived values with their parent values, with cursors
   oriented backwards, pointing to the parents  
![graph example][graph-visualization-img]

Without any GET arguments, the API returns the rendering of the graph in its current state. Alternatively, it is possible to pass argument `txn=<seq-num>`, to display the graph state as it was when the given transaction has just finalized, highlighting the vertices updated by the transaction with a yellow border. For example, to display the state of the graph after the first transaction, access URL: 
```
http://<host>:9191/scheduler/graph?txn=0
```

We find the graph visualization tremendously helpful for debugging. It provides an instantaneous global-viewpoint on the system state, often helping to quickly pinpoint the source of a potential problem (for example: why is my object not configured?). 

### Understanding the graph walk (advanced)

To observe and understand how KVScheduler walks through the graph to process transactions, define environment variable `KVSCHED_LOG_GRAPH_WALK` before the agent is started, which will enable verbose logs showing how the graph nodes are visited by the scheduling algorithm.

The scheduler may visit a graph node in one of the transaction processing stages:

1. graph refresh
2. transaction simulation
3. transaction execution

### Graph refresh

During the graph refresh, some or all the registered descriptors are asked to `Retrieve` the values currently created in the SB. Nodes corresponding to the retrieved values are refreshed by the method `refreshValue()`. The method propagates the call further to `refreshAvailNode()` for the node itself and for every value which is derived from it and therefore must be also refreshed. The method updates the value state and its content to reflect the retrieved data. Obsolete derived values (previously derived, but not anymore with the latest retrieved revision of the value), are visited with `refreshUnavailNode()`, marking them as unavailable in the SB. Finally, the graph refresh procedure visits all nodes for which the values were not retrieved and marks them as unavailable through method `refreshUnavailNode()`. The control-flow is depicted by the following diagram:

![Graph refresh diagram][graph-refresh]

Example verbose log of graph refresh as printed by the scheduler to stdout:
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

### Transaction simulation / execution

Both the transaction simulation and the execution follow the same algorithm. The only difference is that during the simulation, the CRUD operations provided by descriptors are not actually executed, but only pretended to be called with a nil error value returned. Also, all the graph updates performed during the simulation are thrown away at the end. If a transaction executes without any errors, however, the path taken through the graph by the scheduling algorithm will be the same for both the execution and the simulation.

The main for-cycle of the transaction processing engine visits every value to be changed by the transaction using the method `applyValue()`. The method determines which of the `applyCreate()` / `applyUpdate()` / `applyDelete()` methods to execute, based on the current and the new value data to be applied.

Update of a value often requires some related values to be updated as well - this is handled through *recursion*. For example, `applyCreate()` will use `applyDerived()` method to call `applyValue()` for every derived value to be created as well. Additionally, once the value is created, `applyCreate()` will call `runDepUpdates()` to recursively call `applyValue()` for values which are depending on the created vale and are currently in the PENDING state from previous transaction, but now with the dependency satisfied are ready to be created.
Similarly, `applyUpdate()` and `applyDelete()` may also cause the scheduling engine to recursively continue and *walk* through the edges of the graph to update related values.
The control-flow of transaction processing is depicted by the following diagram:
 
![KVScheduler diagram][graph-wal-img]

Example verbose log of transaction processing as printed by the scheduler to stdout:
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
[logmanager-readme]: ../plugins/infra-plugins.md#log-manager
[plugin-interface]: https://github.com/ligato/cn-infra/blob/425b8dd352626b88fb36713d7589ac9fc678bdb7/infra/infra.go#L8-L16
[rest-plugin-readme]: https://github.com/ligato/cn-infra/blob/master/rpc/rest/README.md
[resync]: ../developer-guide/kvscheduler.md#resync
[resync-with-error-img]: ../img/developer-guide/resync-with-error.png
[retry-txn-img]: ../img/developer-guide/retry-txn.png
[txn-update-img]: ../img/developer-guide/txn-update.png
[understand-transaction-logs]: kvs-troubleshooting.md#understanding-the-kvscheduler-transaction-log
[verification-error]: https://github.com/ligato/vpp-agent/blob/de1a2254298d61c5712b8e4d6a4b24648b229f04/plugins/kvscheduler/api/errors.go#L162-L213

*[KVDB]: Key-Value Database
*[REST]: Representational State Transfer
*[URL]: Uniform Resource Locator
*[VPP]: Vector Packet Processing
