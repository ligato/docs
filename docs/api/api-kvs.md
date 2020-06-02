# KV Scheduler 

This section describes the REST APIs exposed by the KV Scheduler. The URLs and sample responses (in some cases partial) were generated using the [Quickstart Guide][quickstart-guide] environment.

Reference: [KV Scheduler plugin rest.go file][kvs-rest-go]

!!! Note
    `9191` is the default port number for Ligato REST APIs, however it can be changed using one of the [REST plugin configuration options][rest-plugin-config-options]. Note that the [Agentctl][agentctl] CLI tool is another option for retrieving KV Scheduler system state.

---

## Dump

Description: GET list of the `descriptors` registered with the KV Scheduler, list of the `key prefixes` under watch in the NB direction, and `view` options from the perspective of the KV Scheduler 
```json
curl -X GET http://localhost:9191/scheduler/dump
```
Sample response:
```json
{
  "Descriptors": [
    "bond-interface",
    "dhcp-proxy",
    "linux-interface-address",
    "linux-interface-watcher",
    "microservice",
    "netalloc-ip-address",
    "linux-interface",
    "linux-arp",
    "linux-ipt-rulechain-descriptor",
    "linux-route",
    "vpp-acl-to-interface",
    "vpp-bd-interface",
    "vpp-interface",
    "vpp-acl",
    "vpp-arp",
    "vpp-bridge-domain",
    "vpp-dhcp",
    "vpp-interface-address",
    "vpp-interface-has-address",
    "vpp-interface-link-state",
    "vpp-interface-rx-mode",
    "vpp-interface-rx-placement",
    "vpp-interface-vrf",
    "vpp-ip-scan-neighbor",
    "vpp-l2-fib",
    "vpp-l3xc",
    "vpp-nat44-dnat",
    "vpp-nat44-global",
    "vpp-nat44-address-pool",
    "vpp-nat44-global-address",
    "vpp-nat44-global-interface",
    "vpp-nat44-interface",
    "vpp-proxy-arp",
    "vpp-proxy-arp-interface",
    "vpp-punt-exception",
    "vpp-punt-ipredirect",
    "vpp-punt-to-host",
    "vpp-span",
    "vpp-sr-localsid",
    "vpp-sr-policy",
    "vpp-sr-steering",
    "vpp-srv6-global",
    "vpp-stn-rules",
    "vpp-unnumbered-interface",
    "vpp-vrf-table",
    "vpp-route",
    "vpp-xconnect"
  ],
  "KeyPrefixes": [
    "",
    "config/vpp/v2/dhcp-proxy/",
    "",
    "",
    "",
    "config/netalloc/v1/ip/",
    "config/linux/interfaces/v2/interface/",
    "config/linux/l3/v2/arp/",
    "config/linux/iptables/v2/rulechain/",
    "config/linux/l3/v2/route/",
    "",
    "",
    "config/vpp/v2/interfaces/",
    "config/vpp/acls/v2/acl/",
    "config/vpp/v2/arp/",
    "config/vpp/l2/v2/bridge-domain/",
    "",
    "",
    "",
    "",
    "",
    "",
    "",
    "config/vpp/v2/ipscanneigh-global",
    "config/vpp/l2/v2/fib/",
    "config/vpp/v2/l3xconnect/",
    "config/vpp/nat/v2/dnat44/",
    "config/vpp/nat/v2/nat44-global",
    "config/vpp/nat/v2/nat44-pool/",
    "",
    "",
    "config/vpp/nat/v2/nat44-interface/",
    "config/vpp/v2/proxyarp-global",
    "",
    "config/vpp/v2/exception/",
    "config/vpp/v2/ipredirect/",
    "config/vpp/v2/tohost/",
    "config/vpp/v2/span/",
    "config/vpp/srv6/v2/localsid/",
    "config/vpp/srv6/v2/policy/",
    "config/vpp/srv6/v2/steering/",
    "config/vpp/srv6/v2/srv6-global",
    "config/vpp/stn/v2/rule/",
    "",
    "config/vpp/v2/vrf-table/",
    "config/vpp/v2/route/",
    "config/vpp/l2/v2/xconnect/"
  ],
  "Views": ["SB", "NB", "cached"]
}
```

---

## Dump View&Key-Prefix

Description: GET key-value data by view for a specific key prefix. The parameters are: 

- view: `SB` where descriptor retrieve methods are employed to learn the actual, up-to-date state of the system; `NB` examines the key-value space to determine what key-values were requested and assumed by the NB to be applied; `cached` obtains the KV Scheduler's current SB view.
- key-prefix: Key prefix such as `config/vpp/v2/interfaces/` 

This example dumps the system state in the SB direction for the key prefix of `config/vpp/v2/interfaces/`: 
```json
curl -X GET "http://localhost:9191/scheduler/dump?view=SB&key-prefix=config/vpp/v2/interfaces/"
```
Sample response: 
```json
[
    {
        "Key": "config/vpp/v2/interfaces/mem01",
        "Value": {
            "ProtoMsgName": "ligato.vpp.interfaces.Interface",
            "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 memif:<id:2 socket_filename:\"/run/vpp/memif.sock\" > "
        },
        "Metadata": {
            "SwIfIndex": 2,
            "Vrf": 0,
            "IPAddresses": [
                "100.100.100.102/24"
            ],
            "TAPHostIfName": ""
        },
        "Origin": 1
    },
    {
        "Key": "config/vpp/v2/interfaces/UNTAGGED-local0",
        "Value": {
            "ProtoMsgName": "ligato.vpp.interfaces.Interface",
            "ProtoMsgData": "name:\"UNTAGGED-local0\" type:SOFTWARE_LOOPBACK phys_address:\"00:00:00:00:00:00\" "
        },
        "Metadata": {
            "SwIfIndex": 0,
            "Vrf": 0,
            "IPAddresses": null,
            "TAPHostIfName": ""
        },
        "Origin": 2
    },
    {
        "Key": "config/vpp/v2/interfaces/loop1",
        "Value": {
            "ProtoMsgName": "ligato.vpp.interfaces.Interface",
            "ProtoMsgData": "name:\"loop1\" type:SOFTWARE_LOOPBACK enabled:true phys_address:\"de:ad:00:00:00:00\" ip_addresses:\"192.168.1.1/24\" "
        },
        "Metadata": {
            "SwIfIndex": 1,
            "Vrf": 0,
            "IPAddresses": [
                "192.168.1.1/24"
            ],
            "TAPHostIfName": ""
        },
        "Origin": 1
    }
]
```

---

## Transaction History

Description: GET a complete history of planned and executed transactions. In addition, the following parameters can be included to scope the response:

- seq-num: sequence number of the transaction
- since: start of the transaction history time window in [unix timestamp][unix-timestamp] format
- until: end of the transaction history time window in [unix timestamp][unix-timestamp] format

