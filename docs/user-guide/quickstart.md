# Quickstart Guide

---

This section provides a quick start guide for the VPP agent.

---

In this guide, you will learn how to:

- Install the VPP agent
- Install and start etcd
- Run the VPP agent container
- Interact with the VPP agent using etcdctl, REST, VPP CLI and agentctl. 

The figure below illustrates our quickstart guide environment.

![quickstart-topology](../img/user-guide/quickstart-topo-w-agentctl.svg)

!!! Note
    __VPP Agent__ is used when discussing the Ligato VPP Agent in a higher level context with examples being an architecture or functional description. `vpp-agent` is used in code, repository folders, CLI and docker. __VPP__ is used to describe the VPP data plane.  

---

## 1. Prerequisites

- **Docker** (Docker ce [installation manual][docker-install])
- **etcd** 
- **Postman** or **cURL** tool (postman [installation manual][postman-install])  

!!! note
    The steps described in this quickstart guide use Docker 19.03.5. Use the `docker --version` command to determine which version you are running.

## 2. Download Image

Pull the VPP agent image from [DockerHub][dockerhub]. This image contains the VPP agent and a compatible VPP data plane.

```
docker pull ligato/vpp-agent
```



Verify the image has downloaded:

```
docker images
```

Sample output:
```
REPOSITORY              TAG                 IMAGE ID            CREATED             SIZE
ligato/vpp-agent        latest              05ad16109a2b        3 days ago          264MB
```

Verify the container is _NOT_ running. 

```
docker ps
```
Sample output:
```
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

## 3. etcd

[etcd][etcd] is a Key-Value (KV) data store containing VPP configuration information structured as key-value pairs.

### 3.1 Start etcd

Use the following command to start the etcd server in a docker container. If the image is not present on your localhost, docker will download it for you.
```
docker run --rm --name etcd -p 2379:2379 -e ETCDCTL_API=3 quay.io/coreos/etcd /usr/local/bin/etcd -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379
```

Open a new terminal session and verify the etcd container is running:
```sh
docker ps -f name=etcd
```
Sample output:
```
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS              PORTS                              NAMES
f3db6e6d8975        quay.io/coreos/etcd:latest   "/usr/local/bin/etcd…"   16 minutes ago      Up 16 minutes       0.0.0.0:2379->2379/tcp, 2380/tcp   etcd
```

### 3.2 etcdctl

**etcdctl** is an etcd client CLI tool that can list, put, delete and get key-value pairs from the etcd data store.

You can install it locally depending on your OS.

```bash
// Linux users
$ apt-get install etcd-client

