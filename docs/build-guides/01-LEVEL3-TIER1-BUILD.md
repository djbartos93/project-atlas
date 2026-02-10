# Level 3 Tier-1 ISP Configuration Guide

# In human review 

## Overview

This guide walks through building the Level 3 (AS 3356) Tier-1 ISP simulation using Juniper vMX routers. The focus is on understanding **why** we configure each component, not just copying configurations.

### Network Architecture

```
Level 3 (AS 3356)
├── Core-1 (L3-Core-1)
├── Core-2 (L3-Core-2)
├── Border-1 (L3-Border-1)
└── Border-2 (L3-Border-2)

Internal Routing:
- IGP: ISIS (Level 2)
- MPLS: LDP for label distribution
- BGP: iBGP full mesh for route distribution

External Peering:
- IXP: Settlement-free peering with other networks
- Cogent: Tier-1 settlement-free peering
- Transit Customers: Atlas Lab ISP, others
```

### Design Principles

**Separation of Concerns**:
- **Core routers**: Handle internal routing, MPLS transit, route reflection
- **Border routers**: Handle external BGP, traffic engineering, filtering

**Why This Matters**: In production networks, core routers focus on fast packet forwarding while border routers handle the complexity of internet routing policy.

---

## Phase 1: Foundation - Basic Connectivity

### Goal
Establish management access and baseline configuration on all four routers.

### Step 1: Management Interface Configuration

**On each router**, configure management interface and basic system settings:

```junos
set system host-name L3-Core-1
set system root-authentication plain-text-password

set interfaces fxp0 unit 0 family inet address 192.168.100.x/24
set routing-options static route 0.0.0.0/0 next-hop 192.168.100.1
```

**Key Points**:
- `fxp0` is the out-of-band management interface
- Set unique management IPs for each router
- Static default route ensures management reachability

**Verification**:
```junos
show interfaces fxp0 terse
ping 192.168.100.1
```

### Step 2: Loopback Interface Assignment

Each router needs a unique loopback for ISIS router-ID and MPLS LSR-ID:

```junos
set interfaces lo0 unit 0 family inet address 10.255.1.1/32
set interfaces lo0 unit 0 family iso address 49.0001.0102.5500.1001.00
```

**Loopback IP Scheme**:
- L3-Core-1: `10.255.1.1/32`
- L3-Core-2: `10.255.1.2/32`
- L3-Border-1: `10.255.1.11/32`
- L3-Border-2: `10.255.1.12/32`

**ISO Address Breakdown**:
- `49.0001` = Private AFI (Area ID)
- `0102.5500.1001` = System ID (derived from IP: 010.255.001.001)
- `00` = NSEL (always 00)

**Why ISO Addresses?**: ISIS uses ISO/CLNS addressing for the routing protocol itself, even though we're routing IP.

### Step 3: Core Interconnect Links

Configure point-to-point links between core routers:

```junos
# On L3-Core-1
set interfaces ge-0/0/0 unit 0 description "to-L3-Core-2"
set interfaces ge-0/0/0 unit 0 family inet address 10.1.0.0/31
set interfaces ge-0/0/0 unit 0 family iso
set interfaces ge-0/0/0 unit 0 family mpls
```

**Why `/31` subnets?**: RFC 3021 allows /31 for point-to-point links, saving IP space (no network/broadcast addresses needed).

**Why `family iso` and `family mpls`?**:
- ISO: Required for ISIS to run on the interface
- MPLS: Enables MPLS label switching

**Core Topology**:
- L3-Core-1 ↔ L3-Core-2: `10.1.0.0/31` and `10.1.0.2/31`
- L3-Core-1 ↔ L3-Border-1: `10.1.0.4/31`
- L3-Core-1 ↔ L3-Border-2: `10.1.0.6/31`
- L3-Core-2 ↔ L3-Border-1: `10.1.0.8/31`
- L3-Core-2 ↔ L3-Border-2: `10.1.0.10/31`
- L3-Border-1 ↔ L3-Border-2: `10.1.0.12/31`
**Verification**:
```junos
show interfaces terse | match ge-0/0/
ping 10.1.0.1 source 10.1.0.0
```

---

## Phase 2: Interior Gateway Protocol (ISIS)

### Goal
Enable ISIS Level 2 across all internal links to establish reachability between loopbacks.

### Step 1: Enable ISIS Globally

