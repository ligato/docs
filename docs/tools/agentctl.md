# AgentCTL

---

## Intro

Agentctl is a command line client to manage not only vpp-agent, but any agent built with Ligato framework.

![agentctl](../img/tools/agentctl.png)

**Table of contents:**

- [Intro](#intro)
- [Usage](#usage)
- [Setup](#setup)
- [Subcommands](#subcommands)

## Usage

To use pre-built version of agentctl, you can use official docker image `ligato/vpp-agent:dev` (the `:latest` tag will be available in the next release 2.3.0).

```
➤ docker run --rm -it ligato/vpp-agent:dev agentctl
     ___                    __  ________  __
    /   | ____ ____  ____  / /_/ ____/ /_/ /
   / /| |/ __ '/ _ \/ __ \/ __/ /   / __/ / 
  / ___ / /_/ /  __/ / / / /_/ /___/ /_/ /  
 /_/  |_\__, /\___/_/ /_/\__/\____/\__/_/   
       /____/

Usage:
  agentctl [OPTIONS] COMMAND

Commands:
  dump        Dump running state
  generate    Generate config samples
  help        Help about any command
  import      Import config data from file
  kvdb        Manage agent data in KVDB
  log         Manage agent logging
  model       Manage known models
  status      Retrieve agent status
  vpp         Manage VPP instance

Options:
  -D, --debug                    Enable debug mode
  -e, --etcd-endpoints strings   Etcd endpoints to connect to, default from ETCD_ENDPOINTS env var (default [127.0.0.1:2379])
      --grpc-port string         gRPC server port (default "9111")
  -h, --help                     help for agentctl
  -H, --host string              Address on which agent is reachable, default from AGENT_HOST env var (default "127.0.0.1")
      --http-port string         HTTP server port (default "9191")
      --service-label string     Service label for specific agent instance, default from MICROSERVICE_LABEL env var
      --version                  version for agentctl

Run "agentctl COMMAND --help" for more information about a command.
```

The agentctl can also be built using make target `make agentctl`.

### Setup

The agentctl currently uses various NB access provided by Ligato agents:

- Agent host instance running on `127.0.0.1` by default
    * use option `-H`/`--host` or env var `AGENT_HOST` to change this
    * this is used for both GRPC and REST access
- GRPC running on port `9111` by default
    * use option `--grpc-port` to change this
- REST running on port `9191` by default
    * use option `--http-port` to change this
- ETCD datastore used by the agent on `127.0.0.1:2379 `by default 
    * use option `--etcd-endpoints` or env var `ETCD_ENDPOINTS` to change this

## Subcommands

List of agentctl subcommands:

- [model](#model)
- [dump](#dump)
- [status](#status)
- [kvdb](#kvdb)
- [vpp](#vpp)
- [import](#import)
- [generate](#generate)
- [log](#log)

### Model

<details>
<summary>The <code>model</code> subcommand provides information about models.</summary>
<p>

```
Manage known models

Usage:
  agentctl model [OPTIONS] COMMAND

Commands:
  inspect     Display detailed information on one or more models
  ls          List models

Options:
  -h, --help   help for model
```

</p>
</details>
</br>

To list all supported models, run:

```
➤ agentctl model list
MODEL                       KEY PREFIX                             PROTO NAME                        
linux.interfaces.interface  config/linux/interfaces/v2/interface/  linux.interfaces.Interface        
linux.l3.arp                config/linux/l3/v2/arp/                linux.l3.ARPEntry                 
linux.l3.route              config/linux/l3/v2/route/              linux.l3.Route                    
netalloc.ip                 config/netalloc/v1/ip/                 netalloc.IPAllocation             
vpp.abfs.abf                config/vpp/abfs/v2/abf/                vpp.abf.ABF                       
vpp.acls.acl                config/vpp/acls/v2/acl/                vpp.acl.ACL                       
vpp.arp                     config/vpp/v2/arp/                     vpp.l3.ARPEntry                   
vpp.dhcp-proxy              config/vpp/v2/dhcp-proxy/              vpp.l3.DHCPProxy                  
vpp.exception               config/vpp/v2/exception/               vpp.punt.Exception                
vpp.interfaces              config/vpp/v2/interfaces/              vpp.interfaces.Interface          
vpp.ipredirect              config/vpp/v2/ipredirect/              vpp.punt.IPRedirect               
vpp.ipscanneigh-global      config/vpp/v2/ipscanneigh-global       vpp.l3.IPScanNeighbor             
vpp.ipsec.sa                config/vpp/ipsec/v2/sa/                vpp.ipsec.SecurityAssociation     
vpp.ipsec.spd               config/vpp/ipsec/v2/spd/               vpp.ipsec.SecurityPolicyDatabase  
vpp.l2.bridge-domain        config/vpp/l2/v2/bridge-domain/        vpp.l2.BridgeDomain               
vpp.l2.fib                  config/vpp/l2/v2/fib/                  vpp.l2.FIBEntry                   
vpp.l2.xconnect             config/vpp/l2/v2/xconnect/             vpp.l2.XConnectPair               
vpp.nat.dnat44              config/vpp/nat/v2/dnat44/              vpp.nat.DNat44                    
vpp.nat.nat44-global        config/vpp/nat/v2/nat44-global         vpp.nat.Nat44Global               
vpp.proxyarp-global         config/vpp/v2/proxyarp-global          vpp.l3.ProxyARP                   
vpp.route                   config/vpp/v2/route/                   vpp.l3.Route                      
vpp.span                    config/vpp/v2/span/                    vpp.interfaces.Span               
vpp.srv6.localsid           config/vpp/srv6/v2/localsid/           vpp.srv6.LocalSID                 
vpp.srv6.policy             config/vpp/srv6/v2/policy/             vpp.srv6.Policy                   
vpp.srv6.steering           config/vpp/srv6/v2/steering/           vpp.srv6.Steering                 
vpp.stn.rule                config/vpp/stn/v2/rule/                vpp.stn.Rule                      
vpp.tohost                  config/vpp/v2/tohost/                  vpp.punt.ToHost                   
vpp.vrf-table               config/vpp/v2/vrf-table/               vpp.l3.VrfTable                   
```

To show details about specific model, run:

```
➤ agentctl model inspect vpp.interfaces
[
  {
    "Name": "vpp.interfaces",
    "Module": "vpp",
    "Type": "interfaces",
    "Version": "v2",
    "KeyPrefix": "config/vpp/v2/interfaces/",
    "ProtoName": "vpp.interfaces.Interface",
    "Location": "models/vpp/interfaces/interface.proto"
  }
]
```

### Dump

<details>
<summary>The <code>dump</code> subcommand dumps running state from the scheduler.</summary>
<p>

```
Dump running state

Usage:
  agentctl dump MODEL

Aliases:
  dump, d

Examples:

 To dump VPP interfaces run:
  $ agentctl dump vpp.interfaces

 To use different dump view use --view flag:
  $ agentctl dump --view=NB vpp.interfaces

 For a list of all supported models that can be dumped run:
  $ agentctl model list

 To specify the HTTP address of the agent use --host flag:
  $ agentctl --host 172.17.0.3 dump vpp.interfaces


Options:
  -h, --help          help for dump
  -v, --view string   Dump view type: cached, NB, SB (default "cached")
```

</p>
</details>
</br>

It uses model name to specify what to dump and provides 3 different view types (uses <code>cached</code> view by default).

To dump VPP interfaces, run:

```
➤ agentctl dump vpp.interfaces
KEY                                        VALUE                        ORIGIN    METADATA                                                  
config/vpp/v2/interfaces/UNTAGGED-local0   [vpp.interfaces.Interface]   from-SB   map[IPAddresses:<nil> SwIfIndex:0 TAPHostIfName: Vrf:0]   
                                           name: "UNTAGGED-local0"                                                                          
                                           type: SOFTWARE_LOOPBACK                                                                          
```

### Status

This command retrieves agent current status from the scheduler and lists all the key-values inside the agent, including derived keys.

```
➤ agentctl status
MODEL                    NAME                                            STATE        DETAILS            LAST OP   ERROR                                             
vpp.l2.bridge-domain     bd1                                             CONFIGURED                      UPDATE                                                      
                         vpp/bd/bd1/interface/loop1                      CONFIGURED                      UPDATE                                                      
                         vpp/bd/bd1/interface/loop2                      PENDING      interface-exists   CREATE                                                      
vpp.nat.nat44-global                                                     obtained                                                                                    
vpp.interfaces           UNTAGGED-local0                                 obtained                                                                                    
vpp.interfaces           loop1                                           CONFIGURED                      UPDATE                                                      
vpp.interfaces           loop2                                           INVALID                         CREATE    VPP interface type MEMIF must have link defined   
vpp.ipscanneigh-global                                                   obtained                                                                                    
vpp.proxyarp-global                                                      obtained                                                                                    
vpp.route                vrf/0/dst/0.0.0.0/0/gw/0.0.0.0                  obtained                                                                                    
vpp.route                vrf/0/dst/0.0.0.0/32/gw/0.0.0.0                 obtained                                                                                    
vpp.route                vrf/0/dst/224.0.0.0/4/gw/0.0.0.0                obtained                                                                                    
vpp.route                vrf/0/dst/240.0.0.0/4/gw/0.0.0.0                obtained                                                                                    
vpp.route                vrf/0/dst/255.255.255.255/32/gw/0.0.0.0         obtained                                                                                    
vpp.route                vrf/0/dst/::/0/gw/::                            obtained                                                                                    
vpp.route                vrf/0/dst/fe80::/10/gw/::                       obtained                                                                                    
vpp.vrf-table            id/0/protocol/IPV4                              obtained                                                                                    
vpp.vrf-table            id/0/protocol/IPV6                              obtained                                                                                    
                         linux/interface/host-name/eth0                  obtained                                                                                    
                         linux/interface/host-name/lo                    obtained                                                                                    
                         linux/microservice/agent1                       obtained                                                                                    
                         linux/microservice/myagent                      obtained                                                                                    
                         vpp/interface/UNTAGGED-local0/link-state/DOWN   obtained                                                                                    
                         vpp/interface/loop1/link-state/DOWN             obtained                                                                                    
```

### KVDB

<details>
<summary>The <code>kvdb</code> subcommand managed data stored in KV datastore.</summary>
<p>

```
Manage agent data in KVDB

Usage:
  agentctl kvdb [OPTIONS] COMMAND

Aliases:
  kvdb, kv

Commands:
  del         Delete key-value entry from KVDB
  get         Get key-value entry from KVDB
  list        List key-value entries from KVDB
  put         Put key-value entry into KVDB

Options:
  -h, --help   help for kvdb
```

</p>
</details>

The `kvdb` command is similar to using etcdctl, but supports short keys (which require setting `--service-label` option).

### VPP

<details><summary>The <code>vpp</code> subcommand manages the connected VPP instance.</summary>
<p>

```
Manage VPP instance

Usage:
  agentctl vpp [OPTIONS] COMMAND

Commands:
  cli         Execute VPP CLI command
  info        Retrieve info about VPP

Options:
  -h, --help   help for vpp
```

</p>
</details>
</br>

To run VPP CLI command, run:

```
➤ agentctl vpp cli show version
# show version
vpp v19.08-rc2~12-g1c586de48~b37 built by root on 9ce218d4b187 at Wed Aug 21 17:50:31 UTC 2019
```

### Import

<details>
<summary>The <code>import</code> subcommand can be used to import config data from file using GRPC or ETCD.</summary>
<p>

```
Import config data from file

Usage:
  agentctl import file [flags]

Examples:

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


Options:
      --grpc         Enable to import config via gRPC
  -h, --help         help for import
  -t, --time uint    Timeout (in seconds) to wait for server response (default 30)
      --txops uint   Number of ops per transaction (default 128)
```

</p>
</details>
</br>

The import command supports importing via GRPC which can be enabled with `--grpc` option.

To import config data from file, run:

```
➤ cat ./myconfig
config/vpp/v2/interfaces/loop1 {"name":"loop1","type":"SOFTWARE_LOOPBACK"}
config/vpp/v2/interfaces/loop2 {"name":"loop2","type":"MEMIF"}
config/vpp/l2/v2/bridge-domain/bd1 {"name":"bd1","interfaces":[{"name":"loop1"},{"name":"loop2"}]}

➤ agentctl --service-label=agent1 import ./myconfig
importing 3 key vals
 - /vnf-agent/agent1/config/vpp/v2/interfaces/loop1
 - /vnf-agent/agent1/config/vpp/v2/interfaces/loop2
 - /vnf-agent/agent1/config/vpp/l2/v2/bridge-domain/bd1
commiting tx with 3 ops
```

The import file supports full version of keys including agent namespace (microservice label) or short keys which require setting `--service-label` option.

### Generate

<details>
<summary>The <code>generate</code> subcommand can be used to generate config samples for models.</summary>
<p>

```
Generate config samples

Usage:
  agentctl generate MODEL

Aliases:
  generate, gen

Options:
  -f, --format string   Output formats: json, yaml (default "json")
  -h, --help            help for generate
      --oneline         Print output as single line (only json format)
```

</p>
</details>
</br>

To generate config sample for VPP route, run:

```
➤ agentctl generate vpp.route
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

### Log

<details>
<summary>The <code>log</code> subcommand manages the logging of the agent instance.</summary>
<p>

```
Manage agent logging

Usage:
  agentctl log [OPTIONS] COMMAND

Commands:
  list        Show vppagent logs
  set         Set vppagent logger type

Options:
  -h, --help   help for log
```

</p>
</details>
</br>

To list current loggers and their log levels, run `log list`.

To set log level for specific logger, run:

```
➤ agentctl log set kvscheduler debug
logger kvscheduler has been set to level debug
```
