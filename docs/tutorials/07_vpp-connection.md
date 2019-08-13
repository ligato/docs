# VPP Connection

---

Link to code: [Connect the plugin to the VPP][code-link]

This tutorial will illustrate how to utilize the [GoVPPMux plugin][1] in our hello world plugin to connect to VPP. It will also demonstrate how to perform a synchronous, asynchronous and multi-request call in order to put configuration items.

Requirements:

* Complete the ['Hello World Agent'](../tutorials/01_hello-world.md) tutorial
* Complete the ['Plugin Dependencies'](../tutorials/02_plugin-deps.md) tutorial
* Installed VPP, or VPP binary file of the supported version

The main task of this tutorial is to show how to use GoVPPMux to create a VPP connection (southbound oor SB) and use various procedures to put or read its data. Because the intent here is to show how to work with VPP and to keep the example as simple as possible, we do not use any northbound database such as etcd. 

Our first step is to create a reliable connection with VPP. The vpp-agent uses [GoVPP][2] as a VPP adapter. We add the GoVPPMux plugin (the GoVPP wrapper) to our hello world example:
```go
type HelloWorld struct {
	GoVPPMux govppmux.API
}
```   

And initialize it in the `main` right after the hello world plugin struct definition:
```go
func main() {
	p := new(HelloWorld)
	p.GoVPPMux = &govppmux.DefaultPlugin    // initialize with the default plugin instance
	
...
	
}
``` 

The GoVPPMux API allows other plugins to create their own API channels to communicate with VPP using GoVPP core. The channel can be either buffered or unbuffered and is used to pass requests and replies between VPP and our plugin. 

Let's create one in the `Init`. At first, we need a variable of `Channel` type to have the VPP channel available for all hello world methods:
```go
type HelloWorld struct {
	vppChan api.Channel

	GoVPPMux govppmux.API
}
```

Then initialize new VPP unbuffered channel:
```go
func (p *HelloWorld) Init() (err error) {
	log.Println("Hello World!")

	if p.vppChan, err = p.GoVPPMux.NewAPIChannel(); err != nil {
		panic(err)
	}
	return nil
}
```

And also do not forget to close it at the end:
```go
func (p *HelloWorld) Close() error {
	p.vppChan.Close()
	log.Println("Goodbye World!")
	return nil
}
```

Later, we will use the channel to send or receive data. But first, we need to prepare the data and send it in a way that VPP can understand.

#### 1. Binary API

!!! note
    This step is only explanatory and can be skipped.

GoVPP is a toolset for VPP management. It provides a high-level API for communication with the GoVPP core and for sending and receiving messages to/from the VPP via adapter. This is a component between GoVPP and VPP responsible for sending and receiving binary-encoded data via the VPP socket client (by default, but the VPP shared memory is also available when needed). It also provides a bindings generator for VPP JSON binary API definitions referred to as the **binapi-generator**.
 
