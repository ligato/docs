# VPP Connection

---

Tutorial code: [VPP Connection][code-link]

In this tutorial, you will learn how to use the [GoVPPMux plugin][1] with your HelloWorld plugin to connect to VPP. You will also see how to perform synchronous, asynchronous, and multi-request calls in order to put configuration items to the VPP data plane.

Before running through this tutorial, you should complete the [Hello World tutorial](01_hello-world.md) and the [Plugin Dependencies tutorial](02_plugin-deps.md). You will also need to install VPP, or the VPP binary file of the supported VPP version on your local machine. 

This tutorial focuses on the functions performed in the southbound (SB) direction between the VPP agent and the VPP data plane. To keep it simple, this tutorial does not use a KV data store. 

--- 

Let's start by creating a reliable connection with VPP. The VPP uses [GoVPP][2] as a VPP adapter. Add the GoVPPMux plugin to your HelloWorld plugin:
```go
type HelloWorld struct {
	GoVPPMux govppmux.API
}
```   

Initialize GoVPPMux plugin in the `main()` after the HelloWorld plugin struct definition:
```go
func main() {
	p := new(HelloWorld)
	p.GoVPPMux = &govppmux.DefaultPlugin    // initialize with the default plugin instance
	
...
	
}
``` 

The GoVPPMux API lets other plugins create their own API channels to communicate with VPP using the GoVPP core. The channel can be buffered or unbuffered. The channel passes requests and replies between VPP and your plugin. 

Create a channel in the `Init()`. Add a variable of `Channel` type to have the VPP channel available for all HelloWorld methods:
```go
type HelloWorld struct {
	vppChan api.Channel

	GoVPPMux govppmux.API
}
```

Initialize a new VPP unbuffered channel:
```go
func (p *HelloWorld) Init() (err error) {
	log.Println("Hello World!")

	if p.vppChan, err = p.GoVPPMux.NewAPIChannel(); err != nil {
		panic(err)
	}
	return nil
}
```

Close the channel:
```go
func (p *HelloWorld) Close() error {
	p.vppChan.Close()
	log.Println("Goodbye World!")
	return nil
}
```

Later, you will use the channel to send and receive data. But first, you need to prepare the data and send it in a way that VPP can understand.

#### 1. Binary API

!!! note
    This section provides some background on the binary API. This can be skipped, and you can proceed to the _Synchronous VPP call_ section of this tutorial. 

GoVPP is a toolset for VPP management. It provides a high-level API for communication with the GoVPP core, and for sending and receiving messages to/from the VPP via adapter. This is a component between GoVPP and VPP responsible for sending and receiving binary-encoded data via the VPP socket client. The socket is the default, but the VPP shared memory is also available when needed. It also provides a bindings generator for VPP JSON binary API definitions referred to as the **binapi-generator**.
 
