# Agentctl

---
This section describes the Agentctl CLI tool.

---
## Introduction

You can manage and interact with Ligato software components using the agentctl CLI command line tool. Agentctl supports the following functions:

- Manage VPP agent configurations. 
- Inspect and generate models.
- KV data store list, get, put and delete operations.
- Check status.
- Manage services.
- Configure logs.
- Gather stats.
- Perform system dumps.
- Extract runtime reports for troubleshooting.
- Configure event watch.  

![agentctl](../img/tools/agentctl.png)

---

## Installation

### Docker Image Pull

The official docker images for the VPP agent include agentctl. The [Quickstart Guide](quickstart.md) covers the steps from initial image pull to agentctl command execution. 

To install and run the VPP agent that includes agentctl, use the following commands for image pull, start etcd, start VPP agent, and agentctl help. 

Pull the VPP agent image from dockerhub:
```
docker pull ligato/vpp-agent
```
Start etcd:
```
docker run --rm --name etcd -p 2379:2379 -e ETCDCTL_API=3 quay.io/coreos/etcd /usr/local/bin/etcd -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379
```
Start VPP Agent:
```
docker run -it --rm --name vpp-agent -p 5002:5002 -p 9191:9191 --privileged ligato/vpp-agent
``` 
Agentctl help:
```
docker exec -it vpp-agent agentctl --help
```
---

### Build from Source

