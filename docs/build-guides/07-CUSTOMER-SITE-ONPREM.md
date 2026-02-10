# On-Prem Customer Site Configuration Guide

## NOT REVIEWED YET

## Note

All IP addresses and interface names are generic in this guide. You should replace them with your own IP addresses and interface names. See IP_ADDRESSING_PHILOSOPHY.md for more information.

## Overview

This guide walks through configuring a typical on-premises customer site connected to Atlas Lab ISP. This represents a **branch office** or **datacenter** that purchases transit services from your regional ISP.

### Customer Profile: "AcmeCorp Detroit Branch"

```
Customer: AcmeCorp (AS 65001)
Service Type: Business Internet + BGP
ISP: Atlas Lab ISP (AS 65000)
Location: On-premises (Detroit branch)
Bandwidth: 1Gbps
Equipment: 1x Edge Router (VyOS or Cisco)
```

### Why This Matters

**Real-world scenario**: Small-to-medium businesses don't run their own AS numbers typically, but larger enterprises with multiple sites often do. This setup teaches:
- Customer perspective of ISP services
- BGP from the customer side
- How businesses multihome (multiple ISP connections)
- Basic enterprise edge routing

---

## Network Architecture

```
AcmeCorp Detroit Branch (AS 65001)
│
├── Customer-Edge-1 (Acme-Edge-1)
│   └── Services: Internet gateway, firewall integration point
│
└── Internal Network: 10.10.0.0/16

Upstream Connection:
Atlas-Edge-1 (AS 65000) ←→ Acme-Edge-1 (AS 65001)
  100.64.10.1/30              100.64.10.2/30
```

### Design Decisions

**Why BGP instead of static routing?**
- Enterprise wants to announce their public IP space
- Enables multihoming (can add second ISP later)
- Better control over routing policies
- More realistic for learning purposes

**Why single router?**
- Simplified customer site design
- Can expand to dual routers later
- Focuses on ISP connectivity, not internal redundancy

**Platform**: VyOS (lightweight) or Cisco CSR1000v (if you want IOS-XE practice)

---

## Phase 1: Basic Router Setup

### Goal
Configure management access and prepare router for ISP connection.

### Step 1: Initial Configuration

**On Acme-Edge-1** (VyOS):

```bash
set system host-name Acme-Edge-1

# Management interface (connects to your lab management network)
set interfaces ethernet eth0 address '192.168.100.50/24'
set protocols static route 0.0.0.0/0 next-hop 192.168.100.1

# Loopback for BGP router-ID
set interfaces loopback lo address '10.10.0.1/32'
```

**Why loopback?**: Stable router-ID for BGP, doesn't go down if physical interface fails

### Step 2: ISP-Facing Interface

```bash
# Connection to Atlas-Edge-1
set interfaces ethernet eth1 address '100.64.10.2/30'
set interfaces ethernet eth1 description 'to-Atlas-Lab-ISP'
```

**Verification**:
```bash
ping 100.64.10.1
show interfaces
```

**Expected**: Ping successful to ISP router (100.64.10.1)

---

## Phase 2: BGP Configuration - Customer Perspective

### Goal
Establish BGP session with Atlas Lab ISP and receive internet routing.

### Understanding Customer BGP

**What you receive from ISP**:
- Default route (0.0.0.0/0) - simplest option
- OR partial routes (ISP + peers)
- OR full BGP table (rare for small customers)

**What you announce to ISP**:
- Your public IP space
- Optionally, specific prefixes for traffic engineering

**For this lab**: You'll receive default route only (most common for small businesses)

### Step 1: Basic BGP Configuration

```bash
set protocols bgp system-as '65001'
set protocols bgp parameters router-id '10.10.0.1'

# BGP neighbor to Atlas-Edge-1
set protocols bgp neighbor 100.64.10.1 remote-as '65000'
set protocols bgp neighbor 100.64.10.1 description 'Atlas-Lab-ISP'
set protocols bgp neighbor 100.64.10.1 address-family ipv4-unicast
```

**Key Points**:
- `remote-as 65000`: Different AS = eBGP (external BGP)
- Uses physical interface IP, not loopback (unlike iBGP)

