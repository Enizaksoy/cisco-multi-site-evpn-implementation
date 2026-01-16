# BGP EVPN VXLAN Design Overview

## Topology Summary

### POD 1 (Main Fabric with Super-Spines)
```
                    ┌─────────────────────────────────────┐
                    │         SUPER-SPINE LAYER           │
                    │   Super-Spine-1    Super-Spine-2    │
                    │    AS 65000          AS 65000       │
                    │   Lo0: 10.0.0.1    Lo0: 10.0.0.2    │
                    └─────────────────────────────────────┘
                              │  │  │  │
                    ┌─────────────────────────────────────┐
                    │           SPINE LAYER               │
                    │ Spine-1  Spine-2  Spine-3  Spine-4  │
                    │ AS 65001 AS 65002 AS 65003 AS 65004 │
                    │ 10.0.1.1 10.0.1.2 10.0.1.3 10.0.1.4 │
                    └─────────────────────────────────────┘
                              │  │  │  │
                    ┌─────────────────────────────────────┐
                    │           LEAF LAYER (VTEPs)        │
                    │  Leaf-1   Leaf-2   Leaf-3   Leaf-4  │
                    │ AS 65101 AS 65102 AS 65103 AS 65104 │
                    │ 10.0.2.1 10.0.2.2 10.0.2.3 10.0.2.4 │
                    │    VTEP     VTEP     VTEP     VTEP  │
                    └─────────────────────────────────────┘
                              │       │       │       │
                          iosvl2-0 iosvl2-1 iosvl2-2 iosvl2-3
```

### POD 2 (Standalone Spine-Leaf)
```
                    ┌─────────────────────────────────────┐
                    │           SPINE LAYER               │
                    │       Spine-5        Spine-6        │
                    │      AS 65005       AS 65006        │
                    │     10.0.1.5        10.0.1.6        │
                    └─────────────────────────────────────┘
                              │     ╲  ╱     │
                              │      ╲╱      │
                              │      ╱╲      │
                              │     ╱  ╲     │
                    ┌─────────────────────────────────────┐
                    │           LEAF LAYER (VTEPs)        │
                    │       Leaf-5          Leaf-6        │
                    │      AS 65105        AS 65106       │
                    │      10.0.2.5        10.0.2.6       │
                    │        VTEP            VTEP         │
                    └─────────────────────────────────────┘
                              │                │
                          iosvl2-4         iosvl2-5
```

## Device Inventory

### Management IP Addresses
| Device | Management IP | BGP ASN | Role | Loopback0 |
|--------|--------------|---------|------|-----------|
| Super-Spine-1 | 192.168.30.118 | 65000 | Route Reflector | 10.0.0.1/32 |
| Super-Spine-2 | 192.168.30.119 | 65000 | Route Reflector | 10.0.0.2/32 |
| Spine-1 | 192.168.30.110 | 65001 | Spine/Transit | 10.0.1.1/32 |
| Spine-2 | 192.168.30.111 | 65002 | Spine/Transit | 10.0.1.2/32 |
| Spine-3 | 192.168.30.112 | 65003 | Spine/Transit | 10.0.1.3/32 |
| Spine-4 | 192.168.30.113 | 65004 | Spine/Transit | 10.0.1.4/32 |
| Spine-5 | 192.168.30.122 | 65005 | Spine/Transit (POD2) | 10.0.1.5/32 |
| Spine-6 | 192.168.30.123 | 65006 | Spine/Transit (POD2) | 10.0.1.6/32 |
| Leaf-1 | 192.168.30.114 | 65101 | VTEP | 10.0.2.1/32 |
| Leaf-2 | 192.168.30.115 | 65102 | VTEP | 10.0.2.2/32 |
| Leaf-3 | 192.168.30.116 | 65103 | VTEP | 10.0.2.3/32 |
| Leaf-4 | 192.168.30.117 | 65104 | VTEP | 10.0.2.4/32 |
| Leaf-5 | 192.168.30.120 | 65105 | VTEP (POD2) | 10.0.2.5/32 |
| Leaf-6 | 192.168.30.121 | 65106 | VTEP (POD2) | 10.0.2.6/32 |
| iosvl2-0 | 192.168.30.130 | - | Edge Switch | - |
| iosvl2-1 | 192.168.30.131 | - | Edge Switch | - |
| iosvl2-2 | 192.168.30.132 | - | Edge Switch | - |
| iosvl2-3 | 192.168.30.133 | - | Edge Switch | - |
| iosvl2-4 | 192.168.30.134 | - | Edge Switch (POD2) | - |
| iosvl2-5 | 192.168.30.135 | - | Edge Switch (POD2) | - |