Bindings are by default present in the path `/usr/share/vpp/api/` and divided into multiple logical files. Any of the JSON API files can be transformed into the `.go` file using the binapi-generator. Generated API calls or messages can be then directly referenced in the Go code. The VPP agent stores generated binary API files in the [binapi directory](https://github.com/ligato/vpp-agent/tree/master/plugins/vpp/binapi). 

The binapi-generator uses following format:
```bash
binapi-generator --input-file=/usr/share/vpp/api/<name>.api.json --output-dir=<path>
```

Remember that the `*.api.json` files are present only when the VPP is installed on the system. This step needs to be done only once. The binapi-generator must regenerate the files with the introduction of a new VPP version containing API changes.  

In this tutorial, the `binapi-generator` has generated the required messages, so you will not need to perform this step. Only the correct VPP version is mandatory. 

To check on the supported VPP versions, see the [vpp.env file ](https://github.com/ligato/vpp-agent/blob/master/vpp.env) in the vpp-agent repository.

To read more about GoVPP, go to the [GoVPP Project wiki](https://wiki.fd.io/view/GoVPP). Or see the discussion on the [GoVPPMux VPP plugin_ section][govppmux] of the _User Guide_.

#### 2. Synchronous VPP call

Let's return to the tutorial. So far, you have injected the GoVPPMux plugin into the HelloWorld plugin, and prepared the VPP channel. In this section, you will configure the loopback interface from the `interfaces.api.json` bindings synchronously. 

Define a new method where the message will be processed, and prepare the request:
```go
func (p *HelloWorld) syncVppCall() {
	request := &interfaces.CreateLoopback{}
}
```

The request struct is from the generated VPP API file. The request type defines a configuration item which will be created. In this case,the configuration item is a loopback interface. The majority of the request messages contain one or more fields which can be set to a given value. 

The `CreateLoopback()` method has one field,`MacAddress`, of type `[]byte`. Filling in this value lets you create a new loopback interface with the MAC address already assigned. 

Since the most convenient method would be to define a MAC address as a `string`, prepare a short helper method which transforms your `string` type MAC address to a byte array using the built-in `net` package:
```go
func macParser(mac string) []byte {
	hw, err := net.ParseMAC(mac)
	if err != nil {
		panic(err)
	}
	return hw
}
```

Add the MAC address to the request body:
```go
func (p *HelloWorld) syncVppCall() {
	request := &interfaces.CreateLoopback{
		MacAddress: macParser("00:00:00:00:00:01"),
	}
}
```

Sending the request can result in success or failure. To learn the request state, define a reply value. The rule with the VPP API is that the message reply has the same message name + `Reply` suffix:
```go
func (p *HelloWorld) syncVppCall() {
	request := &interfaces.CreateLoopback{
		MacAddress: macParser("00:00:00:00:00:01"),
	}
	reply := &interfaces.CreateLoopbackReply{}
}
```
The message name is `CreateLoopback` and the reply is `CreateLoopbackReply`.

After you have prepared the request and reply messages, make a VPP call using your VPP channel. 

This is performed in two steps:

- Send the request message using `SendRequest`.
- Receive the reply message by calling `ReceiveReply` on request context.

The call can be done in a single line. Do not forget to handle the error:
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

It is worth noting that the `ReceiveReply` is a blocking call. VPP will return the reply, only after it has fully processed the request. This could take up to (order milliseconds), depending on the request.  

The reply message always provides you with a reply value called `Retval`. It contains a return code in case something went wrong with the API call. You do not need to inspect the `Retval` separately. This is because the call will always return an error. 

There can be other useful data in the reply. The interface index generated within VPP is considered useful data. The interface index is a unique identifier for the interface that was just created.

Add code to print the index of your loopback:
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

Start `syncVppCall` from the `main()`:
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

Start the tutorial example together with VPP. Verify the expected output using the VPP CLI console:
```bash
vpp# sh int
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
local0                            0     down          0/0/0/0       
loop0                             1     down         9000/0/0/0        
vpp# 
``` 

Your loopback interface, internally named 'loop0', is created. Currently, it is down since the admin status changes with another binary API you did not use. However, the procedure is the same. 

Check the mac address:
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

This output shows you that the MAC address equals the value defined in the binary API call.

#### 3. Asynchronous VPP call

In this section, you will configure several loopback interfaces asynchronously. You won't process API calls one-by-one, but rather put all request messages out at once, and then process the replies later. 

Start with another method and prepare two more requests. Use different MAC addresses:
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

Send the requests and maintain request contexts:
```go
func (p *HelloWorld) asyncVppCall() {
	
	...
	
	reqCtx1 := p.vppChan.SendRequest(request1)
    reqCtx2 := p.vppChan.SendRequest(request2)
}
```

At this point, your code has sent every request and VPP request processing is underway. In the meantime, your example can allocate responses:
```go
func (p *HelloWorld) asyncVppCall() {
	
	...
	
	reply1 := &interfaces.CreateLoopbackReply{}
    reply2 := &interfaces.CreateLoopbackReply{}
}
```

Both replies can be obtained using the correct request context:
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

Add the `asyncVppCall` to the `main()`:
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

Then start the example as before. The code configured three loopback interfaces. The MAC addresses can be verified the same way as performed above.
```bash
vpp# sh int
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
local0                            0     down          0/0/0/0       
loop0                             1     down         9000/0/0/0     
loop1                             2     down         9000/0/0/0     
loop2                             3     down         9000/0/0/0         
vpp# 
```

The advantage of this approach is that you do not need to wait for every single request at the time when it is sent. Instead, you can send multiple requests at once and "collect" the responses later, potentially in a different order. Do not forget that the `ReceiveReply` is still a blocking call.

#### 4. Multi-request VPP call

The VPP binary API also defines multi-request messages, where a single request generates multiple 
replies. Programs  will often use such a call for reading data from VPP. 

Multi-request messages contain a `Dump` suffix for a request message, and a `Details` suffix for a reply. Let's use a multi-request call to retrieve configured interfaces.

Prepare a new method and define the request of type `SwInterfaceDump`. Call this request using `SendMultiRequest` and retain the retrieved context: 
```go
func (p *HelloWorld) multiRequest() {
	request := &interfaces.SwInterfaceDump{}
	multiReqCtx := p.vppChan.SendMultiRequest(request)
}
```

Since you are expecting multiple replies, process them in a loop and define new reply messages for each iteration:
```go
func (p *HelloWorld) multiRequest() {
	request := &interfaces.SwInterfaceDump{}
	multiReqCtx := p.vppChan.SendMultiRequest(request)

	for {
		reply := &interfaces.SwInterfaceDetails{}
	}
}
```

Use the same context in every iteration to retrieve the return value using `ReceiveReply`. The `ReceiveReply` that is called in the multi-request context also returns a boolean flag to indicate if the obtained message is the last one received:
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

Add the `multiRequestCall` method to the `main()`:
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

The log output of this call is a repeated message. This indicates that the VPP agent received the VPP interface with a given index. In the multi-request scenario, the reply message usually contains several fields where some are VPP specific such as interface admin status, default MTU, and internal name.

[1]: ../plugins/vpp-plugins.md#govppmux-plugin
[2]: https://wiki.fd.io/view/GoVPP
[govppmux]: ../plugins/vpp-plugins.md#govppmux-plugin
[code-link]: https://github.com/ligato/vpp-agent/tree/master/examples/tutorials/07_vpp-connection

*[CLI]: Command-Line Interface