```junos
set protocols isis level 1 disable
set protocols isis level 2 wide-metrics-only
set protocols isis reference-bandwidth 100g

set interfaces lo0 unit 0 family iso address 49.0001.0102.5500.1001.00
```

**Configuration Breakdown**:
- `level 1 disable`: We only need Level 2 (inter-area routing)
- `wide-metrics-only`: Use 32-bit metrics (default is 6-bit, insufficient for modern networks)
- `reference-bandwidth 100g`: Calculate metrics based on 100G links (adjust cost calculation)

### Step 2: Enable ISIS on Interfaces

On each internal interface:

```junos
set interfaces ge-0/0/0 unit 0 family iso
set protocols isis interface ge-0/0/0.0 point-to-point
set protocols isis interface ge-0/0/0.0 level 2 metric 10
```

**Enable on loopback** (so other routers learn about it):

```junos
set protocols isis interface lo0.0 passive
```

**Why `passive`?**: Loopbacks don't need to send ISIS hellos, but must be advertised.

### Step 3: ISIS Authentication (Optional but Recommended)

```junos
set protocols isis level 2 authentication-key "$9$secure-key-here"
set protocols isis level 2 authentication-type md5
```

**Why Authenticate?**: Prevents rogue routers from injecting routes into your IGP.

### Verification

Check ISIS adjacencies:
```junos
show isis adjacency
show isis database
show isis route
```

**Expected Output**: All four routers should see each other as adjacent, and ISIS should have routes to all loopbacks.

Test reachability:
```junos
ping 10.255.1.2 source 10.255.1.1
traceroute 10.255.1.12 source 10.255.1.1
```

---

## Phase 3: MPLS with LDP

### Goal
Enable MPLS label switching to support future L3VPN services and traffic engineering.

### Step 1: Enable MPLS on Interfaces

On each core-facing interface:

```junos
set interfaces ge-0/0/0 unit 0 family mpls
set protocols mpls interface ge-0/0/0.0
```

**Why MPLS?**: Even if not using VPNs immediately, MPLS provides traffic engineering capabilities and prepares the network for future services.

### Step 2: Configure LDP

```junos
set protocols ldp interface ge-0/0/0.0
set protocols ldp interface lo0.0
```

**What is LDP?**: Label Distribution Protocol - automatically distributes MPLS labels for IGP routes.

### Step 3: Synchronize ISIS with LDP

```junos
set protocols isis interface ge-0/0/0.0 ldp-synchronization
```

**Why Synchronization?**: Prevents ISIS from using a link before LDP has converged, avoiding blackholes.

### Verification

Check MPLS interfaces:
```junos
show mpls interface
```

Check LDP neighbors:
```junos
show ldp neighbor
show ldp database
```

Verify label assignments:
```junos
show route table inet.3
```

**Expected Output**: You should see MPLS labels assigned to each loopback route in `inet.3`.

Test MPLS forwarding:
```junos
ping mpls ldp 10.255.1.12 source 10.255.1.1
traceroute mpls ldp 10.255.1.12
```

---

## Phase 4: Internal BGP (iBGP)

### Goal
Establish iBGP sessions between all routers to exchange external routes learned by border routers.

### Architecture Decision: Route Reflectors

**Traditional Full Mesh**: All routers peer with each other (N*(N-1)/2 sessions)
- 4 routers = 6 iBGP sessions ✅ (manageable)

**Route Reflector**: Core routers act as RRs, border routers are clients
- 4 routers = 5 sessions ✅ (more scalable)

**Choice for This Lab**: Route Reflectors (demonstrates production-like design)

### Step 1: Configure Route Reflectors (Core Routers)

On L3-Core-1 and L3-Core-2:

```junos
set routing-options router-id 10.255.1.1
set routing-options autonomous-system 3356

set protocols bgp group INTERNAL type internal
set protocols bgp group INTERNAL local-address 10.255.1.1
set protocols bgp group INTERNAL family inet unicast
set protocols bgp group INTERNAL cluster 0.0.0.1

# Add RR clients (border routers)
set protocols bgp group INTERNAL neighbor 10.255.1.11
set protocols bgp group INTERNAL neighbor 10.255.1.12

# Peer with other RR
set protocols bgp group INTERNAL neighbor 10.255.1.2
```

**Key Parameters**:
- `local-address`: Use loopback (more stable than physical interface)
- `cluster`: Identifies RR cluster (prevent loops)
- `type internal`: iBGP (AS 3356 to AS 3356)

