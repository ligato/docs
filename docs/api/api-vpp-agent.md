# VPP Agent

---

This section describes the REST APIs supported by the VPP agent. 

---

!!! Note
    Ligato REST APIs use `9191` as the default port number. You can change this value using one of the [REST plugin configuration options][rest-plugin-config-options]. 

---

##OpenAPI

[OpenAPI](https://www.openapis.org/) (swagger) definitions provide additional details for describing, producing and consuming VPP agent REST APIs.  

<div class="swagbutton">
    <a href="https://app.swaggerhub.com/apis/chrisM9/ligato-vpp_agent_rest_ap_is/3">View OpenAPI definitions</a>
</div>

---

## RESTAPI Plugin

Description: Returns an html-formatted index of REST APIs defined in the restapi plugin [urls.go](https://github.com/ligato/vpp-agent/blob/master/plugins/restapi/resturl/urls.go).


```
curl -X GET http://localhost:9191/
```

For more details on the RESTAPI plugin, see [RESTAPI](https://github.com/ligato/vpp-agent/tree/master/plugins/restapi).

---


!!! Note
    Northbound (NB) refers to the communication between external clients and a VPP agent. You manage the __desired VPP agent configuration__ across the __NB__. See [configuration APIs](#configuration).<br></br>Southbound (SB) refers to the communication between the VPP agent and VPP data plane. VPP data plane events, notifications and __runtime VPP configuration__ dumps occur across the __SB__. See [dump APIs](#dump).


---


## Dump

The VPP agent's /dump APIs retrieve the actual VPP runtime configuration by calling the VPP binary API/CLI API in the SB.  

**VPP Interfaces**

* [GET /dump/vpp/v2/interfaces](#vpp-interfaces)
* [GET /dump/vpp/v2/interfaces/loopback](#vpp-interfacesloopback)
* [GET /dump/vpp/v2/interfaces/ethernet](#vpp-interfacesethernet)
* [GET /dump/vpp/v2/interfaces/vxlan](#vpp-interfacesvxlan)
* [GET /dump/vpp/v2/interfaces/tap](#vpp-interfacestap)
* [GET /dump/vpp/v2/interfaces/memif](#vpp-interfacesmemif)
* [GET /dump/vpp/v2/interfaces/afpacket](#vpp-interfacesafpacket)

---

**ACL/ABF**

* [GET /dump/vpp/v2/acl/ip](#vpp-acl-ip)
* [GET /dump/vpp/v2/acl/macip](#vpp-acl-macip)
* [GET /dump/vpp/v2/abf](#vpp-abf)

---

**L2**

* [GET /dump/vpp/v2/bd](#vpp-l2-bridge-domain)
* [GET /dump/vpp/v2/fib](#vpp-l2-fib)
* [GET /dump/vpp/v2/xc](#vpp-l2-x-connect)

---

**L3**

* [GET /dump/vpp/v2/routes](#vpp-l3-routes)
* [GET /dump/vpp/v2/arps](#vpp-l3-arps)
* [GET /dump/vpp/v2/ipscanneigh](#vpp-l3-ip-scan-neighbor)
* [GET /dump/vpp/v2/proxyarp/interfaces](#vpp-l3-proxy-arp-interfaces)
* [GET /dump/vpp/v2/proxyarp/ranges](#vpp-l3-proxy-arp-ranges)
* [GET /dump/vpp/v2/vrrps](#vpp-l3-vrrp)

---

**NAT**

* [GET /dump/vpp/v2/nat/global](#vpp-nat-global)
* [GET /dump/vpp/v2/nat/dnat](#vpp-nat-dnat)
* [GET /dump/vpp/v2/nat/interfaces](#vpp-nat-interfaces)
* [GET /dump/vpp/v2/nat/pools](#vpp-nat-pools)

---

**VPP IPsec**

* [GET /dump/vpp/v2/ipsec/spds](#vpp-ipsec-spd)
* [GET /dump/vpp/v2/ipsec/sas](#vpp-ipsec-sa)
* [GET /dump/vpp/v2/ipsec/sps](#vpp-ipsec-sp)

---

**VPP Punt Socket**

* [GET /dump/vpp/v2/punt/sockets](#vpp-punt-socket)

**Linux**

* [GET /dump/linux/v2/interfaces](#linux-interfaces)
* [GET /dump/linux/v2/arps](#linux-l3-arps)
* [GET /dump/linux/v2/routes](#linux-l3-routes)
* [GET /dump/stats/linux/interfaces](#linux-interface-stats)


---


### VPP Interfaces

Description: GET all interfaces.

```
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces
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

### VPP Interfaces/Loopback

Description: GET loopback interfaces.

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

### VPP Interfaces/Ethernet

Description: GET ethernet interfaces.

```json
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/ethernet
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

### VPP Interfaces/Vxlan

Description: GET vxlan interfaces.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/vxlan
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

### VPP Interfaces/Tap

Description: GET tap interfaces.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/tap
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

### VPP Interfaces/Memif

Description: GET memif interfaces.
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

### VPP Interfaces/Afpacket

Description: GET afpacket interfaces.

```json
curl -X GET http://localhost:9191/dump/vpp/v2/interfaces/afpacket
```
---

### VPP ACL IP

Description: GET IP ACL entries.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/acl/ip
```

Sample response:
```json
[
    {
        "acl": {
            "name": "acl1",
            "rules": [
                {
                    "action": 1,
                    "ip_rule": {
                        "ip": {
                            "destination_network": "10.20.1.0/24",
                            "source_network": "192.168.1.2/32"
                        },
                        "tcp": {
                            "destination_port_range": {
                                "lower_port": 1150,
                                "upper_port": 1250
                            },
                            "source_port_range": {
                                "lower_port": 150,
                                "upper_port": 250
                            },
                            "tcp_flags_mask": 20,
                            "tcp_flags_value": 10
                        }
                    }
                }
            ],
            "interfaces": {}
        },
        "acl_meta": {
            "acl_index": 0,
            "acl_tag": "acl1"
        }
    }
]
```

---

### VPP ACL MACIP

Description: GET MACIP ACL entries.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/acl/macip
```
---

### VPP ABF

Description: GET ABF entries.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/abf
```
---

### VPP L2 Bridge Domain

Description: GET bridge domain information.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/bd
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

### VPP L2 FIB

Description: GET L2 FIB information.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/fib
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

### VPP L2 X-connect

Description: GET L2 x-connect information.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/xc
```
---

### VPP L3 Routes

Description: GET L3 routing table entries.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/routes
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


### VPP L3 ARPs

Description: GET ARP entries.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/arps
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

### VPP L3 IP Scan Neighbor

Description: GET L3 IP scan neighbor data.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/ipscanneigh
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

### VPP L3 Proxy ARP Interfaces

Description: GET proxy ARP interface information.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/proxyarp/interfaces
```

---

### VPP L3 Proxy ARP Ranges

Description: GET proxy ARP range information.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/proxyarp/ranges
```

---

### VPP L3 VRRP

Description: GET VRRP

```
curl -X GET http://localhost:9191/dump/vpp/v2/vrrps
```

---

### VPP NAT Global

Description: GET NAT global configuration information.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/nat/global
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

### VPP NAT DNAT

Description: GET destination-based NAT entries.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/nat/dnat
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

### VPP NAT Interfaces

Description: GET interface-specific NAT information.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/nat/interfaces
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

### VPP NAT Pool

Description: GET NAT address pool information.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/nat/pools
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

### VPP IPsec SPD

Description: GET IPsec security policy database (SPD) entries. 
```json
curl -X GET http://localhost:9191/dump/vpp/v2/ipsec/spds
```
Sample response:
```json
[
  {
    "index": 1,
    "interfaces": [
      {
        "name": "tap1"
      }
    ]
  }
]
```

---

### VPP IPsec SA

Description: GET IPsec security association (SA) entries.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/ipsec/sas
```
Sample response:
```json
[
  {
    "Sa": {
      "index": 1,
      "spi": 1001,
      "protocol": 1,
      "crypto_alg": 1,
      "crypto_key": "4a506a794f574265564551694d653768",
      "integ_alg": 2,
      "integ_key": "4339314b55523947594d6d3547666b45764e6a58"
    },
    "Meta": {
      "SaID": 1,
      "Interface": "",
      "IfIdx": 4294967295,
      "CryptoKeyLen": 0,
      "IntegKeyLen": 0,
      "Salt": 0,
      "SeqOutbound": 0,
      "LastSeqInbound": 0,
      "ReplayWindow": 0,
      "TotalDataSize": 0
    }
  }
]

```

---

### VPP IPsec SP

Description: GET IPsec security policies
```json
curl -X GET http://localhost:9191/dump/vpp/v2/ipsec/sps
``` 

Sample response:
```json
[
  {
    "spd_index": 1,
    "sa_index": 1,
    "priority": 5,
    "is_outbound": true,
    "remote_addr_start": "0.0.0.0",
    "remote_addr_stop": "255.255.255.255",
    "local_addr_start": "0.0.0.0",
    "local_addr_stop": "255.255.255.255",
    "protocol": 4,
    "remote_port_start": 65535,
    "local_port_start": 65535,
    "action": 3
  }
]
```

---

### VPP Punt Socket

Description: GET punt socket data.
```json
curl -X GET http://localhost:9191/dump/vpp/v2/punt/sockets
```
---

---

### Linux Interfaces

Description: GET all Linux interfaces.
```json
curl -X GET http://localhost:9191/dump/linux/v2/interfaces
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

### Linux L3 ARPs

Description: GET Linux L3 ARP entries.
```json
curl -X GET http://localhost:9191/dump/linux/v2/arps
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

### Linux Interface Stats

Description: GET Linux interface stats.
```json
curl -X GET http://localhost:9191/stats/linux/interfaces
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

### Linux L3 Routes

Description: GET Linux L3 routing table entries.
```json
curl -X GET http://localhost:9191/dump/linux/v2/routes
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

## Configuration

The northbound (NB) VPP agent /configuration APIs support the GET, PUT and validate operations of VPP configuration information in the NB. This occurs between an external REST client and the agent's plugins that expose REST APIs.

- [PUT /configuration](#config-update)
- [PUT /configuration?replace](#config-replace)
- [GET /configuration](#config-get)
- [GET /info/configuration/jsonschema](#config-json-schema)
- [POST /configuration/validate](#config-validate) 

---

### Config Update

Description: PUT NB VPP agent configuration update
```
curl -X PUT -H "Content-Type: application/yaml" --data-binary @loop1.yaml http://localhost:9191/configuration
```
Sample `loop1.yaml` configuration
```
# loop1.yaml 
vppConfig:
  interfaces:
  - name: loop1
    type: SOFTWARE_LOOPBACK
    enabled: true
    ipAddresses:
    - 192.168.1.1/24
```

You can embed the yaml configuration in an http request body.

---

### Config Replace

Description: PUT NB Agent configuration update with a replacement
```
curl -X PUT -H "Content-Type: application/yaml" --data-binary @loop3.yaml "http://localhost:9191/configuration?replace=true"
```
Sample `loop3.yaml` file containing the replacement configuration
```
vppConfig:
  interfaces:
  - name: "loop1"
    type: SOFTWARE_LOOPBACK
    enabled: true
    ipAddresses:
    - 192.168.1.3/24
```

You can embed the yaml configuration in an http request body.


---

### Config Get

Description: GET NB VPP agent configuration
```
curl -X GET http://localhost:9191/configuration
``` 
Sample output:
```
netallocConfig: {}
linuxConfig: {}
vppConfig:
  interfaces:
  - name: loop1
    type: SOFTWARE_LOOPBACK
    enabled: true
    ipAddresses:
    - 192.168.1.3/24
```

---

### Config JSON Schema

Description: GET JSON Schema for VPP agent configuration
```
curl -X GET http://localhost:9191/info/configuration/jsonschema
```

Sample response:
```
                // Partial response
               "bridgeDomains": {
                    "items": {
                        "$schema": "http://json-schema.org/draft-04/schema#",
                        "properties": {
                            "name": {
                                "type": "string"
                            },
                            "flood": {
                                "type": "boolean"
                            },
                            "unknown_unicast_flood": {
                                "type": "boolean"
                            },
                            "unknownUnicastFlood": {
                                "type": "boolean"
                            },
                            "forward": {
                                "type": "boolean"
                            },
                            "learn": {
                                "type": "boolean"
                            },
                            "arp_termination": {
                                "type": "boolean"
                            },
                            "arpTermination": {
                                "type": "boolean"
                            },
                            "mac_age": {
                                "type": "integer"
                            },
                            "macAge": {
                                "type": "integer"
                            },
                            "interfaces": {
                                "items": {
                                    "$schema": "http://json-schema.org/draft-04/schema#",
```

---

### Config Validate

Description: Validates the yaml configuration
```
curl -X POST -H "Content-Type: application/yaml" --data-binary @good-yaml-config.yaml "http://localhost:9191/configuration/validate"
```

Response is nil if the validation check passes.

Sample good yaml config file:
```
# good yaml file
netallocConfig: {}
linuxConfig: {}
vppConfig:
  interfaces:
    - name: vpp-tap1
      type: TAP
      enabled: true
      ipAddresses:
        - 10.10.10.1/24
      tap:
        version: 2
        toMicroservice: test-microservice1
    - name: red
      type: MEMIF
      enabled: true
      ipAddresses:
        - 100.0.0.1/24
      mtu: 9200
      memif:
        id: 1
        socketFilename: /var/run/memif_k8s-master.sock
    - name: black
      type: MEMIF
      enabled: true
      ipAddresses:
        - 192.168.20.1/24
      mtu: 9200
      memif:
        id: 2
        socketFilename: /var/run/memif_k8s-master.sock
    - name: loop-test-1
      type: SOFTWARE_LOOPBACK
      enabled: true
      ipAddresses:
        - 10.10.1.1/24
      mtu: 1500
  srv6Global:
    encapSourceAddress: 10.1.1.1
```

---

Response contains errors if the validation check fails.

Sample bad yaml config file:
```
netallocConfig: {}
linuxConfig: {}
vppConfig:
  routes:
    - type: INTRA_VRF
      dstNetwork: 10.1.0.0/16
      nextHopAddr: 4.10.x10.1
      outgoingInterface: cgnat-to-gw
    - type: INTRA_VRF
      dstNetwork: 10.2.0.0/16
      nextHopAddr: 5.10.10.1x
      outgoingInterface: cgnat-to-gw
  srv6Localsids:
    - sid: A::0
      endFunctionDx6:
        outgoingInterface: test
        nextHop: B::X
  proxyArp:
    ranges:
      - firstIpAddr: 10.0.0.1x      
        lastIpAddr: 10.0.0.2
      - firstIpAddr: 10.0.0.3      
        lastIpAddr: 10.0.0.4
```

Sample response after running the validate API against the bad yaml file:
```
[
  {
    "Path": "vppConfig.routes*.gw_addr",
    "Error": "invalid IP address: 4.10.x10.1",
    "ErrorConfigPart": "dstNetwork: 10.1.0.0/16\nnextHopAddr: 4.10.x10.1\noutgoingInterface: cgnat-to-gw\n"
  },
  {
    "Path": "vppConfig.routes*.gw_addr",
    "Error": "invalid IP address: 5.10.10.1x",
    "ErrorConfigPart": "dstNetwork: 10.2.0.0/16\nnextHopAddr: 5.10.10.1x\noutgoingInterface: cgnat-to-gw\n"
  },
  {
    "Path": "vppConfig.proxyArp.ranges.firstIpAddr",
    "Error": "invalid IP address"
  },
  {
    "Path": "vppConfig.srv6Localsids*.EndFunctionDX6.NextHop",
    "Error": "failed to parse next hop B::X, should be a valid ipv6 address:  \"B::X\" is not ip address",
    "ErrorConfigPart": "endFunctionDx6:\n  nextHop: B::X\n  outgoingInterface: test\nsid: A::0\n"
  }
]
```

You can embed the yaml configuration in an http request body. 

---

## Telemetry

**VPP Telemetry**

* [GET /dump/vpp/telemetry](#vpp-telemetry)
* [GET /dump/vpp/telemetry/memory](#vpp-telemetrymemory)
* [GET /dump/vpp/telemetry/runtime](#vpp-telemetryruntime)
* [GET /dump/vpp/telemetry/nodecount](#vpp-telemetrynodecount)

### VPP Telemetry

Description: GET VPP telemetry information.
```json
curl -X GET http://localhost:9191/vpp/telemetry
```
---

### VPP Telemetry/Memory

Description: GET VPP telemetry memory stats.
```json
curl -X GET http://localhost:9191/vpp/telemetry/memory
```

Sample response:
```json
{
  "threads": [
    {
      "id": 0,
      "name": "north1",
      "size": 0,
      "objects": 0,
      "used": 0,
      "total": 0,
      "free": 0,
      "reclaimed": 0,
      "overhead": 0,
      "pages": 0,
      "page_size": 0
    }
  ]
}
```

---

### VPP Telemetry/Runtime

Description: GET VPP telemetry runtime stats.
```json
curl -X GET http://localhost:9191/vpp/telemetry/runtime
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

### VPP Telemetry/Nodecount

Description: GET VPP telemetry nodecount stats.
```json
curl -X GET http://localhost:9191/vpp/telemetry/nodecount
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

---

## Version

Description: GET VPP Agent Version Info
```json
curl -X GET http://localhost:9191/info/version
```

Sample Response:
```json
{
  "App": "vpp-agent",
  "Version": "v3.2.0-alpha-22-ge9aa3556d",
  "GitCommit": "e9aa3556defe818904670e3f5051246fdd11746d",
  "GitBranch": "HEAD",
  "BuildUser": "root",
  "BuildHost": "06a7eb7fd825",
  "BuildTime": 1593781318,
  "GoVersion": "go1.14.4",
  "OS": "linux",
  "Arch": "amd64"
}
```
---

## VPP CLI

Description: Execute VPP CLI commands through a REST API.

For a `show version` command, use

```json
curl -X POST   'http://localhost:9191/vpp/command?Content-Type=application/json'   -H 'Content-Type: application/json' -d '{"vppclicommand":"show version"}'
```
Sample response:
```json
"vpp v20.01-rc2~11-gfce396738~b17 built by root on b81dced13911 at 2020-01-29T21:07:15\n"
```

---

## Stats/Configurator

Description: GET configurator operations stats. 
```json
curl -X GET http://localhost:9191/stats/configurator
```
---









[agentctl]: ../user-guide/agentctl.md
[quickstart-guide]: ../user-guide/quickstart.md
[rest-plugin-config-options]: ../plugins/connection-plugins.md#configuration