# Examples

---

This [VPP agent examples folder][vpp-agent-examples-folder] in the repo contains several examples that illustrate VPP agent functionality. Each example is structured as an individual executable with its own `main.go` file. Each example focuses on a simple use case. 

!!! note
    All examples use etcd, GoVPP and Kafka. Please make sure there are running instances of etcd, Kafka and VPP before running through an example. The [quickstart guide][quickstart-guide] can guide you through a setup, or follow the [instructions][setup-instructions] below using the pre-built docker image.

Current examples:

* **[govpp_call][example-govpp-call]** is an example of a plugin with a configurator and a channel to send/receive data to and from VPP. It shows how to transform northbound model data to VPP binary API calls.
<br />
<br />
          
* **[localclient_vpp_plugins][example-localclient-vpp-plugins]** demonstrates how to use the localclient package to push example configuration information into VPP plugins running in the same agent instance (i.e. in the same OS process). The configuration data is transported via go channels.   
<br />


* **[localclient_vpp_nat][example-localclient-vpp-nat]** demonstrates how to set up NAT global configuration and Destination NAT. The example uses localclient to communicate example configuration data to the respective VPP plugins.   
<br />
    
* **[localclient_linux_tap][example-localclient-linux-tap]** configures 
  simple topology consisting of VPP Tap interfaces with the linux host 
  counterparts. The example demonstrates how to use the localclient package 
  to push example configuration data for the respective interface types into linux 
  and VPP plugins running within the same agent instance (i.e. within 
  the same OS process). The configuration data 
  is transported via go channels.
<br />
<br />     
* **[localclient_linux_veth][example-localclient-linux-veth]** configures 
  simple topology consisting of VPP af-packet interfaces attached to 
  linux Veth pairs. This example uses the localclient package to push 
  the configuration to VPP agent plugins.   
<br />   
* **[grpc_vpp_remote_client][example-grpc-vpp-remote]** demonstrates how to
  use the remoteclient package to push example configuration data into
  VPP plugins running within different VPP agent OS processes.   
<br />  
* **[grpc_vpp_notifications][example-grpc-vpp-notifications]** demonstrates how to
  use the notifications package to receive VPP notifications streamed by different 
  VPP agent processes.   
<br />  
* **[CN-Infra  examples][cn-infra-examples]** demonstrates how to use the Ligato Infra framework
  plugins.
  
## How to run an example
 
**1. Download Image**

```
docker pull ligato/vpp-agent
```
 

**2. Start etcd server on localhost**
 
```
docker run --rm --name etcd -p 2379:2379 -e ETCDCTL_API=3 quay.io/coreos/etcd /usr/local/bin/etcd -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379
```
  </br>
 
 **2b. (Optional) start Kafka on localhost**

 ```
 sudo docker run -p 2181:2181 -p 9092:9092 --name kafka --rm \
  --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 spotify/kafka
 ```
</br>

  
Note: **For ARM64 see the information for [kafka][kafka-arm64]**.

 **3. Start VPP**
 ```
 vpp unix { interactive } plugins { plugin dpdk_plugin.so { disable } }
 ```
 
 **4. Start desired example**

 Example can be started now from the specific directory.
 
```
go run main.go  \
--etcd-config=/opt/vpp-agent/dev/etcd.conf \
--kafka-config=/opt/vpp-agent/dev/kafka.conf
```
[cn-infra-examples]: https://github.com/ligato/cn-infra/tree/master/examples 
[example-govpp-call]: https://github.com/ligato/vpp-agent/tree/master/examples/govpp_call
[example-grpc-vpp-notifications]: https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications
[example-grpc-vpp-remote]: https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_client
[example-localclient-linux-tap]: https://github.com/ligato/vpp-agent/tree/master/examples/localclient_linux/tap
[example-localclient-linux-veth]: https://github.com/ligato/vpp-agent/tree/master/examples/localclient_linux/veth
[example-localclient-vpp-nat]: https://github.com/ligato/vpp-agent/tree/master/examples/localclient_vpp/nat
[example-localclient-vpp-plugins]: https://github.com/ligato/vpp-agent/tree/master/examples/localclient_vpp/plugins
[kafka-arm64]: arm64.md#arm64-and-kafka
[etcd-arm64]: arm64.md#arm64-and-etcd-server
[quickstart-guide]: ../user-guide/quickstart.md
[setup-instructions]: ../user-guide/examples.md#how-to-run-an-example
[vpp-agent-examples-folder]: https://github.com/ligato/vpp-agent/tree/master/examples
