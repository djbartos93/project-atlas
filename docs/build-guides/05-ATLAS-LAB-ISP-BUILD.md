# Atlas Lab ISP (Tier-2) Configuration Guide

## NOT REVIEWED YET

## Note

All IP addresses and interface names are generic in this guide. You should replace them with your own IP addresses and interface names. See IP_ADDRESSING_PHILOSOPHY.md for more information.

## Overview

This guide walks through building Atlas Lab ISP (AS 65000) - your on-prem Tier-2 regional ISP. This is the "middleman" ISP that:
- Buys transit from both Tier-1 ISPs (Level 3 and Cogent)
- Peers at the IXP for free peering
- Provides services to on-prem customers

### Network Architecture

```
Atlas Lab ISP (AS 65000)
├── Core-1 (Atlas-Core-1)
├── Core-2 (Atlas-Core-2)
├── Edge-1 (Atlas-Edge-1) - Customer facing
└── Edge-2 (Atlas-Edge-2) - Customer facing

Internal Routing:
- IGP: OSPF (Area 0)
- MPLS: LDP for L3VPN services
- BGP: iBGP full mesh (only 4 routers)

External Peering:
- Level 3: Transit (we pay them)
- Cogent: Transit (we pay them)
- IXP: Public peering (free)
- Customers: Service delivery
```

### Key Differences from Tier-1

**Why OSPF instead of ISIS?**
- Tier-1s use ISIS (faster, more scalable)
- Regional ISPs often use OSPF (more common, easier to find engineers)
- Both work fine for this scale

**Why Full Mesh iBGP?**
- Only 4 routers = 6 sessions (manageable)
- Route reflectors add complexity we don't need yet

**Platform**: VyOS (lightweight, open-source, perfect for learning)

---

## Phase 1: Foundation - Basic Connectivity

### Goal
Establish management access and basic Layer 3 connectivity between all routers.

### Step 1: Management and Loopback Configuration

**On Atlas-Core-1**:

```bash
set system host-name Atlas-Core-1

# Management interface
set interfaces ethernet eth0 address '192.168.100.30/24'
set protocols static route 0.0.0.0/0 next-hop 192.168.100.1

# Loopback for OSPF router-ID and MPLS
set interfaces loopback lo address '10.0.0.1/32'
```

**Loopback IP Scheme**:
- Atlas-Core-1: `10.0.0.1/32`
- Atlas-Core-2: `10.0.0.2/32`
- Atlas-Edge-1: `10.0.0.11/32`
- Atlas-Edge-2: `10.0.0.12/32`

### Step 2: Core Interconnect Links

**Atlas Internal Network**: `10.0.1.0/24` subnet

```bash
# Atlas-Core-1 to Core-2
set interfaces ethernet eth1 address '10.0.1.1/30'
set interfaces ethernet eth1 description 'to-Atlas-Core-2'

# Atlas-Core-1 to Edge-1
set interfaces ethernet eth2 address '10.0.1.5/30'
set interfaces ethernet eth2 description 'to-Atlas-Edge-1'

# Atlas-Core-1 to Edge-2
set interfaces ethernet eth3 address '10.0.1.9/30'
set interfaces ethernet eth3 description 'to-Atlas-Edge-2'
```

**Complete Internal Topology**:
- Core-1 ↔ Core-2: `10.0.1.0/30`
- Core-1 ↔ Edge-1: `10.0.1.4/30`
- Core-1 ↔ Edge-2: `10.0.1.8/30`
- Core-2 ↔ Edge-1: `10.0.1.12/30`
- Core-2 ↔ Edge-2: `10.0.1.16/30`

**Verification**:
```bash
ping 10.0.1.2 source-address 10.0.1.1
show interfaces
```

---

## Phase 2: Interior Gateway Protocol (OSPF)

### Goal
Enable OSPF Area 0 across all internal links for loopback reachability.

### Understanding OSPF vs ISIS

