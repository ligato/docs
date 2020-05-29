# VPP Agent

## REST API Index

```
curl http://localhost:9191/
```
---

## Interfaces
```
curl http://localhost:9191/dump/vpp/v2/interfaces
```
Sample response:
```
{
    "0": {
        "interface": {
            "name": "UNTAGGED-local0",
            "type": "SOFTWARE_LOOPBACK"
        },
        "interface_meta": {
            "sw_if_index": 0,
            "sub_sw_if_index": 0,
            "l2_address": "",
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
            "name": "GigabitEthernet0/8/0",
            "type": "DPDK",
            "enabled": true,
            "physAddress": "08:00:27:e9:b9:9b",
            "ipAddresses": [
                "192.168.16.1/24"
            ],
            "mtu": 9206,
            "rxModes": [
                {
                    "mode": "POLLING"
                }
            ],
            "rxPlacements": [
                {
                    "mainThread": true
                }
            ]
        },
        "interface_meta": {
            "sw_if_index": 1,
            "sub_sw_if_index": 1,
            "l2_address": "CAAn6bmb",
            "internal_name": "GigabitEthernet0/8/0",
            "is_admin_state_up": true,
            "is_link_state_up": true,
            "link_duplex": 2,
            "link_mtu": 9206,
            "mtu": [
                9000,
                0,
                0,
                0
            ],
            "link_speed": 1000000,
            "sub_id": 0,
            "tag": "",
            "dhcp": null,
            "vrf_ipv4": 0,
            "vrf_ipv6": 0,
            "pci": 0
        }
    },
// additional interfaces
...
```
---

## VPP Interfaces/Loopback

```
curl http://localhost:9191/dump/vpp/v2/interfaces/loopback
```
Sample response:
```json
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
// additional loopbacks
...
```
---

## VPP Interfaces/Ethernet
```json
curl http://localhost:9191/dump/vpp/v2/interfaces/ethernet
```
Sample response:
```json
{
  "1": {
    "interface": {
      "name": "GigabitEthernet0/8/0",
      "type": "DPDK",
      "enabled": true,
      "physAddress": "08:00:27:da:0d:6b",
      "ipAddresses": [
        "192.168.16.1/24"
      ],
      "mtu": 9206,
      "rxModes": [
        {
          "mode": "POLLING"
        }
      ],
      "rxPlacements": [
        {
          "mainThread": true
        }
      ]
    },
    "interface_meta": {
      "sw_if_index": 1,
      "sub_sw_if_index": 1,
      "l2_address": "CAAn2g1r",
      "internal_name": "GigabitEthernet0/8/0",
      "is_admin_state_up": true,
      "is_link_state_up": true,
      "link_duplex": 2,
      "link_mtu": 9206,
      "mtu": [
        9000,
        0,
        0,
        0
      ],
      "link_speed": 1000000,
      "sub_id": 0,
      "tag": "",
      "dhcp": null,
      "vrf_ipv4": 0,
      "vrf_ipv6": 0,
      "pci": 0
    }
  }
}
// additional ethernet interfaces
...
```
---

## VPP Interfaces/Vxlan
```json
curl http://localhost:9191/dump/vpp/v2/interfaces/vxlan
```
Sample response:
```json
{
    "8": {
        "interface": {
            "name": "vxlan-default-2",
            "type": "VXLAN_TUNNEL",
            "enabled": true,
            "vxlan": {
                "srcAddress": "192.168.16.1",
                "dstAddress": "192.168.16.2",
                "vni": 10
            }
        },
        "interface_meta": {
            "sw_if_index": 8,
            "sub_sw_if_index": 8,
            "l2_address": "",
            "internal_name": "vxlan_tunnel0",
            "is_admin_state_up": true,
            "is_link_state_up": true,
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
            "tag": "vxlan-default-2",
            "dhcp": null,
            "vrf_ipv4": 0,
            "vrf_ipv6": 0,
            "pci": 0
        }
    }
}
// other vxlan interfaces
...
```
---