// MAC users
$ brew install etcd
```

However, it's easier and __recommended__ to use etcdctl that comes with the etcd image. Use this command to check the version of etcdctl you are running:
```
docker exec -it etcd etcdctl version
```
Sample output:
```
etcdctl version: 3.3.8
API version: 3.3
```
Use this command to verify etcd server endpoint health:
```json
docker exec -it etcd etcdctl endpoint health
```
Sample output:
```json
127.0.0.1:2379 is healthy: successfully committed proposal: took = 1.6141ms
```


## 4. Start VPP Agent

Open a new terminal session. Start the VPP agent and compatible VPP data plane version in a new docker container:
```
docker run -it --rm --name vpp-agent -p 5002:5002 -p 9191:9191 --privileged ligato/vpp-agent
``` 

Open a new terminal session. Verify the container called `vpp-agent` is running:

```
docker ps -f name=vpp-agent
```

Sample output:
```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                            NAMES
66556668533d        ligato/vpp-agent    "/bin/sh -c 'rm -f /…"   19 seconds ago      Up 19 seconds       0.0.0.0:5002->5002/tcp, 0.0.0.0:9191->9191/tcp   vpp-agent
```


## 5. Managing the VPP Agent

In this section, you will learn: 

- How to configure the VPP data plane through the etcd data store and VPP agent using etcdctl.
- How to read VPP configuration data with a VPP agent REST API.
- How to connect to the VPP CLI and show the VPP configuration.
- How to use agentctl to manage the VPP agent.

---

### 5.1 etcdctl

VPP agent entries in the etcd data store use a prefix of `/vnf-agent/`. List those key-value pairs using this command:
```
docker exec -it etcd etcdctl get --prefix /vnf-agent/
```
Sample output:
```
/vnf-agent/vpp1/check/status/v1/agent
{"build_version":"v2.1.1-1-g80401e6","build_date":"2019-05-27T23:57-07:00","state":"OK","start_time":"1563220274","last_change":"1563220280","last_update":"1563220475","commit_hash":"80401e66a0ef370ac61c404590de51d86f78bf4c","plugins":[{"name":"govpp","state":"OK"},{"name":"etcd","state":"OK"},{"name":"vpp-ifplugin","state":"OK"}]}
/vnf-agent/vpp1/check/status/v1/plugin/etcd
{"state":"OK","last_change":"1563220279","last_update":"1563220475"}
/vnf-agent/vpp1/check/status/v1/plugin/govpp
{"state":"OK","last_change":"1563220274","last_update":"1563220475"}
/vnf-agent/vpp1/check/status/v1/plugin/vpp-abfplugin
{"last_change":"1563220274","last_update":"1563220475"}
/vnf-agent/vpp1/check/status/v1/plugin/vpp-aclplugin
{"last_change":"1563220274","last_update":"1563220475"}
/vnf-agent/vpp1/check/status/v1/plugin/vpp-ifplugin
{"state":"OK","last_change":"1563220280","last_update":"1563220475"}
/vnf-agent/vpp1/check/status/v1/plugin/vpp-ipsec-plugin
{"last_change":"1563220274","last_update":"1563220475"}
/vnf-agent/vpp1/check/status/v1/plugin/vpp-l2plugin
{"last_change":"1563220274","last_update":"1563220475"}
/vnf-agent/vpp1/check/status/v1/plugin/vpp-l3plugin
{"last_change":"1563220274","last_update":"1563220475"}
/vnf-agent/vpp1/check/status/v1/plugin/vpp-natplugin
{"last_change":"1563220274","last_update":"1563220475"}
/vnf-agent/vpp1/check/status/v1/plugin/vpp-srplugin
{"last_change":"1563220274","last_update":"1563220475"}
```



Configure a loopback interface with an IP address by putting the key-value pair into the etcd data store:
```
docker exec etcd etcdctl put /vnf-agent/vpp1/config/vpp/v2/interfaces/loop1 \
'{"name":"loop1","type":"SOFTWARE_LOOPBACK","enabled":true,"ip_addresses":["192.168.1.1/24"]}'
```
This command returns an `OK` message. Note that the key-value pairs contain a loopback interface configuration.

Configure a **bridge domain**:
```
docker exec etcd etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1 \
'{"name":"bd1","forward":true,"learn":true,"interfaces":[{"name":"loop1"}]}'
```
This command returns an `OK` message. Same idea as above, but this time for a bridge domain.

Verify the **loopback interface** configuration exists in the etcd data store:
```
docker exec etcd etcdctl get /vnf-agent/vpp1/config/vpp/v2/interfaces/loop1
```

Sample output:
```
/vnf-agent/vpp1/config/vpp/v2/interfaces/loop1
{"name":"loop1","type":"SOFTWARE_LOOPBACK","enabled":true,"ip_addresses":["192.168.1.1/24"]}
```

Verify the **bridge domain configuration** exists in the etcd data store:
```
docker exec etcd etcdctl get /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1
```
Sample output:
```
/vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1
{"name":"bd1","forward":true,"learn":true,"interfaces":[{"name":"loop1"}]}
```

---

### 5.2 REST API

!!! Note
    You cannot use REST to add, modify or delete configuration data. You can use REST to read or retrieve configuration information.

Get VPP interfaces:
```
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces
```

The response describes two interfaces:

- loopback interface `"name": "UNTAGGED-local0"` present in VPP by default
- loopback interface `"name": "loop1"`configured in the previous step.

Sample response:

```
{
  "0": {
    "interface": {
      "name": "UNTAGGED-local0",
      "type": "SOFTWARE_LOOPBACK",
      "physAddress": "00:00:00:00:00:00"
    },
    "interface_meta": {
      "sw_if_index": 0,
      "sub_sw_if_index": 0,
      "l2_address": "AAAAAAAA",
      "internal_name": "local0",
      "is_admin_state_up": false,
      "is_link_state_up": false,
      "link_duplex": 0,
      "link_mtu": 0,
      "mtu": [
        0,
        0,
        0,
        0
      ],
      "link_speed": 0,
      "sub_id": 0,
      "tag": "",
      "dhcp": null,
      "vrf_ipv4": 0,
      "vrf_ipv6": 0,
      "pci": 0
    }
  },
  "1": {
    "interface": {
      "name": "loop1",
      "type": "SOFTWARE_LOOPBACK",
      "enabled": true,
      "physAddress": "de:ad:00:00:00:00",
      "ipAddresses": [
        "192.168.1.1/24"
      ]
    },
    "interface_meta": {
      "sw_if_index": 1,
      "sub_sw_if_index": 1,
      "l2_address": "3q0AAAAA",
      "internal_name": "loop0",
      "is_admin_state_up": true,
      "is_link_state_up": true,
      "link_duplex": 0,
      "link_mtu": 9216,
      "mtu": [
        9000,
        0,
        0,
        0
      ],
      "link_speed": 0,
      "sub_id": 0,
      "tag": "loop1",
      "dhcp": null,
      "vrf_ipv4": 0,
      "vrf_ipv6": 0,
      "pci": 0
    }
  }
}
```


Get bridge domain configuration information:
```
curl -X GET http://localhost:9191/dump/vpp/v2/bd
```
Sample response:
```
[
  {
    "bridge_domain": {
      "name": "bd1",
      "forward": true,
      "learn": true,
      "interfaces": [
        {
          "name": "loop1"
        }
      ]
    },
    "bridge_domain_meta": {
      "bridge_domain_id": 1
    }
  }
]

