# Custom Plugin

---

This section provides information on how to create a custom VPP plugin.

---

**References:**

- [Tutorials](../tutorials/00_tutorial-setup.md) beginning with the Hello World plugin.
<br></br> 
- [Custom VPP Plugin][custom-vpp-plugin] contains a working example of a VPP agent that implements a custom VPP plugin. 

---

Ligato lets you develop custom VPP plugins. Your implementation depends on how you use the VPP agent code.

You have three options for using the VPP agent code.The following discusses each option.

---

**Using the official VPP agent** 

!!! Note
    Use this option for only testing and development purposes.
    
You should skip this option. Adding support for a custom VPP plugin to the VPP agent is __NOT possible__ without code modifications.

---

**Using a forked VPP agent**

You can fork the [VPP agent repository](https://github.com/ligato/vpp-agent) and modify the code to build your custom VPP plugin. This option might lead to project maintenance complications due to conflicts when updating the fork.

--- 

!!! Note
    If possible, you should avoid this option.

---

However, you might have to use this option, depending on your custom VPP plugin's functionality.    
For example, if your custom VPP plugin defines a new VPP interface type, you will need to modify VPP agent code:

- [VPP interface proto](https://github.com/ligato/vpp-agent/blob/master/proto/ligato/vpp/interfaces/interface.proto) requires changes to the `vpp_interfaces.Interface_Type enum` along with any interface configuration options.
<br></br>
- [VPP interface plugin](https://github.com/ligato/vpp-agent/tree/master/plugins/vpp/ifplugin) Go code that requires handling of interface-type specific code you add to existing interface descriptors.

---

The following lists two PRs you can use as reference for **adding support for new VPP interface types**:

- [#1560](https://github.com/ligato/vpp-agent/pull/1560): GTP-U tunnel.
<br></br>
- [#1457](https://github.com/ligato/vpp-agent/pull/1457): VxLAN-GPE.

---

The following lists two PRs you can use as reference for **adding support for new VPP functions**:

- [#1602](https://github.com/ligato/vpp-agent/pull/1602): L3 Cross Connect.
<br></br>
- [#1430](https://github.com/ligato/vpp-agent/pull/1430): SPAN.

---

**Using the VPP agent as a library**

You can develop your own agent and custom VPP plugin using a dependency on the `go.ligato.io/vpp-agent/v3 Go module`. 

!!! Note
    Use this option.

This allows you to use existing VPP agent plugins without modifying existing code. In addition, Go module dependency updates become straightforward.

---

**Summary of steps for adding a custom VPP plugin**

- Generate binapi (Go code) for your custom VPP plugin API.
<br></br>
- Define VPP handler API vppcalls that will be universal across VPP versions. No import of generated binapi needed.
<br></br>
- Implement the VPP handler for the VPP version you are working with.
<br></br>
- Define a protobuf file representing your functionality.
<br></br>
- Register your proto model and use it in KV descriptors.
<br></br>
- Implement KV descriptors for your custom VPP plugin's objects and resources.
<br></br>
- Create a new plugin for your custom VPP functionality and add it to your main plugin.
<br></br>
- Initialize the VPP handler and register KV descriptors during your plugin's `Init()`.







 

 




 
  











[custom-vpp-plugin]: https://github.com/ligato/vpp-agent/tree/master/examples/customize/custom_vpp_plugin 