# IXP Route Server Configuration Guide (Arista vEOS)

# NOT REVIEWED YET

## Overview

This guide walks through configuring an Internet Exchange Point (IXP) using Arista vEOS-lab. The IXP provides a shared Layer 2 peering fabric where multiple networks can peer with each other through route servers.

## Note

All IP addresses and interface names are generic in this guide. You should replace them with your own IP addresses and interface names. See IP_ADDRESSING_PHILOSOPHY.md for more information.

### What is a Route Server?

**Traditional Bilateral Peering**:
```
Network A ←→ Network B
    ↓           ↓
    └─────→ Network C
```
= 3 BGP sessions for 3 networks

**Route Server Multilateral Peering**:
```
Network A ──┐
Network B ──┤→ Route Server → distributes routes
Network C ──┘
```
= 3 BGP sessions total (one per network to RS)

**Key Difference**: Route server does NOT modify AS-path or next-hop, making it transparent.

### Architecture

```
┌────────────────────────────────────────┐
│  IXP-SW1 (Arista vEOS)                 │
│                                        │
│  VLAN 100: 198.51.100.0/24             │
│  Route Server IP: 198.51.100.251 (SVI) │
│                                        │
│  [Eth1]  [Eth2]  [Eth3]  [Eth4] [Eth5] │
└────┬──────┬───────┬───────┬──────┬─────┘
     │      │       │       │      │
  L3-BR1 L3-BR2 Cogent Atlas Second
  .51    .61    .52   .53   .54
```

**Your Setup**:
- 1 Arista vEOS switch (IXP-SW1) to start
- Layer 2 peering fabric (VLAN 100)
- Route server runs on SVI (switched virtual interface)
- Single link from each ISP border router
- All participants on shared subnet: `198.51.100.0/24`
- Second switch (IXP-SW2) can be added later for redundancy

---

## Arista EOS Platform Primer

### Command Line Basics

**Modes** (similar to Cisco IOS):
```bash
Router>                    # User EXEC mode
Router#                    # Privileged EXEC mode
Router(config)#            # Global configuration mode
Router(config-if-Et1)#     # Interface configuration mode
Router(config-router-bgp)# # BGP configuration mode
```

**Key Commands**:
```bash
enable                     # Enter privileged mode
configure                  # Enter config mode (or 'config terminal')
show running-config        # View current config
show ip bgp summary        # BGP neighbor status
write memory               # Save config (or 'copy run start')
```

### Arista vs Cisco/Juniper

| Feature | Arista EOS | Cisco IOS | Junos |
|---------|-----------|-----------|-------|
| Config mode | `configure` | `config t` | `configure` |
| BGP config | Flat | Flat | Hierarchical |
| Save config | `write` | `copy run start` | `commit` |
| Interface naming | `Ethernet1` | `GigE0/0` | `ge-0/0/0` |
| Show BGP | `show ip bgp` | `show ip bgp` | `show bgp` |