## Interface Connections

### POD 1 - Super-Spine-1
| Interface | Connects To | IP Address |
|-----------|-------------|------------|
| E1/1 | Spine-1 E1/5 | 10.1.1.0/31 |
| E1/2 | Spine-2 E1/5 | 10.1.1.2/31 |
| E1/3 | Spine-3 E1/5 | 10.1.1.4/31 |
| E1/4 | Spine-4 E1/5 | 10.1.1.6/31 |

### POD 1 - Super-Spine-2
| Interface | Connects To | IP Address |
|-----------|-------------|------------|
| E1/1 | Spine-1 E1/6 | 10.1.2.0/31 |
| E1/2 | Spine-2 E1/6 | 10.1.2.2/31 |
| E1/3 | Spine-3 E1/6 | 10.1.2.4/31 |
| E1/4 | Spine-4 E1/6 | 10.1.2.6/31 |

### POD 1 - Spine-1
| Interface | Connects To | IP Address |
|-----------|-------------|------------|
| E1/5 | Super-Spine-1 E1/1 | 10.1.1.1/31 |
| E1/6 | Super-Spine-2 E1/1 | 10.1.2.1/31 |
| E1/1 | Leaf-1 E1/1 | 10.2.1.0/31 |
| E1/2 | Leaf-2 E1/1 | 10.2.1.2/31 |
| E1/3 | Leaf-3 E1/1 | 10.2.1.4/31 |
| E1/4 | Leaf-4 E1/1 | 10.2.1.6/31 |

### POD 1 - Spine-2
| Interface | Connects To | IP Address |
|-----------|-------------|------------|
| E1/5 | Super-Spine-1 E1/2 | 10.1.1.3/31 |
| E1/6 | Super-Spine-2 E1/2 | 10.1.2.3/31 |
| E1/1 | Leaf-1 E1/2 | 10.2.2.0/31 |
| E1/2 | Leaf-2 E1/2 | 10.2.2.2/31 |
| E1/3 | Leaf-3 E1/2 | 10.2.2.4/31 |
| E1/4 | Leaf-4 E1/2 | 10.2.2.6/31 |

### POD 1 - Spine-3
| Interface | Connects To | IP Address |
|-----------|-------------|------------|
| E1/5 | Super-Spine-1 E1/3 | 10.1.1.5/31 |
| E1/6 | Super-Spine-2 E1/3 | 10.1.2.5/31 |
| E1/1 | Leaf-1 E1/3 | 10.2.3.0/31 |
| E1/2 | Leaf-2 E1/3 | 10.2.3.2/31 |
| E1/3 | Leaf-3 E1/3 | 10.2.3.4/31 |
| E1/4 | Leaf-4 E1/3 | 10.2.3.6/31 |

### POD 1 - Spine-4
| Interface | Connects To | IP Address |
|-----------|-------------|------------|
| E1/5 | Super-Spine-1 E1/4 | 10.1.1.7/31 |
| E1/6 | Super-Spine-2 E1/4 | 10.1.2.7/31 |
| E1/1 | Leaf-1 E1/4 | 10.2.4.0/31 |
| E1/2 | Leaf-2 E1/4 | 10.2.4.2/31 |
| E1/3 | Leaf-3 E1/4 | 10.2.4.4/31 |
| E1/4 | Leaf-4 E1/4 | 10.2.4.6/31 |

