# Cogent Tier-1 ISP Configuration Guide

# In human review

## Overview

This guide walks through building the Cogent (AS 174) Tier-1 ISP simulation using Cisco IOS-XRv 9000 routers. The architecture mirrors Level 3 but uses Cisco IOS-XR syntax and best practices.

## Note

All IP addresses and interface names are generic in this guide. You should replace them with your own IP addresses and interface names. See IP_ADDRESSING_PHILOSOPHY.md for more information.

### Network Architecture

```
Cogent (AS 174)
├── Core-1 (Cogent-Core-1)
├── Core-2 (Cogent-Core-2)
├── Border-1 (Cogent-Border-1)
└── Border-2 (Cogent-Border-2)

Internal Routing:
- IGP: ISIS (Level 2)
- MPLS: LDP for label distribution
- BGP: iBGP with route reflectors

External Peering:
- Level 3: Direct settlement-free peering + IXP
- IXP: Public peering fabric
- Transit Customers: Second ISP, others
```

### IOS-XR vs Junos Key Differences

| Feature | Junos | IOS-XR |
|---------|-------|--------|
| Config mode | `edit` hierarchy | Flat config mode |
| Commit | `commit` | `commit` |
| Interface naming | `ge-0/0/0` | `GigabitEthernet0/0/0/0` |
| Show commands | `show isis adjacency` | `show isis adjacency` (same!) |
| Config application | Immediate on commit | Immediate on commit |

**IOS-XR Tip**: Use `commit replace` to replace entire config, or just `commit` to merge changes.

---

## Phase 1: Foundation - Basic Connectivity

### Step 1: Management Interface Configuration

**IOS-XR Configuration**:

```cisco
hostname Cogent-Core-1

interface MgmtEth0/RP0/CPU0/0
 description Management
 ipv4 address 192.168.100.21 255.255.255.0
 no shutdown
!

router static
 address-family ipv4 unicast
  0.0.0.0/0 192.168.100.1
 !
!
```

**Management IP Scheme**:
- Cogent-Core-1: `192.168.100.21/24`
- Cogent-Core-2: `192.168.100.22/24`
- Cogent-Border-1: `192.168.100.23/24`
- Cogent-Border-2: `192.168.100.24/24`

**Verification**:
```cisco
show ipv4 interface brief
ping 192.168.100.1
```

### Step 2: Loopback Interface Assignment

```cisco
interface Loopback0
 description Router-ID for ISIS and BGP
 ipv4 address 10.255.2.1 255.255.255.255
!
```

**Loopback IP Scheme**:
- Cogent-Core-1: `10.255.2.1/32`
- Cogent-Core-2: `10.255.2.2/32`
- Cogent-Border-1: `10.255.2.11/32`
- Cogent-Border-2: `10.255.2.12/32`

**ISIS NET Address** (configured later):
- Format: `49.0002.0102.5500.2001.00` (derived from 010.255.002.001)

### Step 3: Core Interconnect Links

```cisco
interface GigabitEthernet0/0/0/0
 description to-Cogent-Core-2
 ipv4 address 10.2.0.0 255.255.255.254
 no shutdown
!
```

**Core Topology** (using /31 subnets):
- Cogent-Core-1 ↔ Cogent-Core-2: `10.2.0.0/31` and `10.2.0.2/31`
- Cogent-Core-1 ↔ Cogent-Border-1: `10.2.0.4/31`
- Cogent-Core-1 ↔ Cogent-Border-2: `10.2.0.6/31`
- Cogent-Core-2 ↔ Cogent-Border-1: `10.2.0.8/31`
- Cogent-Core-2 ↔ Cogent-Border-2: `10.2.0.10/31`
- Cogent-Border-1 ↔ Cogent-Border-2: `10.2.0.12/31`

**Verification**:
```cisco
show ipv4 interface brief
ping 10.2.0.1 source 10.2.0.0
```

---

## Phase 2: Interior Gateway Protocol (ISIS)

### Goal
Enable ISIS Level 2 across all internal links for loopback reachability.

### Step 1: Enable ISIS Globally

```cisco
router isis COGENT
 is-type level-2-only
 net 49.0002.0102.5500.2001.00
 address-family ipv4 unicast
  metric-style wide
  mpls traffic-eng level-2-only
  mpls traffic-eng router-id Loopback0
 !
!
```

