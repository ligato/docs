# Quickstart Guide

In this guide you will learn how to:
- Install the vpp-agent
- Install and use the ETCD and Kafka
- Run the vpp-agent container
- Interact with the vpp-agent

The result of this guide prepares following topology:

![quickstart-diagram](https://user-images.githubusercontent.com/15376606/54192164-a5a0e600-44b7-11e9-9a8c-c034bf6b7443.png)

---

**Table of contents**
- [1. Prerequisites](#prerequisites)
- [2. Get the agent image from the dockerhub](#image)
- [3. Start ETCD](#etcd)
  - [3.1 Get the ETCD image](#etcdget)
  - [3.2 Use ETCD client (etcdctl)](#etcdclient)
- [4. Start Kafka](#kafka)  
- [5. Run the vpp-agent](#startagent)
- [6. Manage the vpp-agent](#manage)
  - [6.1 Configure the VPP using the vpp-agent](#manageconf)
  - [6.2 Read the VPP configuration via the vpp-agent API](#manageread)
  - [6.3 Read the status provided by the vpp-agent](#managestatus)
  - [6.4 Connect to the VPP in the container](#managevpp)
- [Troubleshooting](#troubleshooting)

## <a name="prerequisites">1. Prerequisites</a>

- **Docker** (docker-ce [installation manual](https://docs.docker.com/cs-engine/1.13/))
- **ETCD** 
- **Kafka** (optional)
- **cURL** (or **Postman** [installation manual](https://learning.getpostman.com/docs/postman/launching_postman/installation_and_updates/)) 

## <a name="image">2. Download the vpp-agent image</a>

To download the vpp-agent image from the [DockerHub](https://hub.docker.com/u/ligato) use the following command:
```
$ docker pull ligato/vpp-agent
```

This image contains the vpp-agent and also compatible VPP.

The command `docker images` shows us that the image exists:

```
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE   
ligato/vpp-agent            latest              17b7db787662        18 hours ago        175MB
```

The `docker ps` should not contain any mention about the vpp-agent in the output, since we did not start the image yet.

### <a name="etcd">3. Start ETCD</a>

#### <a name="etcdget">3.1 Get the ETCD image</a>

The following command starts the ETCD in a docker container. If the image is not present on your local machine, docker will download it first:

```
$ docker run --rm --name etcd -p 2379:2379 quay.io/coreos/etcd -e ETCDCTL_API=3 /usr/local/bin/etcd -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379
```

Verify the ETCD container is running:
```
$ docker ps -f name=etcd
CONTAINER ID        IMAGE                        COMMAND                  CREATED             STATUS              PORTS                              NAMES
f3db6e6d8975        quay.io/coreos/etcd:latest   "/usr/local/bin/etcd…"   16 minutes ago      Up 16 minutes       0.0.0.0:2379->2379/tcp, 2380/tcp   etcd
```

### <a name="etcdclient">3.2 Use the ETCD client</a>

The `etcdctl` is the official ETCD client which can be used to put/delete/get key-value pairs from the ETCD. 

You can install it locally:

```bash
// Linux users
$ apt-get install etcd-client

// MAC users
$ brew install etcd
```

However it's easier (and recommended) to use the one that comes with the ETCD image:

```
$ docker exec etcd etcdctl version
etcdctl version: 3.3.8
API version: 3.3
```

> Note: according to the ETCD documentation, the API version must be set via environmental variable: `ETCDCTL_API=3`. However we do not need to explicitly set it because we defined it when starting the etcd container.

With the following command you can list all key-value pairs related to the vpp-agent:

```
$ docker exec etcd etcdctl get --prefix /vnf-agent/
```

## <a name="kafka">4. Start Kafka</a>

**Note:** Since vpp-agent v2.0.0, running Kafka is not required by default.

Following command starts Kafka in a docker container:

```
$ docker run --rm --name kafka -p 2181:2181 -p 9092:9092 --env ADVERTISED_HOST=172.17.0.1 --env ADVERTISED_PORT=9092 spotify/kafka
```

Verify the Kafka container is running with following command:
```
$ docker ps -f name=kafka
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                                            NAMES
dd9bb1a482c5        spotify/kafka       "supervisord -n"    25 seconds ago      Up 24 seconds       0.0.0.0:2181->2181/tcp, 0.0.0.0:9092->9092/tcp   kafka
```

## <a name="startagent">5. Run the vpp-agent</a>

The command starts the Ligato vpp-agent together with the compatible version of the VPP in new docker container:
```
$ docker run -it --rm --name vpp-agent -p 5002:5002 -p 9191:9191 --privileged ligato/vpp-agent
``` 

Verify the vpp-agent container is running with following command:
```
$ docker ps -f name=vpp-agent
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                            NAMES
77df69266072        ligato/vpp-agent    "/bin/sh -c 'rm -f /…"   26 seconds ago      Up 25 seconds       0.0.0.0:5002->5002/tcp, 0.0.0.0:9191->9191/tcp   vpp-agent
```

## <a name="manage">6. Manage the vpp-agent</a>

This section is divided into several parts:
- How to configure the VPP providing data to the vpp-agent
- How to read VPP configuration asking the vpp-agent
- How to obtain the status (stored in the ETCD) provided by the vpp-agent
- How to connect to the VPP cli and verify the configuration 

### <a name="manageconf">6.1 Configure the VPP using the vpp-agent</a>

In this step we configure a simple loopback interface with an IP address putting the key-value pair to the ETCD:
```
$ docker exec etcd etcdctl put /vnf-agent/vpp1/config/vpp/v2/interfaces/loop1 \
'{"name":"loop1","type":"SOFTWARE_LOOPBACK","enabled":true,"ip_addresses":["192.168.1.1/24"]}'
```

Let's add a bridge domain:
```
$ docker exec etcd etcdctl put /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1 \
'{"name":"bd1","forward":true,"learn":true,"interfaces":[{"name":"loop1"}]}'
```

Verify the configuration is present in the ETCD:
```
$ docker exec etcd etcdctl get /vnf-agent/vpp1/config/vpp/v2/interfaces/loop1
$ docker exec etcd etcdctl get /vnf-agent/vpp1/config/vpp/l2/v2/bridge-domain/bd1
```
The output should return the data configured.

### <a name="manageread">6.2 Read the VPP configuration via the vpp-agent API</a>

This command returns the list of VPP interfaces accessing the vpp-agent REST API:
```
$ curl -X GET http://localhost:9191/dump/vpp/v2/interfaces
```

Two interfaces are returned in the output - the loopback interface with internal name `loop0` present in the vpp by default, and another loopback interface configured in the previous step.

Another command to read the bridge domain:
```
$ curl -X GET http://localhost:9191/dump/vpp/v2/bd
```

URLs can be used to get the same data via postman:
```
http://localhost:9191/dump/vpp/v2/interfaces
http://localhost:9191/dump/vpp/v2/bd
```
### <a name="managestatus">6.3 Read the status provided by the vpp-agent</a>

The vpp-agent publishes interface status to the ETCD. To obtain the status of the `loop1` interface (configured in 5.1) run following command:
```
$ docker exec etcd etcdctl get /vnf-agent/vpp1/vpp/status/v2/interface/loop1
```

The output should look like this:
```json
{"name":"loop1","internal_name":"loop0","if_index":1,"admin_status":"UP","oper_status":"UP","last_change":"1551866265","phys_address":"de:ad:00:00:00:00","mtu":9216,"statistics":{}}
```

Notice the `internal_name` assigned by the VPP as well as the `if_index`. The `statistics` section contain traffic data (received/transmitted packets, bytes, ...). Since there is no traffic at the VPP, statistics are empty.

**Note:** state is exposed only for interfaces (including default interfaces).

### <a name="managevpp">6.4 Connect to the VPP in the container</a>

The following command starts the VPP CLI in vpp-agent container:
```
$ docker exec -it vpp-agent vppctl -s localhost:5002
    _______    _        _   _____  ___ 
 __/ __/ _ \  (_)__    | | / / _ \/ _ \
 _/ _// // / / / _ \   | |/ / ___/ ___/
 /_/ /____(_)_/\___/   |___/_/  /_/    

vpp# 
```

The CLI is ready to accept VPP commands. For example: 

The command `show interface` lists configured interfaces:
```
vpp# show interface
              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count     
local0                            0     down          0/0/0/0       
loop0                             1      up          9000/0/0/0     
```

We can see the default `local0` interface and `loop0` configured by the vpp-agent.

The command `show bridge-domain` lists configured bridge domains:
```
vpp# show bridge-domain
  BD-ID   Index   BSN  Age(min)  Learning  U-Forwrd   UU-Flood   Flooding  ARP-Term   BVI-Intf 
    1       1      0     off        on        on        drop       off       off        N/A    
```

As an alternative, for sending single commands you can run the commands directly:
```
$ docker exec -it vpp-agent vppctl -s localhost:5002 show interface
$ docker exec -it vpp-agent vppctl -s localhost:5002 show interface addr
$ docker exec -it vpp-agent vppctl -s localhost:5002 show bridge-domain
```

## <a name="troubleshooting">Troubleshooting</a>

- **The vpp-agent container was started and immediately closed.**
  
  The ETCD or Kafka containers are not running. Please verify they are running using command `docker ps`.

- **The etcdctl command returns "Error:  100: Key not found".**

  The etcdctl API version was not correctly set. Check the output of the appropriate environment variable with command `echo $ETCDCTL_API`. If the version is not set to "3", make it so with `export ETCDCTL_API=3`.

- **The cURL or Postman command to access vpp-agent API does not work (connection refused).**

  The command starting docker container exports port 9191 to allow access from the host. Make sure that the vpp-agent docker container is started with parameter `-p 9191:9191`.  

- **The cURL or Postman command to access VPP-cli does not work (connection refused).**

  The command starting docker container exports port 5002 (the VPP default port) to allow access from the host. Make sure that the vpp-agent docker container is started with parameter `-p 5002:5002`.