Bindings are by default present in the path `/usr/share/vpp/api/` and divided into multiple logical files. Any of the JSON API files can be transformed into the `.go` file using binapi-generator. Generated API calls or messages can be then directly referenced in the go code. The vpp-agent stores generated binary API files in the [binapi directory](https://github.com/ligato/vpp-agent/tree/master/plugins/vpp/binapi) (the tutorial example uses interface bindings from this directory as well)

The binapi-generator uses following format:
```bash
binapi-generator --input-file=/usr/share/vpp/api/<name>.api.json --output-dir=<path>
```

Remember that the `*.api.json` files are present only when the VPP is installed in the system. This step needs to be done only once, and files must be re-generated only when a new VPP (version) with changes in the API is introduced. 

In the tutorial this step is not needed since all required messages are already generated. Only the correct VPP version is mandatory (see [vpp.env](https://github.com/ligato/vpp-agent/blob/master/vpp.env) file for currently supported VPP versions).

Read more about the [GoVPP project](https://wiki.fd.io/view/GoVPP).

#### 2. Synchronous VPP call

Let's return to the tutorial. Till now we have GoVPPMux plugin injected into the `HelloWorld` plugin and the VPP channel prepared. In this step, we will configure the loopback interface (from the `interfaces.api.json` bindings) synchronously. 

At first, define a new method where the message will be processed and prepare the request:
```go
func (p *HelloWorld) syncVppCall() {
	request := &interfaces.CreateLoopback{}
}
```

The request struct is from the generated VPP API file. The request type defines a configuration item which will be created which in this case is a loopback interface. The majority of the request messages contain one or more fields which can be set to a given value. In our example, the `CreateLoopback` has one field `MacAddress` of type `[]byte`. Filling it will help us to create a new loopback interface with the MAC address already assigned. Since the most convenient method would be to define a MAC address as `string`, we will prepare short helper method which transforms our `string` type MAC address to a byte array using the built-in `net` package:
```go
func macParser(mac string) []byte {
	hw, err := net.ParseMAC(mac)
	if err != nil {
		panic(err)
	}
	return hw
}
```

Now we can add the MAC address to the request body:
```go
func (p *HelloWorld) syncVppCall() {
	request := &interfaces.CreateLoopback{
		MacAddress: macParser("00:00:00:00:00:01"),
	}
}
```

Sending the request can result in success or failure. To learn about the state of our request, we must define a reply value. The rule within the VPP API is that the message reply has the same name + `Reply` suffix:
```go
func (p *HelloWorld) syncVppCall() {
	request := &interfaces.CreateLoopback{
		MacAddress: macParser("00:00:00:00:00:01"),
	}
	reply := &interfaces.CreateLoopbackReply{}
}
```

When the request and reply messages are prepared, we can make a VPP call using our VPP channel. This is done in two steps:

- Send the request message using `SendRequest`
- Receive reply message calling `ReceiveReply` on request context

The call can be done in a single line (do not forget to handle the error):
```go
func (p *HelloWorld) syncVppCall() {
	request := &interfaces.CreateLoopback{
		MacAddress: macParser("00:00:00:00:00:01"),
	}
	reply := &interfaces.CreateLoopbackReply{}
	err := p.vppChan.SendRequest(request).ReceiveReply(reply)
	if err != nil {
		panic(err)
	}
}
```

It is important to mention, that **the `ReceiveReply` is a blocking call**. The reply is provided only when the request is fully processed in VPP which may take some time (order milliseconds), depending on the request.  

The reply message always provides us with the reply value called `Retval`. It contains a return code in case something went wrong with the API call. It is not required to inspect separately since in such a case the call always returns error. 

There can be other useful data in the reply. In our case, the useful data is represented by the interface index generated within VPP. This is used as a unique identifier for the interface just created.

Add code to print the index of our loopback:
```go
func (p *HelloWorld) syncVppCall() {
	request := &interfaces.CreateLoopback{
		MacAddress: macParser("00:00:00:00:00:01"),
	}
	reply := &interfaces.CreateLoopbackReply{}
	err := p.vppChan.SendRequest(request).ReceiveReply(reply)
	if err != nil {
		panic(err)
	}
	log.Printf("Sync call created loopback with index %d", reply.SwIfIndex)
}
```

Last step is to start `syncVppCall` from the `main` :
```go
func main() {
	// Create an instance of our plugin.
	p := new(HelloWorld)
	p.GoVPPMux = &govppmux.DefaultPlugin

	// Create new agent with our plugin instance.
	a := agent.NewAgent(agent.AllPlugins(p))

	// Run starts the agent with plugins, wait until shutdown
	// and then stops the agent and its plugins.
	if err := a.Start(); err != nil {
		log.Fatalln(err)
	}
	
	p.syncVppCall()

	if err := a.Stop(); err != nil {
		log.Fatalln(err)
	}
}
```

Now we can start the tutorial example together with VPP. Verify the expected output using the VPP CLI console:
```bash
vpp# sh int
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
local0                            0     down          0/0/0/0       
loop0                             1     down         9000/0/0/0        
vpp# 
``` 

Our loopback interface (internally named 'loop0') was created. Currently, it is down since the admin status changes via another binary API we did not use. But the procedure is the same. 

The mac address can be also checked:
```bash
pp# sh hardware
              Name                Idx   Link  Hardware
local0                             0    down  local0
  Link speed: unknown
  local
loop0                              1    down  loop0
  Link speed: unknown
  Ethernet address 00:00:00:00:00:01
vpp# 
``` 

This output shows us that the MAC address was set to the value we defined in the binary API call.

#### 3. Asynchronous VPP call

Here we will configure several loopback interfaces asynchronously. This means we do not process API calls one-by-one but rather put all request messages out at once and then process the replies later. 

Let's start with another method and prepare two more requests. We will use different MAC addresses for better resolution:
```go
func (p *HelloWorld) asyncVppCall() {
	request1 := &interfaces.CreateLoopback{
		MacAddress: macParser("00:00:00:00:00:02"),
	}
	request2 := &interfaces.CreateLoopback{
		MacAddress: macParser("00:00:00:00:00:03"),
	}
}
```

Now let's send those requests and maintain request contexts:
```go
func (p *HelloWorld) asyncVppCall() {
	
	...
	
	reqCtx1 := p.vppChan.SendRequest(request1)
    reqCtx2 := p.vppChan.SendRequest(request2)
}
```

At this point, every request was sent and VPP request processing is underway. In the meantime, our example can allocate responses:
```go
func (p *HelloWorld) asyncVppCall() {
	
	...
	
	reply1 := &interfaces.CreateLoopbackReply{}
    reply2 := &interfaces.CreateLoopbackReply{}
}
```

Then both replies can be obtained using the correct request context:
```go
func (p *HelloWorld) asyncVppCall() {
	
	...
	
	if err := reqCtx1.ReceiveReply(reply1); err != nil {
        panic(err)
    }
    if err := reqCtx2.ReceiveReply(reply2); err != nil {
        panic(err)
    }
}
```

The last step is to do all mandatory checks and printing of the interface indexes as before. This is how the complete method looks:
```go
func (p *HelloWorld) ``() {
	request1 := &interfaces.CreateLoopback{
		MacAddress: macParser("00:00:00:00:00:02"),
	}
	request2 := &interfaces.CreateLoopback{
		MacAddress: macParser("00:00:00:00:00:03"),
	}
	reqCtx1 := p.vppChan.SendRequest(request1)
	reqCtx2 := p.vppChan.SendRequest(request2)
	
	reply1 := &interfaces.CreateLoopbackReply{}
	reply2 := &interfaces.CreateLoopbackReply{}
	
	if err := reqCtx1.ReceiveReply(reply1); err != nil {
		panic(err)
	}
	if err := reqCtx2.ReceiveReply(reply2); err != nil {
		panic(err)
	}
	log.Printf("Async call created loopbacks with indexes %d, %d and %d",
		reply1.SwIfIndex, reply2.SwIfIndex, reply3.SwIfIndex)
}
```

Add the `asyncVppCall` to the `main`:
```go
func main() {
	// Create an instance of our plugin.
	p := new(HelloWorld)
	p.GoVPPMux = &govppmux.DefaultPlugin

	// Create new agent with our plugin instance.
	a := agent.NewAgent(agent.AllPlugins(p))

	// Run starts the agent with plugins, wait until shutdown
	// and then stops the agent and its plugins.
	if err := a.Start(); err != nil {
		log.Fatalln(err)
	}
	
	p.syncVppCall()
	p.asyncVppCall()

	if err := a.Stop(); err != nil {
		log.Fatalln(err)
	}
}
```

Then start the example as before. Now we expect three loopback interfaces configured. MAC addresses can be also verified the same way as before.
```bash
vpp# sh int
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
local0                            0     down          0/0/0/0       
loop0                             1     down         9000/0/0/0     
loop1                             2     down         9000/0/0/0     
loop2                             3     down         9000/0/0/0         
vpp# 
```

The advantage of this approach is that we do not need to wait for every single request at the time when it is sent. Rather we can send multiple requests at once and "collect" the responses later potentially in a different order. Do not forget that the `ReceiveReply` is still a blocking call.

#### 4. Multi-request VPP call

The VPP binary API also defines multi-request messages, where a single request generates multiple 
replies. Such a call is often used for reading data from VPP. Multi-request messages are usually distinguished with suffix `Dump` for request message and `Details` for a reply. In the tutorial, we will use a multi-request call to retrieve configured interfaces.

Prepare a new method and define request of type `SwInterfaceDump`. Call this request using `SendMultiRequest` and retain the retrieved context: 
```go
func (p *HelloWorld) multiRequest() {
	request := &interfaces.SwInterfaceDump{}
	multiReqCtx := p.vppChan.SendMultiRequest(request)
}
```

Since we are expecting multiple replies, we must process them in a loop and define new reply messages for every iteration:
```go
func (p *HelloWorld) multiRequest() {
	request := &interfaces.SwInterfaceDump{}
	multiReqCtx := p.vppChan.SendMultiRequest(request)

	for {
		reply := &interfaces.SwInterfaceDetails{}
	}
}
```

In the next step, we use the same context in every iteration to retrieve the return value using `ReceiveReply`. The `ReceiveReply` called in the multi-request context also returns a boolean flag on whether the obtained message was the last one or not:
```go
func (p *HelloWorld) multiRequest() {
	request := &interfaces.SwInterfaceDump{}
	multiReqCtx := p.vppChan.SendMultiRequest(request)

	for {
		reply := &interfaces.SwInterfaceDetails{}
		last, err := multiReqCtx.ReceiveReply(reply)
		if err != nil {
			panic(err)
		}
		if last {
			break
		}
		log.Printf("received VPP interface with index %d", reply.SwIfIndex)
	}
}
```

Let's add the `multiRequestCall` to the `main`:
```go
func main() {
	// Create an instance of our plugin.
	p := new(HelloWorld)
	p.GoVPPMux = &govppmux.DefaultPlugin

	// Create new agent with our plugin instance.
	a := agent.NewAgent(agent.AllPlugins(p))

	// Run starts the agent with plugins, wait until shutdown
	// and then stops the agent and its plugins.
	if err := a.Start(); err != nil {
		log.Fatalln(err)
	}
	
	p.syncVppCall()
	p.asyncVppCall()
	p.multiRequestCall()

	if err := a.Stop(); err != nil {
		log.Fatalln(err)
	}
}
```

The output of this call will be shown in the log as a repeated message indicating that the VPP interface with a given index was received. In the multi-request scenario, the reply message usually contains several fields where some are VPP specific such as interface admin status, default MTU, internal names, etc.).

[1]: ../plugins/vpp-plugins.md#govppmux-plugin
[2]: https://wiki.fd.io/view/GoVPP
[code-link]: https://github.com/ligato/vpp-agent/tree/master/examples/tutorials/07_vpp-connection

*[CLI]: Command-Line Interface