To GET the complete transaction history, use: 
```json
curl -X GET http://localhost:9191/scheduler/txn-history
```
Sample response:
```json
[
    {
        "Start": "2020-05-28T16:50:52.752025508Z",
        "Stop": "2020-05-28T16:50:52.769882769Z",
        "SeqNum": 0,
        "TxnType": "NBTransaction",
        "ResyncType": "FullResync",
        "Values": [
            {
                "Key": "config/vpp/ipfix/v2/ipfix",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.ipfix.IPFIX",
                    "ProtoMsgData": "collector:<address:\"0.0.0.0\" > source_address:\"0.0.0.0\" vrf_id:4294967295 "
                },
                "Origin": 2
            },
            {
                "Key": "config/vpp/nat/v2/nat44-global",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.nat.Nat44Global",
                    "ProtoMsgData": ""
                },
                "Origin": 2
            },
            {
                "Key": "config/vpp/v2/interfaces/UNTAGGED-local0",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"UNTAGGED-local0\" type:SOFTWARE_LOOPBACK phys_address:\"00:00:00:00:00:00\" "
                },
                "Origin": 2
            },
            {
                "Key": "config/vpp/v2/proxyarp-global",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.l3.ProxyARP",
                    "ProtoMsgData": ""
                },
                "Origin": 2
            },
            {
                "Key": "config/vpp/v2/route/vrf/0/dst/0.0.0.0/0/gw/0.0.0.0",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.l3.Route",
                    "ProtoMsgData": "type:DROP dst_network:\"0.0.0.0/0\" next_hop_addr:\"0.0.0.0\" weight:1 "
                },
                "Origin": 2
            },
            {
                "Key": "config/vpp/v2/route/vrf/0/dst/0.0.0.0/32/gw/0.0.0.0",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.l3.Route",
                    "ProtoMsgData": "type:DROP dst_network:\"0.0.0.0/32\" next_hop_addr:\"0.0.0.0\" weight:1 "
                },
                "Origin": 2
            },
            {
                "Key": "config/vpp/v2/route/vrf/0/dst/224.0.0.0/4/gw/0.0.0.0",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.l3.Route",
                    "ProtoMsgData": "type:DROP dst_network:\"224.0.0.0/4\" next_hop_addr:\"0.0.0.0\" weight:1 "
                },
                "Origin": 2
            },
            {
                "Key": "config/vpp/v2/route/vrf/0/dst/240.0.0.0/4/gw/0.0.0.0",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.l3.Route",
                    "ProtoMsgData": "type:DROP dst_network:\"240.0.0.0/4\" next_hop_addr:\"0.0.0.0\" weight:1 "
                },
                "Origin": 2
            },
            {
                "Key": "config/vpp/v2/route/vrf/0/dst/255.255.255.255/32/gw/0.0.0.0",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.l3.Route",
                    "ProtoMsgData": "type:DROP dst_network:\"255.255.255.255/32\" next_hop_addr:\"0.0.0.0\" weight:1 "
                },
                "Origin": 2
            },
            {
                "Key": "config/vpp/v2/route/vrf/0/dst/::/0/gw/::",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.l3.Route",
                    "ProtoMsgData": "type:DROP dst_network:\"::/0\" next_hop_addr:\"::\" weight:1 "
                },
                "Origin": 2
            },
            {
                "Key": "config/vpp/v2/route/vrf/0/dst/fe80::/10/gw/::",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.l3.Route",
                    "ProtoMsgData": "dst_network:\"fe80::/10\" next_hop_addr:\"::\" weight:1 "
                },
                "Origin": 2
            },
            {
                "Key": "config/vpp/v2/vrf-table/id/0/protocol/IPV4",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.l3.VrfTable",
                    "ProtoMsgData": "label:\"ipv4-VRF:0\" "
                },
                "Origin": 2
            },
            {
                "Key": "config/vpp/v2/vrf-table/id/0/protocol/IPV6",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.l3.VrfTable",
                    "ProtoMsgData": "protocol:IPV6 label:\"ipv6-VRF:0\" "
                },
                "Origin": 2
            },
            {
                "Key": "linux/interface/host-name/eth0",
                "Value": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "Origin": 2
            },
            {
                "Key": "linux/interface/host-name/lo",
                "Value": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "Origin": 2
            },
            {
                "Key": "vpp/interface/UNTAGGED-local0/link-state/DOWN",
                "Value": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "Origin": 2
            }
        ]
    },
    {
        "Start": "2020-05-28T17:05:48.16322751Z",
        "Stop": "2020-05-28T17:05:48.166554253Z",
        "SeqNum": 1,
        "TxnType": "NBTransaction",
        "Values": [
            {
                "Key": "config/vpp/v2/interfaces/loop1",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"loop1\" type:SOFTWARE_LOOPBACK enabled:true ip_addresses:\"192.168.1.1/24\" "
                },
                "Origin": 1
            }
        ],
        "Executed": [
            {
                "Operation": "CREATE",
                "Key": "config/vpp/v2/interfaces/loop1",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"loop1\" type:SOFTWARE_LOOPBACK enabled:true ip_addresses:\"192.168.1.1/24\" "
                }
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/loop1/vrf/0/ip-version/v4",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/loop1/address/static/192.168.1.1/24",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/loop1/has-IP-address",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true
            }
        ]
    },
    {
        "Start": "2020-05-28T17:05:48.166962782Z",
        "Stop": "2020-05-28T17:05:48.16711382Z",
        "SeqNum": 2,
        "TxnType": "SBNotification",
        "Values": [
            {
                "Key": "vpp/interface/loop1/link-state/UP",
                "Value": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "Origin": 2
            }
        ],
        "Executed": [
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/loop1/link-state/UP",
                "NewState": "OBTAINED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                }
            }
        ]
    },
    {
        "Start": "2020-05-28T18:53:19.211937573Z",
        "Stop": "2020-05-28T18:53:19.214134123Z",
        "SeqNum": 3,
        "TxnType": "NBTransaction",
        "Values": [
            {
                "Key": "config/vpp/v2/interfaces/mem01",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "Origin": 1
            }
        ],
        "Executed": [
            {
                "Operation": "CREATE",
                "Key": "config/vpp/v2/interfaces/mem01",
                "NewState": "RETRYING",
                "NewValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "NewErr": {}
            }
        ]
    },
    {
        "WithSimulation": true,
        "Start": "2020-05-28T18:53:20.230484533Z",
        "Stop": "2020-05-28T18:53:20.233056755Z",
        "SeqNum": 4,
        "TxnType": "RetryFailedOps",
        "RetryForTxn": 3,
        "RetryAttempt": 1,
        "Values": [
            {
                "Key": "config/vpp/v2/interfaces/mem01",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "Origin": 1
            }
        ],
        "Planned": [
            {
                "Operation": "CREATE",
                "Key": "config/vpp/v2/interfaces/mem01",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "PrevState": "RETRYING",
                "PrevValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "PrevErr": {},
                "IsRetry": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/rx-modes",
                "NewState": "PENDING",
                "NewValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF rx_modes:<mode:ADAPTIVE > "
                },
                "NOOP": true,
                "IsDerived": true,
                "IsRetry": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/vrf/0/ip-version/v4",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true,
                "IsRetry": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/address/static/100.100.100.102/24",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true,
                "IsRetry": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/has-IP-address",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true,
                "IsRetry": true
            }
        ],
        "Executed": [
            {
                "Operation": "CREATE",
                "Key": "config/vpp/v2/interfaces/mem01",
                "NewState": "RETRYING",
                "NewValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "NewErr": {},
                "PrevState": "RETRYING",
                "PrevValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "PrevErr": {},
                "IsRetry": true
            }
        ]
    },
    {
        "WithSimulation": true,
        "Start": "2020-05-28T18:53:22.246068802Z",
        "Stop": "2020-05-28T18:53:22.248180185Z",
        "SeqNum": 5,
        "TxnType": "RetryFailedOps",
        "RetryForTxn": 3,
        "RetryAttempt": 2,
        "Values": [
            {
                "Key": "config/vpp/v2/interfaces/mem01",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "Origin": 1
            }
        ],
        "Planned": [
            {
                "Operation": "CREATE",
                "Key": "config/vpp/v2/interfaces/mem01",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "PrevState": "RETRYING",
                "PrevValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "PrevErr": {},
                "IsRetry": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/rx-modes",
                "NewState": "PENDING",
                "NewValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF rx_modes:<mode:ADAPTIVE > "
                },
                "NOOP": true,
                "IsDerived": true,
                "IsRetry": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/vrf/0/ip-version/v4",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true,
                "IsRetry": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/address/static/100.100.100.102/24",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true,
                "IsRetry": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/has-IP-address",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true,
                "IsRetry": true
            }
        ],
        "Executed": [
            {
                "Operation": "CREATE",
                "Key": "config/vpp/v2/interfaces/mem01",
                "NewState": "RETRYING",
                "NewValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "NewErr": {},
                "PrevState": "RETRYING",
                "PrevValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "PrevErr": {},
                "IsRetry": true
            }
        ]
    },
    {
        "WithSimulation": true,
        "Start": "2020-05-28T18:53:26.267778173Z",
        "Stop": "2020-05-28T18:53:26.27082962Z",
        "SeqNum": 6,
        "TxnType": "RetryFailedOps",
        "RetryForTxn": 3,
        "RetryAttempt": 3,
        "Values": [
            {
                "Key": "config/vpp/v2/interfaces/mem01",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "Origin": 1
            }
        ],
        "Planned": [
            {
                "Operation": "CREATE",
                "Key": "config/vpp/v2/interfaces/mem01",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "PrevState": "RETRYING",
                "PrevValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "PrevErr": {},
                "IsRetry": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/rx-modes",
                "NewState": "PENDING",
                "NewValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF rx_modes:<mode:ADAPTIVE > "
                },
                "NOOP": true,
                "IsDerived": true,
                "IsRetry": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/vrf/0/ip-version/v4",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true,
                "IsRetry": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/address/static/100.100.100.102/24",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true,
                "IsRetry": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/has-IP-address",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true,
                "IsRetry": true
            }
        ],
        "Executed": [
            {
                "Operation": "CREATE",
                "Key": "config/vpp/v2/interfaces/mem01",
                "NewState": "FAILED",
                "NewValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "NewErr": {},
                "PrevState": "RETRYING",
                "PrevValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "PrevErr": {},
                "IsRetry": true
            }
        ]
    },
    {
        "Start": "2020-05-28T19:10:44.973288991Z",
        "Stop": "2020-05-28T19:10:44.979247779Z",
        "SeqNum": 7,
        "TxnType": "NBTransaction",
        "Values": [
            {
                "Key": "config/vpp/v2/interfaces/mem01",
                "Value": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 > "
                },
                "Origin": 1
            }
        ],
        "Executed": [
            {
                "Operation": "CREATE",
                "Key": "config/vpp/v2/interfaces/mem01",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 > "
                },
                "PrevState": "FAILED",
                "PrevValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 socket_filename:\"/var/run/contiv/memif_ts1-host4.sock\" > "
                },
                "PrevErr": {}
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/rx-modes",
                "NewState": "PENDING",
                "NewValue": {
                    "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                    "ProtoMsgData": "name:\"mem01\" type:MEMIF rx_modes:<mode:ADAPTIVE > "
                },
                "NOOP": true,
                "IsDerived": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/vrf/0/ip-version/v4",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/address/static/100.100.100.102/24",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true
            },
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/has-IP-address",
                "NewState": "CONFIGURED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "IsDerived": true
            }
        ]
    },
    {
        "Start": "2020-05-28T19:10:44.980035739Z",
        "Stop": "2020-05-28T19:10:44.980288903Z",
        "SeqNum": 8,
        "TxnType": "SBNotification",
        "Values": [
            {
                "Key": "vpp/interface/mem01/link-state/DOWN",
                "Value": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                },
                "Origin": 2
            }
        ],
        "Executed": [
            {
                "Operation": "CREATE",
                "Key": "vpp/interface/mem01/link-state/DOWN",
                "NewState": "OBTAINED",
                "NewValue": {
                    "ProtoMsgName": "google.protobuf.Empty",
                    "ProtoMsgData": ""
                }
            }
        ]
    }
]
```