## VPP Interfaces/Tap
```json
curl http://localhost:9191/dump/vpp/v2/interfaces/tap
```
Sample response:
```json
{
  "3": {
    "interface": {
      "name": "tap-vpp2",
      "type": "TAP",
      "enabled": true,
      "physAddress": "34:3c:00:00:00:01",
      "ipAddresses": [
        "172.30.1.1/24"
      ],
      "mtu": 1450,
      "rxModes": [
        {
          "mode": "POLLING"
        }
      ],
      "rxPlacements": [
        {
          "mainThread": true
        }
      ],
      "tap": {
        "version": 2,
        "hostIfName": "tap-1755813745",
        "rxRingSize": 1024,
        "txRingSize": 1024
      }
    },
    "interface_meta": {
      "sw_if_index": 3,
      "sub_sw_if_index": 3,
      "l2_address": "NDwAAAAB",
      "internal_name": "tap0",
      "is_admin_state_up": true,
      "is_link_state_up": true,
      "link_duplex": 0,
      "link_mtu": 1450,
      "mtu": [
        1450,
        0,
        0,
        0
      ],
      "link_speed": 0,
      "sub_id": 0,
      "tag": "tap-vpp2",
      "dhcp": null,
      "vrf_ipv4": 0,
      "vrf_ipv6": 0,
      "pci": 0
    }
  },
// additional tap interfaces
...
 
```
---

## VPP Interfaces/Memif
```json
http://localhost:9191/dump/vpp/v2/interfaces/memif
```
Sample response:
```json
{
"2": {
    "interface": {
        "name": "mem01",
        "type": "MEMIF",
        "enabled": true,
        "physAddress": "02:00:00:00:00:03",
        "ipAddresses": [
            "100.100.100.102/24"
        ],
        "mtu": 1500,
        "memif": {
            "id": 2,
            "socketFilename": "/run/vpp/memif.sock",
            "ringSize": 1
        }
    },
    "interface_meta": {
        "sw_if_index": 2,
        "sub_sw_if_index": 2,
        "l2_address": "AgAAAAAD",
        "internal_name": "memif0/2",
        "is_admin_state_up": true,
        "is_link_state_up": false,
        "link_duplex": 0,
        "link_mtu": 1500,
        "mtu": [
            1500,
            0,
            0,
            0
        ],
        "link_speed": 0,
        "sub_id": 0,
        "tag": "mem01",
        "dhcp": null,
        "vrf_ipv4": 0,
        "vrf_ipv6": 0,
        "pci": 0
    },
}
// additional memif interfaces
...
```
---

## VPP Interfaces/Afpacket
```json
curl http://localhost:9191/dump/vpp/v2/interfaces/afpacket
```
---

## VPP ACL IP
```json
curl http://localhost:9191/dump/vpp/v2/acl/ip
```

---

## VPP ACL MACIP
```json
curl http://localhost:9191/dump/vpp/v2/acl/macip
```
---

## VPP ARPs
```json
curl http://localhost:9191/dump/vpp/v2/arps
```
Sample response:
```json
[
    {
        "Arp": {
            "interface": "GigabitEthernet0/8/0",
            "ip_address": "192.168.16.100",
            "phys_address": "08:00:27:c9:c3:7a"
        },
        "Meta": {
            "SwIfIndex": 1
        }
    },
    {
        "Arp": {
            "interface": "vxlanBVI",
            "ip_address": "192.168.30.2",
            "phys_address": "12:2b:00:00:00:02",
            "static": true
        },
        "Meta": {
            "SwIfIndex": 4
        }
    },
    {
        "Arp": {
            "interface": "tap-vpp2",
            "ip_address": "172.30.1.2",
            "phys_address": "96:55:cc:5b:35:6b"
        },
        "Meta": {
            "SwIfIndex": 3
        }
    },
    {
        "Arp": {
            "interface": "vpp-tap-054072f2c3954ab93b479d36b391f264f0096b1c1d374a4afb45ef4",
            "ip_address": "10.1.1.2",
            "phys_address": "02:fe:e3:99:d0:4c",
            "static": true
        },
        "Meta": {
            "SwIfIndex": 5
        }
    },
    {
        "Arp": {
            "interface": "vpp-tap-400dc32e50d03aeff4718c568bf5520cdc0addc1196f3c9d58dc1ec",
            "ip_address": "10.1.1.3",
            "phys_address": "02:fe:d4:0d:5f:64",
            "static": true
        },
        "Meta": {
            "SwIfIndex": 6
        }
    },
    {
        "Arp": {
            "interface": "vpp-tap-83dc59e9763c663bc2693ae1bd50a4982a536ccac3034518ea9e6a2",
            "ip_address": "10.1.1.4",
            "phys_address": "02:fe:a2:51:a7:65",
            "static": true
        },
        "Meta": {
            "SwIfIndex": 7
        }
    }
]
// additional ARP entries
...
```
---

