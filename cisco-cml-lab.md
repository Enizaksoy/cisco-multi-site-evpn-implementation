# CML Lab Automation - EVPN VXLAN Multi-Site

## Lab Overview
- **Lab Name**: VXLAN_MANUAL
- **Lab ID**: 69a8835a-c3ca-46f8-b0f4-0d57f4934d29
- **Topology**: 5-Stage Clos EVPN VXLAN Multi-Site

## Access Information

### CML Server
- **URL**: https://192.168.20.65
- **Username**: admin
- **Password**: Elma12743??
- **Console Server**: SSH to admin@192.168.20.65

### NX-OS Devices (Spines, Leafs)
- **Username**: admin
- **Password**: Versa@123!!

### CSR1000v Devices (Access Layer)
- **Username**: admin
- **Password**: Versa@123!!

## Device Inventory

### Spine Switches (NX-OS 9000v)
| Device | Management IP | Role |
|--------|--------------|------|
| Spine-1 | 192.168.30.110 | POD1 Spine |
| Spine-2 | 192.168.30.111 | POD1 Spine |
| Spine-3 | 192.168.30.112 | POD2 Spine |
| Spine-4 | 192.168.30.113 | POD2 Spine |
| Spine-5 | 192.168.30.118 | POD3 Spine |
| Spine-6 | 192.168.30.119 | POD3 Spine |

### Leaf Switches (NX-OS 9000v) - vPC Pairs
| Device | Management IP | vPC Domain | vPC Role | Loopback0 |
|--------|--------------|------------|----------|-----------|
| Leaf-1 | 192.168.30.114 | 1 | Primary | 10.0.2.1 |
| Leaf-2 | 192.168.30.115 | 1 | Secondary | 10.0.2.2 |
| Leaf-3 | 192.168.30.116 | 2 | Primary | 10.0.2.3 |
| Leaf-4 | 192.168.30.117 | 2 | Secondary | 10.0.2.4 |
| Leaf-5 | 192.168.30.120 | 3 | Primary | 10.0.2.5 |
| Leaf-6 | 192.168.30.121 | 3 | Secondary | 10.0.2.6 |

### Super Spines (NX-OS 9000v)
| Device | Management IP | Role |
|--------|--------------|------|
| Super-Spine-1 | 192.168.30.122 | DCI |
| Super-Spine-2 | 192.168.30.123 | DCI |

### CSR1000v Access Routers (Replaced IOL)
| Device | Management IP | Connected Leafs | Port-Channel |
|--------|--------------|-----------------|--------------|
| csr-0 | 192.168.30.130 | Leaf-1, Leaf-2 | Po1 (Gig2+Gig3) |
| csr-1 | 192.168.30.131 | Leaf-1, Leaf-2 | Po1 (Gig2+Gig3) |
| csr-2 | 192.168.30.132 | Leaf-3, Leaf-4 | Po1 (Gig2+Gig3) |
| csr-3 | 192.168.30.133 | Leaf-3, Leaf-4 | Po1 (Gig2+Gig3) |
| csr-4 | 192.168.30.134 | Leaf-5, Leaf-6 | Po1 (Gig2+Gig3) |
| csr-5 | 192.168.30.135 | Leaf-5, Leaf-6 | Po1 (Gig2+Gig3) |

## CSR Interface Mapping
Each CSR has:
- **GigabitEthernet1**: Management (connected to mngsw)
- **GigabitEthernet2**: To Leaf switch 1 (LACP member)
- **GigabitEthernet3**: To Leaf switch 2 (LACP member)
- **GigabitEthernet4**: Spare

## VLANs and VRFs
| VLAN | VRF Name | Subnet | Gateway |
|------|----------|--------|---------|
| 10 | vlan10 | 192.168.10.0/24 | 192.168.10.1 |
| 20 | vlan20 | 192.168.20.0/24 | 192.168.20.1 |
| 30 | vlan30 | 192.168.30.0/24 | 192.168.30.1 |
| 40 | vlan40 | 192.168.40.0/24 | 192.168.40.1 |

## vPC Configuration