```

For more information on VPP agent REST APIs including sample responses, refer to the [VPP Agent API Documentation](../api/api-vpp-agent.md) 

---

### 5.3 VPP CLI

Start the VPP CLI in the vpp-agent container:
```
docker exec -it vpp-agent vppctl -s localhost:5002
```
output:
```
    _______    _        _   _____  ___ 
 __/ __/ _ \  (_)__    | | / / _ \/ _ \
 _/ _// // / / / _ \   | |/ / ___/ ___/
 /_/ /____(_)_/\___/   |___/_/  /_/    

vpp# 
```


List configured interfaces:
```
vpp# show interface
```
Sample output:
```
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count
local0                            0     down          0/0/0/0
loop0                             1      up          9000/0/0/0
```

The vpp-agent automatically creates the default `local0` interface.  You configured the `loop0` interface in [section 5.1](#51-etcdctl). 

Note that the command output uses the `internal_name`of the interface contained in the metadata. You can confirm this by looking at the `interface_meta` in the `GET /dump/vpp/v2/interfaces` REST API response. 

Show bridge domains:
```
vpp# show bridge-domain
```
 Sample output:
```
 BD-ID   Index   BSN  Age(min)  Learning  U-Forwrd   UU-Flood   Flooding  ARP-Term  arp-ufwd   BVI-Intf
    1       1      0     off        on        on        drop       off       off       off        N/A
```

As an alternative, you can run the commands from the terminal CLI:
```
docker exec -it vpp-agent vppctl -s localhost:5002 show interface
docker exec -it vpp-agent vppctl -s localhost:5002 show interface addr
docker exec -it vpp-agent vppctl -s localhost:5002 show bridge-domain
```

Help for VPP CLI commands options:
```
vpp# help
```

Use the `quit` command to exit the VPP CLI.

For more information on VPP CLI commands, see the [CLI guide of the FD.io wiki](https://wiki.fd.io/view/VPP/Command-line_Interface_(CLI)_Guide), and the [VPP CLI section](vpp-cli.md).

### 5.4 Agentctl

Agenctl is a CLI command line tool for managing and interacting with the software components of the Ligato framework.

Agentctl help:
```
docker exec -it vpp-agent agentctl --help
```
Output:
```

                      __      __  __
  ___ ____ ____ ___  / /_____/ /_/ /
 / _ '/ _ '/ -_) _ \/ __/ __/ __/ /
 \_,_/\_, /\__/_//_/\__/\__/\__/_/
     /___/

COMMANDS
  config      Manage agent configuration
  dump        Dump running state
  generate    Generate config samples
  import      Import config data from file
  kvdb        Manage agent data in KVDB
  log         Manage agent logging
  metrics     Get runtime metrics
  model       Manage known models
  report      Create error report
  service     Manage agent services
  status      Retrieve agent status and version info
  values      Retrieve values from scheduler
  vpp         Manage VPP instance