## VPP L2 Bridge Domain
```json
curl http://localhost:9191/dump/vpp/v2/bd
```
Sample response:
```json
[
    {
        "bridge_domain": {
            "name": "vxlanBD",
            "forward": true,
            "interfaces": [
                {
                    "name": "vxlanBVI",
                    "bridged_virtual_interface": true,
                    "split_horizon_group": 1
                },
                {
                    "name": "vxlan-default-2",
                    "split_horizon_group": 1
                }
            ]
        },
        "bridge_domain_meta": {
            "bridge_domain_id": 1
        }
    }
]
```
---

## VPP L2 FIB
```json
curl http://localhost:9191/dump/vpp/v2/fib
```
Sample response:
```json
{
    "12:2b:00:00:00:01": {
        "fib": {
            "phys_address": "12:2b:00:00:00:01",
            "bridge_domain": "vxlanBD",
            "outgoing_interface": "vxlanBVI",
            "static_config": true,
            "bridged_virtual_interface": true
        },
        "fib_meta": {
            "bridge_domain_id": 1,
            "outgoing_interface_sw_if_idx": 4
        }
    },
    "12:2b:00:00:00:02": {
        "fib": {
            "phys_address": "12:2b:00:00:00:02",
            "bridge_domain": "vxlanBD",
            "outgoing_interface": "vxlan-default-2",
            "static_config": true
        },
        "fib_meta": {
            "bridge_domain_id": 1,
            "outgoing_interface_sw_if_idx": 8
        }
    }
}
```
---

## VPP L2 X-connect
```json
curl http://localhost:9191/dump/vpp/v2/xc
```
---

## VPP L3 Routes
```json
curl http://localhost:9191/dump/vpp/v2/routes
```
Sample response:
```json
[
  {
    "Route": {
      "dst_network": "0.0.0.0/0",
      "next_hop_addr": "192.168.16.100",
      "outgoing_interface": "GigabitEthernet0/8/0",
      "weight": 1
    },
    "Meta": {
      "TableName": "",
      "OutgoingIfIdx": 1,
      "IsIPv6": false,
      "Afi": 0,
      "IsLocal": false,
      "IsUDPEncap": false,
      "IsUnreach": false,
      "IsProhibit": false,
      "IsResolveHost": false,
      "IsResolveAttached": false,
      "IsDvr": false,
      "IsSourceLookup": false,
      "NextHopID": 0,
      "RpfID": 0,
      "LabelStack": []
    }
  },
  {
    "Route": {
      "type": 2,
      "dst_network": "240.0.0.0/4",
      "next_hop_addr": "0.0.0.0",
      "weight": 1
    },
    "Meta": {
      "TableName": "",
      "OutgoingIfIdx": 4294967295,
      "IsIPv6": false,
      "Afi": 0,
      "IsLocal": false,
      "IsUDPEncap": false,
      "IsUnreach": false,
      "IsProhibit": false,
      "IsResolveHost": false,
      "IsResolveAttached": false,
      "IsDvr": false,
      "IsSourceLookup": false,
      "NextHopID": 0,
      "RpfID": 0,
      "LabelStack": []
    },
// additional routes    
...
```
---

## VPP L3 IP Scan Neighbor
```json
curl http://localhost:9191/dump/vpp/v2/ipscanneigh
```
Sampe response:
```json
{
  "mode": 1,
  "scan_interval": 1,
  "max_proc_time": 20,
  "max_update": 10,
  "scan_int_delay": 1,
  "stale_threshold": 4
}
```

---

## VPP L3 Proxy Arp Interfaces
```json
curl http://localhost:9191/dump/vpp/v2/proxyarp/interfaces
```

---

## VPP L3 Proxy ARP Ranges
```json
curl http://localhost:9191/dump/vpp/v2/proxyarp/ranges
```

---

## VPP NAT Global
```json
curl http://localhost:9191/dump/vpp/v2/nat/global
```
Sample response:
```json
{
  "forwarding": true,
  "virtual_reassembly": {
    "timeout": 2,
    "max_reassemblies": 1024,
    "max_fragments": 5
  }
}
```
---