---

To GET the transaction history for a sequence number = 1, use:
```json
curl -X GET http://localhost:9191/scheduler/txn-history?seq-num=1
```

Sample response:
```json
{
    "Start": "2020-05-28T17:05:48.16322751Z",
    "Stop": "2020-05-28T17:05:48.166554253Z",
    "SeqNum": 1,
    "TxnType": "NBTransaction",
    "Values": [
        {
            "Key": "config/vpp/v2/interfaces/loop1",
            "Value": {
                "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                "ProtoMsgData": "name:\"loop1\" type:SOFTWARE_LOOPBACK enabled:true ip_addresses:\"192.168.1.1/24\" "
            },
            "Origin": 1
        }
    ],
    "Executed": [
        {
            "Operation": "CREATE",
            "Key": "config/vpp/v2/interfaces/loop1",
            "NewState": "CONFIGURED",
            "NewValue": {
                "ProtoMsgName": "ligato.vpp.interfaces.Interface",
                "ProtoMsgData": "name:\"loop1\" type:SOFTWARE_LOOPBACK enabled:true ip_addresses:\"192.168.1.1/24\" "
            }
        },
        {
            "Operation": "CREATE",
            "Key": "vpp/interface/loop1/vrf/0/ip-version/v4",
            "NewState": "CONFIGURED",
            "NewValue": {
                "ProtoMsgName": "google.protobuf.Empty",
                "ProtoMsgData": ""
            },
            "IsDerived": true
        },
        {
            "Operation": "CREATE",
            "Key": "vpp/interface/loop1/address/static/192.168.1.1/24",
            "NewState": "CONFIGURED",
            "NewValue": {
                "ProtoMsgName": "google.protobuf.Empty",
                "ProtoMsgData": ""
            },
            "IsDerived": true
        },
        {
            "Operation": "CREATE",
            "Key": "vpp/interface/loop1/has-IP-address",
            "NewState": "CONFIGURED",
            "NewValue": {
                "ProtoMsgName": "google.protobuf.Empty",
                "ProtoMsgData": ""
            },
            "IsDerived": true
        }
    ]
}
```
To GET the transaction history for a window in time, where the `start time = 1591031902` and `end time = 1591031903`, use:

```json
curl -X GET "http://localhost:9191/scheduler/txn-history?since=1591031902&until=1591031903"
```

---

## Key Timeline

Description: GET the timeline of value changes for a `specific key`.