### POD 1 - Leaf Interfaces (to Spines)
| Leaf | E1/1 (Spine-1) | E1/2 (Spine-2) | E1/3 (Spine-3) | E1/4 (Spine-4) | E1/5 (Edge) |
|------|----------------|----------------|----------------|----------------|-------------|
| Leaf-1 | 10.2.1.1/31 | 10.2.2.1/31 | 10.2.3.1/31 | 10.2.4.1/31 | iosvl2-0 |
| Leaf-2 | 10.2.1.3/31 | 10.2.2.3/31 | 10.2.3.3/31 | 10.2.4.3/31 | iosvl2-1 |
| Leaf-3 | 10.2.1.5/31 | 10.2.2.5/31 | 10.2.3.5/31 | 10.2.4.5/31 | iosvl2-2 |
| Leaf-4 | 10.2.1.7/31 | 10.2.2.7/31 | 10.2.3.7/31 | 10.2.4.7/31 | iosvl2-3 |

### POD 2 - Spine-5
| Interface | Connects To | IP Address |
|-----------|-------------|------------|
| E1/1 | Leaf-5 E1/1 | 10.2.5.0/31 |
| E1/2 | Leaf-6 E1/1 | 10.2.5.2/31 |

### POD 2 - Spine-6
| Interface | Connects To | IP Address |
|-----------|-------------|------------|
| E1/1 | Leaf-5 E1/2 | 10.2.6.0/31 |
| E1/2 | Leaf-6 E1/2 | 10.2.6.2/31 |

### POD 2 - Leaf Interfaces
| Leaf | E1/1 (Spine-5) | E1/2 (Spine-6) | E1/5 (Edge) |
|------|----------------|----------------|-------------|
| Leaf-5 | 10.2.5.1/31 | 10.2.6.1/31 | iosvl2-4 |
| Leaf-6 | 10.2.5.3/31 | 10.2.6.3/31 | iosvl2-5 |

## VXLAN / EVPN Design

### VRF Configuration
- **VRF Name:** mylab
- **VNI (L3VNI):** 50000
- **Route-Target:** 65000:50000

### VLAN to VNI Mapping
| VLAN | VNI | Anycast Gateway | Description |
|------|-----|-----------------|-------------|
| 10 | 10010 | 192.168.10.1/24 | VLAN 10 - Data |
| 20 | 10020 | 192.168.20.1/24 | VLAN 20 - Voice |
| 30 | 10030 | 192.168.30.1/24 | VLAN 30 - Management |
| 40 | 10040 | 192.168.40.1/24 | VLAN 40 - Guest |

### Route-Targets (Common across all VTEPs)
| VNI | Route-Target |
|-----|--------------|
| 10010 | 65000:10010 |
| 10020 | 65000:10020 |
| 10030 | 65000:10030 |
| 10040 | 65000:10040 |
| 50000 | 65000:50000 |

### Anycast Gateway
- **Virtual MAC:** 0000.1111.2222
- Configured on all Leaf switches (VTEPs)
- Same IP and MAC across all leaves for seamless VM mobility

## BGP Design

### Underlay (IPv4)
- eBGP between all layers
- Advertise loopback addresses for VTEP reachability

### Overlay (EVPN)
- eBGP EVPN address-family
- Route-Type 2: MAC/IP advertisement
- Route-Type 3: IMET (Inclusive Multicast Ethernet Tag)
- Route-Type 5: IP Prefix (for L3VNI)

## Device Credentials
- **NX-OS Devices:** admin / Versa@123!!
- **IOSvL2 Switches:** admin / Versa@123!!
- **CML Console:** admin / Elma12743??

## Management Network
- **Subnet:** 192.168.30.0/24
- **Gateway:** 192.168.30.4