OPTIONS:
      --config-dir string        Path to directory with config file.
  -D, --debug                    Enable debug mode
  -e, --etcd-endpoints strings   Etcd endpoints to connect to, default from ETCD_ENDPOINTS env var (default
                                 [127.0.0.1:2379])
      --grpc-port int            gRPC server port (default 9111)
  -H, --host string              Address on which agent is reachable, default from AGENT_HOST env var (default "127.0.0.1")
      --http-basic-auth string   Basic auth for HTTP connection in form "user:pass"
      --http-port int            HTTP server port (default 9191)
      --insecure-tls             Use TLS without server's certificate validation
  -l, --log-level string         Set the logging level ("debug"|"info"|"warn"|"error"|"fatal")
      --service-label string     Service label for specific agent instance, default from MICROSERVICE_LABEL env var
  -v, --version                  Print version info and quit

Run 'agentctl COMMAND --help' for more information on a command.
```

Get the VPP agent config data:
```json
docker exec -it vpp-agent agentctl config get
```
Sample output:
```json
vppConfig:
  interfaces:
  - name: loop1
    type: SOFTWARE_LOOPBACK
    enabled: true
    ipAddresses:
    - 192.168.1.1/24
  bridgeDomains:
  - name: bd1
    forward: true
    learn: true
    interfaces:
    - name: loop1
linuxConfig: {}
netallocConfig: {}
```

Refer to the [agentctl](agentctl.md) section of this user guide for more information and examples.
 
!!! Note
    Attempts to interact with the etcd data store using the `agentctl kvdb` command could encounter a `Failed to connect to Etcd` message. This is because the VPP agent that includes `agentctl` is started in one container, and etcd is started another. Agentctl uses a default address of `127.0.0.1` to reach the etcd server; The etcd server is started with a default address of `172.17.0.2:2379`. The solution is to pass the etcd server address to agentctl using the `e` or `--etcd-endpoints` flags like so: `agentctl -e 172.17.0.2:2379 kvdb <command>`.  

## Troubleshooting

The VPP agent container starts and immediately closes.
  
- The etcd container is not running. Please verify it is running using the `docker ps` command.

---

The etcdctl command returns "Error:  100: Key not found".

- The etcdctl API version was not correctly set. Check the output of the appropriate environment variable with command `echo $ETCDCTL_API`. If the version is not set to "3", change it with `export ETCDCTL_API=3` command.

---

The cURL or postman command to access VPP agent REST APIs does not work (connection refused).

- The command starting the docker container exports port 9191 to allow access from the host. Make sure that the vpp-agent docker container is started with parameter `-p 9191:9191`. Run the `Restart vpp-agent steps`  shown below to modify port numbers.

---

The cURL or postman command to access VPP CLI does not work (connection refused)

- The command starting the docker container exports port 5002 (the VPP default port) to allow access from the host. Make sure that the vpp-agent docker container is started with parameter `-p 5002:5002`. Run the `Restart vpp-agent steps` shown below to modify port numbers.

---

Agentctl kvdb `Failed to connect to Etcd` problem.

- Agentctl uses `127.0.0.1:2379 as the etcd server address. But in the examples above, etcd is running in a separate container. To resolve:

Use this command obtain the IP address of the etcd server:
```json
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' etcd
``` 
Output:
```json
172.17.0.2
```
Then pass this address to agentctl using the `-e` or `--etcd-endpoints` flag like so:
```json
docker exec -it vpp-agent agentctl -e 172.17.0.2:2379 kvdb list
```

---

Restart VPP agent steps:

    docker ps -f name=vpp-agent
    docker stop <XX> ; with <XX> equals first to 2 characters of container ID
    // restart with correct port numbers if needed
    docker run -it --rm --name vpp-agent -p 5002:5002 -p 9191:9191 --privileged ligato/vpp-agent
    
<br/>


[docker-install]: https://docs.docker.com/cs-engine/1.13/
[dockerhub]: https://hub.docker.com/u/ligato
[postman-install]: https://learning.getpostman.com/docs/postman/launching_postman/installation_and_updates/
[etcd]: https://etcd.io/

*[CLI]: Command-Line Interface
*[REST]: Representational State Transfer