To GET the timeline of value changes for `key=config/vpp/v2/interfaces/loop1`, use:
```json
curl -X GET "http://localhost:9191/scheduler/key-timeline?key=config/vpp/v2/interfaces/loop1"
```
Sample response:
```json
[
    {
        "Since": "2020-05-28T17:05:48.166536476Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/v2/interfaces/loop1",
        "Label": "loop1",
        "Value": {
            "ProtoMsgName": "ligato.vpp.interfaces.Interface",
            "ProtoMsgData": "name:\"loop1\" type:SOFTWARE_LOOPBACK enabled:true ip_addresses:\"192.168.1.1/24\" "
        },
        "Flags": {
            "last-update": "TXN-1",
            "value-state": "CONFIGURED",
            "descriptor": "vpp-interface"
        },
        "MetadataFields": {
            "index": [
                "1"
            ],
            "ip_addresses": [
                "192.168.1.1/24"
            ]
        },
        "Targets": [
            {
                "Relation": "derives",
                "Label": "vpp/interface/loop1/address/static/192.168.1.1/24",
                "ExpectedKey": "vpp/interface/loop1/address/static/192.168.1.1/24",
                "MatchingKeys": [
                    "vpp/interface/loop1/address/static/192.168.1.1/24"
                ]
            },
            {
                "Relation": "derives",
                "Label": "vpp/interface/loop1/has-IP-address",
                "ExpectedKey": "vpp/interface/loop1/has-IP-address",
                "MatchingKeys": [
                    "vpp/interface/loop1/has-IP-address"
                ]
            },
            {
                "Relation": "derives",
                "Label": "vpp/interface/loop1/vrf/0/ip-version/v4",
                "ExpectedKey": "vpp/interface/loop1/vrf/0/ip-version/v4",
                "MatchingKeys": [
                    "vpp/interface/loop1/vrf/0/ip-version/v4"
                ]
            }
        ],
        "TargetUpdateOnly": false
    }
]
```

---

## Graph

Description: A graph-based representation of the system state, used internally by the KV Scheduler, can be displayed using any modern web browser supporting SVG.

Reference: [How to visualize the graph][kvs-graph-api] section of the Developer Guide.

---

## Graph Snapshot

Description: GET a snapshot of the KV Scheduler internal graph at a point in time. If there is no parameter passed in the request, then the current state is returned. 

Use the following parameter to specify a snapshot point in time:

- time: in [unix timestamp][unix-timestamp] format   