**Why OSPF?**
- Industry standard for enterprise/regional ISPs
- Easier to find engineers who know it
- Simpler initial configuration than ISIS
- Area-based design (we're using single area for simplicity)

**OSPF Area 0**: Backbone area, all routers in same area

### Step 1: Enable OSPF Globally

```bash
set protocols ospf parameters router-id '10.0.0.1'
set protocols ospf area 0
```

**Router-ID**: Use loopback IP for stability

### Step 2: Enable OSPF on Interfaces

**On loopback** (advertise it to OSPF):

```bash
set protocols ospf area 0 network '10.0.0.1/32'
```

**On internal interfaces**:

```bash
set protocols ospf area 0 network '10.0.1.0/30'
set protocols ospf area 0 network '10.0.1.4/30'
set protocols ospf area 0 network '10.0.1.8/30'
```

**Interface-specific settings**:

```bash
set protocols ospf interface eth1 network 'point-to-point'
set protocols ospf interface eth1 priority '0'
```

**Why point-to-point?**: No need for DR/BDR elections on point-to-point links (faster convergence)

### Step 3: OSPF Authentication (Optional but Recommended)

```bash
set protocols ospf area 0 authentication 'md5'
set interfaces ethernet eth1 ip ospf authentication md5 key-id 1 md5-key 'YourSecretKey'
```

### Verification

```bash
show ip ospf neighbor
show ip ospf database
show ip route ospf
```

**Expected**: All routers see each other as neighbors, loopbacks are reachable

Test reachability:
```bash
ping 10.0.0.11 source-address 10.0.0.1
traceroute 10.0.0.12
```

---

## Phase 3: MPLS with LDP

### Goal
Enable MPLS for future L3VPN customer services.

### Why MPLS for Regional ISP?

**Use Cases**:
- L3VPN: Give customers their own private routing table
- Traffic engineering: Steer traffic through specific paths
- QoS: Different treatment for different customers

### Step 1: Enable MPLS on Interfaces

On all internal interfaces:

```bash
set interfaces ethernet eth1 mpls
set interfaces ethernet eth2 mpls
set interfaces ethernet eth3 mpls
```

### Step 2: Configure LDP

```bash
set protocols mpls ldp interface 'eth1'
set protocols mpls ldp interface 'eth2'
set protocols mpls ldp interface 'eth3'
set protocols mpls ldp interface 'lo'

set protocols mpls ldp router-id '10.0.0.1'
```

**Why LDP?**: Automatically distributes labels for OSPF routes (no manual label config needed)

### Verification

```bash
show mpls ldp neighbor
show mpls ldp binding
show mpls table
```

**Expected**: LDP neighbors established, labels assigned to loopback routes

---

## Phase 4: Internal BGP (iBGP)

### Goal
Establish iBGP full mesh to exchange external routes.

### Why Full Mesh (Not Route Reflectors)?

**With 4 routers**:
- Full mesh = 6 iBGP sessions
- Route reflectors = 5 sessions + complexity

**Decision**: Full mesh is simpler at this scale

### Step 1: Configure iBGP on All Routers

**On Atlas-Core-1**:

```bash
set protocols bgp system-as '65000'
set protocols bgp parameters router-id '10.0.0.1'

# iBGP neighbors (all other routers)
set protocols bgp neighbor 10.0.0.2 remote-as '65000'
set protocols bgp neighbor 10.0.0.2 description 'Atlas-Core-2'
set protocols bgp neighbor 10.0.0.2 update-source 'lo'

set protocols bgp neighbor 10.0.0.11 remote-as '65000'
set protocols bgp neighbor 10.0.0.11 description 'Atlas-Edge-1'
set protocols bgp neighbor 10.0.0.11 update-source 'lo'

set protocols bgp neighbor 10.0.0.12 remote-as '65000'
set protocols bgp neighbor 10.0.0.12 description 'Atlas-Edge-2'
set protocols bgp neighbor 10.0.0.12 update-source 'lo'

# Address family
set protocols bgp neighbor 10.0.0.2 address-family ipv4-unicast
set protocols bgp neighbor 10.0.0.11 address-family ipv4-unicast
set protocols bgp neighbor 10.0.0.12 address-family ipv4-unicast
```

**Key Settings**:
- `update-source lo`: Use loopback (more stable than physical interface)
- `remote-as 65000`: Same AS = iBGP

**Repeat for all routers** (change router-ID and neighbor IPs accordingly)

### Verification

```bash
show bgp summary
show bgp neighbors
```

**Expected**: All 6 iBGP sessions `Established`

---

## Phase 5: External BGP (eBGP) - Transit and Peering

### Goal
Configure transit (paid) with Tier-1s and free peering at IXP.

### External Peering Architecture

**Atlas Lab ISP External Connections**:

| Atlas Router | External Peer | Relationship | Interface | IP Address | Peer IP |
|--------------|---------------|--------------|-----------|------------|---------|
| Core-1 | Level3-Border-2 | Transit (paid) | eth4 | 100.64.0.1/30 | 100.64.0.2 |
| Core-2 | Cogent-Border-2 | Transit (paid) | eth4 | 100.64.0.5/30 | 100.64.0.6 |
| Core-1 | IXP-RS1 | Peering (free) | eth5 | 198.51.100.58/24 | 198.51.100.251 |

**Why These Connections?**:
- **Dual transit**: If Level 3 fails, Cogent provides backup
- **IXP peering**: Free peering with other networks (CDN, etc.)
- **Redundancy**: Multiple paths for reliability

### Step 1: Configure Transit to Level 3

**On Atlas-Core-1**:

```bash
# Interface to Level 3
set interfaces ethernet eth4 address '100.64.0.1/30'
set interfaces ethernet eth4 description 'to-Level3-Transit'

# eBGP session
set protocols bgp neighbor 100.64.0.2 remote-as '3356'
set protocols bgp neighbor 100.64.0.2 description 'Level3-Transit'
set protocols bgp neighbor 100.64.0.2 address-family ipv4-unicast
```

**What is Transit?**:
- We PAY Level 3 to carry our traffic
- They announce full internet table to us
- We announce our customer routes to them

### Step 2: Configure Transit to Cogent

**On Atlas-Core-2**:

```bash
# Interface to Cogent
set interfaces ethernet eth4 address '100.64.0.5/30'
set interfaces ethernet eth4 description 'to-Cogent-Transit'

# eBGP session
set protocols bgp neighbor 100.64.0.6 remote-as '174'
set protocols bgp neighbor 100.64.0.6 description 'Cogent-Transit'
set protocols bgp neighbor 100.64.0.6 address-family ipv4-unicast
```

### Step 3: Configure IXP Peering

**On Atlas-Core-1** (can also use Edge routers):

```bash
# Interface to IXP
set interfaces ethernet eth5 address '198.51.100.58/24'
set interfaces ethernet eth5 description 'to-IXP'

# eBGP session to route server
set protocols bgp neighbor 198.51.100.251 remote-as '64999'
set protocols bgp neighbor 198.51.100.251 description 'IXP-RS1'
set protocols bgp neighbor 198.51.100.251 address-family ipv4-unicast
```

### Step 4: BGP Route Policies

**Import Policy** (what we accept from transit):

```bash
# Accept default route from transit providers
set policy route-map FROM-TRANSIT rule 10 action 'permit'
set policy route-map FROM-TRANSIT rule 10 set local-preference '50'

# Apply to Level 3
set protocols bgp neighbor 100.64.0.2 address-family ipv4-unicast route-map import 'FROM-TRANSIT'
```

**Why local-preference 50?**: Prefer customer routes (200) and peer routes (100) over expensive transit

**Export Policy** (what we announce to transit):

```bash
# Announce only customer routes
set policy route-map TO-TRANSIT rule 10 action 'permit'
set policy route-map TO-TRANSIT rule 10 match community 'CUSTOMER-ROUTES'

set protocols bgp neighbor 100.64.0.2 address-family ipv4-unicast route-map export 'TO-TRANSIT'
```

**Why limit exports?**: Don't announce routes learned from peers/transit (would be providing free transit)

### Verification

```bash
show bgp summary
show ip route bgp
```

**Expected**: BGP sessions to Level 3, Cogent, and IXP all `Established`

**Test internet connectivity** (once receiving routes):
```bash
ping 8.8.8.8 source-address 10.0.0.1
traceroute 1.1.1.1
```

---

## Phase 6: Customer-Facing Configuration

### Goal
Prepare edge routers to deliver services to customers.

### Customer Service Types

**Basic Internet** (default):
- Customer gets public IPs
- Basic BGP or static routing
- No isolation from other customers

**L3VPN** (private):
- Customer gets own routing table (VRF)
- RFC1918 addressing
- Isolated from other customers

### Step 1: Configure Customer Interfaces

**On Atlas-Edge-1**:

```bash
# Interface to Customer-1
set interfaces ethernet eth4 address '100.64.10.1/30'
set interfaces ethernet eth4 description 'to-Customer-1'
```

### Step 2: Basic BGP to Customer

```bash
set protocols bgp neighbor 100.64.10.2 remote-as '65001'
set protocols bgp neighbor 100.64.10.2 description 'Customer-1'
set protocols bgp neighbor 100.64.10.2 address-family ipv4-unicast
```

**Customer Policies**:
```bash
# Accept customer routes and tag them
set policy community-list CUSTOMER-ROUTES rule 10 action 'permit'
set policy community-list CUSTOMER-ROUTES rule 10 regex '65000:100'

set policy route-map FROM-CUSTOMER rule 10 action 'permit'
set policy route-map FROM-CUSTOMER rule 10 set community '65000:100'
set policy route-map FROM-CUSTOMER rule 10 set local-preference '200'
```

**Why local-preference 200?**: Prefer customer routes over everything (we get paid!)

---

## Common Issues and Troubleshooting

### Issue 1: OSPF Neighbors Not Forming

**Check**:
```bash
show ip ospf neighbor
show log | match OSPF
```

**Common Causes**:
- Interface not in OSPF area
- Area mismatch
- Authentication mismatch
- Network type mismatch (broadcast vs point-to-point)

### Issue 2: No MPLS Labels

**Check**:
```bash
show mpls ldp neighbor
show mpls ldp interface
```

**Common Causes**:
- MPLS not enabled on interface
- LDP not configured
- OSPF not converged first

### Issue 3: iBGP Routes Not Installing

**Symptoms**: Routes received via BGP but not in routing table

**Check**:
```bash
show bgp ipv4 unicast summary
show ip route bgp
```

**Common Causes**:
- Next-hop not reachable (need IGP)
- `next-hop-self` not configured on edge routers
- Better route from another protocol

### Issue 4: Transit Not Working

**Check**:
```bash
show bgp summary
ping 8.8.8.8 source-address 10.0.0.1
```

**Common Causes**:
- Transit provider not announcing default route
- Export policy blocking customer routes
- NAT issues (if using RFC1918 internally)

---

## Best Practices

1. **Always use loopbacks for BGP sessions** (more stable)
2. **Set local-preference** to control inbound path selection
3. **Tag routes with communities** for easier policy application
4. **Keep OSPF area simple** (single area 0 is fine)
5. **Monitor BGP session flaps** (`show bgp summary`)
6. **Document all external peering** (who, what, why)

---

## Next Steps

1. Complete Atlas Lab ISP build (all 4 routers)
2. Verify transit from both Level 3 and Cogent
3. Test IXP peering
4. Connect first customer
5. Document in Netbox

---

## Configuration Snippets Reference

### Quick OSPF Enable
```bash
set protocols ospf area 0 network '10.0.0.1/32'
set protocols ospf interface eth1 network 'point-to-point'
```

### Quick MPLS/LDP Enable
```bash
set interfaces ethernet eth1 mpls
set protocols mpls ldp interface 'eth1'
```

### Quick iBGP Neighbor
```bash
set protocols bgp neighbor 10.0.0.X remote-as '65000'
set protocols bgp neighbor 10.0.0.X update-source 'lo'
```

### Quick eBGP Transit
```bash
set protocols bgp neighbor X.X.X.X remote-as 'XXXX'
set protocols bgp neighbor X.X.X.X description 'Transit-Provider'
```

---

*This guide focuses on learning the concepts behind regional ISP operations. Adapt configurations to match your specific topology and requirements.*

## VERSION HISTORY

- v1.0 (2026-02-09): Initial Atlas Lab ISP build guide for VyOS
