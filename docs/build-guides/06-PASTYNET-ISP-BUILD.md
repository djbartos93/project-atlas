# PastyNet ISP (Tier-2) Configuration Guide

## NOT REVIEWED YET

## Note

All IP addresses and interface names are generic in this guide. You should replace them with your own IP addresses and interface names. See IP_ADDRESSING_PHILOSOPHY.md for more information.

## Overview

This guide walks through building PastyNet (AS 65100) - your cloud-side Tier-2 regional ISP. Named after the Michigan UP's local ISP, this is a **simplified** 2-router design representing a smaller regional provider.

### Network Architecture

```
PastyNet (AS 65100)
├── Edge-1 (Pasty-Edge-1) - Primary router
└── Edge-2 (Pasty-Edge-2) - Backup/customer router

Internal Routing:
- IGP: OSPF (Area 0) - just 2 routers!
- MPLS: Optional (not required for 2-router design)
- BGP: iBGP single session

External Peering:
- Cogent OR Level 3: Transit (single provider for simplicity)
- IXP: Public peering (free)
- Cloud Customer: Service delivery
```

### Key Design Decisions

**Why Only 2 Routers?**
- Smaller regional ISP simulation
- Simpler to manage and configure
- Still demonstrates core concepts
- Represents "branch" ISP design

**Why No MPLS?**
- 2 routers don't need label switching
- Direct connectivity between routers
- Saves resources and complexity

**Platform**: VyOS or Cisco CSR1000v (your choice)

---

## Phase 1: Foundation - Basic Connectivity

### Goal
Establish connectivity between the two routers and to external networks.

### Network Topology

```
Pasty-Edge-1 ←─ (10.1.1.0/30) ─→ Pasty-Edge-2
     │                                 │
     ↓                                 ↓
  To Cogent                      To Customer
  (Transit)                      (Cloud-CE)
     │
     ↓
  To IXP
  (Peering)
```

### Step 1: Management and Loopback Configuration

**On Pasty-Edge-1** (VyOS):

```bash
set system host-name Pasty-Edge-1

# Management
set interfaces ethernet eth0 address '192.168.100.35/24'
set protocols static route 0.0.0.0/0 next-hop 192.168.100.1

# Loopback
set interfaces loopback lo address '10.1.0.1/32'
```

**On Pasty-Edge-2**:

```bash
set system host-name Pasty-Edge-2

# Management
set interfaces ethernet eth0 address '192.168.100.36/24'

# Loopback
set interfaces loopback lo address '10.1.0.2/32'
```

### Step 2: Internal Link

**Between the two routers**:

**On Pasty-Edge-1**:
```bash
set interfaces ethernet eth1 address '10.1.1.1/30'
set interfaces ethernet eth1 description 'to-Pasty-Edge-2'
```

**On Pasty-Edge-2**:
```bash
set interfaces ethernet eth1 address '10.1.1.2/30'
set interfaces ethernet eth1 description 'to-Pasty-Edge-1'
```

**Verification**:
```bash
ping 10.1.1.2 source-address 10.1.1.1
```

---

## Phase 2: Interior Gateway Protocol (OSPF)