To GET the current graph snapshot, use:
```json
curl -X GET http://localhost:9191/scheduler/graph-snapshot
```
Sample response:
```json
[
    {
        "Since": "2020-05-28T16:50:52.769880703Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/v2/route/vrf/0/dst/0.0.0.0/32/gw/0.0.0.0",
        "Label": "config/vpp/v2/route/vrf/0/dst/0.0.0.0/32/gw/0.0.0.0",
        "Value": {
            "ProtoMsgName": "ligato.vpp.l3.Route",
            "ProtoMsgData": "type:DROP dst_network:\"0.0.0.0/32\" next_hop_addr:\"0.0.0.0\" weight:1 "
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "vpp-route"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.769881793Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "vpp/interface/UNTAGGED-local0/link-state/DOWN",
        "Label": "vpp/interface/UNTAGGED-local0/link-state/DOWN",
        "Value": {
            "ProtoMsgName": "google.protobuf.Empty",
            "ProtoMsgData": ""
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "vpp-interface-link-state"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T17:05:48.166546952Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "vpp/interface/loop1/vrf/0/ip-version/v4",
        "Label": "vpp/interface/loop1/vrf/0/ip-version/v4",
        "Value": {
            "ProtoMsgName": "google.protobuf.Empty",
            "ProtoMsgData": ""
        },
        "Flags": {
            "last-update": "TXN-1",
            "value-state": "CONFIGURED",
            "descriptor": "vpp-interface-vrf",
            "derived": "config/vpp/v2/interfaces/loop1"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T19:10:44.979182721Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/v2/interfaces/mem01",
        "Label": "mem01",
        "Value": {
            "ProtoMsgName": "ligato.vpp.interfaces.Interface",
            "ProtoMsgData": "name:\"mem01\" type:MEMIF enabled:true phys_address:\"02:00:00:00:00:03\" ip_addresses:\"100.100.100.102/24\" mtu:1500 rx_modes:<mode:ADAPTIVE > memif:<id:2 > "
        },
        "Flags": {
            "last-update": "TXN-7",
            "value-state": "CONFIGURED",
            "descriptor": "vpp-interface"
        },
        "MetadataFields": {
            "index": [
                "2"
            ],
            "ip_addresses": [
                "100.100.100.102/24"
            ]
        },
        "Targets": [
            {
                "Relation": "derives",
                "Label": "vpp/interface/mem01/address/static/100.100.100.102/24",
                "ExpectedKey": "vpp/interface/mem01/address/static/100.100.100.102/24",
                "MatchingKeys": [
                    "vpp/interface/mem01/address/static/100.100.100.102/24"
                ]
            },
            {
                "Relation": "derives",
                "Label": "vpp/interface/mem01/has-IP-address",
                "ExpectedKey": "vpp/interface/mem01/has-IP-address",
                "MatchingKeys": [
                    "vpp/interface/mem01/has-IP-address"
                ]
            },
            {
                "Relation": "derives",
                "Label": "vpp/interface/mem01/rx-modes",
                "ExpectedKey": "vpp/interface/mem01/rx-modes",
                "MatchingKeys": [
                    "vpp/interface/mem01/rx-modes"
                ]
            },
            {
                "Relation": "derives",
                "Label": "vpp/interface/mem01/vrf/0/ip-version/v4",
                "ExpectedKey": "vpp/interface/mem01/vrf/0/ip-version/v4",
                "MatchingKeys": [
                    "vpp/interface/mem01/vrf/0/ip-version/v4"
                ]
            }
        ],
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.769826989Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/v2/route/vrf/0/dst/::/0/gw/::",
        "Label": "config/vpp/v2/route/vrf/0/dst/::/0/gw/::",
        "Value": {
            "ProtoMsgName": "ligato.vpp.l3.Route",
            "ProtoMsgData": "type:DROP dst_network:\"::/0\" next_hop_addr:\"::\" weight:1 "
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "vpp-route"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T19:10:44.979238506Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "vpp/interface/mem01/has-IP-address",
        "Label": "vpp/interface/mem01/has-IP-address",
        "Value": {
            "ProtoMsgName": "google.protobuf.Empty",
            "ProtoMsgData": ""
        },
        "Flags": {
            "last-update": "TXN-7",
            "value-state": "CONFIGURED",
            "descriptor": "vpp-interface-has-address",
            "derived": "config/vpp/v2/interfaces/mem01"
        },
        "MetadataFields": null,
        "Targets": [
            {
                "Relation": "depends-on",
                "Label": "interface-has-IP",
                "ExpectedKey": "vpp/interface/mem01/address/*",
                "MatchingKeys": [
                    "vpp/interface/mem01/address/static/100.100.100.102/24"
                ]
            }
        ],
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.769827446Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/v2/route/vrf/0/dst/fe80::/10/gw/::",
        "Label": "config/vpp/v2/route/vrf/0/dst/fe80::/10/gw/::",
        "Value": {
            "ProtoMsgName": "ligato.vpp.l3.Route",
            "ProtoMsgData": "dst_network:\"fe80::/10\" next_hop_addr:\"::\" weight:1 "
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "vpp-route"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.769867279Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/v2/route/vrf/0/dst/240.0.0.0/4/gw/0.0.0.0",
        "Label": "config/vpp/v2/route/vrf/0/dst/240.0.0.0/4/gw/0.0.0.0",
        "Value": {
            "ProtoMsgName": "ligato.vpp.l3.Route",
            "ProtoMsgData": "type:DROP dst_network:\"240.0.0.0/4\" next_hop_addr:\"0.0.0.0\" weight:1 "
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "vpp-route"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T19:10:44.979234834Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "vpp/interface/mem01/address/static/100.100.100.102/24",
        "Label": "vpp/interface/mem01/address/static/100.100.100.102/24",
        "Value": {
            "ProtoMsgName": "google.protobuf.Empty",
            "ProtoMsgData": ""
        },
        "Flags": {
            "last-update": "TXN-7",
            "value-state": "CONFIGURED",
            "descriptor": "vpp-interface-address",
            "derived": "config/vpp/v2/interfaces/mem01"
        },
        "MetadataFields": null,
        "Targets": [
            {
                "Relation": "depends-on",
                "Label": "interface-assigned-to-vrf-table",
                "ExpectedKey": "vpp/interface/mem01/vrf/*",
                "MatchingKeys": [
                    "vpp/interface/mem01/vrf/0/ip-version/v4"
                ]
            }
        ],
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.76982628Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/v2/route/vrf/0/dst/224.0.0.0/4/gw/0.0.0.0",
        "Label": "config/vpp/v2/route/vrf/0/dst/224.0.0.0/4/gw/0.0.0.0",
        "Value": {
            "ProtoMsgName": "ligato.vpp.l3.Route",
            "ProtoMsgData": "type:DROP dst_network:\"224.0.0.0/4\" next_hop_addr:\"0.0.0.0\" weight:1 "
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "vpp-route"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T17:05:48.166546244Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "vpp/interface/loop1/has-IP-address",
        "Label": "vpp/interface/loop1/has-IP-address",
        "Value": {
            "ProtoMsgName": "google.protobuf.Empty",
            "ProtoMsgData": ""
        },
        "Flags": {
            "last-update": "TXN-1",
            "value-state": "CONFIGURED",
            "descriptor": "vpp-interface-has-address",
            "derived": "config/vpp/v2/interfaces/loop1"
        },
        "MetadataFields": null,
        "Targets": [
            {
                "Relation": "depends-on",
                "Label": "interface-has-IP",
                "ExpectedKey": "vpp/interface/loop1/address/*",
                "MatchingKeys": [
                    "vpp/interface/loop1/address/static/192.168.1.1/24"
                ]
            }
        ],
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.769871655Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "linux/interface/host-name/lo",
        "Label": "linux/interface/host-name/lo",
        "Value": {
            "ProtoMsgName": "google.protobuf.Empty",
            "ProtoMsgData": ""
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "linux-interface-watcher"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.769877741Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/ipfix/v2/ipfix",
        "Label": "",
        "Value": {
            "ProtoMsgName": "ligato.vpp.ipfix.IPFIX",
            "ProtoMsgData": "collector:<address:\"0.0.0.0\" > source_address:\"0.0.0.0\" vrf_id:4294967295 "
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "vpp-ipfix"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.769878645Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/nat/v2/nat44-global",
        "Label": "config/vpp/nat/v2/nat44-global",
        "Value": {
            "ProtoMsgName": "ligato.vpp.nat.Nat44Global",
            "ProtoMsgData": ""
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "vpp-nat44-global"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.769867907Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/v2/vrf-table/id/0/protocol/IPV6",
        "Label": "id/0/protocol/IPV6",
        "Value": {
            "ProtoMsgName": "ligato.vpp.l3.VrfTable",
            "ProtoMsgData": "protocol:IPV6 label:\"ipv6-VRF:0\" "
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "vpp-vrf-table"
        },
        "MetadataFields": {
            "index": [
                "0"
            ]
        },
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.769827916Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/v2/vrf-table/id/0/protocol/IPV4",
        "Label": "id/0/protocol/IPV4",
        "Value": {
            "ProtoMsgName": "ligato.vpp.l3.VrfTable",
            "ProtoMsgData": "label:\"ipv4-VRF:0\" "
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "vpp-vrf-table"
        },
        "MetadataFields": {
            "index": [
                "0"
            ]
        },
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.769866017Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/v2/proxyarp-global",
        "Label": "config/vpp/v2/proxyarp-global",
        "Value": {
            "ProtoMsgName": "ligato.vpp.l3.ProxyARP",
            "ProtoMsgData": ""
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "vpp-proxy-arp"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T19:10:44.979177611Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "vpp/interface/mem01/vrf/0/ip-version/v4",
        "Label": "vpp/interface/mem01/vrf/0/ip-version/v4",
        "Value": {
            "ProtoMsgName": "google.protobuf.Empty",
            "ProtoMsgData": ""
        },
        "Flags": {
            "last-update": "TXN-7",
            "value-state": "CONFIGURED",
            "descriptor": "vpp-interface-vrf",
            "derived": "config/vpp/v2/interfaces/mem01"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T17:05:48.166544965Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "vpp/interface/loop1/address/static/192.168.1.1/24",
        "Label": "vpp/interface/loop1/address/static/192.168.1.1/24",
        "Value": {
            "ProtoMsgName": "google.protobuf.Empty",
            "ProtoMsgData": ""
        },
        "Flags": {
            "last-update": "TXN-1",
            "value-state": "CONFIGURED",
            "descriptor": "vpp-interface-address",
            "derived": "config/vpp/v2/interfaces/loop1"
        },
        "MetadataFields": null,
        "Targets": [
            {
                "Relation": "depends-on",
                "Label": "interface-assigned-to-vrf-table",
                "ExpectedKey": "vpp/interface/loop1/vrf/*",
                "MatchingKeys": [
                    "vpp/interface/loop1/vrf/0/ip-version/v4"
                ]
            }
        ],
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T19:10:44.980286279Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "vpp/interface/mem01/link-state/DOWN",
        "Label": "vpp/interface/mem01/link-state/DOWN",
        "Value": {
            "ProtoMsgName": "google.protobuf.Empty",
            "ProtoMsgData": ""
        },
        "Flags": {
            "last-update": "TXN-8",
            "value-state": "OBTAINED",
            "descriptor": "vpp-interface-link-state"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.769823066Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/v2/interfaces/UNTAGGED-local0",
        "Label": "UNTAGGED-local0",
        "Value": {
            "ProtoMsgName": "ligato.vpp.interfaces.Interface",
            "ProtoMsgData": "name:\"UNTAGGED-local0\" type:SOFTWARE_LOOPBACK phys_address:\"00:00:00:00:00:00\" "
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "vpp-interface"
        },
        "MetadataFields": {
            "index": [
                "0"
            ]
        },
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.76987092Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "linux/interface/host-name/eth0",
        "Label": "linux/interface/host-name/eth0",
        "Value": {
            "ProtoMsgName": "google.protobuf.Empty",
            "ProtoMsgData": ""
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "linux-interface-watcher"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T17:05:48.166536476Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/v2/interfaces/loop1",
        "Label": "loop1",
        "Value": {
            "ProtoMsgName": "ligato.vpp.interfaces.Interface",
            "ProtoMsgData": "name:\"loop1\" type:SOFTWARE_LOOPBACK enabled:true ip_addresses:\"192.168.1.1/24\" "
        },
        "Flags": {
            "last-update": "TXN-1",
            "value-state": "CONFIGURED",
            "descriptor": "vpp-interface"
        },
        "MetadataFields": {
            "index": [
                "1"
            ],
            "ip_addresses": [
                "192.168.1.1/24"
            ]
        },
        "Targets": [
            {
                "Relation": "derives",
                "Label": "vpp/interface/loop1/address/static/192.168.1.1/24",
                "ExpectedKey": "vpp/interface/loop1/address/static/192.168.1.1/24",
                "MatchingKeys": [
                    "vpp/interface/loop1/address/static/192.168.1.1/24"
                ]
            },
            {
                "Relation": "derives",
                "Label": "vpp/interface/loop1/has-IP-address",
                "ExpectedKey": "vpp/interface/loop1/has-IP-address",
                "MatchingKeys": [
                    "vpp/interface/loop1/has-IP-address"
                ]
            },
            {
                "Relation": "derives",
                "Label": "vpp/interface/loop1/vrf/0/ip-version/v4",
                "ExpectedKey": "vpp/interface/loop1/vrf/0/ip-version/v4",
                "MatchingKeys": [
                    "vpp/interface/loop1/vrf/0/ip-version/v4"
                ]
            }
        ],
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.76987227Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/v2/route/vrf/0/dst/0.0.0.0/0/gw/0.0.0.0",
        "Label": "config/vpp/v2/route/vrf/0/dst/0.0.0.0/0/gw/0.0.0.0",
        "Value": {
            "ProtoMsgName": "ligato.vpp.l3.Route",
            "ProtoMsgData": "type:DROP dst_network:\"0.0.0.0/0\" next_hop_addr:\"0.0.0.0\" weight:1 "
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "vpp-route"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T16:50:52.769881236Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "config/vpp/v2/route/vrf/0/dst/255.255.255.255/32/gw/0.0.0.0",
        "Label": "config/vpp/v2/route/vrf/0/dst/255.255.255.255/32/gw/0.0.0.0",
        "Value": {
            "ProtoMsgName": "ligato.vpp.l3.Route",
            "ProtoMsgData": "type:DROP dst_network:\"255.255.255.255/32\" next_hop_addr:\"0.0.0.0\" weight:1 "
        },
        "Flags": {
            "last-update": "TXN-0",
            "value-state": "OBTAINED",
            "descriptor": "vpp-route"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T17:05:48.167112121Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "vpp/interface/loop1/link-state/UP",
        "Label": "vpp/interface/loop1/link-state/UP",
        "Value": {
            "ProtoMsgName": "google.protobuf.Empty",
            "ProtoMsgData": ""
        },
        "Flags": {
            "last-update": "TXN-2",
            "value-state": "OBTAINED",
            "descriptor": "vpp-interface-link-state"
        },
        "MetadataFields": null,
        "Targets": null,
        "TargetUpdateOnly": false
    },
    {
        "Since": "2020-05-28T19:10:44.979173377Z",
        "Until": "0001-01-01T00:00:00Z",
        "Key": "vpp/interface/mem01/rx-modes",
        "Label": "vpp/interface/mem01/rx-modes",
        "Value": {
            "ProtoMsgName": "ligato.vpp.interfaces.Interface",
            "ProtoMsgData": "name:\"mem01\" type:MEMIF rx_modes:<mode:ADAPTIVE > "
        },
        "Flags": {
            "last-update": "TXN-7",
            "value-state": "PENDING",
            "unavailable": "",
            "descriptor": "vpp-interface-rx-mode",
            "derived": "config/vpp/v2/interfaces/mem01"
        },
        "MetadataFields": null,
        "Targets": [
            {
                "Relation": "depends-on",
                "Label": "interface-link-is-UP",
                "ExpectedKey": "vpp/interface/mem01/link-state/UP",
                "MatchingKeys": []
            }
        ],
        "TargetUpdateOnly": false
    }
]
```
---

