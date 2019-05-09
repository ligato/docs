# Examples

---

This folder contains several examples that illustrate various aspects of VPP Agent's functionality. Each example is structured as an individual executable with its own `main.go` file. Each example focuses on a very simple use case. All examples use ETCD, GOVPP and Kafka, so please make sure there are running instances of ETCD, Kafka and VPP before starting an example.

Current examples:
* **[govpp_call][example-govpp-call]** is an example of a plugin with a configurator and a channel to send/receive data to VPP. The example shows how to transform northbound model data to VPP binary API calls. 
* **[localclient_vpp_plugins][example-localclient-vpp-plugins]** demonstrates how to use the localclient package to push example configuration into VPP plugins that run in the same agent instance (i.e. in the same OS process). Behind the scenes, configuration data is transported via go channels.
* **[localclient_vpp_nat][example-localclient-vpp-nat]** demonstrates how to set up NAT global configuration and Destination NAT. The example uses localclient to put example data to the respective VPP plugins.
* **[localclient_linux_tap][example-localclient-linux-tap]** configures 
  simple topology consisting of VPP Tap interfaces with linux host 
  counterparts. Example demonstrates how to use the localclient package 
  to push example configuration for those interface types into linux 
  and VPP plugins running within the same agent instance (i.e. within 
  the same OS process). Behind the scenes the configuration data 
  is transported via go channels. 
* **[localclient_linux_veth][example-localclient-linux-veth]** configures 
  simple topology consisting of VPP af-packet interfaces attached to 
  linux Veth pairs. As before, this example also uses localclient to push 
  the configuration to vpp-agent plugins.  
* **[grpc_vpp_remote_client][example-grpc-vpp-remote]** demonstrates how to
  use the remoteclient package to push example configuration into
  VPP default plugins running within different vpp-agent OS process.
* **[grpc_vpp_notifications][example-grpc-vpp-notifications]** demonstrates how to
  use the notifications package to  receive VPP notifications streamed by different 
  vpp-agent process.

* **[CN-Infra  examples][cn-infra-examples]** demonstrate how to use the CN-Infra framework
  plugins.
  
## How to run an example
 
 **1. Start the ETCD server on localhost**
 
  ```
  sudo docker run -p 2379:2379 --name etcd --rm 
  quay.io/coreos/etcd:v3.1.0 /usr/local/bin/etcd \
  -advertise-client-urls http://0.0.0.0:2379 \
  -listen-client-urls http://0.0.0.0:2379
  ```
  Note: **For ARM64 see the information for [etcd][3]**.
  
 **2. (Optional) start Kafka on localhost**

 ```
 sudo docker run -p 2181:2181 -p 9092:9092 --name kafka --rm \
  --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 spotify/kafka
 ```
  Note: **For ARM64 see the information for [kafka][kafka-arm64]**.

 **3. Start VPP**
 ```
 vpp unix { interactive } plugins { plugin dpdk_plugin.so { disable } }
 ```
 
 **4. Start desired example**

 Example can be started now from particular directory.
 ```
 go run main.go  \
 --etcd-config=/opt/vpp-agent/dev/etcd.conf \
 --kafka-config=/opt/vpp-agent/dev/kafka.conf
 ```
[cn-infra-examples]: https://github.com/ligato/cn-infra/tree/master/examples 
[example-govpp-call]: https://github.com/ligato/vpp-agent/tree/master/examples/govpp_call
[example-grpc-vpp-notifications]: https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/notifications
[example-grpc-vpp-remote]: https://github.com/ligato/vpp-agent/tree/master/examples/grpc_vpp/remote_clien
[example-localclient-linux-tap]: https://github.com/ligato/vpp-agent/tree/master/examples/localclient_linux/tap
[example-localclient-linux-veth]: https://github.com/ligato/vpp-agent/tree/master/examples/localclient_linux/veth
[example-localclient-vpp-nat]: https://github.com/ligato/vpp-agent/tree/master/examples/localclient_vpp/nat
[example-localclient-vpp-plugins]: https://github.com/ligato/vpp-agent/tree/master/examples/localclient_vpp/plugins
[kafka-arm64]: arm64.md#arm64-and-kafka
[etcd-arm64]: arm64.md#arm64-and-etcd-server
