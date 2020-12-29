# KVS Troubleshooting

---

This section contains troubleshooting information for the KV Scheduler.

---

## Tools

You have a number of tools to help troubleshoot KV Scheduler.    

* [agentctl](../user-guide/agentctl.md)

* [KV Scheduler](../api/api-kvs.md) and [VPP agent](../api/api-vpp-agent.md) REST APIs

* Transaction Logs.

* Agent log levels.

* KV data store access using `agentctl kvdb`. 

* Environment Variables

* Source code

* Other system tools including kubectl and vppctl.

!!! Note
    To identify and resolve problems as quickly as possible, you should first run [agentctl report](../user-guide/agentctl.md#report). In most cases, the information contained in the subreports describes information on the error and cause. 

---

## Problems

You might encounter one or more problems described in this section. Follow the troubleshooting checklist to identify and fix the problem. 

---

### NB value not configured in SB

Before you look into this problem, gather the following information: 

* Run [agentctl report](../user-guide/agentctl.md#report).
<br>
</br>  
* Set log levels for the `kvscheduler` logger and related component loggers to `debug`. Re-create or re-run your application.
<br>
</br> 
* Print and [review the transaction logs][understand-transaction-logs]. Locate the transaction triggered to configure the value.  

---

1. **Transaction NOT triggered** (not found in the logs), or the **value is
   missing** in the transaction input. 
   
Checklist:

- Review the `_failed-reports.txt` subreport for errors.  
</br>
    
- Register model.
   
!!! danger "Important"
    Not using a model is considered a programming error.
    
- [Load plugin implementing the value][debug-plugin-lookup].
<br>
</br>
- [Register descriptor associated with the value][how-to-descriptors].
<br>
</br>
- [Watching key prefix][how-to-descriptors].
<br>
</br>
- Use correct key to put value to the NB key value data store.
<br>
</br>
- Use correct key prefix. Note that an incorrect `suffix` renders the value `UNIMPLEMENTED`, but still includes value in the transaction input.
<br>
</br>
- [Check the debug logs][debug-logs] of the Orchestrator (NB of KV Scheduler). Logs show the set of key-value pairs received with each NB event.

---

RESYNC orchestrator logger example set to `debug`:
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

---

NB event CHANGE example:
```
DEBU[0012] => received CHANGE event (1 changes)          loc="orchestrator/orchestrator.go(121)" logger=orchestrator.dispatcher
DEBU[0012] Pushing data with 1 KV pairs (source: watcher)  loc="orchestrator/dispatcher.go(67)" logger=orchestrator.dispatcher
DEBU[0012]  - UPDATE: "config/mock/v1/interfaces/tap2"   loc="orchestrator/dispatcher.go(93)" logger=orchestrator.dispatcher
```

---

2. **Transaction containing the value was triggered**, yet the value not configured in SB. 

Possible causes of this problem consist of the following:

* Value is `PENDING`. <br></br> Checklist:

    - Run [agentctl report](../user-guide/agentctl.md#report). Review the `_failed-reports.txt` subreport for errors.<br></br>
    - [GET /scheduler/txn-history](../api/api-kvs.md#transaction-history). <br></br> 
    - [GET /scheduler/graph-snapshot](../api/api-kvs.md#graph-snapshot).<br></br>
    - [Display the KV Scheduler internal graph][how-to-graph].<br></br>  
    - Check dependencies state. Missing dependency shows `NONEXISTENT`, or failed state indicated by `INVALID`/`FAILED`/`RETRYING`.<br></br>
    - [Load plugin][debug-plugin-lookup] implementing the dependency. Dependency state shows `UNIMPLEMENTED`.<br></br>
    - Unintended dependency. Verify your implementation of the [dependencies method](kvdescriptor.md#descriptor-api) for the value descriptor.

---

* Value in the `UNIMPLEMENTED` state.<br></br> Checklist:

     - Run [agentctl report](../user-guide/agentctl.md#report). Review the `_failed-reports.txt` subreport for errors.<br></br>
     - [GET /scheduler/dump](../api/api-kvs.md#dump-viewkey-prefix) for the key prefix in the NB view.<br></br>
   - Use correct key suffix to put value to the NB key value data. You might have a malformed key suffix for a valid prefix watched by the VPP agent. You may have a mismatch between the key and the descriptor's `KeySelector`.<br></br> 
    - Descriptor's `KeySelector` or `NBKeyPrefix` does not use the model, or does use it, but incorrectly. You could have the descriptor's `NBKeyPrefix`, or the `NBKePrefix` of another descriptor select the value, but `KeySelector` does not.

---

* Value SB apply *failed*<br></br> Checklist: 

    - Run [agentctl report](../user-guide/agentctl.md#report). Review the `_failed-reports.txt` subreport for errors.<br></br>
    - Review the `agent-transaction-history.txt` subreport, or transaction logs, and check the value state.  `FAILED`, `RETRYING` or `INVALID` state indicates SB apply did not properly complete.<br></br> 
    - Look for a `value has invalid type for key` error. This indicates a possible mismatch between the descriptor and the model.<br></br>
    - Incomplete set of value dependencies listed by the descriptor. Look over the descriptor/ folders of your VPP or Linux plugins for additional dependencies needed to properly apply the SB value.

---

* Treat derived value as a `PROPERTY`<br></br> Checklist:

    - Instead, you should assign CRUD operations.<br></br> 
    - You _DO NOT_ have programming techniques to define models for derived values. You must implement your own key building/parsing methods. If UT does not cover this, errors likely appear. In certain corner cases, a mismatch assigning derived values to descriptors could occur.

---

### Resync triggers operations with SB in-sync with NB

Checklist:

- [Run KV Scheduler in verification mode][crud-verification] to check for CRUD inconsistencies.
<br>
</br>
- Resync updates of value descriptors do not consider attribute value equivalency inside [ValueComparator](kvdescriptor.md#descriptor-api). For example, you should configure NB-defined MTU 0 with SB default MTU. You avoid an `update()` operation trigger to switch from 0 to 1500 or vice-versa.
<br></br>  
- If you implement `ValueComparator`as a separate method, and not as a function literal inside the descriptor structure, do not forget to plug it in via reference.

 

---

### Resync tries to create objects that already exist

Checklist:

- You implement the `Retrieve` method for the descriptor of the created objects duplicates, or the method returns an error:
```
 ERRO[0005] failed to retrieve values, refresh for the descriptor will be skipped  descriptor=mock-interface loc="kvscheduler/refresh.go(104)" logger=kvscheduler
```
- You should avoid implementing an empty `Retrieve()` method for an unsupported SB operation.


---

### Resync removes item that is not configured by NB

Checklist:

- Use origin `FromSB` for an automatically created SB object. Default route is an example.<br></br>
- You can use `UnknownOrigin`. KV Scheduler searches the transaction history to determine if the VPP agent configured the specific value.<br></br>
- Defaults to `FromSB` for value upon first resync, and transaction history is empty.

---

### Retrieve cannot find dependency metadata

Checklist:

- As an example, the KV Scheduler dumps VPP routes. The route descriptor reads interfaces metadata to translate `sw_if_index` from the routes dump into NB model logical interface names. Therefore, the interface dump must occur _before_ the routes dump.
<br></br>  
- `Dependencies()` descriptor method defines the ordering of `create()`, `update()` and `delete()` operations between values.
<br></br>
- `RetrieveDependencies()` method determines the order of the `Retrieve()` operations between descriptors.<br></br>
- If your `Retrieve()` method implementation reads another descriptor's metadata, you must mention this in your `RetrieveDependencies()` CRUD callback. 

---

### Value re-created when update should be called

Checklist:

- Verify your `Update()` method implementation plugs into the descriptor structure.
<br></br>  
- Without `Update()`, re-creation becomes the only way to apply changes.
<br></br>
- Check your `UpdateWithRecreate()` implementation. An unintentional re-creation for the given update will occur. 

---

### Nil metadata passed to `Update()` or `Delete()`

Checklist:

- You did not set the `WithMetadata` to `true`. You cannot define the factory using only the `MetadataMapFactory`.
<br></br>
- Metadata for derived values is not supported.
<br></br>
- You should return new metadata in `Update()`, even if the derived values do not change. 

---

### Incorrect transaction plan (wrong ordering, missing operations)

Checklist:

 - Run [agentctl report](../user-guide/agentctl.md#report). Review the `_failed-reports.txt` subreport for errors. 
  <br></br>
- Review transaction logs for inconsistencies.
 <br></br> 
- Call [GET /scheduler/tnx-history](../api/api-kvs.md#transaction-history). You can hone in on the specific txn sequence number using the `seq-num` query parameter.<br></br>  
- [Display the graph visualization][how-to-graph]. Verify you have correct derived values and dependencies, and that you have correct value states before and after the transaction.
 <br></br>
- [Follow the KV Scheduler as it walks through the graph][graph-walk] during the transaction processing. Locate the point where it diverges from the expected path. The descriptor of the value where this occurs likely contains bugs.


---

### Common Returned Errors

The KV Scheduler returns error messages that appear in transaction logs, and agentctl report subreports. In most cases, a programming error or definition omission caused the error.

To review returned errors and the likely cause, see the [errors.go file][errors-go-file]. 

To review errors encountered during refresh, see the [refresh.go file][refresh-go-file].      

---

### Common Programming Mistakes / Bad Practices

This section discusses programming mistakes and bad practices you should avoid in your development efforts.

---

*  **Changing value content inside descriptor methods**

    - You should treat values as mutable objects. Otherwise, you could confuse the scheduling algorithm, and generate incorrect transaction plans.
<br>
</br>
    - Example code block with bug:
``` golang
    func (d *InterfaceDescriptor) Create(key string, iface *interfaces.Interface) (metadata *ifaceidx.IfaceMetadata, err error) {
	    if iface.Mtu == 0 {
    		// MTU not set - work with the default MTU 1500
	    	iface.Mtu = 1500 // BUG - do not change content of "iface" (the value)
    	}
        //...
    }
```
You can only edit an input argument by passing metadata to the `Update()` method. 

Example code block:
       
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

    - Descriptors are stateless. Inside callbacks, descriptors should only operate with method input arguments, as received from the KV Scheduler. 
</br>
</br>
 
    - You can implement CRUD operations as methods of a structure.
</br>
</br>
 
    - However, the structure should only act as a "static context" for the descriptor. Storing references to the logger and SB handler(s) are examples. 
 These items:
 
        - do not change once you construct the descriptor.
        - received as input arguments for the descriptor constructor.  
 </br>
     
    - All key-value pairs and associated metadata are stored inside the KV Scheduler graph, and exposed through [transaction logs](../developer-guide/kvs-troubleshootingmd#understanding-the-kv-scheduler-transaction-log), [REST APIs](../api/api-kvs.md#graph-snapshot), and [agentctl dump](../user-guide/agentctl.md#dump).
<br>
</br>
    
    - If descriptors do not hide any state internally, you have external system visibilty via logs, REST, and agentctl. You can more easily reproduce issues.
</br>
</br>
 
    - Use metadata to maintain extra run-time data alongside values. Do _NOT_ use context.
</br>
</br>
    
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

    - You can't associate metadata with a derived value. Use the parent value instead.
</br>
</br>
 
    - This limitation exists because the KV Scheduler cannot directly retrieve derived values.  These values are derived from the parent values.
</br>
</br>
  
    - Parent values may have metadata for additional run-time state.
</br>
</br>
  
    - Derived values cannot carry additional state data, beyond any parent value metadata. 

---


*  **Same key derived from different values**

    - Include the parent value ID inside a derived key.
</br>
</br>
 
    - This ensures derived values do not collide with key-value pairs across the entire system.

---

* **Not using models for non-derived values; mixing with custom key building/parsing methods**

      - Models are _NOT_ supported with derived values.
<br>
</br>
      - Non-derived values require models. You need models to define these four descriptor fields: `NBKeyPrefix`, `ValueTypeName`, `KeySelector`, and `KeySelector`
<br>
</br>
      - Eventually these fields will all be replaced with a single `Model` reference. It is recommended to prepare and use the models for an easy transition to a new release.
<br></br>
      - It is bad practice to only partially use a model in your descriptor.
<br>
</br>

      - Code block shows bad practice:
      
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

      - Careful with copy/paste from [prepared skeletons][descriptor-skeleton]. You might retain unused callback leftovers.
<br>
</br> 
      - For example, do not define the `Update()` method if you do not need it. You add no value by retaining this method, even if empty.
<br>
</br>

      - Note that the KV Scheduler defines descriptors as structures, and not as interfaces. Unused methods can remain undefined, rather than defined and empty.

---

*  **Using unsupported `Retrieve()` to return an empty set of values**

      - Do not define the `Retrieve()` callback in your descriptor, if the SB does not support this operation for a particular value type.
<br>
</br>

      - KV Scheduler skips refresh for these values.
<br>
</br>
 
      - KV Scheduler assumes the value state set through previous transactions corresponds with the current SB state.
<br>
</br>
 
      - If your `Retrieve()` callback always returns an empty value set, then the KV Scheduler re-creates each NB-defined value, with each resync. The KV Scheduler believes the values are missing.
<br>
</br>
 
      - You could encounter duplicate value errors. 

---

*  **Manipulating (derived) value attributes**

      - If you have a value attribute derived into a separate key-value pair, and handled by CRUD operations of another descriptor, you can think of this value attribute as a "slice" carved out from the original value.
<br>
</br>
 
      - An implicit dependency on its original value remains. However, this value is no longer part of the original value.
<br>
</br>
 
      - For example, [you might define a separate derived value for every IP address to assign to an interface][derived-if-img]. A separate descriptor implements assignments composed of `create()` to add IP, `delete()` to unassign an IP, and `update()` for another operation. Your interfaces descriptor should no longer:
         - consider IP addresses when comparing interfaces in `ValueComparator`
         - Assign or unassign IP addresses in `create()`/`update()`/`delete()` operations.
         - Consider IP addresses for interface dependencies.

---


* **Implementing a `Retrieve()` method for a descriptor with only derived values in scope**

      - You should never return derived values using a `Retrieve()` operation.
<br>
</br>
      - Instead, use the `DerivedValues()` callback of the descriptor with retrieved parent values.

---


* **Not implementing a `Retrieve()` method for values announced to the KV Scheduler as `OBTAINED` via notifications**

      - You must refresh `OBTAINED` values, even if resync does not touch them.
<br>
</br>
      - You may have NB-defined values that depend on `OBTAINED` values. The KV Scheduler needs to know their state to determine if the dependencies are satisfied, and then plan the resync.

---

* **Sleeping/blocking inside descriptor methods**

     - Transaction processing is synchronous. Sleeping inside a CRUD method delays remaining transaction operations, and queued transactions.
<br>
</br>
     - If you have a CRUD operation that waits for "something", then you should define that "something" as a separate key-value pair, and add it to the dependencies list.
<br>
</br> 
     - When it becomes available, send notifications using `KVScheduler.PushSBNotification()`. The KV Scheduler will automatically apply pending operations ready for execution.
<br>
</br>
 
     - You should not hide dependencies inside CRUD operations. 

---

* **Exposing metadata map with write access**

     - KV Scheduler owns and maintains metadata maps.
<br>
</br> 
     - Descriptors do not create custom metadata maps.
<br>
</br>
     - Descriptors convey custom metadata maps to the KV Scheduler as factories `KVDescriptor.MetadataMapFactory`.
 <br>
 </br>
      - Maps retrieved from the KV Scheduler using `KVScheduler.GetMetadataMap()` should remain read-only, and exposed to other plugins.

---

## Debugging

This section covers how-to and other techniques for debugging KV Scheduler problems.

---

### How to set up logging

The [log manager plugin](../plugins/infra-plugins.md#log-manager) allows you to view and modify agent loggers and log levels. You can change the agent's per-logger and global log levels using a conf file, environment variable, agentctl, or REST.

The log level choices consist of one of the following: 

- `debug`
- `info`
- `warning`
- `error`
- `fatal`
- `panic`


---
 
**Using the conf file or env variable**

* To set a default for all loggers, per-logger levels, and external link hooks, use the [logs.conf][logs-conf-file] configuration file.
</br>
</br>
* To override the entries in the `logs.conf` file, use the `INITIAL_LOGLVL=<level>` environment variable.

---

**Using agentctl log**

* To manage individual log levels, use [agentctl log](../user-guide/agentctl.md#log).

List individual loggers and their log levels:
```
agentctl log list
```

Example of setting the KV Scheduler log level to `debug`:
```
agentctl log set kvscheduler debug
```

---

**Using a REST API**

* To set an individual log level, use the log-level REST API.

```
curl -X POST http://localhost:9191/log/<logger-name>/<log-level>
```

The KV Scheduler prints the [transaction log][understand-transaction-logs] to stdout. The logs provide information and visibility to debug and resolve KV Scheduler issues.

KV Scheduler internal debug messages, which require some knowledge of the underlying implementation, are logged to the `kvscheduler` logger.

For more details managing agent log levels, see the [log manager plugin documentation][logmanager-readme].

---

### How to debug agent plugin lookup

You can determine if [plugin lookup](plugin-lookup.md) can find and initialize your plugin by enabling verbose logs:

* Before you start the agent, set the `DEBUG_INFRA` environment variable as follows:
``` 
  export DEBUG_INFRA=lookup
```
* Search for `FOUND PLUGIN: <your-plugin-name>` in the logs. 

If you do not find a log entry for your plugin, check the following:

* You did not include the plugin in your agent's dependencies list.
  
* Your plugin does not implement the [plugin interface][plugin-interface].

---

### How to list registered descriptors and watched key prefixes

You can list registered descriptors, and NB-watched key prefixes using [GET /scheduler/dump](../api/api-kvs.md#dump):

```json
curl -X GET http://localhost:9191/scheduler/dump
```
Sample output:
```
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

The KV Scheduler will not receive the value if you do not watch the value's key prefix, or register the associated descriptor.

---

### Understanding the KV Scheduler transaction log

The KV Scheduler exposes a summary of every executed transaction. You can retrieve the transaction log using one of the following:

* Print to `stdout`.

* [agentctl config history](../user-guide/agentctl.md#config-history)

* [agentctl report](../user-guide/agentctl.md#report), and the `agent-transaction-history.txt` subreport.

!!! Note
    [GET /scheduler/txn-history](../api/api-kvs.md#transaction-history) returns all transaction information in a detailed JSON format. You may find it difficult to read and parse.

The transaction log output describes the following:

- Transaction type
- Assigned sequence number
- Values to be changed
- Transaction plan prepared by the scheduling algorithm
- Actual sequence of executed operations, which may differ from the plan if errors occur.

---

#### Examples

!!! Note
    For easier viewing of the examples below, open the svg figure in your browser.  

---

**Transaction log output with explanations:**

![NB transaction][txn-update-img]

<br>
</br>

---


**Resync transaction that failed to apply one value:**

![Full Resync with error][resync-with-error-img]

<br>
</br>

---

**Retry transaction automatically triggered for the failed operation**. This example uses the resync transaction shown above:

![Retry of failed operations][retry-txn-img]


---

In addition, before a [full or downstream Resync][resync], or after a transaction error, the KV Scheduler dumps the graph state *after* a refresh:

![Graph dump][graph-dump-img]

---

### CRUD verification mode

The KV Scheduler supports a verification mode to check the correctness of CRUD operations provided by descriptors. If you enable this function, the KV Scheduler triggers verification inside the post-processing stage of every transaction. 

Verification mode consists of the following:

* Values changed by transaction CRUD create, update, or delete operations are re-read using the descriptor's `Retrieve()` methods.
<br>
</br> 
* Compares changed values to the intended values to verify they have been correctly applied. 

A failed check can indicate an external entity modified the affected values, or you have not correctly implemented the CRUD operations for the corresponding descriptor or descriptors. 

!!! Note
    Since verification mode re-reads the SB values after changes have been applied, it's _not likely_ an external entity modified these values.
    
You add overhead when performing verification mode. `Retrieve()` operations run after every transaction for descriptors with changed values. Because of this overhead, verification mode is disabled by default, and not recommended for use in production environments.

However, for your development and testing purposes, you will find this feature handy. You can quickly discover CRUD implementation bugs. You should run verification mode for new descriptors prior to production release, or with regression test suites.

Values with read-write inconsistencies appear in the transaction log with the [verification error information][verification-error] attached.

You can enable verification mode with the `KVSCHED_VERIFY_MODE=1` environment variable before starting the agent:
```
export KVSCHED_VERIFY_MODE=1
```

---

### How to visualize the graph

You can display a graph-based representation of the KV Scheduler's internal system state. You just need a modern web browser that supports SVG images, and the graphviz `dot` renderer that ships with the Graphviz package. 

For information on installing and customizing the graphviz `dot` renderer, see the [Graphviz - Graph Visualization Software](http://graphviz.org) website. You must install and run the graphviz `dot` renderer on the host running the agent. 

You retrieve the KV Scheduler internal system for input to the Graphviz tool by calling [GET /scheduler/graph](../api/api-kvs.md#graph):
```
curl -X GET http://localhost:9191/scheduler/graph
```

An example of a rendered graph is shown below. Graph vertices, drawn as rectangles, represent key-value pairs. Derived values have rounded corners. Different fill-colors represent different value states. If you hover over a graph node, a tooltip pops up, describing the state and content of the corresponding value.

The edges depict the relationships between values:

 * Black arrows point to the value dependencies.
 <br>
 </br>
 * Gold arrows connect derived values with their parent values, with cursors pointing to the parents in the backwards direction.

![graph example][graph-visualization-img]

If you call GET /scheduler/graph without query parameters, it returns a rendering of the graph in its current state. 

Alternatively, if you include `txn=<seq-num>` query parameter, you can display the graph state after the given transaction is finished. A yellow border illustrates the vertices updated by the transaction.  

GET /scheduler/graph API to display graph state after the first transaction: 
```
http://<host>:9191/scheduler/graph?txn=0
```

This tool provides you with an instantaneous global view of the system state. You can quickly pinpoint and debug potential problems. This will save you time and effort. 

---

### Understanding the graph walk (advanced)

You can observe how the KV Scheduler walks through the graph to process transactions using the log graph walk function. The KV Scheduler visits a graph node in one of the following transaction processing stages:

1. Graph refresh
2. Transaction simulation
3. Transaction execution

You can enable this function with the `KVSCHED_LOG_GRAPH_WALK` environment variable before starting the agent:
```
export `KVSCHED_LOG_GRAPH_WALK`
```
The log graph walk function generates verbose logs showing how the KV Scheduler visits the graph nodes. 

---

### Graph refresh

During [graph refresh][graph-refresh-code], some or all of the registered descriptors `Retrieve()` the current SB values. The `refreshValue()` method refreshes the node state corresponding to the retrieved values. This method propagates the call to the `refreshAvailNode()` method for the node itself, and for every value derived from the node. The `refreshAvailNode()`method updates the value state and its content to reflect the retrieved data.

Graph refresh uses `refreshUnavailNode()` method to visit nodes with obsolete derived values, and mark these values as unavailable in the SB. The same method marks values no longer present in the SB as unavailable.  
 
!!! Note
    Obsolete derived values were previously derived, but no longer relevant because a new value has been retrieved.    

Diagram the graph refresh control flow:

![Graph refresh diagram][graph-refresh]


---

Example graph refresh verbose log printed by the KV Scheduler to the transaction log:
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

Transaction simulation and execution follow the same algorithm. During simulation, the CRUD operations provided by descriptors are _not executed_. Instead, the scheduling algorithm simulates the calls with a nil error value returned. 

In addition, the algorithm discards graph refresh updates. If a transaction executes without any errors, the algorithm uses the same path through the graph for simulation and execution. 

Transaction processing uses the `applyValue()` method for every value to modify during the transaction. This method determines which of the `applyCreate()` / `applyUpdate()` / `applyDelete()` methods to execute, based on the current, and the new value data to apply.

Recursion handles the situation when value updates require updates of some related values. For example, `applyCreate()` uses the `applyDerived()` method to call `applyValue()` for every derived value to create. 

Once the KV Scheduler creates the value, `applyCreate()` calls `runDepUpdates()` to recursively call `applyValue()`, for values dependent on the created value. The KV Scheduler marked the dependent values as `PENDING` in the previous transaction. Now, having recursed through the graph to satisfy dependencies, the KV Scheduler creates the dependent values.    

Similarly, `applyUpdate()` and `applyDelete()` could tell the KV Scheduler to recursively continue and walk through the edges of the graph to update related values.

---

Diagram showing the transaction processing control flow:
 
![KVScheduler diagram][graph-wal-img]


---

Example transaction processing verbose log printed by the KV Scheduler to the transaction log:
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
[errors-go-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/kvscheduler/api/errors.go
[graph-dump-img]: ../img/developer-guide/graph-dump.png
[graph-refresh]: ../img/developer-guide/graph-refresh.svg
[graph-refresh-code]: https://github.com/ligato/vpp-agent/blob/master/plugins/kvscheduler/refresh.go
[graph-visualization-img]: ../img/developer-guide/graph-visualization.svg
[graph-walk]: kvs-troubleshooting.md#understanding-the-graph-walk-advanced
[graph-wal-img]: ../img/developer-guide/graph-walk.svg
[how-to-descriptors]: kvs-troubleshooting.md#how-to-list-registered-descriptors-and-watched-key-prefixes
[how-to-graph]: kvs-troubleshooting.md#how-to-visualize-the-graph
[logs-conf-file]: https://github.com/ligato/cn-infra/blob/master/logging/logmanager/logs.conf
[logmanager-readme]: ../plugins/infra-plugins.md#log-manager
[plugin-interface]: https://github.com/ligato/cn-infra/blob/425b8dd352626b88fb36713d7589ac9fc678bdb7/infra/infra.go#L8-L16
[refresh-go-file]: https://github.com/ligato/vpp-agent/blob/master/plugins/kvscheduler/refresh.go
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
