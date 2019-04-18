The `govppmux` is a core Agent plugin which **allows other plugins to access the VPP** independently at each other by means of connection multiplexing.

Any plugin (core or external) that interacts with the VPP can ask `govppmux` to get its own, potentially customized communication channel to access running VPP instance. Behind the scene, **all channels share the same connection** created during the plugin initialization using `govpp` core.

### Connection

Tho GoVPP multiplexer supports two connection types:
  - using socket client
  - using shared memory
    
By default the GoVPP connects uses the **socket client**. This option can be changed using environment variable `GOVPPMUX_NOSOCK` which fallback to the **shared memory** - in that case, the plugin connects to that instance of the VPP which uses the default shared memory segment prefix.
 
The **default behaviour assumes that there is only a single VPP running** in a sand-boxed environment together with the agent (e.g. through containerization). In case the VPP runs with customized SHM prefix, or **there are several VPP instances** running side-by-side, the **GoVPP needs to know the prefix in order to connect to the desired VPP 
instance**. The prefix has to be put into the GoVPPMux configuration file (called `govpp.conf`) with the key `shm-prefix` and value matching the VPP shared memory prefix name.

### Multiplexing

The `NewAPIChannel` call returns a new API channel for communication with VPP via the `govpp` core. It uses default buffer sizes for the request and reply go channels (by default both are 100 messages long).

If it is expected that the VPP may get overloaded at peak loads (for example if the user plugin sends configuration requests in bulks) then it is recommended to use `NewAPIChannelBuffered` and increase the buffer size for requests appropriately. Similarly, `NewAPIChannelBuffered` allows to configure the size of the buffer for responses. This is also useful since the buffer for responses is also used to carry VPP notifications and statistics which may temporarily rapidly grow in size and frequency. By increasing the reply channel size, the probability of dropping messages from VPP decreases at the cost of increased memory footprint.

### Trace
 
Duration of the VPP binary api call can be measured using trace feature. These data are logged after every event (any resynchronization, new interfaces, bridge domains, fib entries etc.). Enable trace in the `govpp.conf`: 
 
`trace-enabled: true` or  `trace-enabled: false`
  
The trace feature is disabled by default (if there is no config available). 

### Configuration

The plugin allows to configure parameters of the VPP health-check probe. Possible items to configure:

  - `health check probe interval`: time between health check probes
  - `health check reply timeout`: if the reply does not arrive until timeout elapses probe is considered failed
  - `health check threshold`: number of consequent failed health checks until an error is reported
  - `reply timeout`: if the reply from channel does not arrive until timeout elapses, the request fails
  - `shm-prefix`: used for connection to a VPP instance which is not using default shared memory prefix
  - `resync-after-reconnect`: allows to run resync after the VPP reconnection