## VPP NAT DNAT
```json
curl http://localhost:9191/dump/vpp/v2/nat/dnat
```
Sample response:
```json
[
  {
    "label": "DNAT-identities",
    "id_mappings": [
      {
        "ip_address": "192.168.16.1",
        "port": 4789,
        "protocol": 1
      },
      {
        "vrf_id": 1,
        "ip_address": "192.168.16.1",
        "protocol": 1
      },
      {
        "ip_address": "192.168.16.1",
        "protocol": 1
      }
    ]
  },
  {
    "label": "default/kubernetes",
    "st_mappings": [
      {
        "external_ip": "10.96.0.1",
        "external_port": 443,
        "local_ips": [
          {
            "local_ip": "10.20.0.2",
            "local_port": 6443
          }
        ],
        "twice_nat": 2
      }
    ]
  },
// additional DNAT Mappings
...
```
---

## VPP NAT Interfaces
```json
curl http://localhost:9191/dump/vpp/v2/nat/interfaces
```
Sample response:
```json
{
  "name": "tap-vpp2",
  "nat_inside": true,
  "nat_outside": true
},
{
  "name": "vxlanBVI",
  "nat_inside": true,
  "nat_outside": true
},
{
  "name": "vpp-tap-61c1f434be0396de2d2347cbebc51a822799f3ee78f0f98a101a816",
  "nat_outside": true
},
{
  "name": "vpp-tap-860ca2dd92a3a4bf0e330affcedbe17135b7695e6ccd70e06c3eacb",
  "nat_inside": true,
  "nat_outside": true
},
{
  "name": "vpp-tap-9653fece511f2e658bcb95ec986b52532acdf540bece3652e01ff32",
  "nat_outside": true
},
{
  "name": "GigabitEthernet0/8/0",
  "nat_outside": true,
  "output_feature": true
}
```
---

## VPP NAT Pool
```json
curl http://localhost:9191/dump/vpp/v2/nat/pools
```

Sample response:
```json
[
  {
    "vrf_id": 4294967295,
    "first_ip": "192.168.16.1"
  },
  {
    "vrf_id": 4294967295,
    "first_ip": "10.1.1.254",
    "twice_nat": true
  }
]
```
---

## Linux Interfaces
```json
curl http://localhost:9191/dump/linux/v2/interfaces
```
Sample response:
```json
[
  {
    "interface": {
      "name": "tap-vpp1",
      "type": "TAP_TO_VPP",
      "hostIfName": "vpp1",
      "enabled": true,
      "ipAddresses": ["172.30.1.2/24"],
      "physAddress": "ca:aa:bc:5d:76:9c",
      "mtu": 1450,
      "tap": {
        "vppTapIfName": "tap-vpp2"
      }
    },
    "interface_meta": {
      "linux_if_index": 6,
      "parent_index": 0,
      "master_index": 0,
      "oper_state": 0,
      "flags": 69699,
      "encapsulation": "ether",
      "num_rx_queue": 0,
      "num_tx_queue": 0,
      "tx_queue_len": 500
    }
  },
// additional linux interfaces
...
```
---

## Linux ARPs
```json
curl http://localhost:9191/dump/linux/v2/arps
```
Sample response:
```json
[
  {
    "linux_arp": {
      "interface": "linux-tap-61c1f434be0396de2d2347cbebc51a822799f3ee78f0f98a101a8",
      "ip_address": "10.1.1.1",
      "hw_address": "02:fe:af:aa:51:e6"
    },
    "linux_arp_meta": {
      "interface_index": 7,
      "ip_family": 2,
      "vni": 0
    }
  },
...
```
---

## Linux Interface Stats
```json
curl http://localhost:9191/stats/linux/interfaces
```
Sample response:
```json
{
  "interface_name": "tap-vpp1",
  "interface_type": 2,
  "linux_if_index": 6,
  "rx_packets": 58919,
  "tx_packets": 54273,
  "rx_bytes": 83203332,
  "tx_bytes": 85581834,
  "rx_errors": 0,
  "tx_errors": 0,
  "rx_dropped": 0,
  "tx_dropped": 0
},
{
  "interface_name": "linux-loop-61c1f434be0396de2d2347cbebc51a822799f3ee78f0f98a101a",
  "interface_type": 3,
  "linux_if_index": 1,
  "rx_packets": 50540,
  "tx_packets": 50540,
  "rx_bytes": 4033092,
  "tx_bytes": 4033092,
  "rx_errors": 0,
  "tx_errors": 0,
  "rx_dropped": 0,
  "tx_dropped": 0
},
...
```
---

