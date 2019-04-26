# Tutorial: GRPC

The tutorial shows how to use CN-Infra/VPP-Agent plugins in order to implement GRPC CRUD client or notification watcher. The main advantage of this solution is that the implementation is easy and short (in terms of LoC). 

Requirements:

* Complete and understand the ['Hello World Agent'](../tutorials/01_hello-world) tutorial
* Complete and understand the ['Plugin Dependencies'](../tutorials/02_plugin-deps) tutorial
* Installed VPP, or VPP binary file of the supported version

In the first part, we create a generic GRPC plugin providing a client object which allows to send the configuration to the VPP-Agent and also can act as notification watcher which can listen on the VPP-Agent notifications. Next part shows how the plugin works with some example data for both scenarios. 

The tutorial uses following folder structure:

```
src/grpc
│
└───cmd
│   │
│   └───client
│          main.go     
│        
└───plugins
    │
    └───grpc
           grpc_client.go
           options.go
```   

The `grpc_client.go` will contain GRPC plugin definition, GRPC connection and client initialization. In the tutorial we use this file also to generate test data. The `options.go` file will be the plugin adapter (the application uses it to set up the plugin) based on the VPP-Agent convention. The `main.go` will contain all code necessary to set up the plugin.            

### The GRPC plugin

In this section is described the procedure of creating the generic GRPC plugin. 

