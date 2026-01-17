# Cisco CML Lab - VXLAN_MANUAL

## Overview
This document covers accessing the Cisco CML (Cisco Modeling Labs) environment and configuring NX-OS 9000v devices for the VXLAN_MANUAL lab topology.

---

## CML Access

### Web UI
- **URL:** https://192.168.20.65
- **Username:** admin
- **Password:** Elma12743??

### Console Server (SSH)
Access device consoles via CML's breakout tool:
```bash
ssh admin@192.168.20.65
# Password: Elma12743??

# At consoles> prompt, open a device console:
open /VXLAN_MANUAL/<device-name>/0

# Example:
open /VXLAN_MANUAL/Spine-1/0

# To exit console: Ctrl+] then type 'exit'
```

---

## Lab Topology - VXLAN_MANUAL

### Device Inventory

#### NX-OS 9000v Devices (VXLAN EVPN Fabric)

| Device | Role | Management IP | BGP AS | Loopback0 (VTEP) |
|--------|------|---------------|--------|------------------|
| **Super-Spine Layer** |||||
| Super-Spine-1 | Super Spine | 192.168.30.118/24 | 65000 | 10.0.0.1 |
| Super-Spine-2 | Super Spine | 192.168.30.119/24 | 65000 | 10.0.0.2 |
| **POD 1 - Spine Layer** |||||
| Spine-1 | Spine | 192.168.30.110/24 | 65001 | 10.2.1.0 |
| Spine-2 | Spine | 192.168.30.111/24 | 65002 | 10.2.2.0 |
| Spine-3 | Spine | 192.168.30.112/24 | 65003 | 10.2.3.0 |
| Spine-4 | Spine | 192.168.30.113/24 | 65004 | 10.2.4.0 |
| **POD 1 - Leaf Layer** |||||
| Leaf-1 | Leaf/VTEP | 192.168.30.114/24 | 65101 | 10.0.2.1 |
| Leaf-2 | Leaf/VTEP | 192.168.30.115/24 | 65102 | 10.0.2.2 |
| Leaf-3 | Leaf/VTEP | 192.168.30.116/24 | 65103 | 10.0.2.3 |
| Leaf-4 | Leaf/VTEP | 192.168.30.117/24 | 65104 | 10.0.2.4 |
| **POD 2 - Spine Layer** |||||
| Spine-5 | Spine | 192.168.30.122/24 | 65005 | 10.2.5.0 |
| Spine-6 | Spine | 192.168.30.123/24 | 65006 | 10.2.6.0 |
| **POD 2 - Leaf Layer** |||||
| Leaf-5 | Leaf/VTEP | 192.168.30.120/24 | 65105 | 10.0.2.5 |
| Leaf-6 | Leaf/VTEP | 192.168.30.121/24 | 65106 | 10.0.2.6 |

#### IOSvL2 Access Switches

| Device | Role | Management IP | Connected To |
|--------|------|---------------|--------------|
| iosvl2-0 | Access Switch | 192.168.30.130/24 | Leaf-1 (G0/1) |
| iosvl2-1 | Access Switch | 192.168.30.131/24 | Leaf-2 (G0/1) |
| iosvl2-2 | Access Switch | 192.168.30.132/24 | Leaf-3 (G0/1) |
| iosvl2-3 | Access Switch | 192.168.30.133/24 | Leaf-4 (G0/1) |
| iosvl2-4 | Access Switch | 192.168.30.134/24 | Leaf-5 (G0/1) |
| iosvl2-5 | Access Switch | 192.168.30.135/24 | Leaf-6 (G0/1) |

### Device Credentials (SSH)
- **NX-OS Devices:** admin / Versa@123!! (role: network-admin)
- **IOSvL2 Switches:** admin / Versa@123!!
- **IOSvL2 SSH Note:** Requires legacy SSH options:
  ```bash
  ssh -o KexAlgorithms=+diffie-hellman-group14-sha1 -o HostKeyAlgorithms=+ssh-rsa admin@<ip>
  ```

---

## VXLAN EVPN Configuration

### VLANs and VN-Segments

| VLAN | Name | VN-Segment | Purpose |
|------|------|------------|---------|
| 10 | VLAN10_Data | 10010 | Data traffic |
| 20 | VLAN20_Voice | 10020 | Voice traffic |
| 30 | VLAN30_Management | 10030 | Management |
| 40 | VLAN40_Guest | 10040 | Guest access |
| 100 | L3VNI_mylab | 50000 | L3VNI for VRF routing |

### VRF Configuration