## Linux Routes
```json
curl http://localhost:9191/dump/linux/v2/routes
```
Sample response:
```json
[
  {
    "Route": {
      "outgoing_interface": "tap-vpp1",
      "dst_network": "10.1.0.0/16",
      "gw_addr": "172.30.1.1"
    },
    "Meta": {
      "interface_index": 6,
      "link_scope": 0,
      "protocol": 3,
      "mtu": 0
    }
  },
  {
    "Route": {
      "outgoing_interface": "tap-vpp1",
      "dst_network": "10.96.0.0/12",
      "gw_addr": "172.30.1.1"
    },
    "Meta": {
      "interface_index": 6,
      "link_scope": 0,
      "protocol": 3,
      "mtu": 0
    }
  },
// more Linux routes
...
```

---

## VPP IPsec SDP
```json
curl http://localhost:9191/dump/vpp/v2/ipsec/spds
```
---

## VPP IPsec SA
```json
curl http://localhost:9191/dump/vpp/v2/ipsec/sas
```
---

## VPP Punt Socket
```json
curl http://localhost:9191/dump/vpp/v2/punt/sockets
```
---

## VPP Telemetry
```json
curl http://localhost:9191/vpp/telemetry
```
---

## VPP Telemetry/Memory
```json
curl http://localhost:9191/vpp/telemetry/memory
```
---

## VPP Telemetry/Runtime
```json
curl http://localhost:9191/vpp/telemetry/runtime
```
Sample response:
```json
{
    "threads": [
        {
            "id": 0,
            "name": "ALL",
            "time": 0,
            "avg_vectors_per_node": 0,
            "last_main_loops": 0,
            "vectors_per_main_loop": 0,
            "vector_length_per_node": 0,
            "vector_rates_in": 0,
            "vector_rates_out": 0,
            "vector_rates_drop": 0,
            "vector_rates_punt": 0,
            "items": [
                {
                    "index": 0,
                    "name": "null-node",
                    "state": "",
                    "calls": 0,
                    "vectors": 0,
                    "suspends": 0,
                    "clocks": 0,
                    "vectors_per_call": 0
                },
                {
                    "index": 1,
                    "name": "vmxnet3-input",
                    "state": "",
                    "calls": 0,
                    "vectors": 0,
                    "suspends": 0,
                    "clocks": 0,
                    "vectors_per_call": 0
                },
// more items
...
```

---

## VPP Telemetry/Nodecount
```json
curl http://localhost:9191/vpp/telemetry/nodecount
```
Sample response:
```json
{
    "counters": [
        {
            "value": 0,
            "node": "/err/null-node",
            "name": "blackholed packets"
        },
        {
            "value": 0,
            "node": "/err/vmxnet3-input",
            "name": "buffer alloc error"
        },
        {
            "value": 0,
            "node": "/err/vmxnet3-input",
            "name": "Rx packet error - no SOP"
        },
// more counters
...
```

## Stats/Configurator
```json
curl http://localhost:9191/stats/configurator
```

---

# KV Scheduler 

## Dump
```json
curl http://localhost:9191/scheduler/dump
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

## Dump View&Key-Prefix 
```json
curl "http://localhost:9191/scheduler/dump?view=SB&key-prefix=config/vpp/v2/interfaces/"
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

## TXN-History
```json
curl http://localhost:9191/scheduler/txn-history
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
```json
curl http://localhost:9191/scheduler/txn-history?seq-num=1
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


## Key Timeline
```json
curl "http://localhost:9191/scheduler/key-timeline?key=config/vpp/v2/interfaces/loop1"
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

## Graph  Snapshot
```json
curl http://localhost:9191/scheduler/graph-snapshot
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
```json
curl http://localhost:9191/scheduler/status
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
```json
http://localhost:9191/scheduler/status?key=config/vpp/v2/interfaces/loop1
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
```json
curl http://localhost:9191/scheduler/flag-stats?flag=descriptor
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
```json
curl http://localhost:9191/scheduler/flag-stats?flag=last-update
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
```json
curl http://localhost:9191/scheduler/flag-stats?flag=value-state
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