**Configuration Breakdown**:
- `is-type level-2-only`: Only Level 2 routing (inter-area)
- `net 49.0002...`: ISO NET address (unique per router)
- `metric-style wide`: Use 32-bit metrics
- `mpls traffic-eng`: Prepare for MPLS (configured in Phase 3)

**NET Address Format**:
- `49.0002` = Private AFI + Area ID (different from Level 3's 0001)
- `0102.5500.2001` = System ID from loopback IP
- `00` = NSEL (always 00)

### Step 2: Enable ISIS on Interfaces

On each core-facing interface:

```cisco
router isis COGENT
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
   metric 10
  !
 !
!
```

**Enable on Loopback**:

```cisco
router isis COGENT
 interface Loopback0
  passive
  address-family ipv4 unicast
  !
 !
!
```

**Why `passive`?**: Advertises loopback into ISIS without sending hellos.

### Step 3: ISIS Authentication (Recommended)

```cisco
router isis COGENT
 lsp-password hmac-md5 encrypted <password>
!
```

**IOS-XR Note**: Use `encrypted` keyword and IOS-XR will hash the password automatically.

### Verification

Check ISIS adjacencies:
```cisco
show isis adjacency
show isis database
show isis route
```

**Expected Output**: All four routers adjacent, ISIS routes to all loopbacks visible.

Test reachability:
```cisco
ping 10.255.2.2 source 10.255.2.1
traceroute 10.255.2.12 source 10.255.2.1
```

---

## Phase 3: MPLS with LDP

### Goal
Enable MPLS label switching across the core for future L3VPN and traffic engineering.

### Step 1: Enable MPLS Globally

```cisco
mpls ldp
 router-id 10.255.2.1
 address-family ipv4
 !
!
```

### Step 2: Enable MPLS on Interfaces

On each core-facing interface:

```cisco
interface GigabitEthernet0/0/0/0
 no shutdown
!

mpls ldp
 interface GigabitEthernet0/0/0/0
  address-family ipv4
  !
 !
!
```

**Critical**: MPLS must be enabled on ALL internal ISIS interfaces (not external eBGP interfaces).

### Step 3: ISIS-LDP Synchronization

```cisco
router isis COGENT
 interface GigabitEthernet0/0/0/0
  address-family ipv4 unicast
   mpls ldp sync
  !
 !
!
```

**Why Sync?**: Prevents ISIS from using link before LDP converges (avoids blackholes).

### Verification

Check MPLS status:
```cisco
show mpls interfaces
show mpls ldp neighbor brief
show mpls ldp bindings
```

**Expected Output**: LDP neighbors established, labels assigned to loopback routes.

Check label forwarding:
```cisco
show mpls forwarding
```

Test MPLS connectivity:
```cisco
ping mpls ipv4 10.255.2.12/32 source 10.255.2.1
traceroute mpls ipv4 10.255.2.12/32
```

---

## Phase 4: Internal BGP (iBGP) with Route Reflectors

### Goal
Establish iBGP sessions using Core routers as Route Reflectors.

### Step 1: Configure Route Reflectors (Core Routers)

On Cogent-Core-1 and Cogent-Core-2:

```cisco
router bgp 174
 bgp router-id 10.255.2.1
 bgp cluster-id 0.0.0.2
 address-family ipv4 unicast
 !

 neighbor-group RR-CLIENTS
  remote-as 174
  update-source Loopback0
  address-family ipv4 unicast
   route-reflector-client
  !
 !

 neighbor 10.255.2.2
  use neighbor-group RR-CLIENTS
  description Cogent-Core-2-RR-Peer
  address-family ipv4 unicast
   ! Not a client - this is peer RR
   route-reflector-client disable
  !
 !

 neighbor 10.255.2.11
  use neighbor-group RR-CLIENTS
  description Cogent-Border-1-RR-Client
 !

 neighbor 10.255.2.12
  use neighbor-group RR-CLIENTS
  description Cogent-Border-2-RR-Client
 !
!
```

**Key Parameters**:
- `bgp cluster-id`: Identifies RR cluster (use different value than Level 3)
- `route-reflector-client`: Makes neighbor a client
- `update-source Loopback0`: Use stable loopback IP

**IOS-XR Feature**: Neighbor groups simplify configuration (like Junos groups).

### Step 2: Configure RR Clients (Border Routers)

On Cogent-Border-1 and Cogent-Border-2:

```cisco
router bgp 174
 bgp router-id 10.255.2.11
 address-family ipv4 unicast
 !

 neighbor 10.255.2.1
  remote-as 174
  description Cogent-Core-1-RR
  update-source Loopback0
  address-family ipv4 unicast
  !
 !

 neighbor 10.255.2.2
  remote-as 174
  description Cogent-Core-2-RR
  update-source Loopback0
  address-family ipv4 unicast
  !
 !
!
```

**Note**: Clients don't need `route-reflector-client` config - only RRs do.

### Verification

Check BGP sessions:
```cisco
show bgp summary
show bgp neighbors 10.255.2.2
```

**Expected Output**: All iBGP sessions `Established`.

Check route reflection:
```cisco
show bgp advertised neighbor 10.255.2.11
```

---

## Phase 5: External BGP (eBGP) Configuration

### Goal
Configure external peering with IXP route servers, Level 3 (settlement-free peering), and transit customers.

### External Peering Architecture

**Both border routers peer externally for redundancy**:

| Cogent Router | External Peer | Peer Type | Interface | IP Address | Peer IP |
|---------------|---------------|-----------|-----------|------------|---------|
| Cogent-Border-1 | IXP-RS1 | Public Peering | Gi0/0/0/3 | 198.51.100.57/24 | 198.51.100.251 |
| Cogent-Border-1 | Level3-Border-1 | Private Peering | Gi0/0/0/4 | 203.0.113.18/30 | 203.0.113.17 |
| Cogent-Border-2 | IXP-RS1 | Public Peering | Gi0/0/0/3 | 198.51.100.62/24 | 198.51.100.251 |
| Cogent-Border-2 | Level3-Border-2 | Private Peering | Gi0/0/0/4 | 203.0.113.22/30 | 203.0.113.21 |

**Why Dual Border Peering?**:
- ✅ Redundancy: If Border-1 fails, Border-2 maintains all external peering
- ✅ Load balancing: Traffic distributed across both paths
- ✅ Traffic engineering: Can prefer one border over another using local-preference/MED

### Step 1: Configure External Interfaces

**On Cogent-Border-1**:

```cisco
# Interface to IXP (shared subnet)
interface GigabitEthernet0/0/0/3
 description to-IXP
 ipv4 address 198.51.100.57 255.255.255.0
 no shutdown
!

# Interface to Level3-Border-1 (point-to-point)
interface GigabitEthernet0/0/0/4
 description to-Level3-Border1
 ipv4 address 203.0.113.18 255.255.255.252
 no shutdown
!
```

**On Cogent-Border-2**:

```cisco
# Interface to IXP (shared subnet, different IP)
interface GigabitEthernet0/0/0/3
 description to-IXP
 ipv4 address 198.51.100.62 255.255.255.0
 no shutdown
!

# Interface to Level3-Border-2 (point-to-point)
interface GigabitEthernet0/0/0/4
 description to-Level3-Border2
 ipv4 address 203.0.113.22 255.255.255.252
 no shutdown
!
```

### Step 2: eBGP to IXP Route Server

**On both border routers** (same config):

```cisco
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

### Step 3: eBGP to Level 3 (Settlement-Free Peering)

**On Cogent-Border-1**:

```cisco
router bgp 174
 neighbor 203.0.113.17
  remote-as 3356
  description Level3-Border1
  address-family ipv4 unicast
   route-policy FROM-TIER1-PEER in
   route-policy TO-TIER1-PEER out
  !
 !
!
```

**On Cogent-Border-2**:

```cisco
router bgp 174
 neighbor 203.0.113.21
  remote-as 3356
  description Level3-Border2
  address-family ipv4 unicast
   route-policy FROM-TIER1-PEER in
   route-policy TO-TIER1-PEER out
  !
 !
!
```

**Peering Type**: Settlement-free (Tier-1 to Tier-1, no payment either direction)

### Step 4: BGP Route Policies

**IOS-XR requires explicit route policies** (no default accept):

**Import Policies**:

```cisco
route-policy FROM-IXP
  # Accept all for now (add filtering in production)
  set community (174:200) additive
  set local-preference 100
  pass
end-policy
!

route-policy FROM-TIER1-PEER
  # Accept from Tier-1 peer (Level 3)
  set community (174:300) additive
  set local-preference 100
  pass
end-policy
!
```

**Export Policies** (announce own routes + customer routes, NOT peer routes):

```cisco
# Define communities
community-set LEARNED-FROM-CUSTOMER
  174:100
end-set
!

community-set LEARNED-FROM-PEER
  174:200
end-set
!

# Export policy - don't provide free transit for peer routes
route-policy TO-PEERS
  # Announce customer routes (we get paid)
  if community matches-any LEARNED-FROM-CUSTOMER then
    pass
  # Announce own routes
  elseif protocol eq static then
    pass
  # Don't announce peer routes (would be free transit)
  else
    drop
  endif
end-policy
!

# Apply to both IXP and Tier-1 peering
router bgp 174
 neighbor 198.51.100.251
  address-family ipv4 unicast
   route-policy FROM-IXP in
   route-policy TO-PEERS out
  !
 !

 neighbor 203.0.113.17
  address-family ipv4 unicast
   route-policy FROM-TIER1-PEER in
   route-policy TO-PEERS out
  !
 !
!
```

**Why Export Policy Matters**:
- ✅ Announce own routes (originated by Cogent)
- ✅ Announce customer routes (we get paid to transit)
- ❌ Do NOT announce peer routes (won't provide free transit for Level 3's customers)

**Production Additions**:
- Prefix length limits (no /32s, no > /24)
- Bogon filtering (RFC1918, etc.)
- AS-path filtering
- Max-prefix limits

### Verification

**Check all eBGP sessions**:
```cisco
show bgp summary
```

**Expected Output**:
```
Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
198.51.100.251    0 64999    1234    1235        0    0    0 01:23:45        150
203.0.113.17      0  3356    5678    5679        0    0    0 02:15:30        200
```

**Check specific peers**:
```cisco
show bgp neighbors 198.51.100.251
show bgp neighbors 203.0.113.17
```

**Check received/advertised routes**:
```cisco
show bgp neighbors 198.51.100.251 routes
show bgp neighbors 203.0.113.17 advertised-routes
```

**Verify Tier-1 peering path** (from Border-1 to Border-2 via Level 3):
```cisco
traceroute 10.255.2.12 source Loopback0
```

**Expected**: Should see path through Level 3 routers (203.0.113.17 → Level 3 network → 203.0.113.21)

---

## Phase 6: Traffic Engineering with Local-Preference and MED

### Local Preference (Inbound Path Selection)

Higher = more preferred:

```cisco
route-policy FROM-CUSTOMERS
  set local-preference 200
  pass
end-policy
!

route-policy FROM-PEERS
  set local-preference 100
  pass
end-policy
!

route-policy FROM-TRANSIT
  set local-preference 50
  pass
end-policy
!
```

**Apply to neighbors**:

```cisco
router bgp 174
 neighbor 203.0.113.17
  address-family ipv4 unicast
   route-policy FROM-PEERS in
  !
 !
!
```

### MED (Outbound Path Influence)

Lower = more preferred by neighbor:

```cisco
route-policy TO-PEERS-MED100
  set med 100
  pass
end-policy
!

router bgp 174
 neighbor 198.51.100.251
  address-family ipv4 unicast
   route-policy TO-PEERS-MED100 out
  !
 !
!
```

---

## Phase 7: Verification and Testing

### Comprehensive Health Checks

**1. IGP Status**:
```cisco
show isis adjacency
show isis route
show route isis
```

**2. MPLS Functionality**:
```cisco
show mpls ldp neighbor brief
show mpls interfaces
show mpls forwarding
ping mpls ipv4 10.255.2.12/32
```

**3. iBGP Status**:
```cisco
show bgp summary
show bgp ipv4 unicast
show route bgp
```

**4. eBGP Status**:
```cisco
show bgp summary
show bgp neighbors 198.51.100.251 routes | count
```

**5. Route Reflection Verification**:
```cisco
# On Core-1
show bgp neighbors 10.255.2.11 advertised-routes
```

**6. End-to-End Connectivity**:
```cisco
traceroute 1.1.1.1 source Loopback0
```

---

## Common Issues and Troubleshooting

### Issue 1: ISIS Adjacency Not Forming

**Symptoms**: `show isis adjacency` empty

**Check**:
```cisco
show isis interface
show isis protocol
show logging
```

**Common Causes**:
- Interface not added to ISIS process
- IS-type mismatch (Level 1 vs Level 2)
- MTU mismatch
- Wrong NET address

### Issue 2: No MPLS Labels

**Symptoms**: `show mpls forwarding` empty

**Check**:
```cisco
show mpls ldp neighbor
show mpls ldp interface
show isis route
```

**Common Causes**:
- LDP not enabled on interface
- ISIS not converged first
- No IGP routes to assign labels to

### Issue 3: iBGP Sessions Not Establishing

**Symptoms**: BGP shows `Active` or `Idle`

**Check**:
```cisco
show bgp neighbors 10.255.2.2
show route 10.255.2.2
ping 10.255.2.2 source Loopback0
```

**Common Causes**:
- Loopback not reachable (ISIS problem)
- Wrong `update-source` configured
- Firewall blocking TCP 179

### Issue 4: eBGP Routes Not in Routing Table

**Symptoms**: BGP receives routes but not in `show route`

**Check**:
```cisco
show bgp
show route bgp
show bgp neighbors 198.51.100.251 routes
```

**Common Causes**:
- Route policy blocking routes
- Next-hop unreachable
- Better route from another protocol

### Issue 5: IOS-XR Commit Failures

**Symptoms**: Config changes rejected on `commit`

**Check**:
```cisco
show configuration failed
show configuration failed inheritance
```

**Common Causes**:
- Incomplete configuration block
- Missing `end-policy` or closing `!`
- Invalid route-policy syntax

**Fix**: Use `abort` to discard changes, fix syntax, retry

---

## IOS-XR Specific Tips

### Configuration Mode Navigation

```cisco
configure
 router bgp 174
  neighbor 10.255.2.2
   address-family ipv4 unicast
   !
  !
 !
commit
exit
```

**Tip**: Use `!` to exit one level, `exit` to leave config mode entirely.

### Viewing Uncommitted Changes

```cisco
show configuration
show configuration merge
```

### Rollback Feature

```cisco
show configuration commit list
rollback configuration to 2
commit
```

**Use Case**: Undo last change if it breaks connectivity.

### Running Configuration

```cisco
show running-config
show running-config router bgp
show running-config interface GigabitEthernet0/0/0/0
```

---

## Best Practices Summary

1. **Always commit frequently** to save progress
2. **Use descriptive interface descriptions** for documentation
3. **Test each phase** before moving forward
4. **Check logs** when troubleshooting: `show logging`
5. **Use `show configuration failed`** to debug commit errors
6. **Document IP addressing** in Netbox as you build
7. **Save configs externally**: `show running-config > tftp://...`

---

## Next Steps

Once Cogent is fully operational:

1. **Document in Netbox**: All interfaces, IPs, BGP sessions
2. **Establish Tier-1 Peering**: Connect Cogent ↔ Level 3 directly + at IXP
3. **Configure IXP Route Servers**: Enable multi-lateral peering
4. **Build Atlas Lab ISP**: Start your Tier-2 regional ISP
5. **Add monitoring**: SNMP, NetFlow, syslog

---

## Configuration Snippets Reference

### Quick ISIS Enable (IOS-XR)
```cisco
router isis COGENT
 interface GigabitEthernet0/0/0/X
  point-to-point
  address-family ipv4 unicast
  !
 !
!
```

### Quick MPLS/LDP Enable
```cisco
mpls ldp
 interface GigabitEthernet0/0/0/X
  address-family ipv4
  !
 !
!
```

### Quick iBGP Neighbor Add
```cisco
router bgp 174
 neighbor 10.255.2.X
  remote-as 174
  update-source Loopback0
  address-family ipv4 unicast
  !
 !
!
```

### Quick eBGP Peer Add
```cisco
router bgp 174
 neighbor X.X.X.X
  remote-as XXXXX
  description Peer-Name
  address-family ipv4 unicast
   route-policy FROM-PEER in
   route-policy TO-PEER out
  !
 !
!
```

---

## Syntax Comparison: Junos vs IOS-XR

| Task | Junos | IOS-XR |
|------|-------|--------|
| Enter config | `configure` | `configure` |
| Set hostname | `set system host-name` | `hostname X` |
| Interface IP | `set interfaces ge-0/0/0 unit 0 family inet address` | `interface GigE0/0/0/0`<br>`ipv4 address` |
| Enable ISIS | `set protocols isis interface` | `router isis X`<br>`interface Y` |
| BGP neighbor | `set protocols bgp neighbor X` | `router bgp Y`<br>`neighbor X` |
| Commit config | `commit` | `commit` |
| Show BGP | `show bgp summary` | `show bgp summary` |

---

*This guide provides IOS-XR equivalent configurations for the Level 3 Junos build. The concepts are identical; only syntax differs.*

## VERSION HISTORY

- v1.0 (2026-02-09): Initial Cogent Tier-1 build guide for Cisco IOS-XR
