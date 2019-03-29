# VPP multi-version support

The vpp-agent is highly dependent on the version of the VPP binary API used to send and receive various types of configuration messages. The VPP API is changing over time, adding new binary calls or modifying or removing existing ones. The latter is the most crucial from the vpp-agent perspective since it introduces incompatibilities between vpp-agent and the VPP.
For that reason the vpp-agent does a compatibility check in the GoVPP multiplex plugin (an adapter of the GoVPP - the GoVPP component provides API for communication with the VPP). The compatibility check attempts to read an ID of the provided message. If just one message ID is not found (validation if the cyclic redundancy code), the VPP is considered incompatible and the vpp-agent will not connect. The message validation is essential for successful connection. Now it is clear that the vpp-agent is tightly bound to the version of the VPP (for that reason the vpp-agent image is shipped together with compatible VPP version to make it easier for users).

This concept may cause some inconveniences during manual setup, and especially in scenarios where the vpp-agent needs to be quickly switched to different VPP which is not compatible.

### VPP compatibility

The vpp-agent bindings are generated from the VPP JSON API definitions. Those can be found in the path `/usr/share/vpp/api`. Full VPP installation is required if definitions should be generated. The json API definition is then transformed to the `*.ba.go` file using `binapi-generater` which is a part of the GoVPP project. All generated structures implement the GoVPP `Message` interface which allows to get message name, CRC or message type, and represents generic type for all messages which could be sent via the VPP channel.

Example:
```go
type CreateLoopback struct {
	MacAddress []byte `struc:"[6]byte"`
}

func (*CreateLoopback) GetMessageName() string {
	return "create_loopback"
}
func (*CreateLoopback) GetCrcString() string {
	return "3b54129c"
}
func (*CreateLoopback) GetMessageType() api.MessageType {
	return api.RequestMessage
}
```

The code above is generated from `create_loopback` within the `interface.api.json`. The structure represents the binary API request call, and usually contains a set of fields which can be set to required values (like MAC address of the loopback interface in this example). Every VPP request call requires a response:

```go
type CreateLoopbackReply struct {
	Retval    int32
	SwIfIndex uint32
}

func (*CreateLoopbackReply) GetMessageName() string {
	return "create_loopback_reply"
}
func (*CreateLoopbackReply) GetCrcString() string {
	return "fda5941f"
}
func (*CreateLoopbackReply) GetMessageType() api.MessageType {
	return api.ReplyMessage
}
``` 

The response has a `Retval` field, which is `0` if the API call was successful and without errors. In case of any error, the field contains numerical index of VPP-defined error message. Other fields can be present (usually some information generated within the VPP, as the `SwIfIndex` of the created interface in this case). Notice that the response message has the same name but with `Reply` suffix. Other types of messages may serve to obtain various information from the VPP - those API calls have suffix `Dump` (and for them, the reply message is with `Details` suffix).

If the json API was changed, it must be re-generated in the vpp-agent in order to be compatible again (and all changes caused by the modified binary api structures must be resolved, like added, removed or renamed fields). This process can be time-consuming sometimes (depends on the size of the difference). In order to minimize updates for various VPP versions, the vpp-agent introduced multi-version support. 

### Multi-version

The vpp-agent multi-version support allows to switch to the different VPP version (with different API) without any changes to the vpp-agent itself and without any need to rebuild the vpp-agent binary. Plugins can now obtain the version of the VPP and the vpp-agent is trying to connect and initialize correct set of `vppcalls` - base methods preparing API requests for the VPP. 

Every `vppcalls` handler registers itself with the VPP version intended to support (like `vpp1810`, `vpp1901`, etc.). During initialization, the vpp-agent does compatibility check with with all available handlers, until it founds some which is compatible with required messages. The chosen handled must be in line with all messages, it is not possible (and reasonable) to use multiple handlers for single VPP. When the compatibility check found any handler, it is returned to the main plugin for use. 

The little drawback of this solution is a lot of duplicated code across `vppcalls`, since there is not much of significant API changes between versions.


 






 

   

