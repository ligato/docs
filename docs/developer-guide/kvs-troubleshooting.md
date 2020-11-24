# KVS Troubleshooting

---

This section contains troubleshooting information for the KV Scheduler.

---

## Tools

You have a number of tools to help troubleshoot KV Scheduler.    

* [Agentctl](../user-guide/agentctl.md)

* [KV Scheduler](../api/api-kvs.md) and [VPP agent](../api/api-vpp-agent.md) REST APIs

* Transaction Logs.

* Agent log levels.

* KV data store access using `agentctl kvdb`. 

* Environment Variables:
    - export KVSCHED_VERIFY_MODE=1
    - export DEBUG_INFRA=lookup

* Source code

* Other system tools including kubectl and vppctl.

!!! Note
    To identify and resolve problems as quickly as possible, you should first run the [agentctl report command](../user-guide/agentctl.md#report). In most cases, the information contained in the subreports describes specific information on the error and cause.

---

## Problems

You might encounter one or more problems described in this section. Follow the troubleshooting checklist to identify the problem source. 

### NB value not configured in SB

Data collect:

* Run the [agentctl report command](../user-guide/agentctl.md#report).  
* Set log levels for the `kvscheduler` and related component loggers to `debug`. 
* [Review the transaction logs][understand-transaction-logs]. Locate the transaction triggered to configure the value.  

1. **Transaction is not triggered** (not found in the logs), or the **value is
   missing** in the transaction input. 
   
Checklist:

- Review the `_failed-reports.txt` subreport for errors.  
</br>
    
- Make sure you register the model.
   
!!! danger "Important"
    Not using a model is considered a programming error.
    
- [Check if plugin implementing the value is loaded][debug-plugin-lookup].
- [Check if descriptor associated with the value is registered][how-to-descriptors].
- [Check if the key prefix is being watched][how-to-descriptors].
- For NB KV data store, verify the key used to put the value is correct
- Check if an incorrect key prefix is used. Note that an incorrect `suffix` renders the value `UNIMPLEMENTED`, but is still included in the transaction input.
- [check the debug logs][debug-logs] of the Orchestrator (NB of KV Scheduler). Logs show the set of key-value pairs received with each NB event.

RESYNC example orchestrator logger set to `debug`:
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
NB event CHANGE example:
```
DEBU[0012] => received CHANGE event (1 changes)          loc="orchestrator/orchestrator.go(121)" logger=orchestrator.dispatcher
DEBU[0012] Pushing data with 1 KV pairs (source: watcher)  loc="orchestrator/dispatcher.go(67)" logger=orchestrator.dispatcher
DEBU[0012]  - UPDATE: "config/mock/v1/interfaces/tap2"   loc="orchestrator/dispatcher.go(93)" logger=orchestrator.dispatcher
```

2. **Transaction containing the value was triggered**, yet the value is not configured in SB. Possible causes of this problem consist of the following:

* Value is `PENDING`. Checklist:

    - GET [KV Scheduler tnx history](../api/api-kvs.md#transaction-history) 
    - GET [KV Scheduler graph snapshot](../api/api-kvs.md#graph-snapshot)
    - Run the [agentctl report command](../user-guide/agentctl.md#report). Review the `_failed-reports.txt` subreport for errors.
    - [Display the KV Scheduler internal graph][how-to-graph].
    - Check dependencies state.
    - Missing dependency is `NONEXISTENT`, or failed state indicated by `INVALID`/`FAILED`/`RETRYING`.
    - Plugin implementing the dependency [is not loaded][debug-plugin-lookup]. Dependency state is `UNIMPLEMENTED`
    - You added an unintended dependency. Verify your implementation of the [dependencies method](kvdescriptor.md#descriptor-api) for the value descriptor.

---

* Value in the `UNIMPLEMENTED` state. Checklist:

     - Run the [agentctl report command](../user-guide/agentctl.md#report). Review the `_failed-reports.txt` subreport for errors.
     - GET [KV Scheduler Dump](../api/api-kvs.md#dump-viewkey-prefix) for the key prefix in the NB view.
   - Verify a correct key suffix for the value put to the NB KV data store. You might have a malformed key suffix for a valid prefix watched by the VPP agent. You may have a mismatch between the key and the descriptor's `KeySelector`. 
    - Check if the descriptor's `KeySelector` or `NBKeyPrefix` does not use the model, or does use it, but incorrectly. You could have the descriptor's `NBKeyPrefix`, or the `NBKePrefix` of another descriptor select the value, but `KeySelector` does not.

---

* Value SB apply *failed* 

    - Run the [agentctl report command](../user-guide/agentctl.md#report). Review the `_failed-reports.txt` subreport for errors.
    - Review the `agent-transaction-history.txt` subreport or transaction logs. Check the value state. A `FAILED`, `RETRYING` or `INVALID` indicates SB apply did not properly complete. 
    - Look for a `value has invalid type for key` error. This error indicates a possible mismatch between the descriptor and the model.
    - You might have an incomplete Descriptor's value dependencies list. Look over the descriptor/ folders of your VPP or Linux plugins for additional dependencies needed for proper SB value apply.

---

* Treat derived value as `PROPERTY`

    - Run the [agentctl report command](../user-guide/agentctl.md#report). Review the `_failed-reports.txt` subreport for errors.
    - Review the `agent-transaction-history.txt` subreport or transaction logs. Check the value state. A `FAILED`, `RETRYING` or `INVALID` indicates SB apply did not properly complete. &&&   
    - Instead, you should assign CRUD operations. 
    - You _DO NOT_ have programming techniques to define models for derived values.
     - You must implement your own key building/parsing methods. If UT does not cover this, errors likely appear.  
     - A mismatch assigning derived values to descriptors could occur.

---

### Resync triggers operations with SB in-sync with NB

- [Run KV Scheduler in verification mode][crud-verification] to check for CRUD inconsistencies.

- Resync updates of value descriptors do not consider attribute value value equivalency inside [ValueComparator](kvdescriptor.md#descriptor-api). For example, you should configure NB-defined MTU 0 with SB default MTU. You then avoid an `update` operation trigger to switch from 0 to 1500 or vice-versa. 
 

- You implement `ValueComparator`as a separate method.
- You _DO NOT_ implement `ValueComparator` as a function literal inside the descriptor structure
- Do not forget to plug it in via reference.

 

---

### Resync tries to create objects which already exist

- Check that you implement `Retrieve` method for the descriptor of the created objects duplicat
-  `Retrieve` method returns an error:
```
 ERRO[0005] failed to retrieve values, refresh for the descriptor will be skipped  descriptor=mock-interface loc="kvscheduler/refresh.go(104)" logger=kvscheduler
```
- You should avoid implementing an empty `Retrieve()` method for an unsupported SB operation.


---

### Resync removes item not configured by NB

- check REFS
- Use origin `FromSB` orig for an automatically created in SB object. Default route is an example.
- You can use `UnknownOrigin`. KV Scheduler searches transaction history to determine if the VPP agent configured the specific value.
- defaults to `FromSB` when the first resync and transactions history is empty
- `FromSB` defaults for value upon first resync. Transaction history is empty.

---

### Retrieve cannot find dependency metadata

- REFS
- KV Scheduler dumps VPP routes. Route descriptor reads interfaces metadata to translate `sw_if_index` from the routes dump into NB model logical interface names. Therefore, interface dump must occur before routes dump.  
- `Dependencies` descriptor method defines the ordering of `Create`, `Update` and `Delete` operations between values.
- `RetrieveDependencies` method determines the order of the `Retrieve` operations between descriptors.
- If your `Retrieve` method implementation reads another descriptor's metadata, you must mention this in your `RetrieveDependencies` CRUD callback. 

---

### Value re-created when update should be called

- REFS in code
- Verify your `Update` method implementation plugs into the descriptor structure.  
- Without `Update`, re-creation becomes the only way to apply changes.
- Verify your `UpdateWithRecreate` implementation. An unintentional re-creation for the given update  will occur. 

---

### Metadata passed to `Update` or `Delete` is nil


- You did not set the `WithMetadata` to `true`. You cannot define the factory using only the `MetadataMapFactory`.
- Metadata for derived values is not supported.
- You should return new metadata in `Update` even if the derived values are unchanged. 

---

### Unexpected transaction plan (wrong ordering, missing operations)


- Tnx plan reveals wrong ordering or mission operations.
- [display the graph visualization][how-to-graph] REF
- check that you have correct derived values and dependencies.
- Check that you have correct value states before and after the transaction.

- [follow the KV Scheduler as it walks through the graph][graph-walk] during the transaction processing.
- Locate the point where it diverges from the expected path. The descriptor of the value where this occurs likely contains.


---

### Common Returned Errors

 

* **value (...) has invalid type for key: ...**

    - Transaction Error.
    - Mismatch between the proto message registered with the model, and the value type name defined for the [descriptor adapter][descriptor-adapter].

---


*  **failed to retrieve values, refresh for the descriptor will be skipped**
    - Logs
    - Descriptor `Retreive()` method fails and returns an error.
    - KV Scheduler treats failed retrieval as non-fatal. Error prints to the log as a warning. Graph refresh skips for the descriptor values.
    - If you observe this often for a given descriptor, check your `Retrieve` operation implentation, and ensure  `RetrieveDependencies` mentions all of the dependencies

---

 * **Create operation is not implemented**
 
    - Transaction Error*
    - Value descriptor is missing the `Create` method, or `Create` method is not plugged into the descriptor structure. REFS
    - 

---

 * **Delete operation is not implemented** 
 
    - Transaction error
    - Value descriptor is missing the `Create` method, or `Create` method is not plugged into the descriptor structure. REFS

---

*  **descriptor already exists**
 
    - Returned by `KVScheduler.RegisterKVDescriptor()`
    - Same descriptor is registered more than once.
    - Verify descriptor `Name` attribute is unique across all descriptors for all initialized plugins.

---

### Common Programming Mistakes / Bad Practices

*  **changing value content inside descriptor methods**

    - You should treat values as mutable objects. Otherwise, you could confuse the scheduling algorithm, and incorrect transaction plans will result.
    - Example code block showing a bug:
``` golang
    func (d *InterfaceDescriptor) Create(key string, iface *interfaces.Interface) (metadata *ifaceidx.IfaceMetadata, err error) {
	    if iface.Mtu == 0 {
    		// MTU not set - work with the default MTU 1500
	    	iface.Mtu = 1500 // BUG - do not change content of "iface" (the value)
    	}
        //...
    }
```

   - Passing metadata to the `Update` method is the only exception in which you can edit an input argument.
   - Example code block:
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

 
* **Implementing stateful descriptors**

    - Descriptors are stateless. Inside callbacks, descriptors should operate only with method input arguments, as received from the KV Scheduler. 
</br>
</br>
 
    - You can implement CRUD operations as methods of a structure.
</br>
</br>
 
    - However, the structure should only act as a "static context" for the descriptor. Storing references to the logger and SB handler(s) are examples. These items:
        - do not change once you construct the the descriptor.
        - typically received as input arguments for the descriptor constructor.  
 </br>
 </br>
     
    - all key-value pairs and associated metadata:
         - stored inside the graph
         - exposed through [transaction logs](../developer-guide/kvs-troubleshootingmd#understanding-the-kv-scheduler-transaction-log) and [REST APIs][rest-kv-system]. 
     - If descriptors do not hide any state internally, you have external system visibilty. You can more easily reproduce issues.
    - Use metadata to maintain extra run-time data alongside values. Do _NOT_ use context.
    - Code block shows bad practice:

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

*  **Using metadata with unsupported derived values**

    - Associating metadata with a derived value is not supported. Use the parent value instead.
    - limitation exists because the KV Scheduler cannot directly retrieve derived values. 
    - Derived values derive from the retrieved parent values. 
    - Parent values may have metadata for additional run-time state. 
    - Derived values cannot carry additional state-data, beyond any metadata associated with their parent value. 

---


*  **Same key derived from different values**
    - include the parent value ID inside a derived key.
    - Ensures derived values do not collide with key-value pairs across the entire system

---

* **not using models for non-derived values; mixing with custom key building/parsing methods**
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


*  **Defining unused descriptor methods**

      - Careful with copy/paste from [prepared skeletons][descriptor-skeleton]. 
      - You might retain unused callback skeleton leftovers. 
      - For example, if you do not require the `Update` method, then do not define it. Retaining the method in your code, even if emtpy, adds no value.
      - Note that descriptors are defined as structures, and not as interfaces.
      - Unused methods remain undefined, rather than defined and empty.

---

*  **Using unsupported `Retrieve` to return empty set of values**

      - Do not define the `Retrieve` callback in your descriptor, if the SB does not support this operation for a particular value type.
      - KV Scheduler skips refresh for these values. 
      - KV Scheduler assumes value state set through previous transactions corresponds with current SB state. 
      - If your `Retrieve` callback always returns an empty value set, then the KV Scheduler re-creates each NB-defined value, with each resync. KV Scheduler believes the values are missing. 
      - You could encounter duplicate value errors. 

---

*  **Manipulating (derived) value attributes**

      - If you have a value attribute derived into a separate key-value pair, and handled by CRUD operations of another descriptor, you should think of this value attribute as a slice carved out from the original value. 
      - An implicit dependency on its original value remains. However, this value is no longer a part of the original value. 
      - For example, [if you define a separate derived value for every IP address to assign to an interface][derived-if-img], and a separate descriptor which implements these assignments (i.e. `Create` = add IP, `Delete` = unassign IP, etc.), then the interfaces descriptor should no longer:
         - consider IP addresses when comparing interfaces in `ValueComparator`
         - Assign or unassign IP addresses in `Create`/`Update`/`Delete` operations.
         - Consider IP addresses for interface dependencies

---


* **Implementing a Retrieve method for a descriptor with only derived values in scope**
      - You should never return derived values using a `Retrieve` operation.
      - Instead, use the `DerivedValues()` callback of the descriptor with retrieved parent values.

---


* **Not implementing a Retrieve() method for values announced to the KV Scheduler as `OBTAINED` via notifications**

      - Note you must refresh `OBTAINED` values, even if resync does not touch them. 
      - You may have NB-defined values that depend on `OBTAINED` values. The KV Scheduler must know their state to determine if the dependencies are satisfied, and plan the resync.

---

* **Sleeping/blocking inside descriptor methods**

     - Transaction processing is synchronous. Sleeping inside a CRUD method delays remaining transaction operations, and queued transactions.
     - If you have a CRUD operation that wait for "something", then you should define that "something" as a separate key-value pair, and add it to the dependencies list.
     - When it becomes available, send notifications using `KVScheduler.PushSBNotification()`. The KV Scheduler will automatically apply pending operations ready for execution.
     - Do not hide dependencies inside CRUD operations. 

---

* **Exposing metadata map with write access**

     - KV Scheduler owns and maintains metadata maps.
     - Descriptors do not create custom metadata maps.
     - Descriptors convey custom metadata maps to the KV Scheduler as factories `KVDescriptor.MetadataMapFactory`.
      - Maps retrieved from the KV Scheduler using `KVScheduler.GetMetadataMap()` should remain read-only, and exposed to other plugins.

---

## Debugging

### Logging

You can change the agent's log level globally, or individually per logger.

The log level choice consist of one of the following: `debug`,`info`,`warning`,`error`,`fatal`, and `panic`.
 
**Using the conf file or env variable**

* To set a default for all loggers, per-logger levels, and external link hooks, use the [logs.conf][logs-conf-file] configuration file.
</br>
</br>
* To override the entries in the `logs.conf` file, use the `INITIAL_LOGLVL=<level>` environment variable.

The KV Scheduler prints [transaction logs][understand-transaction-logs] to stdout. The logs provide information and visibility to debug and resolve KV Scheduler issues.

KV Scheduler internal debug messages require some knowledge of the underlying implementation. The `kvscheduler` logger prints internal debug messages. 


**Using agentctl log**

* To manage individual log levels, use the [agentctl log command](../user-guide/agentctl.md#log).

list individual loggers and their log levels:
```
agentctl log list
```

Example of setting the KV Scheduler log level to `debug`:
```
agentctl log set kvscheduler debug
```

**Using a REST API**

* To set an individual log level, use the logger REST API.

```
curl -X POST http://localhost:9191/log/<logger-name>/<log-level>
```

The KV Scheduler prints [transaction logs][understand-transaction-logs] to stdout. The logs can provide sufficient information and visibility to debug and resolve KV Scheduler issues.

KV Scheduler internal debug messages, which require some knowledge of the underlying implementation, are logged to the `kvscheduler` logger.

To review more details about managing agent log levels, see the [log manager plugin documentation][logmanager-readme].



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