## Status

Description: GET value state by descriptor, by key, or all. The parameters are:

- descriptor
- key 

To GET all value states, use: 
```json
curl -X GET http://localhost:9191/scheduler/status
```
Sample response:
```json
[
    {
        "value": {
            "key": "config/vpp/ipfix/v2/ipfix",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "config/vpp/nat/v2/nat44-global",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "config/vpp/v2/interfaces/UNTAGGED-local0",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "config/vpp/v2/interfaces/loop1",
            "state": "CONFIGURED",
            "lastOperation": "CREATE"
        },
        "derived_values": [
            {
                "key": "vpp/interface/loop1/address/static/192.168.1.1/24",
                "state": "CONFIGURED",
                "lastOperation": "CREATE"
            },
            {
                "key": "vpp/interface/loop1/has-IP-address",
                "state": "CONFIGURED",
                "lastOperation": "CREATE"
            },
            {
                "key": "vpp/interface/loop1/vrf/0/ip-version/v4",
                "state": "CONFIGURED",
                "lastOperation": "CREATE"
            }
        ]
    },
    {
        "value": {
            "key": "config/vpp/v2/interfaces/mem01",
            "state": "CONFIGURED",
            "lastOperation": "CREATE"
        },
        "derived_values": [
            {
                "key": "vpp/interface/mem01/address/static/100.100.100.102/24",
                "state": "CONFIGURED",
                "lastOperation": "CREATE"
            },
            {
                "key": "vpp/interface/mem01/has-IP-address",
                "state": "CONFIGURED",
                "lastOperation": "CREATE"
            },
            {
                "key": "vpp/interface/mem01/rx-modes",
                "state": "PENDING",
                "lastOperation": "CREATE",
                "details": [
                    "interface-link-is-UP"
                ]
            },
            {
                "key": "vpp/interface/mem01/vrf/0/ip-version/v4",
                "state": "CONFIGURED",
                "lastOperation": "CREATE"
            }
        ]
    },
    {
        "value": {
            "key": "config/vpp/v2/proxyarp-global",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "config/vpp/v2/route/vrf/0/dst/0.0.0.0/0/gw/0.0.0.0",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "config/vpp/v2/route/vrf/0/dst/0.0.0.0/32/gw/0.0.0.0",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "config/vpp/v2/route/vrf/0/dst/224.0.0.0/4/gw/0.0.0.0",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "config/vpp/v2/route/vrf/0/dst/240.0.0.0/4/gw/0.0.0.0",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "config/vpp/v2/route/vrf/0/dst/255.255.255.255/32/gw/0.0.0.0",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "config/vpp/v2/route/vrf/0/dst/::/0/gw/::",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "config/vpp/v2/route/vrf/0/dst/fe80::/10/gw/::",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "config/vpp/v2/vrf-table/id/0/protocol/IPV4",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "config/vpp/v2/vrf-table/id/0/protocol/IPV6",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "linux/interface/host-name/eth0",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "linux/interface/host-name/lo",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "vpp/interface/UNTAGGED-local0/link-state/DOWN",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "vpp/interface/loop1/link-state/UP",
            "state": "OBTAINED"
        }
    },
    {
        "value": {
            "key": "vpp/interface/mem01/link-state/DOWN",
            "state": "OBTAINED"
        }
    }
]
```

---

To GET the value state for the `config/vpp/v2/interfaces/loop1` key, use:
```json
curl -X GET http://localhost:9191/scheduler/status?key=config/vpp/v2/interfaces/loop1
```
Sample response:
```json
{
    "value": {
        "key": "config/vpp/v2/interfaces/loop1",
        "state": "CONFIGURED",
        "lastOperation": "CREATE"
    },
    "derived_values": [
        {
            "key": "vpp/interface/loop1/address/static/192.168.1.1/24",
            "state": "CONFIGURED",
            "lastOperation": "CREATE"
        },
        {
            "key": "vpp/interface/loop1/has-IP-address",
            "state": "CONFIGURED",
            "lastOperation": "CREATE"
        },
        {
            "key": "vpp/interface/loop1/vrf/0/ip-version/v4",
            "state": "CONFIGURED",
            "lastOperation": "CREATE"
        }
    ]
}
```
---

## Flag Stats

Description: GET total and per-value counts by value flag. The following parameters are used to specify a value flag:  