### Step 2: Configure RR Clients (Border Routers)

On L3-Border-1 and L3-Border-2:

```junos
set routing-options router-id 10.255.1.11
set routing-options autonomous-system 3356

set protocols bgp group INTERNAL type internal
set protocols bgp group INTERNAL local-address 10.255.1.11
set protocols bgp group INTERNAL family inet unicast

# Peer with both route reflectors
set protocols bgp group INTERNAL neighbor 10.255.1.1
set protocols bgp group INTERNAL neighbor 10.255.1.2
```

**Note**: Clients don't need `cluster` configuration - only RRs do.

### Verification

Check BGP sessions:
```junos
show bgp summary
show bgp neighbor 10.255.1.2
```

**Expected Output**: All iBGP sessions should be `Established`.

Check route reflection:
```junos
show route advertising-protocol bgp 10.255.1.11
```

---

## Phase 5: External BGP (eBGP) - Border Routers Only

### Goal
Configure external peering with IXP route servers, Cogent (settlement-free peering), and transit customers.

### External Peering Architecture

**Both border routers peer externally for redundancy**:

| L3 Router | External Peer | Peer Type | Interface | IP Address | Peer IP |
|-----------|---------------|-----------|-----------|------------|---------|
| L3-Border-1 | IXP-RS1 | Public Peering | ge-0/0/3 | 198.51.100.56/24 | 198.51.100.251 |
| L3-Border-1 | Cogent-Border-1 | Private Peering | ge-0/0/4 | 203.0.113.17/30 | 203.0.113.18 |
| L3-Border-2 | IXP-RS1 | Public Peering | ge-0/0/3 | 198.51.100.61/24 | 198.51.100.251 |
| L3-Border-2 | Cogent-Border-2 | Private Peering | ge-0/0/4 | 203.0.113.21/30 | 203.0.113.22 |

**Why Dual Border Peering?**:
- ✅ Redundancy: If Border-1 fails, Border-2 maintains all external peering
- ✅ Load balancing: Traffic distributed across both paths
- ✅ Traffic engineering: Can prefer one border over another using MED/communities

### Step 1: Configure External Interfaces

**On L3-Border-1**:

```junos
# Interface to IXP (shared subnet)
set interfaces ge-0/0/3 unit 0 description "to-IXP"
set interfaces ge-0/0/3 unit 0 family inet address 198.51.100.56/24

# Interface to Cogent-Border-1 (point-to-point)
set interfaces ge-0/0/4 unit 0 description "to-Cogent-Border1"
set interfaces ge-0/0/4 unit 0 family inet address 203.0.113.17/30
```

**On L3-Border-2**:

```junos
# Interface to IXP (shared subnet, different IP)
set interfaces ge-0/0/3 unit 0 description "to-IXP"
set interfaces ge-0/0/3 unit 0 family inet address 198.51.100.61/24

# Interface to Cogent-Border-2 (point-to-point)
set interfaces ge-0/0/4 unit 0 description "to-Cogent-Border2"
set interfaces ge-0/0/4 unit 0 family inet address 203.0.113.21/30
```

### Step 2: eBGP Session to IXP

**On both border routers** (same config):

```junos
set protocols bgp group IXP-PEERS type external
set protocols bgp group IXP-PEERS peer-as 64999
set protocols bgp group IXP-PEERS family inet unicast
set protocols bgp group IXP-PEERS neighbor 198.51.100.251 description "IXP-RS1"
```

**Key Differences from iBGP**:
- `type external`: Different AS numbers
- `peer-as`: Remote AS (IXP route server AS)
- No `local-address` needed (uses physical interface IP)

### Step 3: eBGP to Cogent (Settlement-Free Peering)

**On L3-Border-1**:

```junos
set protocols bgp group TIER1-PEERS type external
set protocols bgp group TIER1-PEERS peer-as 174
set protocols bgp group TIER1-PEERS family inet unicast
set protocols bgp group TIER1-PEERS neighbor 203.0.113.18 description "Cogent-Border1"
```

**On L3-Border-2**:

```junos
set protocols bgp group TIER1-PEERS type external
set protocols bgp group TIER1-PEERS peer-as 174
set protocols bgp group TIER1-PEERS family inet unicast
set protocols bgp group TIER1-PEERS neighbor 203.0.113.22 description "Cogent-Border2"
```