### Goal
Enable OSPF for internal routing (even though it's just 2 routers, good practice).

### Why OSPF for 2 Routers?

**Could use static routes**, but OSPF provides:
- Automatic failover if link goes down
- Practice with IGP configuration
- Easier to expand later

### Configuration

**On Both Routers**:

```bash
# Enable OSPF
set protocols ospf parameters router-id '10.1.0.1'  # Use loopback IP
set protocols ospf area 0

# Advertise loopback
set protocols ospf area 0 network '10.1.0.1/32'

# Advertise internal link
set protocols ospf area 0 network '10.1.1.0/30'

# Set interface to point-to-point
set protocols ospf interface eth1 network 'point-to-point'
```

**Verification**:
```bash
show ip ospf neighbor
show ip route ospf
ping 10.1.0.2 source-address 10.1.0.1
```

**Expected**: One OSPF neighbor, loopback reachable

---

## Phase 3: Internal BGP (iBGP)

### Goal
Single iBGP session between the two routers.

### Why iBGP for 2 Routers?

**Alternative**: IGP redistribution
**Chosen**: iBGP because it's cleaner for BGP networks

### Configuration

**On Pasty-Edge-1**:

```bash
set protocols bgp system-as '65100'
set protocols bgp parameters router-id '10.1.0.1'

set protocols bgp neighbor 10.1.0.2 remote-as '65100'
set protocols bgp neighbor 10.1.0.2 description 'Pasty-Edge-2'
set protocols bgp neighbor 10.1.0.2 update-source 'lo'
set protocols bgp neighbor 10.1.0.2 address-family ipv4-unicast
```

**On Pasty-Edge-2**:

```bash
set protocols bgp system-as '65100'
set protocols bgp parameters router-id '10.1.0.2'

set protocols bgp neighbor 10.1.0.1 remote-as '65100'
set protocols bgp neighbor 10.1.0.1 description 'Pasty-Edge-1'
set protocols bgp neighbor 10.1.0.1 update-source 'lo'
set protocols bgp neighbor 10.1.0.1 address-family ipv4-unicast
```

**Verification**:
```bash
show bgp summary
```

**Expected**: One iBGP session `Established`

---

## Phase 4: External BGP - Transit Provider

### Goal
Configure transit from either Cogent OR Level 3 (pick one for simplicity).

### Design Decision: Single Transit Provider

**Why only one?**
- Simpler configuration
- Lower cost (in real world)
- Still functional for lab purposes

**Chosen**: Cogent (AS 174)

### External Connection

**On Pasty-Edge-1**:

| Connection | Interface | IP Address | Peer IP | Peer AS |
|------------|-----------|------------|---------|---------|
| Cogent Transit | eth2 | 100.64.1.1/30 | 100.64.1.2 | 174 |
| IXP Peering | eth3 | 198.51.100.59/24 | 198.51.100.251 | 64999 |

### Step 1: Configure Transit to Cogent

```bash
# Interface
set interfaces ethernet eth2 address '100.64.1.1/30'
set interfaces ethernet eth2 description 'to-Cogent-Transit'

# BGP session
set protocols bgp neighbor 100.64.1.2 remote-as '174'
set protocols bgp neighbor 100.64.1.2 description 'Cogent-Transit'
set protocols bgp neighbor 100.64.1.2 address-family ipv4-unicast
set protocols bgp neighbor 100.64.1.2 address-family ipv4-unicast route-map import 'FROM-TRANSIT'
set protocols bgp neighbor 100.64.1.2 address-family ipv4-unicast route-map export 'TO-TRANSIT'
```

### Step 2: Configure IXP Peering

```bash
# Interface
set interfaces ethernet eth3 address '198.51.100.59/24'
set interfaces ethernet eth3 description 'to-IXP'

# BGP session to route server
set protocols bgp neighbor 198.51.100.251 remote-as '64999'
set protocols bgp neighbor 198.51.100.251 description 'IXP-RS1'
set protocols bgp neighbor 198.51.100.251 address-family ipv4-unicast
set protocols bgp neighbor 198.51.100.251 address-family ipv4-unicast route-map import 'FROM-IXP'
set protocols bgp neighbor 198.51.100.251 address-family ipv4-unicast route-map export 'TO-IXP'
```

### Step 3: BGP Route Policies

**Import from Transit**:

```bash
set policy route-map FROM-TRANSIT rule 10 action 'permit'
set policy route-map FROM-TRANSIT rule 10 set local-preference '50'
```

**Import from IXP**:

```bash
set policy route-map FROM-IXP rule 10 action 'permit'
set policy route-map FROM-IXP rule 10 set local-preference '100'
```

**Why local-pref 100 > 50?**: Prefer free IXP peering over paid transit when possible

**Export to Transit** (announce customer routes only):

```bash
set policy community-list CUSTOMER-ROUTES rule 10 action 'permit'
set policy community-list CUSTOMER-ROUTES rule 10 regex '65100:100'

set policy route-map TO-TRANSIT rule 10 action 'permit'
set policy route-map TO-TRANSIT rule 10 match community 'CUSTOMER-ROUTES'

set policy route-map TO-TRANSIT rule 20 action 'deny'
```

**Export to IXP**:

```bash
set policy route-map TO-IXP rule 10 action 'permit'
set policy route-map TO-IXP rule 10 match community 'CUSTOMER-ROUTES'

set policy route-map TO-IXP rule 20 action 'deny'
```

**Why deny at end?**: Don't announce transit routes to IXP (would be providing free transit)

### Verification

```bash
show bgp summary
show ip route bgp
```

**Expected**:
- BGP to Cogent: Established, receiving default route
- BGP to IXP: Established

Test internet:
```bash
ping 8.8.8.8 source-address 10.1.0.1
```

---

## Phase 5: Customer Connection (Cloud Site)

### Goal
Connect cloud customer site for multi-site setup.

### Customer Connection Options

**Option 1: Static Routing** (Simple):
- Customer gets subnet
- Static routes point to customer

**Option 2: BGP** (Realistic):
- Customer gets AS number (AS 65002)
- BGP session for dynamic routing

### Using BGP (Recommended)

**On Pasty-Edge-2**:

```bash
# Interface to customer
set interfaces ethernet eth2 address '100.64.20.1/30'
set interfaces ethernet eth2 description 'to-Cloud-Customer'

# BGP to customer
set protocols bgp neighbor 100.64.20.2 remote-as '65002'
set protocols bgp neighbor 100.64.20.2 description 'Cloud-Customer-AS65002'
set protocols bgp neighbor 100.64.20.2 address-family ipv4-unicast
set protocols bgp neighbor 100.64.20.2 address-family ipv4-unicast route-map import 'FROM-CUSTOMER'
set protocols bgp neighbor 100.64.20.2 address-family ipv4-unicast route-map export 'TO-CUSTOMER'
```

**Customer route policy**:

```bash
# Accept customer routes and tag them
set policy route-map FROM-CUSTOMER rule 10 action 'permit'
set policy route-map FROM-CUSTOMER rule 10 set community '65100:100'
set policy route-map FROM-CUSTOMER rule 10 set local-preference '200'

# Announce default route to customer
set policy route-map TO-CUSTOMER rule 10 action 'permit'
set policy route-map TO-CUSTOMER rule 10 match ip address prefix-list 'DEFAULT-ONLY'

set policy prefix-list DEFAULT-ONLY rule 10 action 'permit'
set policy prefix-list DEFAULT-ONLY rule 10 prefix '0.0.0.0/0'
```

**Why announce default only?**: Customer doesn't need full BGP table, just default route

---

## Phase 6: Cloud Interconnect (WireGuard Tunnel)

### Goal
Establish secure tunnel from cloud (OCI/AWS) to PastyNet for realistic hybrid cloud.

### Why WireGuard?

- Lightweight VPN (vs IPsec complexity)
- Works in cloud environments
- Easy to configure
- Supports BGP over tunnel

### Step 1: Configure WireGuard Interface

**On Pasty-Edge-1**:

```bash
# Generate keys (do this locally)
# wg genkey | tee privatekey | wg pubkey > publickey

# Configure tunnel
set interfaces wireguard wg0 address '172.16.0.1/30'
set interfaces wireguard wg0 description 'Tunnel-to-OCI'
set interfaces wireguard wg0 port '51820'
set interfaces wireguard wg0 private-key '<private-key-here>'

# Peer configuration (OCI side)
set interfaces wireguard wg0 peer <OCI-public-key>
set interfaces wireguard wg0 peer <OCI-public-key> endpoint '<OCI-public-IP>:51820'
set interfaces wireguard wg0 peer <OCI-public-key> allowed-ips '0.0.0.0/0'
set interfaces wireguard wg0 peer <OCI-public-key> persistent-keepalive '25'
```

### Step 2: BGP Over Tunnel

```bash
# BGP to cloud gateway
set protocols bgp neighbor 172.16.0.2 remote-as '65002'
set protocols bgp neighbor 172.16.0.2 description 'Cloud-Gateway-OCI'
set protocols bgp neighbor 172.16.0.2 address-family ipv4-unicast
```

**Cloud side** (OCI VM running VyOS/FRRouting):
- Mirror WireGuard config
- BGP session to 172.16.0.1

**Traffic Flow**:
```
Cloud K8s Pod → OCI VM → WireGuard → PastyNet → Internet/On-prem
```

---

## Simplified Configuration Example

### Complete Pasty-Edge-1 Overview

```bash
# System
set system host-name Pasty-Edge-1

# Interfaces
set interfaces ethernet eth0 address '192.168.100.35/24'  # Mgmt
set interfaces ethernet eth1 address '10.1.1.1/30'        # to Pasty-Edge-2
set interfaces ethernet eth2 address '100.64.1.1/30'      # to Cogent
set interfaces ethernet eth3 address '198.51.100.59/24'   # to IXP
set interfaces loopback lo address '10.1.0.1/32'

# OSPF
set protocols ospf parameters router-id '10.1.0.1'
set protocols ospf area 0 network '10.1.0.1/32'
set protocols ospf area 0 network '10.1.1.0/30'

# BGP
set protocols bgp system-as '65100'
set protocols bgp parameters router-id '10.1.0.1'

# iBGP
set protocols bgp neighbor 10.1.0.2 remote-as '65100'
set protocols bgp neighbor 10.1.0.2 update-source 'lo'

# eBGP
set protocols bgp neighbor 100.64.1.2 remote-as '174'     # Cogent
set protocols bgp neighbor 198.51.100.251 remote-as '64999'  # IXP
```

---

## Common Issues and Troubleshooting

### Issue 1: iBGP Session Down

**Symptoms**: BGP not establishing between Pasty-Edge-1 and Edge-2

**Check**:
```bash
show bgp summary
show ip route 10.1.0.2
ping 10.1.0.2 source-address 10.1.0.1
```

**Common Causes**:
- OSPF not converged (loopbacks not reachable)
- Wrong update-source configuration
- Firewall blocking TCP 179

### Issue 2: No Internet via Transit

**Check**:
```bash
show bgp summary
show ip route 0.0.0.0/0
ping 8.8.8.8
```

**Common Causes**:
- Transit provider not announcing default route
- Import policy blocking routes
- Next-hop not reachable

### Issue 3: WireGuard Tunnel Not Connecting

**Check**:
```bash
show interfaces wireguard
ping 172.16.0.2
```

**Common Causes**:
- Incorrect public keys
- Firewall blocking UDP 51820
- Endpoint IP wrong or unreachable
- Cloud security group not allowing traffic

---

## Best Practices

1. **Keep it simple**: 2-router design doesn't need complexity
2. **Use iBGP even for 2 routers**: Cleaner than IGP redistribution
3. **Single transit provider**: Adequate for smaller ISP
4. **IXP peering**: Free traffic exchange with others
5. **Document cloud interconnect**: Critical for hybrid cloud setup

---

## Next Steps

1. Complete PastyNet build (both routers)
2. Verify transit from Cogent
3. Test IXP peering
4. Connect cloud customer (OCI/AWS)
5. Document in Netbox

---

## Configuration Comparison: 2-Router vs 4-Router ISP

| Feature | PastyNet (2 routers) | Atlas ISP (4 routers) |
|---------|----------------------|-----------------------|
| IGP | OSPF (simple) | OSPF (same) |
| MPLS | Not needed | Yes (for L3VPN) |
| iBGP | 1 session | 6 sessions (full mesh) |
| Route Reflectors | No | Optional |
| Transit Providers | 1 (Cogent) | 2 (Level3 + Cogent) |
| Complexity | Low | Medium |
| Resource Usage | ~2GB RAM | ~4-6GB RAM |

---

*This guide demonstrates a simplified regional ISP design suitable for smaller operations or branch offices. The concepts scale to larger networks.*

## VERSION HISTORY

- v1.0 (2026-02-09): Initial PastyNet ISP build guide for VyOS
