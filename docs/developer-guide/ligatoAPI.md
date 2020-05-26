# VPP Agent

## REST API Index

```
curl -X GET http://localhost:9191/
```
## Interfaces
```
curl http://localhost:32500/api/contiv/dump/vpp/v2/interfaces
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
    "2": {
        "interface": {
            "name": "podGwLoop",
            "type": "SOFTWARE_LOOPBACK",
            "enabled": true,
            "physAddress": "de:ad:00:00:00:00",
            "ipAddresses": [
                "10.1.1.1/24"
            ],
            "vrf": 1
        },
        "interface_meta": {
            "sw_if_index": 2,
            "sub_sw_if_index": 2,
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
            "tag": "podGwLoop",
            "dhcp": null,
            "vrf_ipv4": 1,
            "vrf_ipv6": 0,
            "pci": 0
        }
    },
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
    "4": {
        "interface": {
            "name": "vxlanBVI",
            "type": "SOFTWARE_LOOPBACK",
            "enabled": true,
            "physAddress": "12:2b:00:00:00:01",
            "ipAddresses": [
                "192.168.30.1/24"
            ],
            "vrf": 1
        },
        "interface_meta": {
            "sw_if_index": 4,
            "sub_sw_if_index": 4,
            "l2_address": "EisAAAAB",
            "internal_name": "loop1",
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
            "tag": "vxlanBVI",
            "dhcp": null,
            "vrf_ipv4": 1,
            "vrf_ipv6": 0,
            "pci": 0
        }
    },
    "5": {
        "interface": {
            "name": "vpp-tap-054072f2c3954ab93b479d36b391f264f0096b1c1d374a4afb45ef4",
            "type": "TAP",
            "enabled": true,
            "physAddress": "02:fe:1c:66:2f:b3",
            "vrf": 1,
            "mtu": 1450,
            "unnumbered": {
                "interfaceWithIp": "podGwLoop"
            },
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
                "hostIfName": "tap-2008519863",
                "rxRingSize": 1024,
                "txRingSize": 1024
            }
        },
        "interface_meta": {
            "sw_if_index": 5,
            "sub_sw_if_index": 5,
            "l2_address": "Av4cZi+z",
            "internal_name": "tap1",
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
            "tag": "vpp-tap-054072f2c3954ab93b479d36b391f264f0096b1c1d374a4afb45ef4",
            "dhcp": null,
            "vrf_ipv4": 1,
            "vrf_ipv6": 0,
            "pci": 0
        }
    },
    "6": {
        "interface": {
            "name": "vpp-tap-400dc32e50d03aeff4718c568bf5520cdc0addc1196f3c9d58dc1ec",
            "type": "TAP",
            "enabled": true,
            "physAddress": "02:fe:2b:f2:a0:9b",
            "vrf": 1,
            "mtu": 1450,
            "unnumbered": {
                "interfaceWithIp": "podGwLoop"
            },
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
                "hostIfName": "tap-3161137050",
                "rxRingSize": 1024,
                "txRingSize": 1024
            }
        },
        "interface_meta": {
            "sw_if_index": 6,
            "sub_sw_if_index": 6,
            "l2_address": "Av4r8qCb",
            "internal_name": "tap2",
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
            "tag": "vpp-tap-400dc32e50d03aeff4718c568bf5520cdc0addc1196f3c9d58dc1ec",
            "dhcp": null,
            "vrf_ipv4": 1,
            "vrf_ipv6": 0,
            "pci": 0
        }
    },
    "7": {
        "interface": {
            "name": "vpp-tap-83dc59e9763c663bc2693ae1bd50a4982a536ccac3034518ea9e6a2",
            "type": "TAP",
            "enabled": true,
            "physAddress": "02:fe:5d:ae:58:9a",
            "vrf": 1,
            "mtu": 1450,
            "unnumbered": {
                "interfaceWithIp": "podGwLoop"
            },
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
                "hostIfName": "tap-3918432989",
                "rxRingSize": 1024,
                "txRingSize": 1024
            }
        },
        "interface_meta": {
            "sw_if_index": 7,
            "sub_sw_if_index": 7,
            "l2_address": "Av5drlia",
            "internal_name": "tap3",
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
            "tag": "vpp-tap-83dc59e9763c663bc2693ae1bd50a4982a536ccac3034518ea9e6a2",
            "dhcp": null,
            "vrf_ipv4": 1,
            "vrf_ipv6": 0,
            "pci": 0
        }
    },
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
```
## VPP Interfaces/Loopback