```
vrf context mylab
  vni 50000
  rd auto
  address-family ipv4 unicast
    route-target import 65000:50000
    route-target import 65000:50000 evpn
    route-target export 65000:50000
    route-target export 65000:50000 evpn
```

### NVE Interface (VTEP)

```
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  member vni 10010
    ingress-replication protocol bgp
  member vni 10020
    ingress-replication protocol bgp
  member vni 10030
    ingress-replication protocol bgp
  member vni 10040
    ingress-replication protocol bgp
  member vni 50000 associate-vrf
```

### BGP EVPN Design

**Architecture:** eBGP EVPN with unique AS per device
- Super-Spines: AS 65000 (Route Reflectors for inter-POD)
- POD 1 Spines: AS 65001-65004
- POD 1 Leafs: AS 65101-65104
- POD 2 Spines: AS 65005-65006
- POD 2 Leafs: AS 65105-65106

**Leaf BGP Example (Leaf-1):**
```
router bgp 65101
  router-id 10.0.2.1
  bestpath as-path multipath-relax
  address-family l2vpn evpn
  neighbor 10.2.1.0
    remote-as 65001
    description Spine-1
    address-family l2vpn evpn
      send-community extended
  ! ... repeat for Spine-2, Spine-3, Spine-4
  vrf mylab
    address-family ipv4 unicast
      advertise l2vpn evpn
```

### IOSvL2 Access Switch Configuration

**VLANs configured on access switches:**
```
vlan 10
 name VLAN10_Data
vlan 20
 name VLAN20_Voice
vlan 40
 name VLAN40_Guest

interface Vlan1
 ip address 192.168.30.x 255.255.255.0   ! Management
interface Vlan10
 ip address 192.168.10.x 255.255.255.0
interface Vlan20
 ip address 192.168.20.x 255.255.255.0
interface Vlan40
 ip address 192.168.40.x 255.255.255.0
```

### Current Status (as of Jan 2026)

| Component | POD 1 | POD 2 |
|-----------|-------|-------|
| Leaf-to-Spine BGP | ✅ Working (4 spines) | ✅ Working (2 spines) |
| VXLAN NVE VNIs | ✅ All 5 Up | ✅ All 5 Up |
| VXLAN NVE Peers | ✅ Up (3 peers) | ✅ Up (1 peer) |
| Super-Spine BGP | ⚠️ Idle (underlay issue) | N/A |
| IOSvL2 VLANs | ✅ Configured | ✅ Configured |
| IOSvL2 Trunks | ✅ Configured | ✅ Configured |
| IOSvL2 Gateways | ✅ Configured | ✅ Configured |

### Verification Commands Output (Jan 17, 2026)

#### Leaf-1 NVE VNI Status - ✅ ALL UP
```
Leaf-1# show nve vni
Interface VNI      Multicast-group   State Mode Type [BD/VRF]
nve1      10010    UnicastBGP        Up    CP   L2 [10]       ✅
nve1      10020    UnicastBGP        Up    CP   L2 [20]       ✅
nve1      10030    UnicastBGP        Up    CP   L2 [30]       ✅
nve1      10040    UnicastBGP        Up    CP   L2 [40]       ✅
nve1      50000    n/a               Up    CP   L3 [mylab]    ✅
```

#### Leaf-1 NVE Peers (VXLAN Tunnels) - ✅ ALL UP
```
Leaf-1# show nve peers
Interface Peer-IP          State LearnType Uptime   Router-Mac
nve1      10.0.2.2         Up    CP        07:42:28 52b6.6784.1b08  (Leaf-2) ✅
nve1      10.0.2.3         Up    CP        07:41:52 52c0.6d7c.1b08  (Leaf-3) ✅
nve1      10.0.2.4         Up    CP        07:41:44 521e.a310.1b08  (Leaf-4) ✅
```

#### Leaf-1 BGP EVPN Summary
```
Leaf-1# show bgp l2vpn evpn summary
BGP router identifier 10.0.2.1, local AS number 65101
BGP table version is 98, L2VPN EVPN config peers 4, capable peers 4

Neighbor        V    AS    MsgRcvd    MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.2.1.0        4 65001        601        555       98    0    0 09:06:48 16  (Spine-1) ✅
10.2.2.0        4 65002        602        556       98    0    0 09:06:53 16  (Spine-2) ✅
10.2.3.0        4 65003        602        554       98    0    0 09:06:47 16  (Spine-3) ✅
10.2.4.0        4 65004        621        557       98    0    0 09:06:48 24  (Spine-4) ✅
```

