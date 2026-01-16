# Session Notes - January 16, 2026

## What Was Completed

### 1. BGP EVPN VXLAN Configuration - POD 1 (Original Fabric)
- **Super-Spine-1/2** (192.168.30.118/119) - AS 65000
- **Spine-1/2/3/4** (192.168.30.110-113) - AS 65001-65004
- **Leaf-1/2/3/4** (192.168.30.114-117) - AS 65101-65104

### 2. BGP EVPN VXLAN Configuration - POD 2 (NEW - Standalone Fabric)
- **Spine-5** (192.168.30.122) - AS 65005
- **Spine-6** (192.168.30.123) - AS 65006
- **Leaf-5** (192.168.30.120) - AS 65105
- **Leaf-6** (192.168.30.121) - AS 65106

### 3. Issues Found and Fixed (POD 1)

#### Issue 1: NVE Interface Down
- **Problem:** `show nve interface nve1` showed State: Down, Source-Interface loopback1 had IP 0.0.0.0
- **Cause:** Same IP configured on both loopback0 and loopback1 (NX-OS rejected duplicate)
- **Fix:** Changed NVE source-interface to loopback0, removed loopback1
- **Script:** `/tmp/fix_nve.exp`

#### Issue 2: EVPN Routes Not Being Exchanged (0 PfxRcd)
- **Problem:** BGP L2VPN EVPN neighbors UP but receiving 0 prefixes
- **Cause:** Route-Target mismatch - "rt auto" generates AS-based RT (65101:10010 vs 65102:10010)
- **Fix:** Changed to common RT for all VNIs: 65000:10010, 65000:10020, etc.
- **Script:** `/tmp/fix_rt.exp`

### 4. Current EVPN Status

#### POD 1 (WORKING)
```
Leaf-1# show bgp l2vpn evpn summary
Neighbor        State/PfxRcd
10.2.1.0        16    ← Receiving prefixes!
10.2.2.0        16
10.2.3.0        16
10.2.4.0        24
```

#### POD 2 (WORKING)
```
Leaf-5# show bgp l2vpn evpn summary
Neighbor        State/PfxRcd
10.2.5.0        8    ← Spine-5 neighbor (receiving 8 prefixes)
10.2.6.0        8    ← Spine-6 neighbor (receiving 8 prefixes)

Leaf-6# show bgp l2vpn evpn summary
Neighbor        State/PfxRcd
10.2.5.2        8    ← Spine-5 neighbor
10.2.6.2        8    ← Spine-6 neighbor
```

### 5. Leaf Trunk Ports Configured
- All Leaf E1/5 ports configured as trunk (VLAN 10,20,30,40)
- For connecting to IOSvL2 access switches

### 6. IOSvL2 Edge Switches - ALL CONFIGURED
| Switch | IP | Connected To | Status |
|--------|-----|--------------|--------|
| iosvl2-0 | 192.168.30.130 | Leaf-1 | ✅ Configured |
| iosvl2-1 | 192.168.30.131 | Leaf-2 | ✅ Configured |
| iosvl2-2 | 192.168.30.132 | Leaf-3 | ✅ Configured |
| iosvl2-3 | 192.168.30.133 | Leaf-4 | ✅ Configured |
| iosvl2-4 | 192.168.30.134 | Leaf-5 | ✅ Configured (NEW) |
| iosvl2-5 | 192.168.30.135 | Leaf-6 | ✅ Configured (NEW) |

**IOSvL2 Config Method:** Using VLAN 1 SVI (not Gi0/0 directly because it's L2 switch)
- Script: `/tmp/config_iosvl2_svi.exp`
- Credentials: admin / Versa@123!!
- Gateway: 192.168.30.4

## POD 2 Topology Details

### Physical Connections (Discovered via CDP)
```
POD 2 - Standalone Spine-Leaf
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

### POD 2 Interface IP Addressing
| Connection | IP Address |
|------------|------------|
| Spine-5 E1/1 → Leaf-5 E1/1 | 10.2.5.0/31 ↔ 10.2.5.1/31 |
| Spine-5 E1/2 → Leaf-6 E1/1 | 10.2.5.2/31 ↔ 10.2.5.3/31 |
| Spine-6 E1/1 → Leaf-5 E1/2 | 10.2.6.0/31 ↔ 10.2.6.1/31 |
| Spine-6 E1/2 → Leaf-6 E1/2 | 10.2.6.2/31 ↔ 10.2.6.3/31 |

## Key Files Created

### POD 1 Scripts
- `/tmp/config_superspine.exp` - Super-Spine config script
- `/tmp/config_spine.exp` - Spine config script
- `/tmp/config_leaf.exp` - Leaf VTEP config script
- `/tmp/fix_nve.exp` - NVE source-interface fix
- `/tmp/fix_rt.exp` - Route-Target fix
- `/tmp/config_iosvl2_svi.exp` - IOSvL2 management config

### POD 2 Scripts
- `/tmp/config_spine5.exp` - Spine-5 BGP EVPN config
- `/tmp/config_spine6.exp` - Spine-6 BGP EVPN config
- `/tmp/config_leaf5.exp` - Leaf-5 VTEP config (full EVPN)
- `/tmp/config_leaf6.exp` - Leaf-6 VTEP config (full EVPN)
- `/tmp/config_nxos_cisco.exp` - Fresh NX-OS initial config (cisco/cisco login)

## Device Credentials
- **CML Console:** 192.168.20.65, admin / Elma12743??
- **NX-OS Devices (existing):** admin / Versa@123!!
- **NX-OS Devices (fresh):** cisco / cisco (then add admin user)
- **IOSvL2 Switches:** admin / Versa@123!!

## MobaXterm Sessions Updated
All devices now have SSH sessions in MobaXterm under the SuperSpine folder:
- Super-Spine-1, Super-Spine-2
- Spine-1 through Spine-6
- Leaf-1 through Leaf-6
- iosvl2-0 through iosvl2-5

## Design Notes

### Common Route-Targets for EVPN
All VTEPs use the same Route-Targets to enable EVPN route exchange:
- VNI 10010 (VLAN 10): RT 65000:10010
- VNI 10020 (VLAN 20): RT 65000:10020
- VNI 10030 (VLAN 30): RT 65000:10030
- VNI 10040 (VLAN 40): RT 65000:10040
- VNI 50000 (L3VNI): RT 65000:50000

### Anycast Gateway
- Virtual MAC: 0000.1111.2222
- Same IP on all Leaves per VLAN (192.168.10.1, 192.168.20.1, etc.)

## Known Limitations

### Leaf-4 (POD 1)
- Only Spine-4 is connected (others show Idle in BGP)
- CML topology missing physical links to Spine-1/2/3

### suppress-arp Warning
- NX-OS 9000v shows "Please configure TCAM region for Ingress ARP-Ether ACL"
- This is expected in virtual environment, doesn't affect basic EVPN functionality

## Next Steps
1. Connect POD 1 and POD 2 if inter-POD routing is needed
2. Test EVPN traffic between Leaves within each POD
3. Configure test hosts on IOSvL2 switches
4. Verify MAC learning via EVPN