### Step 2: Announce Your Network

**Assuming AcmeCorp has public IP block**: `203.0.113.0/24` (example prefix)

```bash
# Advertise your network to the ISP
set protocols bgp address-family ipv4-unicast network '203.0.113.0/24'
```

**OR use a route-map for more control**:

```bash
# Create prefix list of what to announce
set policy prefix-list ANNOUNCE-TO-ISP rule 10 action 'permit'
set policy prefix-list ANNOUNCE-TO-ISP rule 10 prefix '203.0.113.0/24'

# Route map to export only your routes
set policy route-map TO-ISP rule 10 action 'permit'
set policy route-map TO-ISP rule 10 match ip address prefix-list 'ANNOUNCE-TO-ISP'

# Apply export policy
set protocols bgp neighbor 100.64.10.1 address-family ipv4-unicast route-map export 'TO-ISP'
```

**Why route-map?**: Prevents accidentally announcing learned routes back to ISP (could cause issues)

### Step 3: Accept Routes from ISP

**Simple approach** (accept default route only):

```bash
# No import policy needed - just accept what they send
# ISP will only send you 0.0.0.0/0 anyway
```

**More restrictive approach**:

```bash
# Only accept default route
set policy prefix-list DEFAULT-ONLY rule 10 action 'permit'
set policy prefix-list DEFAULT-ONLY rule 10 prefix '0.0.0.0/0'

set policy route-map FROM-ISP rule 10 action 'permit'
set policy route-map FROM-ISP rule 10 match ip address prefix-list 'DEFAULT-ONLY'

set policy route-map FROM-ISP rule 20 action 'deny'

set protocols bgp neighbor 100.64.10.1 address-family ipv4-unicast route-map import 'FROM-ISP'
```

**Why restrictive?**: Prevents ISP from accidentally sending you full BGP table (which could overwhelm your router)

### Verification

```bash
show bgp summary
show ip route bgp
ping 8.8.8.8 source-address 10.10.0.1
```

**Expected**:
- BGP session: `Established`
- Route table: Default route via 100.64.10.1
- Ping to 8.8.8.8: Successful (internet connectivity!)

---

## Phase 3: Internal Network Integration

### Goal
Connect customer's internal network and provide internet access.

### Step 1: Internal Interface Configuration

```bash
# Internal network facing interface
set interfaces ethernet eth2 address '10.10.1.1/24'
set interfaces ethernet eth2 description 'Internal-LAN'
```

**Network layout**:
- `10.10.1.0/24`: User workstations
- `10.10.2.0/24`: Servers
- `10.10.3.0/24`: Voice/IoT devices

### Step 2: NAT Configuration (if using private IPs internally)

**Scenario**: Internal network uses RFC1918 (10.10.0.0/16), but internet requires public IPs

```bash
# Source NAT (hide internal IPs behind public IP)
set nat source rule 100 outbound-interface 'eth1'
set nat source rule 100 source address '10.10.0.0/16'
set nat source rule 100 translation address '203.0.113.1'
```

**What this does**: All traffic from 10.10.0.0/16 appears to come from 203.0.113.1 when going to internet

**Why NAT?**: You don't have enough public IPs for every device (most businesses don't)

### Step 3: Internal Routing

**Option 1: Static routes** (simple, good for small sites):

```bash
# Announce internal subnets via BGP if you want ISP to route to you
set protocols bgp address-family ipv4-unicast network '10.10.1.0/24'
set protocols bgp address-family ipv4-unicast network '10.10.2.0/24'
```

**Wait, announce private IPs?**: No! Only if you have public IPs. With NAT, you don't announce internal networks.

**Option 2: OSPF** (for larger internal networks):

```bash
# Enable OSPF for internal routing
set protocols ospf parameters router-id '10.10.0.1'
set protocols ospf area 0 network '10.10.1.0/24'
set protocols ospf area 0 network '10.10.2.0/24'

# Redistribute default route from BGP into OSPF
set protocols ospf default-information originate
```

**Why OSPF?**: If you have multiple internal routers, OSPF distributes routes automatically

---

## Phase 4: Firewall and Security

### Goal
Protect customer network from internet threats.

### Understanding VyOS Firewall