```
http://localhost:32500/api/contiv/dump/vpp/v2/interfaces/loopback
```
Sample response:
```json
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
  "2": {
    "interface": {
      "name": "podGwLoop",
      "type": "SOFTWARE_LOOPBACK",
      "enabled": true,
      "physAddress": "de:ad:00:00:00:00",
      "ipAddresses": [
        "10.1.1.1/24"
      ],
      "vrf": 1
    },
    "interface_meta": {
      "sw_if_index": 2,
      "sub_sw_if_index": 2,
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
      "tag": "podGwLoop",
      "dhcp": null,
      "vrf_ipv4": 1,
      "vrf_ipv6": 0,
      "pci": 0
    }
  },
  "5": {
    "interface": {
      "name": "vxlanBVI",
      "type": "SOFTWARE_LOOPBACK",
      "enabled": true,
      "physAddress": "12:2b:00:00:00:01",
      "ipAddresses": [
        "192.168.30.1/24"
      ],
      "vrf": 1
    },
    "interface_meta": {
      "sw_if_index": 5,
      "sub_sw_if_index": 5,
      "l2_address": "EisAAAAB",
      "internal_name": "loop1",
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
      "tag": "vxlanBVI",
      "dhcp": null,
      "vrf_ipv4": 1,
      "vrf_ipv6": 0,
      "pci": 0
    }
  }
}
```
## VPP Interfaces/Ethernet
```json
http://localhost:32500/api/contiv/dump/vpp/v2/interfaces/ethernet
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
```
## VPP Interfaces/Vxlan
```json
http://localhost:32500/api/contiv/dump/vpp/v2/interfaces/vxlan
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
```
## VPP Interfaces/Tap
```json
http://localhost:32500/api/contiv/dump/vpp/v2/interfaces/tap
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
  "6": {
    "interface": {
      "name": "vpp-tap-61c1f434be0396de2d2347cbebc51a822799f3ee78f0f98a101a816",
      "type": "TAP",
      "enabled": true,
      "physAddress": "02:fe:af:aa:51:e6",
      "vrf": 1,
      "mtu": 1450,
      "unnumbered": {
        "interfaceWithIp": "podGwLoop"
      },
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
        "hostIfName": "tap-283567301",
        "rxRingSize": 1024,
        "txRingSize": 1024
      }
    },
    "interface_meta": {
      "sw_if_index": 6,
      "sub_sw_if_index": 6,
      "l2_address": "Av6vqlHm",
      "internal_name": "tap1",
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
      "tag": "vpp-tap-61c1f434be0396de2d2347cbebc51a822799f3ee78f0f98a101a816",
      "dhcp": null,
      "vrf_ipv4": 1,
      "vrf_ipv6": 0,
      "pci": 0
    }
  },
  "7": {
    "interface": {
      "name": "vpp-tap-860ca2dd92a3a4bf0e330affcedbe17135b7695e6ccd70e06c3eacb",
      "type": "TAP",
      "enabled": true,
      "physAddress": "02:fe:1b:0d:d5:29",
      "vrf": 1,
      "mtu": 1450,
      "unnumbered": {
        "interfaceWithIp": "podGwLoop"
      },
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
        "hostIfName": "tap-1566076365",
        "rxRingSize": 1024,
        "txRingSize": 1024
      }
    },
    "interface_meta": {
      "sw_if_index": 7,
      "sub_sw_if_index": 7,
      "l2_address": "Av4bDdUp",
      "internal_name": "tap2",
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
      "tag": "vpp-tap-860ca2dd92a3a4bf0e330affcedbe17135b7695e6ccd70e06c3eacb",
      "dhcp": null,
      "vrf_ipv4": 1,
      "vrf_ipv6": 0,
      "pci": 0
    }
  },
  "8": {
    "interface": {
      "name": "vpp-tap-9653fece511f2e658bcb95ec986b52532acdf540bece3652e01ff32",
      "type": "TAP",
      "enabled": true,
      "physAddress": "02:fe:63:8c:a0:9e",
      "vrf": 1,
      "mtu": 1450,
      "unnumbered": {
        "interfaceWithIp": "podGwLoop"
      },
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
        "hostIfName": "tap-495714737",
        "rxRingSize": 1024,
        "txRingSize": 1024
      }
    },
    "interface_meta": {
      "sw_if_index": 8,
      "sub_sw_if_index": 8,
      "l2_address": "Av5jjKCe",
      "internal_name": "tap3",
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
      "tag": "vpp-tap-9653fece511f2e658bcb95ec986b52532acdf540bece3652e01ff32",
      "dhcp": null,
      "vrf_ipv4": 1,
      "vrf_ipv6": 0,
      "pci": 0
    }
  }
}
```
## VPP ARPs
```json
http://localhost:32500/api/contiv/dump/vpp/v2/arps
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
```
## VPP Bridge Domain
```json
http://localhost:32500/api/contiv/dump/vpp/v2/bd
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
## VPP L2 FIB
```json
http://localhost:32500/api/contiv/dump/vpp/v2/fib
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
## VPP L3 Routes
```json
http://localhost:32500/api/contiv/dump/vpp/v2/routes
```
Sample response:
```json

```