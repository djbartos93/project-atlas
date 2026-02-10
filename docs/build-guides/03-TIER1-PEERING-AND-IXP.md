# Tier-1 Peering and IXP Architecture Guide

## NOT REVIEWED YET

## Overview

This guide explains how Tier-1 ISPs interconnect and how the Internet Exchange Point (IXP) fits into the topology.

## Note

All IP addresses and interface names are generic in this guide. You should replace them with your own IP addresses and interface names. See IP_ADDRESSING_PHILOSOPHY.md for more information.

## Learning Approach: Two-Phase Deployment

**Phase 1 (This Guide): Open Peering - Pass-All Policies**
- Configure BGP sessions with simple "pass-all" route policies
- **Cisco requirement**: IOS-XR mandates route policies on eBGP (won't exchange routes without them)
- See all routes flow freely between networks (no filtering, just acceptance)
- Understand the "before" state (what happens without restrictive policies)
- Easier to troubleshoot initial connectivity

**Phase 2 (Future Guide): Apply Restrictive Route Policies**
- Replace pass-all policies with selective export/import policies
- Compare before/after to visualize what filtering does
- Understand why Tier-1s don't announce peer routes to other peers
- See how policies prevent becoming unwanted transit

**Why this order?**: In a learning environment, seeing unrestricted routing first helps you understand what the policies are actually filtering. You'll be able to say "I see 10,000 routes from Cogent, but after I apply the filtering policy, I only see 500 customer routes" - much more educational than just following a config template.

**Note**: Junos doesn't require explicit policies (will accept/announce by default), but we'll add them anyway for consistency.

---

## Tier-1 Settlement-Free Peering

### What is Settlement-Free Peering?

**Definition**: Tier-1 ISPs exchange traffic with each other **without payment**. Neither ISP pays the other.

**Why?**: Both networks benefit equally - they need each other to reach customers.

### Level 3 ↔ Cogent Peering Architecture

**Production Design: Dual Border Router Peering**

**Both** border routers from each ISP peer with each other for full redundancy:

```
L3-Border-1 ──────── Cogent-Border-1
    │                     │
    │                     │
    │ (IXP)          (IXP)│
    │                     │
L3-Border-2 ──────── Cogent-Border-2
```

**Peering Links**:

| Link | L3 IP | Cogent IP | Subnet | Purpose |
|------|-------|-----------|--------|---------|
| Border-1 to Border-1 | 203.0.113.17/30 | 203.0.113.18/30 | /30 | Primary direct peering |
| Border-2 to Border-2 | 203.0.113.21/30 | 203.0.113.22/30 | /30 | Secondary direct peering |
| Both via IXP | 198.51.100.56, .61 | 198.51.100.57, .62 | /24 | Public peering fabric |

**Why Dual Peering?**:
- ✅ Redundancy: If one border fails, other maintains connectivity
- ✅ Load balancing: Traffic distributed across both paths
- ✅ Traffic engineering: Can influence path selection via BGP attributes

### Configuration Examples

**On L3-Border-1** (Junos):
```junos
# Direct peering to Cogent-Border-1
set interfaces ge-0/0/4 unit 0 description "to-Cogent-Border1"
set interfaces ge-0/0/4 unit 0 family inet address 203.0.113.17/30

set protocols bgp group TIER1-PEERS type external
set protocols bgp group TIER1-PEERS peer-as 174
set protocols bgp group TIER1-PEERS neighbor 203.0.113.18 description "Cogent-Border1"

# IXP peering
set interfaces ge-0/0/3 unit 0 family inet address 198.51.100.56/24
set protocols bgp group IXP-PEERS neighbor 198.51.100.251 description "IXP-RS1"
```

**On L3-Border-2** (Junos):
```junos
# Direct peering to Cogent-Border-2
set interfaces ge-0/0/4 unit 0 description "to-Cogent-Border2"
set interfaces ge-0/0/4 unit 0 family inet address 203.0.113.21/30

set protocols bgp group TIER1-PEERS type external
set protocols bgp group TIER1-PEERS peer-as 174
set protocols bgp group TIER1-PEERS neighbor 203.0.113.22 description "Cogent-Border2"

# IXP peering
set interfaces ge-0/0/3 unit 0 family inet address 198.51.100.61/24
set protocols bgp group IXP-PEERS neighbor 198.51.100.251 description "IXP-RS1"
```

**On Cogent-Border-1** (IOS-XR):
```cisco
# Phase 1: Simple pass-all route policies (required by IOS-XR)
route-policy PASS-ALL
 pass
end-policy
!

# Direct peering to Level3-Border-1
interface GigabitEthernet0/0/0/4
 description to-Level3-Border1
 ipv4 address 203.0.113.18 255.255.255.252
!

router bgp 174
 neighbor 203.0.113.17
  remote-as 3356
  description Level3-Border1
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
   # NOTE: Using simple pass-all policy for Phase 1 learning
   # Phase 2 will replace with selective filtering policies
  !
 !
!

# IXP peering
interface GigabitEthernet0/0/0/3
 ipv4 address 198.51.100.57 255.255.255.0
!

router bgp 174
 neighbor 198.51.100.251
  remote-as 64999
  description IXP-RS1
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
  !
 !
!
```

**On Cogent-Border-2** (IOS-XR):
```cisco
# Phase 1: Simple pass-all route policies (required by IOS-XR)
route-policy PASS-ALL
 pass
end-policy
!

# Direct peering to Level3-Border-2
interface GigabitEthernet0/0/0/4
 description to-Level3-Border2
 ipv4 address 203.0.113.22 255.255.255.252
!

router bgp 174
 neighbor 203.0.113.21
  remote-as 3356
  description Level3-Border2
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
   # NOTE: Using simple pass-all policy for Phase 1 learning
  !
 !
!

# IXP peering
interface GigabitEthernet0/0/0/3
 ipv4 address 198.51.100.62 255.255.255.0
!

router bgp 174
 neighbor 198.51.100.251
  remote-as 64999
  description IXP-RS1
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
  !
 !
!
```

**Peering Summary**:
- **Private peering**: Primary high-capacity path (dedicated /30 links)
- **IXP peering**: Backup path + connectivity to other IXP participants

### Critical Note: Cisco IOS-XR Route Policy Requirement

**IOS-XR requires route policies on ALL eBGP sessions**. Without both inbound and outbound policies configured, the BGP session will establish but exchange **zero routes**.

**Error you'll see if policies are missing**:
```
Some configured eBGP neighbors do not have both inbound and outbound
policies configured for IPv4 Unicast address family. These neighbors
will default to sending and/or receiving no routes and are marked
with '!' in the output.
```

**Solution for Phase 1** (open peering):
```cisco
route-policy PASS-ALL
 pass
end-policy
```

**Apply to all eBGP neighbors**:
```cisco
address-family ipv4 unicast
 route-policy PASS-ALL in
 route-policy PASS-ALL out
!
```

**This is different from Junos**, which will accept/announce routes by default without explicit policies.

---

## Internet Exchange Point (IXP) Architecture

### What is an IXP?

**Definition**: Neutral Layer 2 switching fabric where multiple networks peer.

**Purpose**:
- Reduce transit costs (peer directly instead of buying transit)
- Lower latency (direct paths instead of going through providers)
- Increase redundancy (multiple paths available)

**Who Runs It**: Independent entity (not any single ISP)

### IXP Components

```
┌─────────────────────────────────────────────────┐
│              IXP Switching Fabric               │
│                                                 │
│  ┌──────────────┐         ┌──────────────┐    │
│  │ Route Server │         │ Route Server │    │
│  │   RS1        │         │   RS2        │    │
│  │ AS 64999     │         │ AS 64999     │    │
│  └──────┬───────┘         └──────┬───────┘    │
│         │                        │             │
│         └────────┬───────────────┘             │
│                  │                             │
├──────────────────┼─────────────────────────────┤
│                  │  Layer 2 Fabric             │
│                  │  (Single Subnet)            │
│         198.51.100.0/24                        │
└─────────┬────┬───┴───┬────┬────┬──────────────┘
          │    │       │    │    │
    ┌─────┴┐ ┌─┴────┐ ┌┴───┐┌───┴──┐
    │Level3│ │Cogent│ │Atlas││Second│
    │ .56  │ │ .57  │ │ .58 ││ .59  │
    └──────┘ └──────┘ └─────┘└──────┘
```

### Route Servers Explained

**Without Route Servers** (Bilateral Peering):
- Each network needs BGP sessions to EVERY other network
- 4 networks = 6 sessions (N*(N-1)/2)
- 10 networks = 45 sessions ❌ (doesn't scale)

**With Route Servers** (Multilateral Peering):
- Each network peers only with route servers
- Route servers distribute routes between all participants
- 10 networks = 20 sessions (2 route servers × 10 networks) ✅

**Route Server Behavior**:
- Receives routes from all participants
- Announces routes to all participants
- Does NOT modify AS-path (remains transparent)
- Does NOT transit traffic (only exchanges routing info)

---

## IXP Configuration

### Network Addressing

**IXP Peering LAN**: `198.51.100.0/24`

**IP Assignments**:
- Route Server 1: `198.51.100.251`
- Route Server 2: `198.51.100.252`
- Level 3 Border-1: `198.51.100.56`
- Cogent Border-1: `198.51.100.57`
- Atlas Lab ISP: `198.51.100.58`
- Second ISP: `198.51.100.59`
- CDN Network: `198.51.100.60`
- Content Provider: `198.51.100.62`
- Eyeball ISP: `198.51.100.63`

### Level 3 IXP Configuration

**On L3-Border-1**:

```junos
set interfaces ge-0/0/3 unit 0 description "to-IXP"
set interfaces ge-0/0/3 unit 0 family inet address 198.51.100.56/24

set protocols bgp group IXP-RS type external
set protocols bgp group IXP-RS multihop ttl 1
set protocols bgp group IXP-RS peer-as 64999
set protocols bgp group IXP-RS neighbor 198.51.100.251 description "IXP-RS1"
set protocols bgp group IXP-RS neighbor 198.51.100.252 description "IXP-RS2"

set protocols bgp group IXP-RS export TO-IXP
set protocols bgp group IXP-RS import FROM-IXP
```

**Why `multihop ttl 1`?**: Route servers are directly connected but Junos requires multihop for external BGP with same subnet.

### Cogent IXP Configuration

**On Cogent-Border-1**:

```cisco
# Phase 1: Simple pass-all route policy (IOS-XR requirement)
route-policy PASS-ALL
 pass
end-policy
!

interface GigabitEthernet0/0/0/3
 description to-IXP
 ipv4 address 198.51.100.57 255.255.255.0
!

router bgp 174
 neighbor 198.51.100.251
  remote-as 64999
  description IXP-RS1
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
   # NOTE: Phase 1 uses pass-all (no filtering)
   # See "Route Policies" section at bottom for Phase 2 implementation
  !
 !

 neighbor 198.51.100.252
  remote-as 64999
  description IXP-RS2
  address-family ipv4 unicast
   route-policy PASS-ALL in
   route-policy PASS-ALL out
  !
 !
!
```

### Route Server Configuration

**Platform**: Arista vEOS-lab (Layer 2 switch with SVI for route server)

**On IXP-SW1** (Arista EOS):

```
! VLAN for IXP peering fabric
vlan 100
 name IXP-Peering-LAN
!

! Layer 2 ports to participants
interface Ethernet1
 description to-Level3-Border1
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
!

interface Ethernet2
 description to-Cogent-Border1
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
!

interface Ethernet3
 description to-Atlas-Edge1
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
!

interface Ethernet4
 description to-PastyNet-Edge1
 switchport mode access
 switchport access vlan 100
 spanning-tree portfast
!

! SVI for route server functionality
interface Vlan100
 description IXP-Route-Server-1
 ip address 198.51.100.251/24
!

! Enable routing
ip routing

! BGP route server configuration
router bgp 64999
 router-id 198.51.100.251

 ! Level 3 peer
 neighbor 198.51.100.56 remote-as 3356
 neighbor 198.51.100.56 description Level3-Border1
 neighbor 198.51.100.56 route-server-client
 neighbor 198.51.100.56 maximum-routes 12000

 ! Cogent peer
 neighbor 198.51.100.57 remote-as 174
 neighbor 198.51.100.57 description Cogent-Border1
 neighbor 198.51.100.57 route-server-client
 neighbor 198.51.100.57 maximum-routes 12000

 ! Atlas Lab ISP peer
 neighbor 198.51.100.58 remote-as 65000
 neighbor 198.51.100.58 description Atlas-Lab-ISP
 neighbor 198.51.100.58 route-server-client
 neighbor 198.51.100.58 maximum-routes 12000

 ! PastyNet ISP peer
 neighbor 198.51.100.59 remote-as 65100
 neighbor 198.51.100.59 description PastyNet-ISP
 neighbor 198.51.100.59 route-server-client
 neighbor 198.51.100.59 maximum-routes 12000
!
```

**Key Settings**:
- `route-server-client`: Preserves AS-path through route server (doesn't add AS 64999)
- `maximum-routes`: Protects against route leaks
- Layer 2 fabric with SVI: Route server runs on switch interface, not separate VM

---

## Complete Peering Topology

### Physical Connections

```
Level 3 Border Routers:
├── L3-Border-1 ─┬─> IXP (198.51.100.56)
│                └─> Cogent-Border-1 Direct (203.0.113.17)
│
└── L3-Border-2 ─┬─> IXP (198.51.100.56 - anycast, or .61 unique)
                 └─> Atlas Lab ISP Transit (TBD)

Cogent Border Routers:
├── Cogent-Border-1 ─┬─> IXP (198.51.100.57)
│                    └─> Level 3 Border-1 Direct (203.0.113.18)
│
└── Cogent-Border-2 ─┬─> IXP (198.51.100.57 - anycast, or .62 unique)
                     └─> Second ISP Transit (TBD)

IXP Route Servers:
├── RS1 (198.51.100.251)
└── RS2 (198.51.100.252)
```

### BGP Peering Matrix

| Network | Type | Peers With | Relationship |
|---------|------|------------|--------------|
| Level 3 | Tier-1 | Cogent (direct), IXP (RS) | Settlement-free |
| Cogent | Tier-1 | Level 3 (direct), IXP (RS) | Settlement-free |
| Atlas Lab ISP | Tier-2 | Level 3 (transit), Cogent (transit), IXP (peer) | Pays Tier-1s |
| Second ISP | Tier-2 | Level 3 or Cogent (transit), IXP (peer) | Pays Tier-1s |
| CDN | Content | IXP (peer), maybe Atlas ISP (PNI) | Settlement-free |

---

## Traffic Flow Examples

### Example 1: Level 3 Customer → Cogent Customer

**Without Direct Peering** (BAD):
```
L3-Customer → L3-Border → Atlas-ISP → Cogent-Border → Cogent-Customer
              (transit through Atlas - Atlas gets paid by both!)
```

**With Direct Peering** (GOOD):
```
L3-Customer → L3-Border ─(direct link)─> Cogent-Border → Cogent-Customer
              (direct path, no intermediary)
```

### Example 2: Atlas ISP Customer → Internet

**Path 1: Via Level 3**
```
Atlas-Customer → Atlas-ISP → L3-Border → L3-Core → Internet
                 (Atlas pays Level 3)
```

**Path 2: Via Cogent**
```
Atlas-Customer → Atlas-ISP → Cogent-Border → Cogent-Core → Internet
                 (Atlas pays Cogent)
```

**Path 3: Via IXP Peer**
```
Atlas-Customer → Atlas-ISP → IXP → CDN-Network
                 (free peering, no transit cost!)
```

---

## Route Policies for Settlement-Free Peering (Phase 2)

**Note**: This section shows example policies for reference, but they are **NOT applied in Phase 1**. After you get basic peering working and can see all routes flowing, a separate guide will walk through applying these policies and observing the changes.

### Level 3 Policy to Cogent

**Export Policy** (what Level 3 announces to Cogent):

```junos
set policy-options policy-statement TO-COGENT term ANNOUNCE-CUSTOMERS from protocol bgp
set policy-options policy-statement TO-COGENT term ANNOUNCE-CUSTOMERS from community LEARNED-FROM-CUSTOMER
set policy-options policy-statement TO-COGENT term ANNOUNCE-CUSTOMERS then accept

set policy-options policy-statement TO-COGENT term ANNOUNCE-OWN from protocol static
set policy-options policy-statement TO-COGENT term ANNOUNCE-OWN then accept

set policy-options policy-statement TO-COGENT term DENY-ALL then reject
```

**What this does**:
- ✅ Announce Level 3's own routes
- ✅ Announce Level 3's customer routes (get paid to carry)
- ❌ Do NOT announce routes learned from other peers (won't transit for free)

**Import Policy** (what Level 3 accepts from Cogent):

```junos
set policy-options policy-statement FROM-COGENT term ACCEPT-ALL then local-preference 100
set policy-options policy-statement FROM-COGENT term ACCEPT-ALL then accept
```

**In production, add**:
- Prefix filtering (no bogons, RFC1918, etc.)
- Max-prefix limits
- AS-path filtering

### Cogent Policy to Level 3

Mirror of above, adjusted for IOS-XR syntax:

```cisco
route-policy TO-LEVEL3
 if community matches-any LEARNED-FROM-CUSTOMER then
  pass
 elseif protocol eq static then
  pass
 else
  drop
 endif
end-policy
!

route-policy FROM-LEVEL3
 set local-preference 100
 pass
end-policy
!
```

---

## Verification and Testing

### Verify Tier-1 Direct Peering

**On L3-Border-1**:
```junos
show bgp neighbor 203.0.113.18
show route receive-protocol bgp 203.0.113.18
```

**Expected**: BGP session `Established`, receiving Cogent routes.

### Verify IXP Peering

**On any router connected to IXP**:
```
show bgp neighbor 198.51.100.251
show route receive-protocol bgp 198.51.100.251 | count
```

**Expected**: Routes from ALL IXP participants visible.

### Test Tier-1 to Tier-1 Traffic Flow

**From L3-Border-1**:
```junos
traceroute 10.255.2.11 source 10.255.1.11
```

**Expected Path**:
```
1  10.1.0.x (L3 core)
2  203.0.113.18 (direct to Cogent-Border-1)
3  10.2.0.x (Cogent core)
4  10.255.2.11 (Cogent-Border-1 loopback)
```

**If path goes through Atlas ISP** → Missing direct peering!

### Test IXP Route Propagation

**From Atlas Lab ISP**:
```
show route protocol bgp | match "via 198.51.100"
```

**Expected**: Routes learned via IXP route servers.

---

## Best Practices

1. **Always peer Tier-1s directly**: Settlement-free peering is realistic
2. **Use both private and IXP peering**: Redundancy and load balancing
3. **Apply proper export policies**: Don't become transit for peers
4. **Monitor BGP sessions**: IXP route servers going down = lost routes
5. **Document peering agreements**: Who peers with whom and why

---

## Next Steps

**Phase 1 (Basic Peering)**:
1. Configure IXP switch (Arista) with Layer 2 fabric and route server
2. Connect both Tier-1s to IXP (BGP sessions to route servers)
3. Establish direct Tier-1 peering (Level 3 ↔ Cogent private link)
4. Verify BGP sessions establish (all routes flow freely)
5. Add Atlas Lab ISP and PastyNet ISP to IXP
6. Test traffic paths and observe full route tables

**Phase 2 (Route Policy Implementation)** - Future Guide:
1. Document current route tables (before policies)
2. Apply export policies (control what you announce)
3. Apply import policies (control what you accept)
4. Compare before/after route tables
5. Verify Tier-1s don't transit for each other
6. Test traffic engineering with local-preference

---

## VERSION HISTORY

- v1.0 (2026-02-09): Initial Tier-1 peering and IXP architecture guide