**Zone-based firewall**:
- OUTSIDE zone: ISP connection (untrusted)
- INSIDE zone: Internal network (trusted)
- LOCAL zone: Router itself

### Step 1: Define Zones

```bash
# Define zones
set zone-policy zone OUTSIDE interface 'eth1'
set zone-policy zone INSIDE interface 'eth2'
set zone-policy zone LOCAL local-zone
```

### Step 2: Firewall Rules

**Allow internal → internet**:

```bash
# Allow established/related traffic back in
set firewall name OUTSIDE-TO-INSIDE rule 10 action 'accept'
set firewall name OUTSIDE-TO-INSIDE rule 10 state established 'enable'
set firewall name OUTSIDE-TO-INSIDE rule 10 state related 'enable'

# Drop everything else from internet
set firewall name OUTSIDE-TO-INSIDE rule 999 action 'drop'

# Apply to zone
set zone-policy zone INSIDE from OUTSIDE firewall name 'OUTSIDE-TO-INSIDE'
```

**Allow internet → internal** (for specific services):

```bash
# Example: Allow HTTPS to internal web server (10.10.2.10)
set nat destination rule 10 inbound-interface 'eth1'
set nat destination rule 10 destination port '443'
set nat destination rule 10 protocol 'tcp'
set nat destination rule 10 translation address '10.10.2.10'

# Firewall rule to allow it
set firewall name OUTSIDE-TO-INSIDE rule 20 action 'accept'
set firewall name OUTSIDE-TO-INSIDE rule 20 destination address '10.10.2.10'
set firewall name OUTSIDE-TO-INSIDE rule 20 destination port '443'
set firewall name OUTSIDE-TO-INSIDE rule 20 protocol 'tcp'
```

**Protect router itself**:

```bash
# Allow BGP from ISP
set firewall name OUTSIDE-TO-LOCAL rule 10 action 'accept'
set firewall name OUTSIDE-TO-LOCAL rule 10 source address '100.64.10.1'
set firewall name OUTSIDE-TO-LOCAL rule 10 destination port '179'
set firewall name OUTSIDE-TO-LOCAL rule 10 protocol 'tcp'

# Allow established/related
set firewall name OUTSIDE-TO-LOCAL rule 20 action 'accept'
set firewall name OUTSIDE-TO-LOCAL rule 20 state established 'enable'

# Drop everything else
set firewall name OUTSIDE-TO-LOCAL rule 999 action 'drop'

set zone-policy zone LOCAL from OUTSIDE firewall name 'OUTSIDE-TO-LOCAL'
```

---

## Phase 5: Monitoring and Troubleshooting

### Goal
Verify everything works and know how to troubleshoot issues.

### Key Verification Commands

**BGP status**:
```bash
show bgp summary                    # Session status
show bgp neighbors 100.64.10.1     # Detailed neighbor info
show ip route bgp                   # Routes learned via BGP
```

**Connectivity tests**:
```bash
ping 8.8.8.8 source-address 10.10.0.1     # Internet from router
ping 8.8.8.8 source-address 10.10.1.10    # Internet from internal
traceroute 1.1.1.1                         # Path to internet
```

**NAT verification**:
```bash
show nat source translations
show nat source statistics
```

**Firewall logs**:
```bash
show firewall name OUTSIDE-TO-INSIDE
show log firewall
```

### Common Issues

**Issue 1: BGP Not Establishing**

**Check**:
```bash
show bgp summary
show log | match BGP
```