**Peering Type**: Settlement-free (Tier-1 to Tier-1, no payment either direction)

### Step 4: BGP Route Policies

**Import Policy** (what we accept):

```junos
# Basic acceptance for now (add filtering in production)
set policy-options policy-statement FROM-IXP-PEERS term DEFAULT-ACCEPT then accept
set policy-options policy-statement FROM-TIER1-PEER term DEFAULT-ACCEPT then accept

# In production, you'd add filtering here:
# - Prefix length limits (no /32s, no > /24)
# - Bogon filtering (RFC1918, etc.)
# - Max-prefix limits
```

**Export Policy** (what we announce):

```junos
# Announce own routes and customer routes (NOT peer routes)
set policy-options policy-statement TO-PEERS term ANNOUNCE-CUSTOMERS from protocol bgp
set policy-options policy-statement TO-PEERS term ANNOUNCE-CUSTOMERS from community LEARNED-FROM-CUSTOMER
set policy-options policy-statement TO-PEERS term ANNOUNCE-CUSTOMERS then accept

set policy-options policy-statement TO-PEERS term ANNOUNCE-OWN from protocol static
set policy-options policy-statement TO-PEERS term ANNOUNCE-OWN then accept

set policy-options policy-statement TO-PEERS term DENY-ALL then reject

# Apply policies to both IXP and Tier-1 peering
set protocols bgp group IXP-PEERS import FROM-IXP-PEERS
set protocols bgp group IXP-PEERS export TO-PEERS

set protocols bgp group TIER1-PEERS import FROM-TIER1-PEER
set protocols bgp group TIER1-PEERS export TO-PEERS
```

**Why Export Policy Matters**:
- ✅ Announce own routes (originated by Level 3)
- ✅ Announce customer routes (we get paid to transit)
- ❌ Do NOT announce peer routes (won't provide free transit for Cogent's customers)

### Step 5: BGP Communities for Traffic Engineering

Define communities for marking routes:

```junos
set policy-options community LEARNED-FROM-PEER members 3356:200
set policy-options community LEARNED-FROM-CUSTOMER members 3356:100
set policy-options community LEARNED-FROM-TIER1-PEER members 3356:300

# Tag routes on import
set policy-options policy-statement FROM-IXP-PEERS term TAG then community add LEARNED-FROM-PEER
set policy-options policy-statement FROM-IXP-PEERS term TAG then accept

set policy-options policy-statement FROM-TIER1-PEER term TAG then community add LEARNED-FROM-TIER1-PEER
set policy-options policy-statement FROM-TIER1-PEER term TAG then accept
```

**Use Case**: These communities help with route selection and policy application throughout the network.

### Verification

**Check all eBGP sessions**:
```junos
show bgp summary
```

**Expected Output**:
```
Peer                AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active
198.51.100.251   64999       1234       1235       0       0       1:23:45 Established
203.0.113.18       174       5678       5679       0       0       2:15:30 Established
```

**Check specific peers**:
```junos
show bgp neighbor 198.51.100.251
show bgp neighbor 203.0.113.18
```

**Check received/advertised routes**:
```junos
show route receive-protocol bgp 198.51.100.251
show route advertising-protocol bgp 203.0.113.18
```

**Verify Tier-1 peering path** (from Border-1 to Border-2 via Cogent):
```junos
traceroute 10.255.1.12 source 10.255.1.11
```

**Expected**: Should see path through Cogent routers (203.0.113.18 → Cogent network → 203.0.113.22)

---

## Phase 6: Multi-Exit Discriminator (MED) and Local Preference

### Goal
Implement basic traffic engineering to prefer certain paths.

### Local Preference (Inbound Traffic Engineering)

Higher local-preference = more preferred:

```junos
# Prefer routes learned from customers over peers
set policy-options policy-statement FROM-CUSTOMERS term DEFAULT then local-preference 200
set policy-options policy-statement FROM-PEERS term DEFAULT then local-preference 100
set policy-options policy-statement FROM-TRANSIT term DEFAULT then local-preference 50
```

**Why It Matters**:
- Customer routes = we get paid to carry traffic ✅
- Peer routes = free exchange ⚠️
- Transit routes = we pay to send traffic ❌

### MED (Outbound Traffic Engineering)

Lower MED = more preferred by neighbor:

```junos
# Prefer traffic to enter via Border-1
set policy-options policy-statement TO-PEERS term SET-MED then metric 100
```