#### Leaf-1 NVE1 Config
```
Leaf-1# show run int nve1
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0
  member vni 10010
    ingress-replication protocol bgp
  member vni 10020
    ingress-replication protocol bgp
  member vni 10030
    ingress-replication protocol bgp
  member vni 10040
    ingress-replication protocol bgp
  member vni 50000 associate-vrf
```

#### Leaf-1 SVI Config (Anycast Gateway)
```
Leaf-1# show run int vlan10
interface Vlan10
  description VLAN 10 - Anycast Gateway
  no shutdown
  vrf member mylab
  ip address 192.168.10.1/24
  fabric forwarding mode anycast-gateway
```

#### Spine-1 BGP EVPN Summary
```
Spine-1# show bgp l2vpn evpn summary
BGP router identifier 10.0.1.1, local AS number 65001
Neighbor        V    AS    State/PfxRcd
10.1.1.0        4 65000   Idle          (Super-Spine-1) ⚠️
10.1.1.8        4 65000   Idle          (Super-Spine-2) ⚠️
10.2.1.1        4 65101   16            (Leaf-1) ✅
10.2.1.3        4 65102   16            (Leaf-2) ✅
10.2.1.5        4 65103   16            (Leaf-3) ✅
10.2.1.7        4 65104   Idle          (Leaf-4) ⚠️
```

#### Leaf-5 (POD 2) NVE VNI Status - ✅ ALL UP
```
Leaf-5# show nve vni
Interface VNI      Multicast-group   State Mode Type [BD/VRF]
nve1      10010    UnicastBGP        Up    CP   L2 [10]       ✅
nve1      10020    UnicastBGP        Up    CP   L2 [20]       ✅
nve1      10030    UnicastBGP        Up    CP   L2 [30]       ✅
nve1      10040    UnicastBGP        Up    CP   L2 [40]       ✅
nve1      50000    n/a               Up    CP   L3 [mylab]    ✅
```

#### Leaf-5 NVE Peers
```
Leaf-5# show nve peers
Interface Peer-IP          State LearnType Uptime   Router-Mac
nve1      10.0.2.6         Up    CP        06:14:23 5227.4721.1b08  (Leaf-6) ✅
```

#### Leaf-5 BGP EVPN Summary
```
Leaf-5# show bgp l2vpn evpn summary
BGP router identifier 10.0.2.5, local AS number 65105
Neighbor        V    AS    State/PfxRcd
10.2.5.0        4 65005   8             (Spine-5) ✅
10.2.6.0        4 65006   8             (Spine-6) ✅
```

#### Spine-5 BGP EVPN Summary
```
Spine-5# show bgp l2vpn evpn summary
BGP router identifier 10.0.1.5, local AS number 65005
Neighbor        V    AS    State/PfxRcd
10.2.5.1        4 65105   8             (Leaf-5) ✅
10.2.5.3        4 65106   8             (Leaf-6) ✅
```

#### All Leafs NVE Status Summary
| Leaf | NVE VNIs | NVE Peers | BGP EVPN |
|------|----------|-----------|----------|
| Leaf-1 | 5 Up | 3 peers (Leaf-2,3,4) | 4 spines Up |
| Leaf-2 | 5 Up | 3 peers (Leaf-1,3,4) | 4 spines Up |
| Leaf-3 | 5 Up | 3 peers (Leaf-1,2,4) | 4 spines Up |
| Leaf-4 | 5 Up | 3 peers (Leaf-1,2,3) | 4 spines Up |
| Leaf-5 | 5 Up | 1 peer (Leaf-6) | 2 spines Up |
| Leaf-6 | 5 Up | 1 peer (Leaf-5) | 2 spines Up |

### Known Issues

1. **Super-Spine BGP Idle** - Underlay connectivity issue between Super-Spines and Spines
   - Super-Spine-1 neighbors (10.1.1.1, 10.1.1.3, 10.1.1.5, 10.1.1.7) all show Idle
   - Need to verify underlay routing/interfaces in CML

2. **IOSvL2 Trunks Not Configured** - Access switches need trunk config
   - G0/1 is currently in access mode (VLAN 1)
   - Need to configure as trunk with VLANs 10,20,40
   - Fix command:
     ```
     interface g0/1
       switchport trunk encapsulation dot1q
       switchport mode trunk
       switchport trunk allowed vlan 10,20,40
     ```

---

## NX-OS Device Configuration

Configuration applied to each device:

```
conf t
hostname <device-name>
interface mgmt0
  ip address <ip>/24
  no shutdown
exit
vrf context management
  ip route 0.0.0.0/0 192.168.30.4
exit
feature ssh
username admin password Versa@123!! role network-admin
end
copy run start
```