**Common causes**:
- Wrong AS number
- IP addressing mismatch (can't ping ISP router)
- Firewall blocking TCP 179
- ISP hasn't configured their side yet

**Issue 2: Can Ping Router But Not Internet**

**Check**:
```bash
show ip route
show nat source translations
```

**Common causes**:
- No default route (BGP not established)
- NAT not configured
- Firewall blocking outbound traffic

**Issue 3: Internet Works from Router But Not Clients**

**Check**:
```bash
ping 8.8.8.8 source-address 10.10.1.10
show firewall name INSIDE-TO-OUTSIDE
```

**Common causes**:
- Clients not using router as default gateway
- Internal interface not configured
- Zone policy blocking traffic

---

## Advanced Topics (Optional)

### Multihoming (Two ISPs)

**Scenario**: Add second ISP for redundancy

```bash
# Second ISP connection (example: to PastyNet)
set interfaces ethernet eth3 address '100.64.20.2/30'
set interfaces ethernet eth3 description 'to-PastyNet-ISP'

# BGP to second ISP
set protocols bgp neighbor 100.64.20.1 remote-as '65100'
set protocols bgp neighbor 100.64.20.1 description 'PastyNet-ISP'
set protocols bgp neighbor 100.64.20.1 address-family ipv4-unicast

# Prefer Atlas as primary using local-pref
set policy route-map FROM-ATLAS rule 10 action 'permit'
set policy route-map FROM-ATLAS rule 10 set local-preference '200'

set policy route-map FROM-PASTYNET rule 10 action 'permit'
set policy route-map FROM-PASTYNET rule 10 set local-preference '100'

set protocols bgp neighbor 100.64.10.1 address-family ipv4-unicast route-map import 'FROM-ATLAS'
set protocols bgp neighbor 100.64.20.1 address-family ipv4-unicast route-map import 'FROM-PASTYNET'
```

**Result**: Atlas is primary, PastyNet is backup

### Traffic Engineering

**Scenario**: Send specific traffic via specific ISP

```bash
# Send traffic to 8.8.8.0/24 via PastyNet
set policy route POLICY-ROUTE rule 10 destination address '8.8.8.0/24'
set policy route POLICY-ROUTE rule 10 set table '100'

set protocols static table 100 route 0.0.0.0/0 next-hop 100.64.20.1

set interfaces ethernet eth2 policy route 'POLICY-ROUTE'
```

---

## Configuration Summary

### Complete Acme-Edge-1 Overview

```bash
# System
set system host-name Acme-Edge-1

# Interfaces
set interfaces ethernet eth0 address '192.168.100.50/24'   # Management
set interfaces ethernet eth1 address '100.64.10.2/30'       # to Atlas ISP
set interfaces ethernet eth2 address '10.10.1.1/24'         # Internal LAN
set interfaces loopback lo address '10.10.0.1/32'

# BGP
set protocols bgp system-as '65001'
set protocols bgp parameters router-id '10.10.0.1'
set protocols bgp neighbor 100.64.10.1 remote-as '65000'
set protocols bgp neighbor 100.64.10.1 description 'Atlas-Lab-ISP'
set protocols bgp address-family ipv4-unicast network '203.0.113.0/24'

# NAT
set nat source rule 100 outbound-interface 'eth1'
set nat source rule 100 source address '10.10.0.0/16'
set nat source rule 100 translation address '203.0.113.1'

# Zones
set zone-policy zone OUTSIDE interface 'eth1'
set zone-policy zone INSIDE interface 'eth2'
```

---

## Best Practices

1. **Always use route-maps**: Don't just accept/announce everything
2. **Implement firewall zones**: Protect your network from threats
3. **Monitor BGP sessions**: Set up alerts for session down
4. **Document ISP contacts**: Know who to call when things break
5. **Test failover**: If you have dual ISPs, test what happens when one fails
6. **Keep configs backed up**: Especially before making changes

---

## Next Steps

1. Configure basic customer router
2. Establish BGP with Atlas ISP
3. Test internet connectivity
4. Add internal network integration
5. Implement firewall policies
6. (Optional) Add second ISP for multihoming

---

## Integration with Project Atlas

**This customer site connects to**:
- Atlas-Edge-1 (AS 65000) via eth4
- Receives transit service (default route)
- Announces 203.0.113.0/24

**In your lab topology**:
```
Atlas-Edge-1 ← BGP → Acme-Edge-1 ← Internal Network (K8s, VMs, etc.)
```

**Traffic flow**:
```
Acme workstation → Acme-Edge-1 (NAT) → Atlas-Edge-1 → Atlas-Core → Transit ISP → Internet
```

---

*This guide demonstrates enterprise customer edge routing from the customer's perspective. It's the "other side" of the ISP configurations, showing how businesses consume ISP services.*

## VERSION HISTORY

- v1.0 (2026-02-09): Initial on-prem customer site configuration guide