**Limitation**: MED only influences **that specific neighbor's** decision. Not all ISPs honor MED.

---

## Phase 7: Verification and Testing

### Comprehensive Checks

**1. IGP Health**:
```junos
show isis adjacency
show isis route
```

**2. MPLS Functionality**:
```junos
show ldp neighbor
show mpls interface
ping mpls ldp 10.255.1.12
```

**3. iBGP Status**:
```junos
show bgp summary group INTERNAL
show route protocol bgp table inet.0
```

**4. eBGP Status**:
```junos
show bgp summary group IXP-PEERS
show route receive-protocol bgp 198.51.100.251 | count
```

**5. Route Reflection**:
```junos
# On Core-1, check what's being reflected
show route advertising-protocol bgp 10.255.1.11 detail
```

**6. End-to-End Connectivity**:
```junos
# From Border-1, test reaching external destination through Core-2
traceroute 8.8.8.8 source 10.255.1.11
```

---

## Common Issues and Troubleshooting

### Issue 1: ISIS Adjacency Not Forming

**Symptoms**: `show isis adjacency` returns empty

**Check**:
```junos
show isis interface
show log messages | match ISIS
```

**Common Causes**:
- Interface not enabled for ISIS (`set protocols isis interface ge-0/0/0.0`)
- Missing `family iso` on interface
- ISIS level mismatch (Level 1 vs Level 2)
- MTU mismatch

### Issue 2: No MPLS Labels

**Symptoms**: `show route table inet.3` is empty

**Check**:
```junos
show ldp neighbor
show ldp interface
show log messages | match LDP
```

**Common Causes**:
- LDP not enabled on interface
- ISIS not converged (LDP waits for IGP)
- Missing `family mpls` on interface

### Issue 3: iBGP Sessions Down

**Symptoms**: `show bgp summary` shows `Active` or `Connect`

**Check**:
```junos
show bgp neighbor 10.255.1.2
show route 10.255.1.2
ping 10.255.1.2 source 10.255.1.1
```

**Common Causes**:
- Loopback not reachable (IGP issue)
- Wrong `local-address` configured
- Firewall blocking TCP 179

### Issue 4: eBGP Routes Not Propagating to Core

**Symptoms**: Border router learns routes but core doesn't see them

**Check**:
```junos
# On border router
show route advertising-protocol bgp 10.255.1.1

# On core router
show route receive-protocol bgp 10.255.1.11
```

**Common Causes**:
- Missing export policy on border router
- Next-hop not in IGP (need `next-hop self` in some cases)
- Route reflector misconfiguration

---

## Best Practices Summary

1. **Always verify each phase** before moving to the next
2. **Use descriptive interface descriptions** for documentation
3. **Save configurations frequently**: `commit and-quit`
4. **Document your IP addressing** in Netbox as you build
5. **Test from multiple routers** to verify end-to-end reachability
6. **Check logs** when troubleshooting: `show log messages | last 50`

---

## Next Steps

Once Level 3 is fully operational:

1. **Document in Netbox**: All interfaces, IP addresses, BGP sessions
2. **Create baseline configs**: Save working configs for disaster recovery
3. **Build Cogent (AS 174)**: Replicate this process for second Tier-1
4. **Establish Tier-1 peering**: Connect Level 3 and Cogent at IXP
5. **Add monitoring**: SNMP, NetFlow, syslog integration

---

## Configuration Snippets Reference

### Quick ISIS Enable
```junos
set protocols isis interface ge-0/0/X.0 point-to-point level 2 metric 10
set interfaces ge-0/0/X unit 0 family iso
```

### Quick MPLS/LDP Enable
```junos
set interfaces ge-0/0/X unit 0 family mpls
set protocols mpls interface ge-0/0/X.0
set protocols ldp interface ge-0/0/X.0
```

### Quick iBGP Neighbor Add
```junos
set protocols bgp group INTERNAL neighbor 10.255.1.X
```

### Quick eBGP Peer Add
```junos
set protocols bgp group EXTERNAL neighbor X.X.X.X peer-as XXXXX description "Peer-Name"
```

---

*This guide focuses on understanding the "why" behind each configuration. Actual production networks will have additional security, redundancy, and policy configurations not covered here.*

## VERSION HISTORY

- v1.0 (2026-02-08): Initial build guide for Level 3 Tier-1 ISP
