# GRPC Handler

---

Tutorial code: [gRPC][code-link]

In this tutorial, you will learn how to implement a gRPC client and notification watcher using plugins. Before running through this tutorial, you should complete the [Hello World tutorial](01_hello-world.md) and the [Plugin Dependencies tutorial](02_plugin-deps.md). 

You should also be familiar with gRPC. To learn about gRPC using Go, check out the [gRPC Go Quick start](https://grpc.io/docs/languages/go/quickstart/).   

---

For the first part of this tutorial, you will create a generic GRPC plugin that allows a gRPC client to send configuration data to the VPP agent. In addition, the gRPC plugin can act as a notification watcher that can listen to VPP agent notifications.

This lays out the folder structure in this tutorial for your gRPC plugin:

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

The `grpc_client.go` file contains a gRPC plugin definition, gRPC connection, and a client initialization function. The gRPC plugin can also generate test data. 

The `options.go` file defines an adapter that is used by the application. The `main.go` file contains all necessary code to set up the plugin.

---

### gRPC plugin

To create the generic gRPC plugin, you'll need to start with the gRPC client plugin skeleton defined in the `/grpc/plugins/grpc/grpc_client.go` file. 

This file contains the following:

- Main plugin structure and external dependencies.
- Three methods required to implement the plugin interface: `Init()`, `Close()` and `String()`:
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

---

Next, add the plugin external dependencies, and local fields for the plugin infrastructure and gRPC. 

The plugin infrastructure defines `PluginName`, enables cn-infra-defined logging, and adds support for a plugin conf file if needed. The plugin defines the gRPC connection and client that will be used for communication with the server.
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

---

Set up the connection to the gRPC server. The connection requires several parameters which can be provided by flags, or by a [conf file](../user-guide/config-files.md#grpc).

To keep it simple, use hard-coded values:
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

---


Next, prepare the gRPC client to perform two tasks: 

- Send northbound configuration data to the VPP agent gRPC server.
- Watch the gRPC server for notifications. 

The gRPC client object implements the configurator client interface that is generated from the [configurator proto file](https://github.com/ligato/vpp-agent/blob/master/proto/ligato/configurator/configurator.proto):
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

---

Finally, make sure you close the plugin connection.

```
// Close function, called on shutdown
func (p *Client) Close() (err error) {
	return p.connection.Close()
}
``` 
---

Here is the complete code for the gRPC plugin:
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

---

As you can see, the gRPC code uses the plugin definition, and an external gRPC dependency. A second dependency on the VPP agent needed for consuming VPP agent proto-definitions will be added. 

The gRPC plugin is ready to use, but first you need to wire it up to the application.

---

### Application wiring

In this section, you will create a running application from your GRPC plugin, and connect it to the server.

Start by preparing the plugin adapter for the application in the `/grpc/plugins/options.go` file. This lets you load the plugin with a default setup, or customize its dependencies:
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

---

Define the App as a top-level plugin. Prepare the struct in the `/grpc/cmd/client/main.go` file with all methods required to implement the plugin interface. To keep it simple, the struct defines only your gRPC plugin:
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

---

You now have two plugins: the gRPC plugin and the top-level application plugin. The top-level plugin uses the gRPC plugin as a direct dependency. By VPP agent convention, a top-level plugin should be created using other plugins as external dependencies. 

Those plugins can be anything: connectors, VPP-configuration plugins, KV data stores. With this approach, all you need is the App top-level plugin and plugin lookup mechanisms handles the rest. 

---

Finish the App by adding the following code to the `main.go` file. The `New()` method creates an instance of the App top-level plugin. The `AllPlugins()` automatically locates and loads our gRPC plugins as an App dependency:
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

---

Start the App with `./client`. This process builds the client agent into an executable binary file, and then initializes it.

The gRPC client communicates with the VPP agent acting as a server. To achieve this, first update the gRPC conf file with the same endpoint and network values that you hard coded in your gRPC client application.  Second, start the VPP agent with the `-grpc-config` flag.

The next two sections in this tutorial explain how to use the gRPC client application to perform the following:

- put the configuration
- watch for notifications 

---

### Put configuration

Let's simulate configuration provisioning from the client plugin `Init()` method. During the init phase of this method, a new Go routine starts and puts the example configuration to the GRPC server.

Add the following code to the `/grpc/plugins/grpc/grpc_client.go` file.

Set up the `configure()` method:
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

---

Update the `Init()` method, and run `configure()` in the Go routine. When plugin initialization occurs, make sure you send the example configuration after the initialization of the client object.
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

---

Start the App with `./client`. This command passes the example interface config to the VPP agent via the gRPC server. The VPP agent then puts the example interface configuration to the VPP data plane.

---

### Notification watcher

The gRPC client can receive automatic notifications from the VPP agent gRPC server.

To prepare a notification watcher, perform the following:

Add a `watchNotif()` method to the `grpc_client.go` code. The gRPC client periodically polls the VPP agent gRPC server for notifications. The server side maintains a history of up to 100 indexed notifications. The gRPC client request provides the next index which is incremented after each iteration. This lets you inspect previous notifications as needed. 
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

---

Start the watcher in a new Go routine inside the `Init()` method:
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
    This approach is for testing purposes only. In practice, a race condition could occur since you cannot tell which will run first: `watchNotif()` or `configure()`. Here, this is not harmful because you are reading notifications from the server cache. In theory, this could be a problem if the client configures >100 interfaces before the watcher starts. The result could be loss of some notifications.

---

After starting the client, the gRPC plugin sends the interface configuration to the VPP agent server. The VPP agent programs the interface configuration into the VPP data plane. The server then sends a notification back to the gRPC client.

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