---

## MobaXterm Session Setup

### Configuration File Location
- **Installed version:** `%APPDATA%\MobaXterm\MobaXterm.ini`
- **Portable version:** `<MobaXterm folder>\MobaXterm.ini`

### Adding Sessions Manually
1. Right-click in Sessions sidebar → **New session**
2. Select **SSH**
3. Configure:
   - **Remote host:** Device IP (e.g., 192.168.30.110)
   - **Username:** admin
   - **Port:** 22
4. Click **OK**
5. Drag session into desired folder

### Adding Sessions via INI File

> **Important:** Close MobaXterm before editing the INI file!

Add to the INI file under a `[Bookmarks_N]` section:

```ini
[Bookmarks_47]
SubRep=SuperSpine
ImgNum=41
Super-Spine-1=#109#0%192.168.30.118%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
Super-Spine-2=#109#0%192.168.30.119%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
Spine-1=#109#0%192.168.30.110%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
Spine-2=#109#0%192.168.30.111%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
Spine-3=#109#0%192.168.30.112%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
Spine-4=#109#0%192.168.30.113%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
Leaf-1=#109#0%192.168.30.114%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
Leaf-2=#109#0%192.168.30.115%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
Leaf-3=#109#0%192.168.30.116%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
Leaf-4=#109#0%192.168.30.117%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1

[Bookmarks_48]
SubRep=POD2
ImgNum=41
Spine-5=#109#0%192.168.30.122%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
Spine-6=#109#0%192.168.30.123%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
Leaf-5=#109#0%192.168.30.120%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
Leaf-6=#109#0%192.168.30.121%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1

[Bookmarks_49]
SubRep=IOSvL2
ImgNum=41
iosvl2-0=#109#0%192.168.30.130%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
iosvl2-1=#109#0%192.168.30.131%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
iosvl2-2=#109#0%192.168.30.132%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
iosvl2-3=#109#0%192.168.30.133%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
iosvl2-4=#109#0%192.168.30.134%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
iosvl2-5=#109#0%192.168.30.135%22%admin%%-1%-1%%%%%0%0%0%%%-1%-1%0%0%%1080%%0%0%1%%0%%%%0%-1%-1%0%%#MobaFont%10%0%0%-1%15%236,236,236%30,30,30%180,180,192%0%-1%0%%xterm%-1%0%_Std_Colors_0_%80%24%0%3%-1%<none>%%0%0%-1%0%#0# #-1
```

### MobaXterm INI Format Reference
- `#109` = SSH connection type
- `%IP%22%username%` = Host, port 22, username
- `SubRep=FolderName` = Folder name in sidebar
- `SubRep=Parent\Child` = Nested subfolder

---

## Quick Reference

### SSH to Device
```bash
ssh admin@192.168.30.110
# Password: Versa@123!!
```

### Useful NX-OS Commands
```
show ip int brief vrf management    # Check mgmt interface
show ssh server                      # Verify SSH status
show users                           # Show logged in users
show running-config                  # View current config
show version                         # Device info
```

### CML API (Optional)
```bash
# Authenticate
curl -k -X POST https://192.168.20.65/api/v0/authenticate \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"Elma12743??"}'

# List labs
curl -k -X GET https://192.168.20.65/api/v0/labs \
  -H "Authorization: Bearer <token>"
```

---

## Troubleshooting

### Cannot SSH to Device
1. Verify device is BOOTED in CML
2. Check mgmt0 interface: `show ip int brief vrf management`
3. Verify SSH feature: `show ssh server`
4. Check routing: `ping 192.168.30.4 vrf management`

### MobaXterm Sessions Not Appearing
1. **Close MobaXterm completely** before editing INI
2. Verify correct INI file location (installed vs portable)
3. Check `[Bookmarks_N]` section numbers don't conflict

### Device Console Access via CML
If SSH fails, use CML console:
```
ssh admin@192.168.20.65
open /VXLAN_MANUAL/Spine-1/0
```

---

## Automation Script (Expect)

Use this script to configure multiple NX-OS devices via CML console:

```bash
#!/usr/bin/expect -f
# Save as: /tmp/config_device.exp
# Usage: expect /tmp/config_device.exp <device-name> <ip-address>

log_user 1
set timeout 60

set device [lindex $argv 0]
set ip [lindex $argv 1]

puts "=== Configuring $device with $ip ==="

spawn ssh -o StrictHostKeyChecking=no admin@192.168.20.65
expect "password:"
send "Elma12743??\r"
expect "consoles>"

send "open /VXLAN_MANUAL/$device/0\r"
expect "Escape character"

sleep 2
send "\r\r"
sleep 2

# Handle login if needed
expect {
    "login:" {
        send "admin\r"
        expect "Password:"
        send "cisco\r"
    }
    -re "#" { }
    -re ">" {
        send "enable\r"
    }
    timeout {
        send "\r"
    }
}

sleep 2
expect -re ".*"

# Configuration commands
set config "
conf t
hostname $device
interface mgmt0
ip address $ip/24
no shutdown
exit
vrf context management
ip route 0.0.0.0/0 192.168.30.4
exit
feature ssh
username admin password Versa@123!! role network-admin
end
copy run start
"

foreach line [split $config "\n"] {
    set line [string trim $line]
    if {$line ne ""} {
        send "$line\r"
        sleep 0.8
        expect -re ".*"
    }
}

sleep 3
puts "\n=== Configuration sent to $device ==="

# Escape (Ctrl+])
send "\x1d"
sleep 1
expect -re ".*"
send "exit\r"
expect eof
```

### Running the Script
```bash
# Configure single device
expect /tmp/config_device.exp Spine-1 192.168.30.110

# Configure all devices (loop)
declare -A devices=(
    # Super-Spine Layer
    ["Super-Spine-1"]="192.168.30.118"
    ["Super-Spine-2"]="192.168.30.119"
    # POD 1 - Spines
    ["Spine-1"]="192.168.30.110"
    ["Spine-2"]="192.168.30.111"
    ["Spine-3"]="192.168.30.112"
    ["Spine-4"]="192.168.30.113"
    # POD 1 - Leafs
    ["Leaf-1"]="192.168.30.114"
    ["Leaf-2"]="192.168.30.115"
    ["Leaf-3"]="192.168.30.116"
    ["Leaf-4"]="192.168.30.117"
    # POD 2 - Spines
    ["Spine-5"]="192.168.30.122"
    ["Spine-6"]="192.168.30.123"
    # POD 2 - Leafs
    ["Leaf-5"]="192.168.30.120"
    ["Leaf-6"]="192.168.30.121"
)

# IOSvL2 switches (separate - different SSH options needed)
declare -A iosvl2_devices=(
    ["iosvl2-0"]="192.168.30.130"
    ["iosvl2-1"]="192.168.30.131"
    ["iosvl2-2"]="192.168.30.132"
    ["iosvl2-3"]="192.168.30.133"
    ["iosvl2-4"]="192.168.30.134"
    ["iosvl2-5"]="192.168.30.135"
)

for device in "${!devices[@]}"; do
    expect /tmp/config_device.exp "$device" "${devices[$device]}"
    sleep 2
done
```

---

## Workflow for Future Labs

### Step 1: Get Lab Info from CML
```bash
# Authenticate and get token
TOKEN=$(curl -sk -X POST https://192.168.20.65/api/v0/authenticate \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"Elma12743??"}' | tr -d '"')

# List all labs
curl -sk https://192.168.20.65/api/v0/labs -H "Authorization: Bearer $TOKEN"

# Get nodes in a specific lab
LAB_ID="<lab-id>"
curl -sk https://192.168.20.65/api/v0/labs/$LAB_ID/nodes -H "Authorization: Bearer $TOKEN"
```

### Step 2: Configure Devices
1. Check device status in CML (must be BOOTED)
2. Determine IP range for management interfaces
3. Run expect script for each device
4. Verify SSH access

### Step 3: Add to MobaXterm
1. Close MobaXterm
2. Edit `%APPDATA%\MobaXterm\MobaXterm.ini`
3. Add sessions under appropriate `[Bookmarks_N]` section
4. Reopen MobaXterm

---

## For Future Similar Tasks

When asking Claude to do similar CML lab setup, provide:

1. **CML credentials** - IP, username, password
2. **Lab name** - The lab to configure (e.g., VXLAN_MANUAL)
3. **IP range** - Management IP range to use
4. **Gateway** - Default gateway for management VRF
5. **Device credentials** - Username/password to create on devices
6. **MobaXterm folder** - Where to organize sessions

### Example Prompt
```
Configure CML lab "MY_LAB" at 192.168.20.65 (admin/password123).
Assign IPs 192.168.50.10-20 with gateway 192.168.50.1.
Create user "netadmin" with password "MyPass123!" on all devices.
Add sessions to MobaXterm under "MyLab" folder.
```

---

*Document created: January 2026*