- last-update: last transaction that changed/updated the value
- value-state: current state of the value
- descriptor: used to look up values by descriptor
- derived: mark derived values
- unavailable: mark NB values which should not be considered when resolving dependencies of other values
- Error: used to store error returned from the last operation, including validation errors 

To GET the flag-stats by `descriptor` flag, use:
```json
curl -X GET http://localhost:9191/scheduler/flag-stats?flag=descriptor
```
Sample response:
```json
{
    "TotalCount": 31,
    "PerValueCount": {
        "linux-interface-watcher": 2,
        "vpp-interface": 7,
        "vpp-interface-address": 2,
        "vpp-interface-has-address": 2,
        "vpp-interface-link-state": 3,
        "vpp-interface-rx-mode": 1,
        "vpp-interface-vrf": 2,
        "vpp-ipfix": 1,
        "vpp-nat44-global": 1,
        "vpp-proxy-arp": 1,
        "vpp-route": 7,
        "vpp-vrf-table": 2
    }
}
```

To GET the flag-stats by `last-update` flag, use:
```json
curl -X GET http://localhost:9191/scheduler/flag-stats?flag=last-update
```
Sample response:
```json
{
    "TotalCount": 31,
    "PerValueCount": {
        "TXN-0": 16,
        "TXN-1": 4,
        "TXN-2": 1,
        "TXN-3": 1,
        "TXN-4": 1,
        "TXN-5": 1,
        "TXN-6": 1,
        "TXN-7": 5,
        "TXN-8": 1
    }
}
```
To GET the flag-stats by `value-state` flag, use:
```json
curl -X GET http://localhost:9191/scheduler/flag-stats?flag=value-state
```
Sample response:
```json
{
    "TotalCount": 31,
    "PerValueCount": {
        "CONFIGURED": 8,
        "FAILED": 1,
        "OBTAINED": 18,
        "PENDING": 1,
        "RETRYING": 3
    }
}
```
To GET the flag-stats by `derived` flag, use:
```json
curl -X GET http://localhost:9191/scheduler/flag-stats?flag=derived
```
---
To GET the flag-stats by `unavailable` flag, use:
```json
curl -X GET http://localhost:9191/scheduler/flag-stats?flag=unavailable
```
---

To GET the flag-stats by `error` flag, use:
```json
curl -X GET http://localhost:9191/scheduler/flag-stats?flag=error
```
---

## Downstream Resync

Description: Triggers a downstream resync

```json
curl -X POST http://localhost:9191/scheduler/downstream-resync
```