### Anycast VTEP (Secondary IP on Loopback0)
| vPC Domain | Leaf Pair | Shared Secondary IP |
|------------|-----------|---------------------|
| 1 | Leaf-1, Leaf-2 | 10.0.2.100 |
| 2 | Leaf-3, Leaf-4 | 10.0.2.200 |
| 3 | Leaf-5, Leaf-6 | 10.0.2.250 |

### vPC Peer-Keepalive
- Uses Loopback0 IPs in **default VRF** (not management)
- Routed through underlay

### vPC Port-Channels on Leafs
| Leaf | Po for CSR-A | Po for CSR-B | Po100 (Peer-Link) |
|------|--------------|--------------|-------------------|
| Leaf-1 | Po6 (csr-0) E1/6 | Po7 (csr-1) E1/7 | E1/8, E1/9 |
| Leaf-2 | Po6 (csr-0) E1/6 | Po7 (csr-1) E1/7 | E1/8, E1/9 |
| Leaf-3 | Po6 (csr-2) E1/6 | Po7 (csr-3) E1/7 | E1/5 |
| Leaf-4 | Po6 (csr-2) E1/6 | Po7 (csr-3) E1/7 | E1/5 |
| Leaf-5 | Po3 (csr-4) E1/3 | Po4 (csr-5) E1/4 | E1/5 |
| Leaf-6 | Po3 (csr-4) E1/3 | Po4 (csr-5) E1/4 | E1/5 |

## CSR Configuration Template

```
! === CSR-X Configuration ===
hostname csr-X

! Management Interface
interface GigabitEthernet1
 ip address 192.168.30.13X 255.255.255.0
 no shutdown

ip route 0.0.0.0 0.0.0.0 192.168.30.4

! VRF Definitions
vrf definition vlan10
 address-family ipv4
vrf definition vlan20
 address-family ipv4
vrf definition vlan30
 address-family ipv4
vrf definition vlan40
 address-family ipv4

! LACP Port-Channel
interface Port-channel1
 no shutdown

interface GigabitEthernet2
 no shutdown
 channel-group 1 mode active

interface GigabitEthernet3
 no shutdown
 channel-group 1 mode active

! VLAN Subinterfaces with VRF
interface Port-channel1.10
 encapsulation dot1Q 10
 vrf forwarding vlan10
 ip address 192.168.10.13X 255.255.255.0

interface Port-channel1.20
 encapsulation dot1Q 20
 vrf forwarding vlan20
 ip address 192.168.20.13X 255.255.255.0

interface Port-channel1.30
 encapsulation dot1Q 30
 vrf forwarding vlan30
 ip address 192.168.30.13X 255.255.255.0

interface Port-channel1.40
 encapsulation dot1Q 40
 vrf forwarding vlan40
 ip address 192.168.40.13X 255.255.255.0

! Default routes per VRF
ip route vrf vlan10 0.0.0.0 0.0.0.0 192.168.10.1
ip route vrf vlan20 0.0.0.0 0.0.0.0 192.168.20.1
ip route vrf vlan30 0.0.0.0 0.0.0.0 192.168.30.1
ip route vrf vlan40 0.0.0.0 0.0.0.0 192.168.40.1

! LLDP
lldp run
```

## Current Status (Updated: 2026-01-18 07:30 UTC)

### Completed ✅
- [x] Replaced IOL devices with CSR1000v
- [x] Created links: CSR -> mngsw (Gig1), CSR -> Leaf (Gig2, Gig3)
- [x] vPC configured on all Leaf pairs with correct secondary IPs
- [x] vPC peer-keepalive using loopback IPs in default VRF
- [x] LLDP enabled on all NX-OS devices
- [x] CSR nodes booted and configured
- [x] CSR devices configured with LACP, VRFs, IPs
- [x] End-to-end connectivity verified (csr-0 can ping csr-1 via VLAN 10)

### Known Issues ⚠️
- **Leaf-4**: 3/4 BGP neighbors in `Idle` state (only Spine-4 connected)
- **Leaf-6**: Po3 and Po4 are DOWN (CSR-4/CSR-5 LACP issue on Leaf-6 side)