You can build a local image from the [VPP agent repository][ligato-vpp-agent-repo]. Follow the [Local Image Build](get-vpp-agent.md#local-image-build) instructions in [VPP Agent Setup](get-vpp-agent.md).

You can run agentctl commands after you start the VPP agent. 

---

## Setup

Run agentctl:
```
agentctl
```
Output:
```json

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
!!! note
    Bears worth repeating: Use `agentctl <ANY COMMAND> --help` for explanations and examples for any agentctl commands and subcommands. 

---

### NB Access Methods

Agentctl uses various NB access methods provided by the Ligato agents. You can modify the default values as needed.

- Agent host instance runs on `127.0.0.1` by default.
    * Use option `-H`/`--host` or env var `AGENT_HOST` to update.
    * Use this option for gRPC and REST access.
<br></br>
- gRPC runs on port `9111` by default.
    * Use option `--grpc-port` to update.
<br></br>
- REST runs on port `9191` by default.
    * use option `--http-port` to update.
<br></br>
- etcd server reachable on `127.0.0.1:2379 `by default. 
    * use option `-e`, `--etcd-endpoints`, or the `ETCD_ENDPOINTS` environment variable to update. For more details, see [kvdb](#kvdb) below.

---

## Commands

Agentctl subcommands:

- [config](#config)
- [dump](#dump)
- [generate](#generate)
- [import](#import)
- [kvdb](#kvdb)
- [log](#log)
- [metrics](#metrics)
- [model](#model)
- [report](#report)
- [service](#service)
- [status](#status)
- [values](#values)
- [vpp](#vpp)

---

### Config

Use `agentctl config` with the appropriate command, to manage VPP agent configuration data.

```
Manage agent configuration

COMMANDS
  delete      Delete config in agent
  get         Get config from agent
  history     Show config history
  resync      Run config resync
  retrieve    Retrieve currently running config
  update      Update config in agent
  watch       Watch events
```

---

#### config get

Get the VPP agent configuration.
```sh
agentctl config get
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

---

#### config history

Show the configuration history for all transactions, or by seqNum.
```json
agentctl config history
```
Sample output:
```golang
  SEQ  TYPE            START  INPUT      OPERATIONS  RESULT  SUMMARY
  0    config replace  26m    16 values  <none>              <none>
  1    config change   4m     1  values  CREATE:4    ok      CONFIGURED:4
  2    status update   4m     1  values  CREATE:1    ok      OBTAINED:1
  3    config change   3m     1  values  CREATE:2    ok      CONFIGURED:2
```

---

Config history for seqNum=3:
```
agentctl config history 3
```

Config history with more details:
```json
agentctl config history --details
```
Config history details for seqNum=3:
```json
agentctl config history 3 --details
```
Config history using transaction log format:
```json
agentctl config history -f log
```
Config history using transaction log format for seqNum=3:
```json
agentctl config history -f log 3
```


---

#### config [resync](../developer-guide/kvscheduler.md#resync)

Peform downstream resync.
```
agentctl config resync
```
Sample output:
```json
{
  "Start": "2020-04-01T23:28:21.3068686Z",
  "Stop": "2020-04-01T23:28:21.5081483Z",
  "SeqNum": 2,
  "TxnType": "NBTransaction",
  "ResyncType": "DownstreamResync"
  // one or more planned and executed operations follow,
}
```
For a detailed example of resync output, see [downstream resync API](../api/api-kvs.md#downstream-resync).

---

#### config retrieve 

Retrieves the runtime VPP configuration.

```sh
agentctl config retrieve
```
Sample output:
```json
dump:
  vppConfig:
    interfaces:
    - name: UNTAGGED-local0
      type: SOFTWARE_LOOPBACK
      physAddress: 00:00:00:00:00:00
    - name: loop1
      type: SOFTWARE_LOOPBACK
      enabled: true
      physAddress: de:ad:00:00:00:00
      ipAddresses:
      - 192.168.1.1/24
    bridgeDomains:
    - name: bd1
      forward: true
      learn: true
      interfaces:
      - name: loop1
    routes:
    - type: DROP
      dstNetwork: ::/0
      nextHopAddr: ::
      weight: 1
    - dstNetwork: fe80::/10
      nextHopAddr: ::
      weight: 1
    - type: DROP
      dstNetwork: 0.0.0.0/0
      nextHopAddr: 0.0.0.0
      weight: 1
    - type: DROP
      dstNetwork: 240.0.0.0/4
      nextHopAddr: 0.0.0.0
      weight: 1
    - type: DROP
      dstNetwork: 224.0.0.0/4
      nextHopAddr: 0.0.0.0
      weight: 1
    - dstNetwork: 192.168.1.1/24
      nextHopAddr: 0.0.0.0
      outgoingInterface: loop1
      weight: 1
    - dstNetwork: 192.168.1.1/32
      nextHopAddr: 192.168.1.1
      outgoingInterface: loop1
      weight: 1
    - type: DROP
      dstNetwork: 0.0.0.0/32
      nextHopAddr: 0.0.0.0
      weight: 1
    - type: DROP
      dstNetwork: 255.255.255.255/32
      nextHopAddr: 0.0.0.0
      weight: 1
    - type: DROP
      dstNetwork: 192.168.1.255/32
      nextHopAddr: 0.0.0.0
      weight: 1
    - type: DROP
      dstNetwork: 192.168.1.0/32
      nextHopAddr: 0.0.0.0
      weight: 1
    nat44Global: {}
  linuxConfig: {}
  netallocConfig: {}
```

---

#### config update

Update a VPP agent configuration. You must define the configuration inside a file. 

```
Update configuration in agent from file

OPTIONS:
  -f, --format string      Format output
  -t, --timeout duration   Timeout for sending updated data (default 5m0s)
```

Update1.yaml file containing configuration:
```json
# update1.yaml
vppConfig:
  interfaces:
  - name: "afpacket1"
    type: AF_PACKET
    enabled: true
    ipAddresses:
    - 192.168.99.1/30
    afpacket:
      host_if_name: "veth-1"
linuxConfig:
  interfaces:
  - name: "veth1"
    type: VETH
    enabled: true
    ip_addresses:
    - 10.0.2.1/30
    host_if_name: "veth-1"
    veth:
      peer_if_name: "veth2"
  - name: "veth2"
    type: VETH
    enabled: true
    ip_addresses:
    - 10.0.3.1/30
    veth:
      peer_if_name: "veth1"
```

Update using the update1.yaml file with a timeout duration of 6m0s:
```json
agentctl config update --timeout=6m0s update1.yaml
```
You can replace an existing configuration with a new configuration using the `--replace` flag. Define your new configuration in a separate file.

Config update replaces the update1.yaml configuration file with an updateNew.yaml configuration file:
```
agentctl config update --replace ./updateNew.yaml
```


---

#### config delete

Delete a VPP agent configuration. You must define your configuration inside a file.

```
Usage:	agentctl config delete

Delete configuration in agent

OPTIONS:
  -f, --format string   Format output
  -v, --verbose         Show verbose output
      --waitdone        Waits until config update is done
```
Config delete using the update1.yaml file:
```
agentctl config delete update1.yaml
```
Config delete using the update1.yaml file with --waitdone flag:
```
agentctl config delete --waitdone update1.yaml
```

!!! note  
    The `--waitdone` flag tells the configurator to block an update or delete request until all transaction keys are non-pending.<br></br>To learn more about this feature, see [configurator proto](https://github.com/ligato/vpp-agent/blob/effbcadbb8d02cf89caaf8bcbfd6d134eae9361d/proto/ligato/configurator/configurator.proto#L26), [issue #1732](https://github.com/ligato/vpp-agent/issues/1732), [PR #1734](https://github.com/ligato/vpp-agent/pull/1734), and [PR #1745](https://github.com/ligato/vpp-agent/pull/1756).  

---

#### config watch

Watch events.

```
Usage:	agentctl config watch

Watch events

OPTIONS:
      --filter stringArray   Filter(s) for notifications (multiple filters are used
                             with AND operator). Value should be JSON data of
                             configurator.Notification.
  -f, --format string        Format output

```

Watch events from VPP interface name of `loop1`:
```
agentctl config watch --filter='{"vpp_notification":{"interface":{"state":{"name":"loop1"}}}}'
```

Watch VPP interface `UPDOWN` events:
```
agentctl config watch --filter='{"vpp_notification":{"interface":{"type":"UPDOWN"}}}'
``` 

Sample output for `loop1` state UP:
```

------------------
 NOTIFICATION #6
------------------
Source: VPP
Value: interface: {
  type: UPDOWN
  state: {
    name: "loop1"
    internal_name: "loop0"
    if_index: 1
    admin_status: UP
    oper_status: UP
    last_change: 1606782814
    phys_address: "de:ad:00:00:00:00"
    mtu: 9216
    statistics: {}
  }
}
```
Sample output for `loop1` state DOWN:

```

------------------
 NOTIFICATION #7
------------------
Source: VPP
Value: interface: {
  type: UPDOWN
  state: {
    name: "loop1"
    internal_name: "loop0"
    if_index: 1
    admin_status: DOWN
    oper_status: DOWN
    last_change: 1606782834
    phys_address: "de:ad:00:00:00:00"
    mtu: 9216
    statistics: {}
  }
}

```

---

### Dump

Use this command to dump the KV Scheduler running state.

```
Usage:	agentctl dump MODEL

Dump running state

EXAMPLES

 To dump all data:
  $ agentctl dump all

 To dump all VPP data in json format run:
  $ agentctl dump -f json vpp.*

 To use different dump view use --view flag:
  $ agentctl dump --view=NB vpp.interfaces

OPTIONS:
  -f, --format string   Format output
      --view string     Dump view type: cached, NB, SB (default "cached")
```


Dump all:
```
agentctl dump all
```

Sample output:
```
+----------------------+---------+-----------------------------------+----------------------+-----------------------------------------+
|        MODEL         | ORIGIN  |               VALUE               |       METADATA       |                   KEY                   |
+----------------------+---------+-----------------------------------+----------------------+-----------------------------------------+
| vpp.ipfix.ipfix      | from-SB | # ligato.vpp.ipfix.IPFIX          |                      | config/vpp/ipfix/v2/ipfix               |
|                      |         | collector:                        |                      |                                         |
|                      |         |   address: 0.0.0.0                |                      |                                         |
|                      |         | sourceAddress: 0.0.0.0            |                      |                                         |
|                      |         | vrfId: 4294967295                 |                      |                                         |
|                      |         |                                   |                      |                                         |
+----------------------+         +-----------------------------------+----------------------+-----------------------------------------+
| vpp.nat.nat44-global |         | # ligato.vpp.nat.Nat44Global      |                      | config/vpp/nat/v2/nat44-global          |
|                      |         | {}                                |                      |                                         |
|                      |         |                                   |                      |                                         |
+----------------------+         +-----------------------------------+----------------------+-----------------------------------------+
| vpp.interfaces       |         | # ligato.vpp.interfaces.Interface | DevType: local       | UNTAGGED-local0                         |
|                      |         | name: UNTAGGED-local0             | IPAddresses: null    |                                         |
|                      |         | type: SOFTWARE_LOOPBACK           | InternalName: local0 |                                         |
|                      |         | physAddress: 00:00:00:00:00:00    | SwIfIndex: 0         |                                         |
|                      |         |                                   | TAPHostIfName: ""    |                                         |
|                      |         |                                   | Vrf: 0               |                                         |
|                      |         |                                   |                      |                                         |
+                      +---------+-----------------------------------+----------------------+-----------------------------------------+
|                      | from-NB | # ligato.vpp.interfaces.Interface | DevType: ""          | loop1                                   |
|                      |         | name: loop1                       | IPAddresses:         |                                         |
|                      |         | type: SOFTWARE_LOOPBACK           | - 192.168.1.1/24     |                                         |
|                      |         | enabled: true                     | InternalName: ""     |                                         |
|                      |         | ipAddresses:                      | SwIfIndex: 1         |                                         |
|                      |         | - 192.168.1.1/24                  | TAPHostIfName: ""    |                                         |
|                      |         |                                   | Vrf: 0               |                                         |
|                      |         |                                   |                      |                                         |
+----------------------+---------+-----------------------------------+----------------------+-----------------------------------------+
| vpp.proxyarp-global  | from-SB | # ligato.vpp.l3.ProxyARP          |                      | config/vpp/v2/proxyarp-global           |
|                      |         | {}                                |                      |                                         |
|                      |         |                                   |                      |                                         |
+----------------------+         +-----------------------------------+----------------------+-----------------------------------------+
| vpp.route            |         | # ligato.vpp.l3.Route             |                      | vrf/0/dst/0.0.0.0/0/gw/0.0.0.0          |
|                      |         | type: DROP                        |                      |                                         |
|                      |         | dstNetwork: 0.0.0.0/0             |                      |                                         |
|                      |         | nextHopAddr: 0.0.0.0              |                      |                                         |
|                      |         | weight: 1                         |                      |                                         |
|                      |         |                                   |                      |                                         |
+                      +         +-----------------------------------+----------------------+-----------------------------------------+
|                      |         | # ligato.vpp.l3.Route             |                      | vrf/0/dst/0.0.0.0/32/gw/0.0.0.0         |
|                      |         | type: DROP                        |                      |                                         |
|                      |         | dstNetwork: 0.0.0.0/32            |                      |                                         |
|                      |         | nextHopAddr: 0.0.0.0              |                      |                                         |
|                      |         | weight: 1                         |                      |                                         |
|                      |         |                                   |                      |                                         |
+                      +         +-----------------------------------+----------------------+-----------------------------------------+
|                      |         | # ligato.vpp.l3.Route             |                      | vrf/0/dst/224.0.0.0/4/gw/0.0.0.0        |
|                      |         | type: DROP                        |                      |                                         |
|                      |         | dstNetwork: 224.0.0.0/4           |                      |                                         |
|                      |         | nextHopAddr: 0.0.0.0              |                      |                                         |
|                      |         | weight: 1                         |                      |                                         |
|                      |         |                                   |                      |                                         |
+                      +         +-----------------------------------+----------------------+-----------------------------------------+
|                      |         | # ligato.vpp.l3.Route             |                      | vrf/0/dst/240.0.0.0/4/gw/0.0.0.0        |
|                      |         | type: DROP                        |                      |                                         |
|                      |         | dstNetwork: 240.0.0.0/4           |                      |                                         |
|                      |         | nextHopAddr: 0.0.0.0              |                      |                                         |
|                      |         | weight: 1                         |                      |                                         |
|                      |         |                                   |                      |                                         |
+                      +         +-----------------------------------+----------------------+-----------------------------------------+
|                      |         | # ligato.vpp.l3.Route             |                      | vrf/0/dst/255.255.255.255/32/gw/0.0.0.0 |
|                      |         | type: DROP                        |                      |                                         |
|                      |         | dstNetwork: 255.255.255.255/32    |                      |                                         |
|                      |         | nextHopAddr: 0.0.0.0              |                      |                                         |
|                      |         | weight: 1                         |                      |                                         |
|                      |         |                                   |                      |                                         |
+                      +         +-----------------------------------+----------------------+-----------------------------------------+
|                      |         | # ligato.vpp.l3.Route             |                      | vrf/0/dst/::/0/gw/::                    |
|                      |         | type: DROP                        |                      |                                         |
|                      |         | dstNetwork: ::/0                  |                      |                                         |
|                      |         | nextHopAddr: ::                   |                      |                                         |
|                      |         | weight: 1                         |                      |                                         |
|                      |         |                                   |                      |                                         |
+                      +         +-----------------------------------+----------------------+-----------------------------------------+
|                      |         | # ligato.vpp.l3.Route             |                      | vrf/0/dst/fe80::/10/gw/::               |
|                      |         | dstNetwork: fe80::/10             |                      |                                         |
|                      |         | nextHopAddr: ::                   |                      |                                         |
|                      |         | weight: 1                         |                      |                                         |
|                      |         |                                   |                      |                                         |
+----------------------+         +-----------------------------------+----------------------+-----------------------------------------+
| vpp.vrf-table        |         | # ligato.vpp.l3.VrfTable          | Index: 0             | id/0/protocol/IPV4                      |
|                      |         | label: ipv4-VRF:0                 | Protocol: 0          |                                         |
|                      |         |                                   |                      |                                         |
+                      +         +-----------------------------------+----------------------+-----------------------------------------+
|                      |         | # ligato.vpp.l3.VrfTable          | Index: 0             | id/0/protocol/IPV6                      |
|                      |         | protocol: IPV6                    | Protocol: 1          |                                         |
|                      |         | label: ipv6-VRF:0                 |                      |                                         |
|                      |         |                                   |                      |                                         |
+----------------------+---------+-----------------------------------+----------------------+-----------------------------------------+


```

---

Dump the `vpp.interfaces` model:
```
agentctl dump vpp.interfaces
```
Sample output:
```
+----------------+---------+-----------------------------------+----------------------+-----------------+
|     MODEL      | ORIGIN  |               VALUE               |       METADATA       |       KEY       |
+----------------+---------+-----------------------------------+----------------------+-----------------+
| vpp.interfaces | from-SB | # ligato.vpp.interfaces.Interface | DevType: local       | UNTAGGED-local0 |
|                |         | name: UNTAGGED-local0             | IPAddresses: null    |                 |
|                |         | type: SOFTWARE_LOOPBACK           | InternalName: local0 |                 |
|                |         | physAddress: 00:00:00:00:00:00    | SwIfIndex: 0         |                 |
|                |         |                                   | TAPHostIfName: ""    |                 |
|                |         |                                   | Vrf: 0               |                 |
|                |         |                                   |                      |                 |
+                +---------+-----------------------------------+----------------------+-----------------+
|                | from-NB | # ligato.vpp.interfaces.Interface | DevType: Loopback    | loop1           |
|                |         | name: loop1                       | IPAddresses:         |                 |
|                |         | type: SOFTWARE_LOOPBACK           | - 192.168.1.1/24     |                 |
|                |         | enabled: true                     | InternalName: loop0  |                 |
|                |         | ipAddresses:                      | SwIfIndex: 1         |                 |
|                |         | - 192.168.1.1/24                  | TAPHostIfName: ""    |                 |
|                |         |                                   | Vrf: 0               |                 |
|                |         |                                   |                      |                 |
+----------------+---------+-----------------------------------+----------------------+-----------------+
```

---

Dump the `vpp.interfaces` model using a NB view:
```
agentctl dump --view=NB vpp.interfaces
```
Sample output:
```
+----------------+---------+-----------------------------------+---------------------+-------+
|     MODEL      | ORIGIN  |               VALUE               |      METADATA       |  KEY  |
+----------------+---------+-----------------------------------+---------------------+-------+
| vpp.interfaces | from-NB | # ligato.vpp.interfaces.Interface | DevType: Loopback   | loop1 |
|                |         | name: loop1                       | IPAddresses:        |       |
|                |         | type: SOFTWARE_LOOPBACK           | - 192.168.1.1/24    |       |
|                |         | enabled: true                     | InternalName: loop0 |       |
|                |         | ipAddresses:                      | SwIfIndex: 1        |       |
|                |         | - 192.168.1.1/24                  | TAPHostIfName: ""   |       |
|                |         |                                   | Vrf: 0              |       |
|                |         |                                   |                     |       |
+----------------+---------+-----------------------------------+---------------------+-------+
```

---

Dump the `vpp.interfaces` model using json format:

```
agentctl dump -f json vpp.interfaces
```
Sample output:
```json
[
  {
    "Key": "config/vpp/v2/interfaces/UNTAGGED-local0",
    "Value": {
      "name": "UNTAGGED-local0",
      "type": "SOFTWARE_LOOPBACK",
      "physAddress": "00:00:00:00:00:00"
    },
    "Metadata": {
      "IPAddresses": null,
      "SwIfIndex": 0,
      "TAPHostIfName": "",
      "Vrf": 0
    },
    "Origin": 2
  },
  {
    "Key": "config/vpp/v2/interfaces/loop1",
    "Value": {
      "name": "loop1",
      "type": "SOFTWARE_LOOPBACK",
      "enabled": true,
      "ipAddresses": [
        "192.168.1.1/24"
      ]
    },
    "Metadata": {
      "IPAddresses": [
        "192.168.1.1/24"
      ],
      "SwIfIndex": 1,
      "TAPHostIfName": "",
      "Vrf": 0
    },
    "Origin": 1
  }
]
```

---

### Model

Use this command to gather information about models.

```
Usage:	agentctl model [options] COMMAND

Manage known models

COMMANDS
  inspect     Display detailed information on one or more models
  ls          List models
```

List all supported models:
```
agentctl model ls
```
Sample output:
```
MODEL                       CLASS    PROTO MESSAGE                            KEY PREFIX
govppmux.stats              metrics  ligato.govppmux.Metrics                  metrics/govppmux/v0/stats
linux.interfaces.interface  config   ligato.linux.interfaces.Interface        config/linux/interfaces/v2/interface/
linux.iptables.rulechain    config   ligato.linux.iptables.RuleChain          config/linux/iptables/v2/rulechain/
linux.l3.arp                config   ligato.linux.l3.ARPEntry                 config/linux/l3/v2/arp/
linux.l3.route              config   ligato.linux.l3.Route                    config/linux/l3/v2/route/
netalloc.ip                 config   ligato.netalloc.IPAllocation             config/netalloc/v1/ip/
vpp.abfs.abf                config   ligato.vpp.abf.ABF                       config/vpp/abfs/v2/abf/
vpp.acls.acl                config   ligato.vpp.acl.ACL                       config/vpp/acls/v2/acl/
vpp.arp                     config   ligato.vpp.l3.ARPEntry                   config/vpp/v2/arp/
vpp.dhcp-proxy              config   ligato.vpp.l3.DHCPProxy                  config/vpp/v2/dhcp-proxy/
vpp.exception               config   ligato.vpp.punt.Exception                config/vpp/v2/exception/
vpp.interfaces              config   ligato.vpp.interfaces.Interface          config/vpp/v2/interfaces/
vpp.ipredirect              config   ligato.vpp.punt.IPRedirect               config/vpp/v2/ipredirect/
vpp.ipscanneigh-global      config   ligato.vpp.l3.IPScanNeighbor             config/vpp/v2/ipscanneigh-global
vpp.ipsec.sa                config   ligato.vpp.ipsec.SecurityAssociation     config/vpp/ipsec/v2/sa/
vpp.ipsec.spd               config   ligato.vpp.ipsec.SecurityPolicyDatabase  config/vpp/ipsec/v2/spd/
vpp.ipsec.tun-protect       config   ligato.vpp.ipsec.TunnelProtection        config/vpp/ipsec/v2/tun-protect/
vpp.l2.bridge-domain        config   ligato.vpp.l2.BridgeDomain               config/vpp/l2/v2/bridge-domain/
vpp.l2.fib                  config   ligato.vpp.l2.FIBEntry                   config/vpp/l2/v2/fib/
vpp.l2.xconnect             config   ligato.vpp.l2.XConnectPair               config/vpp/l2/v2/xconnect/
vpp.l3xconnect              config   ligato.vpp.l3.L3XConnect                 config/vpp/v2/l3xconnect/
vpp.nat.dnat44              config   ligato.vpp.nat.DNat44                    config/vpp/nat/v2/dnat44/
vpp.nat.nat44-global        config   ligato.vpp.nat.Nat44Global               config/vpp/nat/v2/nat44-global
vpp.nat.nat44-interface     config   ligato.vpp.nat.Nat44Interface            config/vpp/nat/v2/nat44-interface/
vpp.nat.nat44-pool          config   ligato.vpp.nat.Nat44AddressPool          config/vpp/nat/v2/nat44-pool/
vpp.proxyarp-global         config   ligato.vpp.l3.ProxyARP                   config/vpp/v2/proxyarp-global
vpp.route                   config   ligato.vpp.l3.Route                      config/vpp/v2/route/
vpp.span                    config   ligato.vpp.interfaces.Span               config/vpp/v2/span/
vpp.srv6.localsid           config   ligato.vpp.srv6.LocalSID                 config/vpp/srv6/v2/localsid/
vpp.srv6.policy             config   ligato.vpp.srv6.Policy                   config/vpp/srv6/v2/policy/
vpp.srv6.srv6-global        config   ligato.vpp.srv6.SRv6Global               config/vpp/srv6/v2/srv6-global
vpp.srv6.steering           config   ligato.vpp.srv6.Steering                 config/vpp/srv6/v2/steering/
vpp.stn.rule                config   ligato.vpp.stn.Rule                      config/vpp/stn/v2/rule/
vpp.tohost                  config   ligato.vpp.punt.ToHost                   config/vpp/v2/tohost/
vpp.vrf-table               config   ligato.vpp.l3.VrfTable                   config/vpp/v2/vrf-table/                   
```

---

Show details about a specific model using `vpp.interfaces` as an example:
```
agentctl model inspect vpp.interfaces
```
Sample output:
```
[
  {
    "Name": "vpp.interfaces",
    "Class": "config",
    "Module": "vpp",
    "Type": "interfaces",
    "Version": "v2",
    "KeyPrefix": "config/vpp/v2/interfaces/",
    "NameTemplate": "{{.Name}}",
    "ProtoName": "ligato.vpp.interfaces.Interface",
    "ProtoFile": "ligato/vpp/interfaces/interface.proto",
    "GoType": "*vpp_interfaces.Interface",
    "PkgPath": "go.ligato.io/vpp-agent/v3/proto/ligato/vpp/interfaces"
  }
]
```
---

### Report

Use this command to generate runtime system reports. The report is packaged as a zipfile containing multiple subreport files. 

Most of the subreport files contain output of individual CLI commands or REST API calls. In addition, the files contain error and problem source information. You will save time and effort, and reduce problem resolution time.

```
Usage:	agentctl report

Create report about running software stack (VPP-Agent, VPP, AgentCtl,...) to allow quicker resolving of problems. The report will be a zip file containing information grouped in multiple files

EXAMPLES

  # Default reporting (creates report file in current directory, whole reporting fails on subreport error)
  agentctl report

  # Reporting into custom directory ("/tmp")
  agentctl report -o /tmp

  # Reporting and ignoring errors from subreports (writing successful reports and
  # errors from failed subreports into zip report file)
  agentctl report -i

OPTIONS:
  -i, --ignore-errors             Ignore subreport errors and create report zip file with all successfully
                                  retrieved/processed information (the errors will be part of the report too)
  -o, --output-directory string   Output directory (as absolute path) where report zip file will be written. Default
                                  is current directory.
```  

Generate report and ignore subreport errors:
```
agentctl report -i
```
Sample output zipfile name:
```
agentctl-report--2020-11-17--01-28-13-403.zip
```
Sample output after unpacking the report zipfile:
```
-rw------- 1 root root     980 Nov 17 01:07  _failed-reports.txt
-rw------- 1 root root    3582 Nov 17 01:07  _report.txt
-rw------- 1 root root     259 Nov 17 01:07  agent-NB-config.yaml
-rw------- 1 root root    2238 Nov 17 01:07  agent-kvscheduler-NB-config-view.txt
-rw------- 1 root root   18818 Nov 17 01:07  agent-kvscheduler-SB-config-view.txt
-rw------- 1 root root   13866 Nov 17 01:07  agent-kvscheduler-cached-config-view.txt
-rw------- 1 root root     375 Nov 17 01:07  agent-status.txt
-rw------- 1 root root    2253 Nov 17 01:07  agent-transaction-history.txt
-rw------- 1 root root    1669 Nov 17 01:07  hardware.txt
-rw------- 1 root root    1039 Nov 17 01:07  software-versions.txt
-rw------- 1 root root     174 Nov 17 01:07  vpp-api-trace.txt
-rw------- 1 root root 1136716 Nov 17 01:07  vpp-event-log.txt
-rw------- 1 root root    8505 Nov 17 01:07  vpp-log.txt
-rw------- 1 root root     198 Nov 17 01:07  vpp-other-srv6.txt
-rw------- 1 root root    1329 Nov 17 01:07 'vpp-running-config(vpp-agent-SB-dump).yaml'
-rw------- 1 root root     331 Nov 17 01:07  vpp-startup-config.txt
-rw------- 1 root root  339360 Nov 17 01:07  vpp-statistics-errors.txt
-rw------- 1 root root    2958 Nov 17 01:07  vpp-statistics-interfaces.txt
```

The `_report.txt` describes each subreport file. For user convenience, errors appear in three places:

* `_failed-reports.txt` file lists all errors from all subreports.
<br></br>
* Console while running `agentctl report`.
<br></br>
* Subreport file contains location where retrieved information should appear when running.   

Example error contained in the `_failed-reports.txt` file:
```json
Retrieving agent kvscheduler NB configuration... failed due to:
dumping kvscheduler data failed due to:
Failed to get data for NB view and key prefix config/vpp/wg/v1/peer/ due to: Error response from daemon: [500] {
  "Error": "unknown key prefix"
}
```

Note this error appears in the `agent-kvscheduler-NB-config-view.txt` subreport.

---
### Status

Use this command to return VPP agent status, version, build and plugin information. 

```
Usage:	agentctl status

Retrieve agent status

OPTIONS:
  -f, --format string   Format output
```

Return status:
```
agentctl status
```
Sample output:
```
AGENT
    App name:    vpp-agent
    Version:     v3.2.0-alpha-22-ge9aa3556d

    State:       OK
    Started:     2020-07-03 15:10:51 +0000 UTC (22m36s ago)
    Last change: 22m30s
    Last update: 3s

    Go version:  go1.14.4
    OS/Arch:     linux/amd64

    Build Info:
        Git commit: e9aa3556defe818904670e3f5051246fdd11746d
        Git branch: HEAD
        User:       root
        Host:       06a7eb7fd825
        Built:      2020-07-03 13:01:58 +0000 UTC

PLUGINS
    VPPAgent: OK
    etcd: OK
    govpp: OK
    vpp-abfplugin: INIT
    vpp-aclplugin: INIT
    vpp-ifplugin: OK
    vpp-ipsec-plugin: INIT
    vpp-l2plugin: INIT
    vpp-l3plugin: INIT
    vpp-natplugin: INIT
    vpp-srplugin: INIT


```

---

### Values

Use this command to retrieve the key-value pairs and derived keys from the KV Scheduler.

```
Usage:	agentctl values [MODEL]

Retrieve values from scheduler

OPTIONS:
  -f, --format string   Format output

```
Retrieve values:
```
agentctl values
```

Sample output:
```
MODEL                  NAME                                                STATE        DETAILS   LAST OP   ERROR
vpp.l2.bridge-domain   bd1                                                 CONFIGURED             CREATE
                       vpp/bd/bd1/interface/loop1                          CONFIGURED             CREATE
vpp.nat.nat44-global                                                       obtained
vpp.interfaces         UNTAGGED-local0                                     obtained
vpp.interfaces         loop1                                               CONFIGURED             CREATE
                       vpp/interface/loop1/address/static/192.168.1.1/24   CONFIGURED             CREATE
                       vpp/interface/loop1/has-IP-address                  CONFIGURED             CREATE
                       vpp/interface/loop1/vrf/0/ip-version/v4             CONFIGURED             CREATE
vpp.proxyarp-global                                                        obtained
vpp.route              vrf/0/dst/0.0.0.0/0/gw/0.0.0.0                      obtained
vpp.route              vrf/0/dst/0.0.0.0/32/gw/0.0.0.0                     obtained
vpp.route              vrf/0/dst/224.0.0.0/4/gw/0.0.0.0                    obtained
vpp.route              vrf/0/dst/240.0.0.0/4/gw/0.0.0.0                    obtained
vpp.route              vrf/0/dst/255.255.255.255/32/gw/0.0.0.0             obtained
vpp.route              vrf/0/dst/::/0/gw/::                                obtained
vpp.route              vrf/0/dst/fe80::/10/gw/::                           obtained
vpp.vrf-table          id/0/protocol/IPV4                                  obtained
vpp.vrf-table          id/0/protocol/IPV6                                  obtained
                       linux/interface/host-name/eth0                      obtained
                       linux/interface/host-name/lo                        obtained
                       vpp/interface/UNTAGGED-local0/link-state/DOWN       obtained
                       vpp/interface/loop1/link-state/UP                   obtained
```

---

### KVDB

Use this command to perform KV data store get, put, del, or list operations. This looks and functions much like `etcdctl`. It supports short form [keys][keys], but you must use the `--service-label` flag in the specific command operation.

```
Usage:	agentctl kvdb [options] COMMAND

Manage agent data in KVDB

ALIASES
  kvdb, kv

COMMANDS
  del         Delete key-value entry
  get         Get key-value entry
  list        List key-value entries
  put         Put key-value entry
```
!!! Note
    You might receive an `ERROR: connecting to KVDB failed: connecting to Etcd failed` message when using `agentctl kvdb` commands. This is because the VPP agent and etcd server start in separate containers. Agentctl uses the `127.0.0.1:2379` default address to reach the etcd server. The etcd server starts with a default `172.17.0.2:2379` address.<br></br>To reach the etcd server container from the agentctl container, pass the etcd server address to agentctl using the `-e` or `--etcd-endpoints` flags.<br></br>Example:  `agentctl -e 172.17.0.2:2379 kvdb list`.

---

kvdb list:
```json
agentctl kvdb list
```
Sample output:
```json
/vnf-agent/vpp1/check/status/v1/agent
{"build_version":"v3.2.0-alpha-1-g615f9fd36","build_date":"Wed Mar 18 17:59:27 UTC 2020","state":"OK","start_time":"1586275960","last_change":"1586275967","last_update":"1586280942","commit_hash":"615f9fd","plugins":[{"name":"govpp","state":"OK"},{"name":"VPPAgent","state":"OK"},{"name":"etcd","state":"OK"},{"name":"vpp-ifplugin","state":"OK"}]}
/vnf-agent/vpp1/check/status/v1/plugin/VPPAgent
{"state":"OK","last_change":"1586275962","last_update":"1586280942"}
...
```
Depending on the number of entries in your KV data store, you could find the returned output large and difficult to read. You can whittle this down by using a more specific key.

---

**list** only the configured interfaces:
```json
agentctl kvdb list /vnf-agent/vpp1/config/vpp/v2/interfaces/ 
``` 
Sample output:
```json
/vnf-agent/vpp1/config/vpp/v2/interfaces/loop1
{"name":"loop1","type":"SOFTWARE_LOOPBACK","enabled":true,"ip_addresses":["192.168.1.1/24"]}
/vnf-agent/vpp1/config/vpp/v2/interfaces/tap
{“name”:”tap1”,”type”:”TAP”,”enabled”:true,”ip_addresses”:[“192.168.1.1/24”]}
```

---

**put** a `loop1` loopback interface:
```json
agentctl kvdb put /vnf-agent/vpp1/config/vpp/v2/interfaces/loop1 '{"name":"loop1","type":"SOFTWARE_LOOPBACK","enabled":true,"ip_addresses":["192.168.1.1/24"]}'
```

---

**get** `loop1` loopback interface using the long form key:
```
agentctl kvdb get /vnf-agent/vpp1/config/vpp/v2/interfaces/loop1
```
Sample output:
```json
{"name":"loop1","type":"SOFTWARE_LOOPBACK","enabled":true,"ip_addresses":["192.168.1.1/24"]}
```
---

**delete** `loop1` loopback interface using the long form key:
```
agentctl kvdb del /vnf-agent/vpp1/config/vpp/v2/interfaces/loop1
```

You will generate a `key not found` message if you attempt to **get** a deleted entry. 



---

### VPP

Use this command to manage the connected VPP instance. You will save time by avoiding the need to "jump to" the VPP CLI. 

```
Usage:	agentctl vpp [options] COMMAND

Manage VPP instance

COMMANDS
  cli         Execute VPP CLI command
  info        Retrieve info about VPP
```
Use the `cli` option to run the VPP `show version` command:
```
agentctl vpp cli show version
```
Sample output:
```
vpp# show version
vpp v20.01-rc2~11-gfce396738~b17 built by root on b81dced13911 at 2020-01-29T21:07:15
```

---

Use the `info` option to show VPP information:
```
agentctl vpp info
```
Sample output:
```
VERSION:
Version:                  v20.01-rc2~11-gfce396738~b17
Compiled by:              root
Compile host:             b81dced13911
Compile date:             2020-01-29T21:07:15
Compile location:         /w/workspace/vpp-merge-2001-ubuntu1804
Compiler:                 GCC 8.3.0
Current PID:              15

CONFIG:
Command line arguments:
  /usr/bin/vpp
  unix
    {
    nodaemon
    cli-listen
    0.0.0.0:5002
    cli-no-pager
    }
  plugins
    {
    plugin
    dpdk_plugin.so
      {
      disable
      }
    }
  socksvr
    {
    default
    }
  statseg
    {
    default
    per-node-counters
    on
    }
```

---

### Import

Use this command to import configuration data from an external file using etcd or gRPC.  



```
Usage:	agentctl import file

Import config data from file

EXAMPLES

 To import file contents into Etcd, run:
  $ cat input.txt
  config/vpp/v2/interfaces/loop1 {"name":"loop1","type":"SOFTWARE_LOOPBACK"}
  config/vpp/l2/v2/bridge-domain/bd1 {"name":"bd1"}

  $ agentctl import input.txt

 To import it via gRPC, include --grpc flag:
  $ agentctl import --grpc=localhost:9111 input.txt

 FILE FORMAT
    Contents of the import file must contain single key-value pair per line:

    <key1> <value1>
    <key2> <value2>
    ...
    <keyN> <valueN>

    Empty lines and lines starting with '#' are ignored.

 KEY FORMAT
    Keys can be defined in two ways:

    - full: 	/vnf-agent/vpp1/config/vpp/v2/interfaces/iface1
    - short:	config/vpp/v2/interfaces/iface1

    For short keys, the import command uses microservice label defined with --service-label.

OPTIONS:
      --grpc         Enable to import config via gRPC
  -t, --time uint    Timeout (in seconds) to wait for server response (default 30)
      --txops uint   Number of ops per transaction (default 128)

```

---

Use short form keys with the microservice label. The `myconfig` file contains the following:
```
config/vpp/v2/interfaces/loop1 {"name":"loop1","type":"SOFTWARE_LOOPBACK"}
config/vpp/v2/interfaces/loop2 {"name":"loop2","type":"MEMIF"}
config/vpp/l2/v2/bridge-domain/bd1 {"name":"bd1","interfaces":[{"name":"loop1"},{"name":"loop2"}]}
```

Import the `myconfig` file contents using the `agent1` service-label:
```
agentctl --service-label=agent1 import ./myconfig
```
Output:
```
importing 3 key vals
 - /vnf-agent/agent1/config/vpp/v2/interfaces/loop1
 - /vnf-agent/agent1/config/vpp/v2/interfaces/loop2
 - /vnf-agent/agent1/config/vpp/l2/v2/bridge-domain/bd1
commiting tx with 3 ops
```

---

### Generate

Use this command to generate a model configuration sample.


```
Usage:	agentctl generate MODEL

Generate config samples

ALIASES
  generate, gen

OPTIONS:
  -f, --format string   Output formats: json, yaml (default "json")
      --oneline         Print output as single line (only json format)
```

---

Generate a configuration sample for a VPP route:

```
agentctl generate vpp.route
```

Sample output:
```
{
  "type": "INTRA_VRF",
  "vrf_id": 0,
  "dst_network": "",
  "next_hop_addr": "",
  "outgoing_interface": "",
  "weight": 0,
  "preference": 0,
  "via_vrf_id": 0
}
```

---

### Log


Use this command to manage the log levels for all loggers in the system.

```
Usage:	agentctl log [options] COMMAND

Manage agent logging

COMMANDS
  list        List agent loggers
  set         Set agent logger level

```

List current loggers and their log levels:
```
agentctl log list
```

Set KV Scheduler log level to `debug`:
```
agentctl log set kvscheduler debug
```
Output:
```
logger kvscheduler has been set to level debug
```

For more details on logging, see [how to setup logging](../developer-guide/kvs-troubleshooting.md#how-to-set-up-logging).

---

### Service

Use this command to manage services.

```
Usage:	agentctl service [options] COMMAND

Manage agent services

COMMANDS
  call        Call methods on services
  list        List remote services
```

---

List remote services including `methods` using the `-m` flag:
```
agentctl service list -m
```
Sample output:
```
service ligato.configurator.ConfiguratorService (ligato/configurator/configurator.proto)
 - rpc Get (GetRequest) returns (GetResponse)
 - rpc Update (UpdateRequest) returns (UpdateResponse)
 - rpc Delete (DeleteRequest) returns (DeleteResponse)
 - rpc Dump (DumpRequest) returns (DumpResponse)
 - rpc Notify (NotifyRequest) returns (stream NotifyResponse)

service ligato.configurator.StatsPollerService (ligato/configurator/statspoller.proto)
 - rpc PollStats (PollStatsRequest) returns (stream PollStatsResponse)

service ligato.generic.ManagerService (ligato/generic/manager.proto)
 - rpc SetConfig (SetConfigRequest) returns (SetConfigResponse)
 - rpc GetConfig (GetConfigRequest) returns (GetConfigResponse)
 - rpc DumpState (DumpStateRequest) returns (DumpStateResponse)
 - rpc Subscribe (SubscribeRequest) returns (stream SubscribeResponse)

service ligato.generic.MetaService (ligato/generic/meta.proto)
 - rpc KnownModels (KnownModelsRequest) returns (KnownModelsResponse)
```

---

Call service and method:
```
agentctl service call SERVICE METHOD
```

---

### Metrics

Use this command to collect and view runtime metrics.
```
Usage:	agentctl metrics [options] COMMAND

Get runtime metrics

COMMANDS
  get         Get metrics data
  list        List metrics
```

---

List metrics:
```
agentctl metrics list
```
Sample output:
```
METRIC          PROTO MESSAGE
govppmux.stats  ligato.govppmux.Metrics
```
---

Get metrics:
```
agentctl metrics get govppmux.stats
```
Sample output:
```json
{
  "channels_created": 35,
  "channels_open": 35,
  "request_replies": 41,
  "requests_done": 78,
  "requests_sent": 78
}
```

---

[ligato-vpp-agent-repo]: https://github.com/ligato/vpp-agent
[keys]: ../user-guide/concepts.md#keys