**Arista Advantages**:
- Very Cisco-like (easy learning curve)
- Powerful Python API (we won't use today)
- Clean, consistent syntax
- Default to Layer 2 switching (perfect for IXP)

---

## Phase 1: Basic Configuration

### Step 1: Management Access

**Initial Connection**:
```bash
# Default login: admin / (no password)
# or admin / admin depending on version

enable
configure
```

**Set Hostname and Management IP**:

```bash
hostname IXP-SW1

interface Management1
 description Out-of-Band Management
 ip address 192.168.100.50/24
 no shutdown
!

ip route 0.0.0.0/0 192.168.100.1
```

**Set Root Password** (Recommended):

```bash
username admin privilege 15 secret <your-password>
```

**Enable SSH** (optional):

```bash
management ssh
 idle-timeout 60
 no shutdown
!
```

**Verification**:
```bash
show ip interface brief
ping 192.168.100.1
```

### Step 2: Loopback Interface

Route server needs a stable router-ID:

```bash
interface Loopback0
 description Router-ID for BGP
 ip address 198.51.100.251/32
!
```

**Why Loopback?**: Provides stable router-ID even if VLAN interface flaps.

---

## Phase 2: Layer 2 Peering Fabric

### Step 1: Create IXP Peering VLAN

```bash
vlan 100
 name IXP-Peering-LAN
!
```

**Why VLAN 100?**: Arbitrary choice - any VLAN ID works. VLAN 100 keeps it separate from management.

### Step 2: Configure Participant Access Ports

**Level 3 Border-1** (Ethernet1):

```bash
interface Ethernet1
 description to-Level3-Border1
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
 no shutdown
!
```

**Level 3 Border-2** (Ethernet2):

```bash
interface Ethernet2
 description to-Level3-Border2
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
 no shutdown
!
```

**Cogent Border-1** (Ethernet3):

```bash
interface Ethernet3
 description to-Cogent-Border1
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
 no shutdown
!
```

**Atlas Lab ISP** (Ethernet4):

```bash
interface Ethernet4
 description to-Atlas-Lab-ISP
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
 no shutdown
!
```

**Second ISP** (Ethernet5):

```bash
interface Ethernet5
 description to-Second-ISP
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
 no shutdown
!
```

**Key Parameters**:
- `switchport mode access`: This is a Layer 2 access port (not trunk)
- `switchport access vlan 100`: Assigns port to peering VLAN
- `spanning-tree portfast`: Speeds up port coming up (no STP delay)
- **NO `no switchport`**: We WANT Layer 2 mode for these ports

**Continue for additional participants** (CDN networks, content providers, etc.)

### Step 3: Create SVI for Route Server

**Switched Virtual Interface** (acts as gateway for the VLAN):

```bash
interface Vlan100
 description IXP-Route-Server-1
 ip address 198.51.100.251/24
 no shutdown
!
```

**What is SVI?**: A Layer 3 interface on a VLAN. All devices on VLAN 100 can reach this IP via Layer 2.

**IP Addressing**:
- IXP-SW1 Route Server: `198.51.100.251/24`
- Level 3 Border-1: `198.51.100.56/24` (configured on ISP side)
- Level 3 Border-2: `198.51.100.61/24`
- Cogent Border-1: `198.51.100.57/24`
- Atlas Lab ISP: `198.51.100.58/24`
- Second ISP: `198.51.100.59/24`

**All on same subnet!** This is Layer 2 magic - everyone can reach everyone else.

### Verification

**Check VLAN status**:
```bash
show vlan
show interfaces status
```

**Expected**: VLAN 100 active, all ports assigned to VLAN 100, ports up.

**Test Layer 2 connectivity** (from ISP router):
```bash
# On Level 3 Border-1
ping 198.51.100.251
```

**Expected**: Should work once ISP configures their interface on 198.51.100.0/24.

---

## Phase 3: BGP Route Server Configuration

### Understanding Route Server Mode

**Normal BGP**: Modifies routes before advertising
- Changes next-hop to self
- Adds own AS to AS-path
- Applies local policies

**Route Server BGP**: Transparent route reflection
- Preserves original next-hop
- Does NOT add RS AS to path (AS 64999 invisible)
- Minimal policy (just sanity filtering)

### Step 1: Configure BGP with Route Server Mode

```bash
router bgp 64999
 router-id 198.51.100.251
 maximum-paths 32
 bgp asn notation asdot

 # Peer with Level 3 Border-1 (AS 3356)
 neighbor 198.51.100.56 remote-as 3356
 neighbor 198.51.100.56 description Level3-Border1
 neighbor 198.51.100.56 route-server-client
 neighbor 198.51.100.56 send-community extended

 # Peer with Level 3 Border-2 (AS 3356)
 neighbor 198.51.100.61 remote-as 3356
 neighbor 198.51.100.61 description Level3-Border2
 neighbor 198.51.100.61 route-server-client
 neighbor 198.51.100.61 send-community extended

 # Peer with Cogent Border-1 (AS 174)
 neighbor 198.51.100.57 remote-as 174
 neighbor 198.51.100.57 description Cogent-Border1
 neighbor 198.51.100.57 route-server-client
 neighbor 198.51.100.57 send-community extended

 # Peer with Atlas Lab ISP (AS 65000)
 neighbor 198.51.100.58 remote-as 65000
 neighbor 198.51.100.58 description Atlas-Lab-ISP
 neighbor 198.51.100.58 route-server-client
 neighbor 198.51.100.58 send-community extended

 # Peer with Second ISP (AS 65100)
 neighbor 198.51.100.59 remote-as 65100
 neighbor 198.51.100.59 description Second-ISP
 neighbor 198.51.100.59 route-server-client
 neighbor 198.51.100.59 send-community extended

 # Address family configuration
 address-family ipv4
  neighbor 198.51.100.56 activate
  neighbor 198.51.100.61 activate
  neighbor 198.51.100.57 activate
  neighbor 198.51.100.58 activate
  neighbor 198.51.100.59 activate
 !
!
```

**Critical Configuration Items**:

**`route-server-client`**:
- ✅ Preserves AS-path (doesn't add AS 64999)
- ✅ Preserves next-hop (doesn't change to 198.51.100.251)
- ✅ Enables transparent route distribution
- **This is the key to route server operation!**

**`send-community extended`**:
- Propagates BGP communities through route server
- Allows participants to use communities for traffic engineering

**`maximum-paths 32`**:
- Allows multiple equal-cost paths in BGP table
- Useful when same prefix received from multiple peers

**`bgp asn notation asdot`**:
- Uses asdot notation for AS numbers (cosmetic)

### Step 2: Configure ISP Side (Example)

**On Level 3 Border-1** (your Level 3 build):

```junos
# Junos config
set interfaces ge-0/0/3 unit 0 description "to-IXP"
set interfaces ge-0/0/3 unit 0 family inet address 198.51.100.56/24

set protocols bgp group IXP-RS type external
set protocols bgp group IXP-RS peer-as 64999
set protocols bgp group IXP-RS neighbor 198.51.100.251 description "IXP-RS1"
```

**On Cogent Border-1** (your Cogent build):

```cisco
! IOS-XR config
interface GigabitEthernet0/0/0/3
 description to-IXP
 ipv4 address 198.51.100.57 255.255.255.0
!

router bgp 174
 neighbor 198.51.100.251
  remote-as 64999
  description IXP-RS1
  address-family ipv4 unicast
   route-policy FROM-IXP in
   route-policy TO-IXP out
  !
 !
!
```

### Verification

**Check BGP sessions**:
```bash
show ip bgp summary
```

**Expected Output**:
```
Neighbor         V    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
198.51.100.56    4  3356       5       6        0    0    0 00:02:15            0
198.51.100.61    4  3356       5       6        0    0    0 00:02:10            0
198.51.100.57    4   174       4       5        0    0    0 00:01:45            0
198.51.100.58    4 65000       3       4        0    0    0 00:01:20            0
```

**All sessions should show Established** (State/PfxRcd column shows prefix count, or blank if zero).

---

## Phase 4: Route Filtering and Security

### Step 1: Prefix Length Filtering

Prevent bogus announcements:

```bash
ip prefix-list SANITY-CHECK seq 10 deny 0.0.0.0/0
ip prefix-list SANITY-CHECK seq 15 deny 0.0.0.0/0 le 7
ip prefix-list SANITY-CHECK seq 20 deny 224.0.0.0/3 le 32
ip prefix-list SANITY-CHECK seq 25 deny 192.0.2.0/24 le 32
ip prefix-list SANITY-CHECK seq 30 deny 198.51.100.0/24 le 32
ip prefix-list SANITY-CHECK seq 35 deny 203.0.113.0/24 le 32
ip prefix-list SANITY-CHECK seq 40 permit 0.0.0.0/0 le 24

route-map FROM-PEERS permit 10
 match ip address prefix-list SANITY-CHECK
!

router bgp 64999
 neighbor 198.51.100.56 route-map FROM-PEERS in
 neighbor 198.51.100.61 route-map FROM-PEERS in
 neighbor 198.51.100.57 route-map FROM-PEERS in
 neighbor 198.51.100.58 route-map FROM-PEERS in
 neighbor 198.51.100.59 route-map FROM-PEERS in
!
```

**What This Blocks**:
- Default routes (`0.0.0.0/0`)
- Very large prefixes (`/0` through `/7`)
- Multicast space (`224.0.0.0/3`)
- Documentation prefixes (RFC5737: 192.0.2.0/24, etc.)
- IXP peering LAN itself (198.51.100.0/24)
- Prefixes longer than `/24` (too specific)

### Step 2: Max-Prefix Limits

Protect against route leaks:

```bash
router bgp 64999
 neighbor 198.51.100.56 maximum-routes 100000 warning-limit 75
 neighbor 198.51.100.61 maximum-routes 100000 warning-limit 75
 neighbor 198.51.100.57 maximum-routes 100000 warning-limit 75
 neighbor 198.51.100.58 maximum-routes 10000 warning-limit 75
 neighbor 198.51.100.59 maximum-routes 10000 warning-limit 75
!
```

**Limits Explained**:
- **Tier-1s** (Level 3, Cogent): 100,000 routes (full internet table)
- **Tier-2s** (Atlas, Second ISP): 10,000 routes (customers + some peers)
- `warning-limit 75`: Log warning at 75% of max

**What Happens**: If limit exceeded, BGP session tears down (prevents route table explosion).

### Step 3: BGP Community Transparency (Optional)

Allow participants to use communities for policy:

```bash
bgp community-list standard DONT-ANNOUNCE permit 64999:0

route-map TO-PEERS permit 10
 match community DONT-ANNOUNCE
 deny
!

route-map TO-PEERS permit 20
 # Everything else, advertise normally
!

router bgp 64999
 neighbor 198.51.100.56 route-map TO-PEERS out
 neighbor 198.51.100.61 route-map TO-PEERS out
 neighbor 198.51.100.57 route-map TO-PEERS out
 neighbor 198.51.100.58 route-map TO-PEERS out
 neighbor 198.51.100.59 route-map TO-PEERS out
!
```

**Use Case**: Participant tags route with `64999:0` = "don't send this through IXP"

---

## Phase 5: Verification and Testing

### Step 1: Check BGP Sessions

**View all neighbors**:
```bash
show ip bgp summary
```

**Check specific neighbor**:
```bash
show ip bgp neighbors 198.51.100.56
```

**Expected**: All sessions `Established`.

### Step 2: Verify Route Server Behavior

**Announce test route from Atlas ISP**:

```bash
# On Atlas Lab ISP
set protocols bgp address-family ipv4-unicast network 10.10.0.0/24
```

**Check route on IXP route server**:

```bash
show ip bgp 10.10.0.0/24 detail
```

**Look for**:
- AS-path should be `65000` (Atlas ISP only, NOT including 64999)
- Next-hop should be `198.51.100.58` (Atlas ISP IP, NOT route server IP)

**Example correct output**:
```
BGP routing table entry for 10.10.0.0/24
Paths: (1 available)
  65000
    198.51.100.58 from 198.51.100.58 (198.51.100.58)
      Origin IGP, valid, external
```

**Check route on Level 3** (received from route server):

```bash
# On Level 3 Border-1
show route receive-protocol bgp 198.51.100.251 | match 10.10.0.0
```

**Expected**: Route present with AS-path `65000`, next-hop `198.51.100.58`.

### Step 3: Test Filtering

**Try to announce bogon**:

```bash
# On Atlas ISP, announce documentation prefix
set protocols bgp address-family ipv4-unicast network 192.0.2.0/24
```

**Check route server**:

```bash
show ip bgp neighbors 198.51.100.58 received-routes | include 192.0.2
```

**Expected**: Route received but NOT in BGP table (filtered).

```bash
show ip bgp 192.0.2.0/24
```

**Expected**: `Network not in table` (successfully filtered).

### Step 4: Verify Multi-Lateral Peering

**Announce route from Level 3**:

```bash
# On Level 3
set protocols bgp address-family ipv4-unicast network 203.0.113.128/25
```

**Check that Cogent receives it via route server**:

```bash
# On Cogent Border-1
show route receive-protocol bgp 198.51.100.251 | match 203.0.113.128
```

**Expected**: Route appears with AS-path `3356`, next-hop `198.51.100.56`.

**Success!** Level 3 and Cogent are peering via IXP route server without direct BGP session.

---

## Phase 6: Monitoring and Logging

### Enable Logging

```bash
logging buffered 50000 informational
logging console warnings

router bgp 64999
 bgp log-neighbor-changes
!
```

**Check logs**:
```bash
show logging last 100
show logging | include BGP
```

### Enable SNMP (Optional)

```bash
snmp-server community public ro
snmp-server location "IXP Peering Facility"
snmp-server contact "noc@example.com"
```

### BGP Statistics

```bash
show ip bgp summary | begin Neighbor
show ip bgp neighbors 198.51.100.56 received-routes | count
show ip bgp neighbors 198.51.100.56 advertised-routes | count
```

---

## Common Issues and Troubleshooting

### Issue 1: BGP Sessions Not Establishing

**Symptoms**: `show ip bgp summary` shows `Idle` or `Active`

**Check Layer 2 connectivity**:
```bash
show interfaces status
show vlan
ping 198.51.100.56
```

**Common Causes**:
- Port not assigned to VLAN 100
- ISP interface not configured on 198.51.100.0/24 subnet
- Wrong remote-as configured
- Firewall blocking TCP 179

**Fix**:
```bash
# Verify port config
show running-config interfaces Ethernet1
# Should show switchport mode access, vlan 100
```

### Issue 2: Routes Not Being Distributed

**Symptoms**: RS receives routes but doesn't advertise to other peers

**Check**:
```bash
show ip bgp neighbors 198.51.100.56 received-routes
show ip bgp neighbors 198.51.100.57 advertised-routes
```

**Common Causes**:
- Missing `route-server-client` configuration
- Route-map blocking advertisement

**Verify route-server mode**:
```bash
show running-config | section bgp | include route-server-client
```

**Expected**: Should see `route-server-client` under each neighbor.

### Issue 3: AS-Path Includes Route Server AS

**Symptoms**: AS-path shows `...64999 65000...` instead of just `65000`

**Problem**: Route server mode not enabled

**Fix**:
```bash
router bgp 64999
 neighbor 198.51.100.58 route-server-client
!
```

### Issue 4: Next-Hop Set to Route Server IP

**Symptoms**: Routes have next-hop of `198.51.100.251` instead of originating router

**Problem**: Missing `route-server-client` or route-map overriding it

**Check**:
```bash
show ip bgp 10.10.0.0/24
```

**Expected next-hop**: `198.51.100.58` (Atlas ISP), NOT `198.51.100.251` (RS).

**Fix**: Ensure `route-server-client` configured and no route-map sets next-hop-self.

### Issue 5: Layer 2 Loops / Broadcast Storm

**Symptoms**: High CPU, network unstable, spanning-tree issues

**Check**:
```bash
show spanning-tree
```

**Cause**: Multiple connections creating loop (if you accidentally dual-homed without LACP)

**Fix**: Remove extra cables or configure proper LACP port-channel.

---

## Arista-Specific Tips

### Configuration Sessions

```bash
configure session IXP-Setup
# Make changes
show session-config diffs
commit
```

**Advantage**: See exactly what changed before committing.

### Configuration Rollback

```bash
copy running-config flash:before-ixp.cfg

# If things go wrong:
configure replace flash:before-ixp.cfg
```

### Show Commands Quick Reference

```bash
show ip bgp summary              # BGP neighbor status
show ip bgp                      # Full BGP table
show ip bgp neighbors X.X.X.X    # Detailed neighbor info
show vlan                        # VLAN status
show interfaces status           # Interface up/down
show running-config | section bgp # BGP config only
```

### Saving Configuration

```bash
write memory                     # Save config
show startup-config              # Verify saved
diff running-config startup-config # Check differences
```

---

## Adding Second Route Server (IXP-SW2) - Phase 7

### When to Add IXP-SW2

Add a second route server after IXP-SW1 is stable and working:
- Provides redundancy (if RS1 fails, RS2 continues peering)
- Load balancing (different paths through different RSs)
- Standard practice (real IXPs always run 2+ route servers)

### Topology with Two Route Servers

```
         ┌──────────────┐
         │  IXP-SW1     │
         │  RS1         │
         │  .251        │
         └──────────────┘
              │
         (All ISPs connect here first)
              │
         ┌──────────────┐
         │  IXP-SW2     │
         │  RS2         │
         │  .252        │
         └──────────────┘
```

**Two Approaches**:

**Option A: Independent Route Servers** (Recommended for learning)
- Each ISP has TWO links: one to SW1, one to SW2
- Two separate Layer 2 fabrics (VLAN 100 on each)
- More cables but clearer separation

**Option B: Shared VLAN with MLAG** (Production-like)
- IXP-SW1 and SW2 in MLAG pair
- Single VLAN 100 spanning both switches
- ISPs can dual-home with LACP
- More complex but more realistic

### Option A: Independent Route Servers (Start Here)

**IXP-SW2 Configuration** (mirror of SW1):

```bash
hostname IXP-SW2

interface Management1
 ip address 192.168.100.51/24
!

interface Loopback0
 ip address 198.51.100.252/32
!

vlan 100
 name IXP-Peering-LAN-RS2
!

# Configure same ports as SW1
interface Ethernet1
 description to-Level3-Border1-Link2
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
!

interface Ethernet2
 description to-Level3-Border2-Link2
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
!

# Continue for all participants...

# SVI for Route Server 2
interface Vlan100
 description IXP-Route-Server-2
 ip address 198.51.100.252/24
!

# BGP config (identical except router-id)
router bgp 64999
 router-id 198.51.100.252
 maximum-paths 32

 neighbor 198.51.100.56 remote-as 3356
 neighbor 198.51.100.56 description Level3-Border1
 neighbor 198.51.100.56 route-server-client
 neighbor 198.51.100.56 send-community extended

 # ... same neighbors as RS1

 address-family ipv4
  neighbor 198.51.100.56 activate
  # ... activate all
 !
!
```

**Key Differences from RS1**:
- Hostname: `IXP-SW2`
- Management IP: `192.168.100.51`
- Loopback: `198.51.100.252`
- SVI IP: `198.51.100.252`
- Router-ID: `198.51.100.252`
- **Everything else identical**

### ISP Configuration for Dual Route Servers

**On Level 3 Border-1** (add second interface + BGP peer):

```junos
# Second link to IXP-SW2
set interfaces ge-0/0/4 unit 0 description "to-IXP-SW2"
set interfaces ge-0/0/4 unit 0 family inet address 198.51.100.56/24

# Peer with both route servers
set protocols bgp group IXP-RS neighbor 198.51.100.251 description "IXP-RS1"
set protocols bgp group IXP-RS neighbor 198.51.100.252 description "IXP-RS2"
```

**Note**: Same IP on both interfaces (198.51.100.56) because both are on same subnet.

### Verification with Two Route Servers

**Check both BGP sessions**:

```bash
# On Level 3
show bgp summary | match IXP
```

**Expected**:
```
198.51.100.251   4 64999  1234  1235        0    0    0 01:23:45          150
198.51.100.252   4 64999  1200  1201        0    0    0 01:20:12          150
```

**Test redundancy**:

```bash
# Shutdown RS1
# On IXP-SW1
router bgp 64999
 shutdown
!
```

**Check from ISP**:

```bash
show bgp summary
```

**Expected**: RS1 down, RS2 still up, routes still learned from RS2.

### Option B: MLAG Configuration (Advanced)

**For future implementation** - this requires:
- MLAG peer link between IXP-SW1 and IXP-SW2
- Shared VLAN 100 across both switches
- LACP port-channels from ISPs to both switches
- More complex but more realistic

**Not covered in this guide** - implement Option A first, add Option B later as stretch goal.

---

## Best Practices Summary

1. **Start with one route server** (IXP-SW1), get it working fully
2. **Always use route-server-client mode** for transparent route distribution
3. **Apply sanity filtering** (prefix length, bogons) but keep it minimal
4. **Set max-prefix limits** to prevent route leaks
5. **Add second route server** after first is stable
6. **Log BGP neighbor changes** for troubleshooting
7. **Monitor BGP session health** and route counts
8. **Save configs after changes**: `write memory`
9. **Document in Netbox**: VLAN, subnet, all BGP sessions

---

## Configuration Template Summary

**Complete IXP-SW1 Minimal Config**:

```bash
hostname IXP-SW1

interface Management1
 ip address 192.168.100.50/24
!

interface Loopback0
 ip address 198.51.100.251/32
!

vlan 100
 name IXP-Peering-LAN
!

interface Ethernet1
 description to-Level3-Border1
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
!

# Repeat for Eth2-5 (other ISPs)

interface Vlan100
 description IXP-Route-Server-1
 ip address 198.51.100.251/24
!

router bgp 64999
 router-id 198.51.100.251
 maximum-paths 32

 neighbor 198.51.100.56 remote-as 3356
 neighbor 198.51.100.56 description Level3
 neighbor 198.51.100.56 route-server-client
 neighbor 198.51.100.56 send-community extended

 # Add all neighbors...

 address-family ipv4
  neighbor 198.51.100.56 activate
  # Activate all
 !
!

ip route 0.0.0.0/0 192.168.100.1

write memory
```

---

## Next Steps

1. ✅ **Configure IXP-SW1**: Follow Phases 1-5 above
2. **Update Tier-1 ISPs**: Add interface on 198.51.100.0/24, peer with RS1
3. **Update Tier-2 ISPs**: Add interface on 198.51.100.0/24, peer with RS1
4. **Test route distribution**: Announce test prefix, verify it appears on all peers
5. **Add filtering**: Implement prefix-list and max-prefix limits (Phase 4)
6. **Add IXP-SW2** (optional): Follow Phase 7 for redundancy
7. **Document in Netbox**: IXP VLAN, subnet, route server IPs, all BGP sessions

---

## VERSION HISTORY

- v1.0 (2026-02-09): Initial IXP route server guide for Arista vEOS
- v1.1 (2026-02-09): Updated to focus on single shared subnet with Layer 2 fabric, added Phase 7 for second route server