**1. Define the GRPC client plugin skeleton** in `/grpc/plugins/grpc/grpc_client.go`, consisting of main plugin structure, external dependencies and three functions `Init()`, `Close()` and `String()` required in order to implement the [plugin interface](https://github.com/ligato/cn-infra/blob/master/infra/infra.go).
```go
package grpc

// The GRPC client plugin structure
type GRPCClient struct {
	Deps // external dependencies	
}

// The structure for the plugin external dependencies  
type Deps struct {

}

// The initialization function, called when the agent in started
func (p *GRPCClient) Init() (err error) {
	return nil
}

// THe close function, called on the shutdown  
func (p *GRPCClient) Close() (err error) {
	return nil
}

// The GRPC plugin string representation
func (p *GRPCClient) String() string {
	return "GRPC-client"
}
```

2. **Provide required plugin external dependencies and local fields** for plugin infrastructure and for the GRPC. The plugin infrastructure defines `PluginName`, enables CN-Infra-defined logging and adds support for plugin config file (if needed). The plugin should define the GRPC connection and client, later used for communication with the server. The connection is kept in order to close it properly later.
```go
import (
	"github.com/ligato/cn-infra/infra"
	"github.com/ligato/vpp-agent/api/configurator"
	"google.golang.org/grpc"
)

...
type GRPCClient struct {
	Deps
	
	connection *grpc.ClientConn
    client     *configurator.ConfiguratorClient
}


type Deps struct {
    infra.PluginDeps
}

...
``` 

3. **Setup the server connection to the GRPC.** The connection requires several parameters which can be provided from the outside (via flags or the config file). For simplicity, let's just use hard-code values.
```go
import (
	"net"
	"time"
	...
)


...

func (p *GRPCClient) Init() (err error) {
	p.connection, err = grpc.Dial("unix",
		// Or any other transport security
		grpc.WithInsecure(),
	    // Hardcoded TCP socket type, IP address and dial timeout (2s)
        grpc.WithDialer(net.DialTimeout("tcp", "0.0.0.0:9111", time.Second*2)),
    )
    if err != nil {
        return err
    }
	
	p.Log.Info("GRPC client is connected")
	
	return nil
}

...

// Dialer function used as a parameter for 'grpc.WithDialer'
func dialer(socket, address string, timeoutVal time.Duration) func(string, time.Duration) (net.Conn, error) {
	return func(addr string, timeout time.Duration) (net.Conn, error) {
		addr, timeout = address, timeoutVal
		return net.DialTimeout(socket, addr, timeoutVal)
	}
}
``` 

4. **Prepare the GRPC client** able to send northbound configuration to the GRPC server (VPP-Agent) or watch notifications from it. The client object implements the configurator client interface generated from the [configurator proto file](https://github.com/ligato/vpp-agent/blob/master/api/configurator/configurator.proto).
```go
func (p *GRPCClient) Init() (err error) {
	p.connection, err = grpc.Dial("unix",
		grpc.WithDialer(dialer("tcp", "0.0.0.0:9111", time.Second*2)),
	)
	if err != nil {
		return err
	}

    p.Log.Info("GRPC client is connected")

    // GRPC client object 
	p.client = configurator.NewConfiguratorClient(p.connection)

	return nil
}
```

5. **Close the connection** on the plugin close. Here is like the complete GRPC plugin code looks like:
```go
package grpc

import (
	"github.com/ligato/cn-infra/infra"
	"github.com/ligato/vpp-agent/api/configurator"
	"google.golang.org/grpc"
	"net"
	"time"
)

// The GRPC client plugin structure
type GRPCClient struct {
	connection *grpc.ClientConn
	client     configurator.ConfiguratorClient

	Deps // external dependencies
}

// Structure for plugin external dependencies
type Deps struct {
	infra.PluginDeps
}

// Initialization function, called when the agent in started
func (p *GRPCClient) Init() (err error) {
	p.connection, err = grpc.Dial("unix",
		grpc.WithDialer(dialer("tcp", "0.0.0.0:9111", time.Second*2)),
	)
	if err != nil {
		return err
	}

    p.Log.Info("GRPC client is connected")

	p.client = configurator.NewConfiguratorClient(p.connection)
    
	return nil
}

// Close function, called on shutdown
func (p *GRPCClient) Close() (err error) {
	return p.connection.Close()
}

// Plugin string representation
func (p *GRPCClient) String() string {
	return "GRPC-client"
}

func dialer(socket, address string, timeoutVal time.Duration) func(string, time.Duration) (net.Conn, error) {
	return func(addr string, timeout time.Duration) (net.Conn, error) {
		addr, timeout = address, timeoutVal
		return net.DialTimeout(socket, addr, timeoutVal)
	}
}
``` 

As you can see, the code only depends on CN-Infra infrastructure (plugin definition) and external GRPC dependency. Later it will consume some VPP-Agent API proto-definitions so it will be dependent at actual VPP-Agent code.

The plugin is ready to use, with the GRPC client ready. However, is still needs to be [wired as the application](#wiring). See also the [testing](#testing) section in order to learn how to use client to manage the configuration or notifications.

### Application wiring

In this part we use CN-Infra to create a running application from our GRPC plugin and connect it to the server.

**1. Prepare the plugin adapter for the application** in the `/grpc/plugins/options.go` file. It allows to load the plugin in default setup or customize its dependencies:
```go
package grpc

// Default instance of the plugin (e.g. without custom dependencies)
var DefaultPlugin = *NewPlugin()

// Method to retrieve plugin instance with options (if defined)
func NewPlugin(opts ...Option) *GRPCClient {
	p := &GRPCClient{}

    // Unique plugin identification
	p.PluginName = "GRPC-client"

    // All custom options are set at this point
	for _, o := range opts {
		o(p)
	}
	
    // Initialize plugin infrastructure dependencies (logger)
	p.PluginDeps.Setup()

	return p
}

type Option func(*GRPCClient)
```

**2. Define the App as a top-level plugin** - prepare the structure in the `/grpc/cmd/client/main.go` with all methods required to implement the plugin interface (as before). For simplicity, the structure defines only our GRPC plugin.
```go
package main

import (
	"grpc/plugins/grpc"
)

type App struct {
	GRPC *grpc.GRPCClient
	
	// Other plugins
}

func New() *App {
	
	// Plugin dependency resolution
	
	return &App{
		GRPC:     &grpc.DefaultPlugin,
	}
}

func (App) Init() error {
	return nil
}

func (App) Close() error {
	return nil
}

func (App) String() string {
	return "App-grpc-client"
}
```

Basically it means we have two plugins now - the GRPC plugin created earlier, and the application plugin using it as direct dependency. In the VPP-Agent, the convention is to always create a top-level plugin which uses other plugin as external dependencies (those plugins can be anyhing, like connectors, VPP-configuration plugins, KVDB, etc.). This approach allows to use only the top-level plugin in the application instantiation and CN-Infra build-in plugin lookup mechanism handles the rest (technically we do not need to laboriously list all plugins we want to use).

**3. Finish the App** adding code below to the `main.go`. The `New()` method creates an instance of App plugin. The `AllPlugins()` automatically finds our GRPC plugins as an App dependency and loads it. For more information about the plugin lifecycle management, see [this article](https://github.com/ligato/cn-infra/wiki/Agent-Plugin-Lookup).
```go
package main

import (
	"github.com/ligato/cn-infra/agent"
	"tutorials/grpc/plugins/grpc"
)

func main() {
	// Prepare the App instance
	a := agent.NewAgent(
		agent.AllPlugins(New()),
	)

    // Start the App
	if err := a.Run(); err != nil {
		panic(err)
	}
}

type App struct {
	GRPC *grpc.GRPCClient
}

func New() *App {
	return &App{
		GRPC:     &grpc.DefaultPlugin,
	}
}

func (App) Init() error {
	return nil
}

func (App) Close() error {
	return nil
}

func (App) String() string {
	return "App-grpc-client"
}
```

**4. Start the App.** Now our client agent can be built to binary file and be started with `./client`. 

The client is designated to communicate with the VPP-Agent acting as a server. Start the VPP-Agent with the `-grpc-config` flag and set the endpoint and the network in the GRPC config file to the same value as in our GRPC client application (currently hardcoded values), so they can communicate with each other. 

# Plugin usage

The GRPC client application can connect to the Ligato-based GRPC server and offers a client object to communicate. In this part we show how to use the client to put GRPC configuration and how to watch on notifications.

### Config publisher

For the tutorial purpose, we simulate configuration provisioning from the client plugin `Init()`. During the init phase, a new go routine starts and puts example configuration to the GRPC server. Add the following code to the `/grpc/plugins/grpc/grpc_client.go`:

1. example setup in `configure()` method

```go
func (p *GRPCClient) configure() {
	time.Sleep(2*time.Second)
	_, err := p.client.Update(context.Background(), &configurator.UpdateRequest{
		Update: &configurator.Config{
			VppConfig: &vpp.ConfigData{
				Interfaces: []*vpp.Interface{
					{
						Name: "interface1",
						Type: vpp_interfaces.Interface_SOFTWARE_LOOPBACK,
						Enabled: true,
						IpAddresses: []string{"10.0.0.1/24"},
					},
				},
			},
		},
		FullResync: true,
	})
	if err != nil {
		p.Log.Errorf("Error putting GRPC data: %v", err)
		return
	}
	p.Log.Infof("GRPC data sent")
}
```

2. Update the `Init()`, run `configure` in the go routine when the plugin is initialized. Make sure the example configuration is sent AFTER the client object is initialized.
```go
func (p *GRPCClient) Init() (err error) {
	p.connection, err = grpc.Dial("unix",
		grpc.WithInsecure(),
		grpc.WithDialer(dialer("tcp", "0.0.0.0:9111", time.Second*2)),
	)
	if err != nil {
		return err
	}

	p.client = configurator.NewConfiguratorClient(p.connection)

	p.Log.Info("GRPC client is connected")

    // Start GRPC configuration
	go p.configure()

	return nil
}
```

After running the `./client`, the example interface config is passed via the GRPC to the VPP-Agent and configured on the VPP the server is connected to.

### Notification watcher

The GRPC client can be also used to receive automatic notifications from the VPP-Agent server. Notifications currently work only for interfaces. Following steps explain how to prepare notification watcher. 

1. Add a `watchNotif()` method to the `grpc_client.go` code. Note that the notifications are polled periodically. Notifications are indexed on the server side (up to 100 notifications to the history), so the request is provided with the next index which is incremented after every iteration. This allows to read previous notifications when needed.
```go
func (p *GRPCClient) watchNotif() {
	p.Log.Info("Notification watcher started")
	var nextIdx uint32
	for {
		request := &configurator.NotificationRequest{
			Idx: nextIdx,
		}
		stream, err := p.client.Notify(context.Background(), request)
		if err != nil {
			p.Log.Error(err)
			return
		}
		var recvNotifs int
		for {
			notif, err := stream.Recv()
			if err == io.EOF {
				if recvNotifs == 0 {
					// Nothing to do
				} else {
					p.Log.Infof("%d new notifications received", recvNotifs)
				}
				break
			}
			if err != nil {
				p.Log.Error(err)
				return
			}

			p.Log.Infof("Notification[%d]: %v", notif.NextIdx-1, notif.Notification)
			nextIdx = notif.NextIdx
			recvNotifs++
		}
		
		time.Sleep(time.Second*1)
	}
}
```

2. Start the watcher in new go routine inside `Init()`:
```go
// Initialization function, called when the agent in started
func (p *GRPCClient) Init() (err error) {
	p.connection, err = grpc.Dial("unix",
		grpc.WithInsecure(),
		grpc.WithDialer(dialer("tcp", "0.0.0.0:9111", time.Second*2)),
	)
	if err != nil {
		return err
	}

	p.client = configurator.NewConfiguratorClient(p.connection)

	p.Log.Info("GRPC client is connected")
	// Start notification watcher
	go p.watchNotif()
	go p.configure()

	return nil
}
```
!!! danger "Important"
    This approach is only for testing purposes, because it creates a race condition, since we cannot tell whether the `watchNotif` or the `configure` will run first. But in this scenario, it is not harmful since notifications are read from the server cache (theoretically it could be a problem if the client would configure >100 interfaces before the watcher starts, so some notifications would be lost but that is not a case here).

After starting the client, the GRPC plugin sends the interface configuration (as before) which is received in the VPP-Agent server and configured to the VPP. After that, the server sends notification back to the client. For illustration, here is the output log from the client application (it's somehow self-explaining):
```bash
INFO[0000] Starting agent version: v0.0.0-dev            BuildDate= CommitHash= loc="agent/agent.go(129)" logger=agent
INFO[0000] Starting agent with 2 plugins                 loc="agent/agent.go(199)" logger=agent
INFO[0000] GRPC client is connected                      loc="grpc/grpc_client.go(40)" logger=GRPC-client
INFO[0000] Agent started with 2 plugins (took 0s)        loc="agent/agent.go(168)" logger=agent
INFO[0000] Notification watcher started                  loc="grpc/grpc_client.go(89)" logger=GRPC-client
INFO[0004] GRPC data sent                                loc="grpc/grpc_client.go(85)" logger=GRPC-client
INFO[0006] Notification[0]: vpp_notification:<interface:<type:UPDOWN state:<name:"interface1" internal_name:"loop0" if_index:1 admin_status:UP oper_status:UP last_change:1550673364 phys_address:"de:ad:00:00:00:00" mtu:9216 statistics:<> > > >   loc="grpc/grpc_client.go(116)" logger=GRPC-client
INFO[0006] 1 new notifications received                  loc="grpc/grpc_client.go(107)" logger=GRPC-client
```