### BGP L2VPN EVPN Status
| Leaf | AS | BGP Neighbors | Status |
|------|-----|---------------|--------|
| Leaf-1 | 65101 | 4/4 capable | ✅ All UP |
| Leaf-2 | 65102 | 4/4 capable | ✅ All UP |
| Leaf-3 | 65103 | 4/4 capable | ✅ All UP |
| Leaf-4 | 65104 | 1/4 capable | ⚠️ 3 Idle |
| Leaf-5 | 65105 | 2/2 capable | ✅ All UP |
| Leaf-6 | 65106 | 2/2 capable | ✅ All UP |

### NVE Peers (VXLAN Tunnels)
| Leaf | Peers | Status |
|------|-------|--------|
| Leaf-1 | 10.0.2.200, 10.0.2.250, 10.1.1.9, 10.1.1.11 | ✅ All UP |
| Leaf-2 | 10.0.2.200, 10.0.2.250, 10.1.1.9, 10.1.1.11 | ✅ All UP |
| Leaf-3 | 10.0.2.100, 10.0.2.250, +3 more | ✅ All UP |
| Leaf-4 | 10.0.2.100, 10.0.2.250, +2 more | ✅ All UP |
| Leaf-5 | 10.0.2.100, 10.0.2.200, +4 more | ✅ All UP |
| Leaf-6 | 10.0.2.100, 10.0.2.200, +4 more | ✅ All UP |

### vPC Status
| Leaf | Domain | Peer Status | Keepalive | Port-Channels |
|------|--------|-------------|-----------|---------------|
| Leaf-1 | 1 | ✅ formed | ✅ alive | Po6(SU) Po7(SU) Po100(SU) |
| Leaf-2 | 1 | ✅ formed | ✅ alive | Po6(SU) Po7(SU) Po100(SU) |
| Leaf-3 | 2 | ✅ formed | ✅ alive | Po6(SU) Po7(SU) Po100(SU) |
| Leaf-4 | 2 | ✅ formed | ✅ alive | Po6(SU) Po7(SU) Po100(SU) |
| Leaf-5 | 3 | ✅ formed | ✅ alive | Po3(SU) Po4(SU) Po100(SU) |
| Leaf-6 | 3 | ✅ formed | ✅ alive | Po3(SD) Po4(SD) Po100(SU) |

### CSR LACP Status
| CSR | Port-Channel | Status |
|-----|--------------|--------|
| csr-0 | Po1(RU) | ✅ Gi2+Gi3 bundled |
| csr-1 | Po1(RU) | ✅ Gi2+Gi3 bundled |
| csr-2 | Po1(RD) | ⚠️ Gi2+Gi3 suspended |
| csr-3 | Po1(RD) | ⚠️ Gi2+Gi3 suspended |
| csr-4 | Po1(RU) | ⚠️ Gi2 bundled, Gi3 suspended |
| csr-5 | Po1(RU) | ⚠️ Gi2 bundled, Gi3 suspended |

## CML API Quick Reference

```bash
# Get token
TOKEN=$(curl -s -k -X POST "https://192.168.20.65/api/v0/authenticate" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"Elma12743??"}' | tr -d '"')

# List labs
curl -s -k -X GET "https://192.168.20.65/api/v0/labs" -H "Authorization: Bearer $TOKEN"

# Get lab nodes
curl -s -k -X GET "https://192.168.20.65/api/v0/labs/${LAB_ID}/nodes" -H "Authorization: Bearer $TOKEN"

# Start node
curl -s -k -X PUT "https://192.168.20.65/api/v0/labs/${LAB_ID}/nodes/${NODE_ID}/state/start" -H "Authorization: Bearer $TOKEN"

# Console access via CML terminal server
ssh admin@192.168.20.65
# Then: open /VXLAN_MANUAL/csr-0/0
```

## Troubleshooting

### vPC Issues
```
show vpc brief
show vpc peer-keepalive
show port-channel summary
show vpc consistency-parameters global
```

### LACP Issues
```
show lacp neighbor
show lacp counters
show etherchannel summary
```

### VXLAN/NVE Issues
```
show nve interface
show nve peers
show bgp l2vpn evpn summary
```