Sample response:
```json
{
    "Start": "2020-05-29T22:22:32.79908837Z",
    "Stop": "2020-05-29T22:22:32.829282034Z",
    "SeqNum": 9,
    "TxnType": "NBTransaction",
    "ResyncType": "DownstreamResync",
    "Executed": [
        {
            "Operation": "DELETE",
            "Key": "vpp/spd/1/interface/tap1",
            "NewState": "REMOVED",
            "PrevState": "PENDING",
            "PrevValue": {
                "ProtoMsgName": "ligato.vpp.ipsec.SecurityPolicyDatabase.Interface",
                "ProtoMsgData": "name:\"tap1\" "
            },
            "NOOP": true,
            "IsDerived": true,
            "IsRecreate": true
        },
        {
            "Operation": "DELETE",
            "Key": "vpp/spd/1/sa/10",
            "NewState": "REMOVED",
            "PrevState": "PENDING",
            "PrevValue": {
                "ProtoMsgName": "ligato.vpp.ipsec.SecurityPolicyDatabase.PolicyEntry",
                "ProtoMsgData": "sa_index:10 priority:10 is_outbound:true remote_addr_start:\"10.0.0.1\" remote_addr_stop:\"10.0.0.1\" local_addr_start:\"10.0.0.2\" local_addr_stop:\"10.0.0.2\" action:PROTECT "
            },
            "NOOP": true,
            "IsDerived": true,
            "IsRecreate": true
        },
        {
            "Operation": "DELETE",
            "Key": "vpp/spd/1/sa/20",
            "NewState": "REMOVED",
            "PrevState": "PENDING",
            "PrevValue": {
                "ProtoMsgName": "ligato.vpp.ipsec.SecurityPolicyDatabase.PolicyEntry",
                "ProtoMsgData": "sa_index:20 priority:10 remote_addr_start:\"10.0.0.1\" remote_addr_stop:\"10.0.0.1\" local_addr_start:\"10.0.0.2\" local_addr_stop:\"10.0.0.2\" action:PROTECT "
            },
            "NOOP": true,
            "IsDerived": true,
            "IsRecreate": true
        },
        {
            "Operation": "DELETE",
            "Key": "config/vpp/ipsec/v2/spd/1",
            "NewState": "REMOVED",
            "PrevState": "CONFIGURED",
            "PrevValue": {
                "ProtoMsgName": "ligato.vpp.ipsec.SecurityPolicyDatabase",
                "ProtoMsgData": "index:1 "
            },
            "IsRecreate": true
        },
        {
            "Operation": "CREATE",
            "Key": "config/vpp/ipsec/v2/spd/1",
            "NewState": "CONFIGURED",
            "NewValue": {
                "ProtoMsgName": "ligato.vpp.ipsec.SecurityPolicyDatabase",
                "ProtoMsgData": "index:1 interfaces:<name:\"tap1\" > policy_entries:<sa_index:20 priority:10 remote_addr_start:\"10.0.0.1\" remote_addr_stop:\"10.0.0.1\" local_addr_start:\"10.0.0.2\" local_addr_stop:\"10.0.0.2\" action:PROTECT > policy_entries:<sa_index:10 priority:10 is_outbound:true remote_addr_start:\"10.0.0.1\" remote_addr_stop:\"10.0.0.1\" local_addr_start:\"10.0.0.2\" local_addr_stop:\"10.0.0.2\" action:PROTECT > "
            },
            "PrevState": "REMOVED",
            "IsRecreate": true
        },
        {
            "Operation": "CREATE",
            "Key": "vpp/spd/1/interface/tap1",
            "NewState": "PENDING",
            "NewValue": {
                "ProtoMsgName": "ligato.vpp.ipsec.SecurityPolicyDatabase.Interface",
                "ProtoMsgData": "name:\"tap1\" "
            },
            "NOOP": true,
            "IsDerived": true,
            "IsRecreate": true
        },
        {
            "Operation": "CREATE",
            "Key": "vpp/spd/1/sa/10",
            "NewState": "PENDING",
            "NewValue": {
                "ProtoMsgName": "ligato.vpp.ipsec.SecurityPolicyDatabase.PolicyEntry",
                "ProtoMsgData": "sa_index:10 priority:10 is_outbound:true remote_addr_start:\"10.0.0.1\" remote_addr_stop:\"10.0.0.1\" local_addr_start:\"10.0.0.2\" local_addr_stop:\"10.0.0.2\" action:PROTECT "
            },
            "NOOP": true,
            "IsDerived": true,
            "IsRecreate": true
        },
        {
            "Operation": "CREATE",
            "Key": "vpp/spd/1/sa/20",
            "NewState": "PENDING",
            "NewValue": {
                "ProtoMsgName": "ligato.vpp.ipsec.SecurityPolicyDatabase.PolicyEntry",
                "ProtoMsgData": "sa_index:20 priority:10 remote_addr_start:\"10.0.0.1\" remote_addr_stop:\"10.0.0.1\" local_addr_start:\"10.0.0.2\" local_addr_stop:\"10.0.0.2\" action:PROTECT "
            },
            "NOOP": true,
            "IsDerived": true,
            "IsRecreate": true
        },
        {
            "Operation": "CREATE",
            "Key": "config/vpp/abfs/v2/abf/1",
            "NewState": "PENDING",
            "NewValue": {
                "ProtoMsgName": "ligato.vpp.abf.ABF",
                "ProtoMsgData": "index:1 acl_name:\"aclip1\" attached_interfaces:<input_interface:\"tap1\" priority:40 > attached_interfaces:<input_interface:\"memif1\" priority:60 > forwarding_paths:<next_hop_ip:\"10.0.0.10\" interface_name:\"loop1\" weight:20 preference:25 > "
            },
            "PrevState": "PENDING",
            "PrevValue": {
                "ProtoMsgName": "ligato.vpp.abf.ABF",
                "ProtoMsgData": "index:1 acl_name:\"aclip1\" attached_interfaces:<input_interface:\"tap1\" priority:40 > attached_interfaces:<input_interface:\"memif1\" priority:60 > forwarding_paths:<next_hop_ip:\"10.0.0.10\" interface_name:\"loop1\" weight:20 preference:25 > "
            },
            "NOOP": true
        },
        {
            "Operation": "CREATE",
            "Key": "config/vpp/l2/v2/xconnect/if1",
            "NewState": "PENDING",
            "NewValue": {
                "ProtoMsgName": "ligato.vpp.l2.XConnectPair",
                "ProtoMsgData": "receive_interface:\"if1\" transmit_interface:\"if2\" "
            },
            "PrevState": "PENDING",
            "PrevValue": {
                "ProtoMsgName": "ligato.vpp.l2.XConnectPair",
                "ProtoMsgData": "receive_interface:\"if1\" transmit_interface:\"if2\" "
            },
            "NOOP": true
        },
        {
            "Operation": "CREATE",
            "Key": "vpp/acl/acl1/interface/egress/tap1",
            "NewState": "PENDING",
            "NewValue": {
                "ProtoMsgName": "google.protobuf.Empty",
                "ProtoMsgData": ""
            },
            "PrevState": "PENDING",
            "PrevValue": {
                "ProtoMsgName": "google.protobuf.Empty",
                "ProtoMsgData": ""
            },
            "NOOP": true,
            "IsDerived": true
        },
        {
            "Operation": "CREATE",
            "Key": "vpp/acl/acl1/interface/egress/tap2",
            "NewState": "PENDING",
            "NewValue": {
                "ProtoMsgName": "google.protobuf.Empty",
                "ProtoMsgData": ""
            },
            "PrevState": "PENDING",
            "PrevValue": {
                "ProtoMsgName": "google.protobuf.Empty",
                "ProtoMsgData": ""
            },
            "NOOP": true,
            "IsDerived": true
        },
        {
            "Operation": "CREATE",
            "Key": "vpp/acl/acl1/interface/ingress/tap3",
            "NewState": "PENDING",
            "NewValue": {
                "ProtoMsgName": "google.protobuf.Empty",
                "ProtoMsgData": ""
            },
            "PrevState": "PENDING",
            "PrevValue": {
                "ProtoMsgName": "google.protobuf.Empty",
                "ProtoMsgData": ""
            },
            "NOOP": true,
            "IsDerived": true
        },
        {
            "Operation": "CREATE",
            "Key": "vpp/acl/acl1/interface/ingress/tap4",
            "NewState": "PENDING",
            "NewValue": {
                "ProtoMsgName": "google.protobuf.Empty",
                "ProtoMsgData": ""
            },
            "PrevState": "PENDING",
            "PrevValue": {
                "ProtoMsgName": "google.protobuf.Empty",
                "ProtoMsgData": ""
            },
            "NOOP": true,
            "IsDerived": true
        },
        {
            "Operation": "UPDATE",
            "Key": "config/vpp/l2/v2/bridge-domain/bd1",
            "NewState": "CONFIGURED",
            "NewValue": {
                "ProtoMsgName": "ligato.vpp.l2.BridgeDomain",
                "ProtoMsgData": "name:\"bd1\" flood:true unknown_unicast_flood:true forward:true learn:true arp_termination:true interfaces:<name:\"if1\" bridged_virtual_interface:true > interfaces:<name:\"if2\" > interfaces:<name:\"if2\" > arp_termination_table:<ip_address:\"192.168.10.10\" phys_address:\"a7:65:f1:b5:dc:f6\" > arp_termination_table:<ip_address:\"10.10.0.1\" phys_address:\"59:6C:45:59:8E:BC\" > "
            },
            "PrevState": "CONFIGURED",
            "PrevValue": {
                "ProtoMsgName": "ligato.vpp.l2.BridgeDomain",
                "ProtoMsgData": "name:\"bd1\" flood:true unknown_unicast_flood:true forward:true learn:true arp_termination:true arp_termination_table:<ip_address:\"192.168.10.10\" phys_address:\"a7:65:9d:c8:c7:7f\" > arp_termination_table:<ip_address:\"10.10.0.1\" phys_address:\"59:6c:9d:c8:c7:7f\" > "
            }
        },
        {
            "Operation": "CREATE",
            "Key": "vpp/bd/bd1/interface/if1",
            "NewState": "PENDING",
            "NewValue": {
                "ProtoMsgName": "ligato.vpp.l2.BridgeDomain.Interface",
                "ProtoMsgData": "name:\"if1\" bridged_virtual_interface:true "
            },
            "PrevState": "PENDING",
            "PrevValue": {
                "ProtoMsgName": "ligato.vpp.l2.BridgeDomain.Interface",
                "ProtoMsgData": "name:\"if1\" bridged_virtual_interface:true "
            },
            "NOOP": true,
            "IsDerived": true
        },
        {
            "Operation": "CREATE",
            "Key": "vpp/bd/bd1/interface/if2",
            "NewState": "PENDING",
            "NewValue": {
                "ProtoMsgName": "ligato.vpp.l2.BridgeDomain.Interface",
                "ProtoMsgData": "name:\"if2\" "
            },
            "PrevState": "PENDING",
            "PrevValue": {
                "ProtoMsgName": "ligato.vpp.l2.BridgeDomain.Interface",
                "ProtoMsgData": "name:\"if2\" "
            },
            "NOOP": true,
            "IsDerived": true
        },
        {
            "Operation": "CREATE",
            "Key": "vpp/bd/bd2/interface/loop1",
            "NewState": "PENDING",
            "NewValue": {
                "ProtoMsgName": "ligato.vpp.l2.BridgeDomain.Interface",
                "ProtoMsgData": "name:\"loop1\" "
            },
            "PrevState": "PENDING",
            "PrevValue": {
                "ProtoMsgName": "ligato.vpp.l2.BridgeDomain.Interface",
                "ProtoMsgData": "name:\"loop1\" "
            },
            "NOOP": true,
            "IsDerived": true
        }
    ]
}
```
[agentctl]: ../user-guide/agentctl.md
[unix-timestamp]: https://www.unixtimestamp.com/
[quickstart-guide]: ../user-guide/quickstart.md
[kvs-rest-go]: https://github.com/ligato/vpp-agent/blob/master/plugins/kvscheduler/rest.go
[kvs-graph-api]: ../developer-guide/kvs-troubleshooting.md#how-to-visualize-the-graph
[rest-plugin-config-options]: ../plugins/connection-plugins.md#configuration