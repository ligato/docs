# GRPC Handler

---

Link to code: [GRPC][code-link]

The tutorial explains how to use Ligato plugins to implement a GRPC CRUD client and notification watcher.

Requirements:

* Complete the ['Hello World'](../tutorials/01_hello-world.md) tutorial
* Complete the ['Plugin Dependencies'](../tutorials/02_plugin-deps.md) tutorial
* Installed VPP, or VPP binary file of the supported version

In the first part, we will create a generic GRPC plugin allowing a client object to send configuration data to the vpp-agent. In addition, it can act as a notification watcher which can listen to vpp-agent notifications.

The tutorial uses this folder structure:

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

The `grpc_client.go` contains a GRPC plugin definition, GRPC connection and a client initialization function, and will be used to generate test data. The `options.go` file defines an adapter used by the application. The `main.go` contains all the necessary code to set up the plugin.

### The GRPC plugin

We will start by creating the generic GRPC plugin.

**1. Define the GRPC client plugin skeleton** in `/grpc/plugins/grpc/grpc_client.go`. This consists of the main plugin structure, external dependencies and three functions  required to implement the [plugin interface:](https://github.com/ligato/cn-infra/blob/master/infra/infra.go) `Init()`, `Close()` and `String()`.
```go
package grpc

// The GRPC client plugin structure
type Client struct {
	Deps // external dependencies	
}

// The structure for the plugin external dependencies  
type Deps struct {

}

// The initialization function, called when the agent in started
func (p *Client) Init() (err error) {
	return nil
}

// THe close function, called on the shutdown  
func (p *Client) Close() (err error) {
	return nil
}

// The GRPC plugin string representation
func (p *Client) String() string {
	return "GRPC-client"
}
```

**2. Provide the required plugin external dependencies and local fields** for the plugin infrastructure and GRPC. The plugin infrastructure defines `PluginName`, enables cn-infra-defined logging, and adds support for a plugin config file if needed.

The plugin should define the GRPC connection and client that will be used later for communication with the server.
```go
import (
	"github.com/ligato/cn-infra/infra"
	"github.com/ligato/vpp-agent/proto/ligato/configurator"
	"google.golang.org/grpc"
)

...
type Client struct {
	Deps
	
	connection *grpc.ClientConn
    client     configurator.ConfiguratorClient
}


type Deps struct {
    infra.PluginDeps
}

...
``` 

**3. Setup the connection to the GRPC server.** The connection requires several parameters which can be provided via flags or the [config file](../user-guide/config-files.md#grpc).

For simplicity, let's use hard-coded values.
```go
import (
	"net"
	"time"
	...
)


...

func (p *Client) Init() (err error) {
	p.connection, err = grpc.Dial("unix",
		// Or any other transport security
		grpc.WithInsecure(),
	    // Hardcoded TCP socket type, IP address and dial timeout (2s)
        grpc.WithDialer(dialer("tcp", "0.0.0.0:9111", time.Second*2)),
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

**4. Prepare the GRPC client** to send northbound configuration data to the vpp-agent GRPC server, or watch the GRPC server for generated notifications. The client object implements the configurator client interface generated from the [configurator proto file](https://github.com/ligato/vpp-agent/blob/master/proto/ligato/configurator/configurator.proto).
```go
func (p *Client) Init() (err error) {
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

**5. Close the plugin connection**. Here is the code:
```go
package grpc

import (
	"github.com/ligato/cn-infra/infra"
    "github.com/ligato/vpp-agent/proto/ligato/configurator"
    "google.golang.org/grpc"
    "net"
    "time"
)

// The GRPC client plugin structure
type Client struct {
	connection *grpc.ClientConn
	client     configurator.ConfiguratorClient

	Deps // external dependencies
}

// Structure for plugin external dependencies
type Deps struct {
	infra.PluginDeps
}

// Initialization function, called when the agent in started
func (p *Client) Init() (err error) {
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
func (p *Client) Close() (err error) {
	return p.connection.Close()
}

// Plugin string representation
func (p *Client) String() string {
	return "GRPC-client"
}

func dialer(socket, address string, timeoutVal time.Duration) func(string, time.Duration) (net.Conn, error) {
	return func(addr string, timeout time.Duration) (net.Conn, error) {
		addr, timeout = address, timeoutVal
		return net.DialTimeout(socket, addr, timeoutVal)
	}
}
``` 

As you can see, the code depends on cn-infra infrastructure (plugin definition) and an external GRPC dependency. Later it will consume vpp-agent API proto-definitions thus making it dependent on actual vpp-agent code.

The GRPC plugin is ready to use; however, it still needs to be [wired to the application](#wiring).

### Application wiring

We will use cn-infra to create a running application from our GRPC plugin, and connect it to the server.

**1. Prepare the plugin adapter for the application** in the `/grpc/plugins/options.go` file. This allows us to load the plugin with a default setup or customize its dependencies:
```go
package grpc

// Default instance of the plugin (e.g. without custom dependencies)
var DefaultPlugin = *NewPlugin()

// Method to retrieve plugin instance with options (if defined)
func NewPlugin(opts ...Option) *Client {
	p := &Client{}

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

type Option func(*Client)
```

**2. Define the App as a top-level plugin** by preparing the structure in the `/grpc/cmd/client/main.go` file with all methods required to implement the plugin interface. For simplicity, the structure defines only our GRPC plugin.
```go
package main

type App struct {
	GRPC *grpc.Client
	
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

We now have two plugins: the GRPC plugin and the top-level application plugin. The GRPC plugin is used by the top-level plugin as a direct dependency. By vpp-agent convention, a top-level plugin should be created using other plugins as external dependencies. Those plugins can be anything: connectors, VPP-configuration plugins, KV data stores, etc. This approach allows us to use just the top-level plugin in the application with cn-infra plugin lookup mechanisms handling the rest.

**3. Finish the App** by adding this code to the `main.go` file. The `New()` method creates an instance of the App (top-level) plugin. The `AllPlugins()` automatically locates and loads our GRPC plugins as an App dependency.

For more information about the plugin lifecycle management, see the [plugin lookup discussion in the developer guide](../developer-guide/plugin-lookup.md).
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
	GRPC *grpc.Client
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

**4. Start the App** with `./client` so our client agent is built into a binary file and initiated.

The client is designed to communicate with the vpp-agent acting as a server. Start the vpp-agent with the `-grpc-config` flag and set the endpoint and the network in the GRPC config file to the same values (hard-coded) used in our GRPC client application. This is so they can communicate with each other.

### Plugin usage

The GRPC client application can connect to the GRPC server. We will now show you how to use the client to put the GRPC configuration, and watch for notifications.

### Config publisher

For tutorial purposes, we simulate configuration provisioning from the client plugin `Init()`. During the init phase, a new go routine starts and puts example configuration to the GRPC server.

Add the following code to the `/grpc/plugins/grpc/grpc_client.go`:

**1. Setup in `configure()` method**

```go
func (p *Client) configure() {
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

**2. Update the `Init()`, and run `configure` in the go routine** when the plugin is initialized. Make sure the example configuration is sent *AFTER* the client object is initialized.
```go
func (p *Client) Init() (err error) {
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

After running `./client`, the example interface config is passed via the GRPC server to the vpp-agent which in turn programs the VPP data plane.

### Notification watcher

The GRPC client can be used to receive automatic notifications from the vpp-agent GRPC server.

Here are the steps to prepare a notification watcher.

**1. Add a `watchNotif()` method to the `grpc_client.go` code.** Note that the notifications are polled periodically. Notifications are indexed on the server side (up to 100 notifications to the history), so the request is provided with the next index which is incremented after every iteration. This allows one to inspect previous notifications if needed.
```go
func (p *Client) watchNotif() {
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

**2. Start the watcher** in a new go routine inside `Init()`:
```go
// Initialization function, called when the agent in started
func (p *Client) Init() (err error) {
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
    This approach is for testing purposes only. In practice, a race condition could occur since we cannot tell which will run first: `watchNotif` or `configure`. This is not harmful here because notifications are read from the server cache. In theory, it could be a problem if the client configures >100 interfaces before the watcher starts. The result could be loss of some notifications.

After starting the client, the GRPC plugin sends the interface configuration which is received in the vpp-agent server and programmed into the VPP. The server then sends a notification back to the client.

For illustrative purposes, here is the output log from the client application:
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

[code-link]: https://github.com/ligato/vpp-agent/tree/master/examples/tutorials/08_grpc
