# Network Configuration Learning Guide

This directory contains router and switch configurations for your ISP simulation.

# This file has not yet been reviewed by a human!

## 🎓 What You're Building

A simulated Tier-2 ISP with:
- **Core network**: MPLS backbone with ISIS + LDP
- **Edge routers**: BGP session terminationwith customers
- **Customer edge**: Your lab's gateway to the "ISP"

## 🌐 Network Topology Overview

```
┌─────────────────────────────────────────────┐
│         Tier-1 ISPs (Optional)              │
│   AS174 (Cogent)     AS3356 (Level3)        │
└─────────────┬───────────────┬───────────────┘
              │               │
         ┌────┴───────────────┴─────┐
         │    MPLS Core (AS65000)   │
         │  Core-CHI     Core-NYC   │
         │  ISIS + LDP + MP-BGP     │
         └────┬───────────────┬─────┘
              │               │
    ┌─────────┴───┐     ┌────┴─────────┐
    │ BGW-CHI-1   │     │  BGW-NYC-1   │
    │ BGW-CHI-2   │     │  BGW-NYC-2   │
    └──────┬──────┘     └──────┬───────┘
           │                   │
           └──────┬────────────┘
                  │
           ┌──────┴───────┐
           │   Lab-CE     │
           │   (VyOS)     │
           └──────────────┘
                  │
           Your Homelab Infrastructure
```

## 🔑 Key Protocols to Learn

### 1. ISIS (Interior Gateway Protocol)
**Purpose**: Distribute routes within the ISP core

**Why ISIS over OSPF?**
- More scalable for large networks
- Used by most large ISPs
- Protocol-independent (not tied to IP)
- Better for MPLS

**What it does**:
- Each router learns about all other routers
- Builds shortest-path tree
- Distributes loopback addresses
- Foundation for MPLS

**Key Concepts**:
- NET address: Network Entity Title (router ID)
- Level 1: within an area
- Level 2: between areas (we'll use Level 2 only)
- Metric: cost of a link

### 2. MPLS with LDP (Label Distribution Protocol)
**Purpose**: Create label-switched paths through the network

**Why MPLS?**
- Faster forwarding than IP routing
- Enables traffic engineering
- Foundation for VPNs
- What real ISPs use

**What it does**:
- Assigns labels to routes
- Routers switch packets based on labels
- Last router (egress) removes label
- Middle routers don't look at IP headers

**Key Concepts**:
- Label: a number (16-99999)
- LDP: protocol to distribute labels
- PHP (Penultimate Hop Popping): remove label before last hop
- Label stack: multiple labels for VPNs

### 3. MP-BGP (Multi-Protocol BGP)
**Purpose**: Exchange VPN routes between edge routers

**Why MP-BGP?**
- Regular BGP can't carry VPN info
- Needs to distinguish between overlapping IP addresses
- Carries both route and VPN identifier

**Key Concepts**:
- VPNv4 address family: special BGP family for VPNs
- Route Distinguisher (RD): makes routes unique
- Route Target (RT): controls import/export
- Next-hop: usually the PE router's loopback

### 4. EBGP (External BGP)
**Purpose**: Connect customer to ISP

**Customer side**:
- Advertises local networks
- Receives default route from ISP
- May have multiple ISP connections (multihoming)

**ISP side**:
- Receives customer routes
- Puts them into VRF
- Redistributes into MP-BGP

## 📝 Configuration Approach

### Strategy: Build Bottom-Up

**1. Physical connectivity first**
```
Core-CHI ←→ Core-NYC
Core-CHI ←→ BGW-CHI-1
Core-CHI ←→ BGW-CHI-2
Core-NYC ←→ BGW-NYC-1
Core-NYC ←→ BGW-NYC-2
```

**2. Then add IGP (ISIS)**
- Configure ISIS on all core and edge routers
- Verify neighbors come up
- Verify loopbacks are reachable

**3. Then add MPLS (LDP)**
- Enable MPLS on all interfaces
- Enable LDP
- Verify label distribution
- Verify LSPs (Label Switched Paths)

**4. Then add MP-BGP**
- Configure BGP between core routers
- Configure VPNv4 address family
- Configure route reflectors (optional)

**5. Finally add customer BGP**
- Create VRF on edge routers
- Configure EBGP with customer
- Test end-to-end connectivity

## 🛠️ Configuration Examples

### Cisco IOS-XR (Core Router)

**File**: `eve-ng-isp/core-chi-iosxr.conf`

```cisco
!! Core-CHI (Cisco IOS-XRv)
!! AS65000
!! Loopback: 10.0.0.1

hostname Core-CHI
domain name lab.local

!! Loopback for router ID
interface Loopback0
 ipv4 address 10.0.0.1 255.255.255.255
 description Router ID and MPLS loopback
!

!! Connection to Core-NYC
interface GigabitEthernet0/0/0/0
 description Link to Core-NYC
 ipv4 address 10.0.1.0 255.255.255.252
 no shutdown
!

!! Connection to BGW-CHI-1
interface GigabitEthernet0/0/0/1
 description Link to BGW-CHI-1
 ipv4 address 10.0.1.4 255.255.255.252
 no shutdown
!

!! Connection to BGW-CHI-2
interface GigabitEthernet0/0/0/2
 description Link to BGW-CHI-2
 ipv4 address 10.0.1.8 255.255.255.252
 no shutdown
!

!! ISIS Configuration
router isis CORE
 is-type level-2-only
 net 49.0001.0100.0000.0001.00
 address-family ipv4 unicast
  metric-style wide
  mpls traffic-eng level-2-only
  mpls traffic-eng router-id Loopback0
 !
 interface Loopback0
  passive
  address-family ipv4 unicast
  !
 !
 interface GigabitEthernet0/0/0/0
  point-to-point
  address-family ipv4 unicast
   metric 10
  !
 !
 ! Repeat for other interfaces
!

!! MPLS and LDP Configuration
mpls ldp
 router-id 10.0.0.1
 interface GigabitEthernet0/0/0/0
 !
 interface GigabitEthernet0/0/0/1
 !
 interface GigabitEthernet0/0/0/2
 !
!

!! MP-BGP Configuration
router bgp 65000
 bgp router-id 10.0.0.1
 address-family vpnv4 unicast
 !
 neighbor 10.0.0.2
  remote-as 65000
  update-source Loopback0
  address-family vpnv4 unicast
  !
 !
!
```

### Cisco CSR1000v (Edge Router)

**File**: `eve-ng-isp/edge-chi-1-csr.conf`

```cisco
! BGW-CHI-1 (Cisco CSR1000v)
! AS65000
! Loopback: 10.0.0.11

hostname BGW-CHI-1

! Loopback
interface Loopback0
 ip address 10.0.0.11 255.255.255.255
 description Router ID

! Uplink to Core-CHI
interface GigabitEthernet1
 description Link to Core-CHI
 ip address 10.0.1.5 255.255.255.252
 no shutdown

! Customer-facing interface
interface GigabitEthernet2
 description Customer Lab-CE
 ip address 100.64.10.1 255.255.255.252
 no shutdown

! ISIS Configuration
router isis CORE
 net 49.0001.0100.0000.0011.00
 is-type level-2-only
 metric-style wide

interface Loopback0
 ip router isis CORE
 isis passive

interface GigabitEthernet1
 ip router isis CORE
 isis network point-to-point
 isis metric 10

! MPLS Configuration
mpls ip
mpls label protocol ldp
mpls ldp router-id Loopback0 force

interface GigabitEthernet1
 mpls ip

! VRF for customer
ip vrf CUSTOMER_LAB
 rd 65000:1
 route-target export 65000:1
 route-target import 65000:1

! Put customer interface in VRF
interface GigabitEthernet2
 ip vrf forwarding CUSTOMER_LAB
 ip address 100.64.10.1 255.255.255.252

! BGP with customer
router bgp 65000
 neighbor 10.0.0.1 remote-as 65000
 neighbor 10.0.0.1 update-source Loopback0
 !
 address-family vpnv4
  neighbor 10.0.0.1 activate
  neighbor 10.0.0.1 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf CUSTOMER_LAB
  neighbor 100.64.10.2 remote-as 65001
  neighbor 100.64.10.2 activate
 exit-address-family
```

### VyOS (Customer Edge)

**File**: `eve-ng-isp/customer-edge-vyos.conf`

```bash
# Lab-CE (VyOS)
# AS65001
# This is the customer side

# Configure system
set system host-name Lab-CE
set system domain-name lab.local

# Loopback
set interfaces loopback lo address 192.168.100.1/32

# Primary uplink to BGW-CHI-1
set interfaces ethernet eth0 address 100.64.10.2/30
set interfaces ethernet eth0 description 'Primary uplink to AS65000'

# Secondary uplink to BGW-NYC-1 (for redundancy)
set interfaces ethernet eth1 address 100.64.20.2/30
set interfaces ethernet eth1 description 'Backup uplink to AS65000'

# Inside interface to homelab
set interfaces ethernet eth2 address 192.168.100.254/24
set interfaces ethernet eth2 description 'Inside network to homelab'

# BGP Configuration
set protocols bgp local-as 65001
set protocols bgp router-id 192.168.100.1

# Neighbor: Primary ISP connection
set protocols bgp neighbor 100.64.10.1 remote-as 65000
set protocols bgp neighbor 100.64.10.1 description 'BGW-CHI-1'
set protocols bgp neighbor 100.64.10.1 address-family ipv4-unicast

# Neighbor: Secondary ISP connection
set protocols bgp neighbor 100.64.20.1 remote-as 65000
set protocols bgp neighbor 100.64.20.1 description 'BGW-NYC-1'
set protocols bgp neighbor 100.64.20.1 address-family ipv4-unicast

# Advertise our networks
set protocols bgp address-family ipv4-unicast network 192.168.100.0/24

# Default route preference (primary vs backup)
# Lower local-preference = less preferred
set protocols bgp neighbor 100.64.20.1 address-family ipv4-unicast route-map import BACKUP
set policy route-map BACKUP rule 10 action permit
set policy route-map BACKUP rule 10 set local-preference 50

# NAT for homelab (if needed)
set nat source rule 100 outbound-interface eth0
set nat source rule 100 source address 192.168.100.0/24
set nat source rule 100 translation address masquerade
```

## 🧪 Verification Commands

### Check ISIS
```bash
# IOS-XR
show isis neighbors
show isis database
show isis route

# IOS/CSR
show isis neighbors
show clns neighbors

# VyOS
show isis neighbor
```

### Check MPLS/LDP
```bash
# IOS-XR
show mpls ldp neighbor
show mpls forwarding
show mpls label table

# IOS/CSR
show mpls ldp neighbor
show mpls forwarding-table

# VyOS (if using MPLS)
show mpls ldp neighbor
```

### Check BGP
```bash
# IOS-XR
show bgp vpnv4 unicast summary
show bgp vpnv4 unicast vrf CUSTOMER_LAB

# IOS/CSR
show bgp vpnv4 unicast all summary
show ip bgp vpnv4 vrf CUSTOMER_LAB

# VyOS
show bgp summary
show bgp neighbors
show ip route bgp
```

### Test Connectivity
```bash
# From customer edge, ping homelab resources through ISP
ping 192.168.100.10 source-address 192.168.100.1

# Traceroute to see MPLS path
traceroute 192.168.100.10

# Check routing table
show ip route
```

## 📚 Learning Resources

### Books
- **MPLS Fundamentals** by Luc De Ghein
- **BGP** by Iljitsch van Beijnum
- **ISIS Network Design Solutions** by Abe Martey

### Online
- Cisco IOS-XR Configuration Guides
- VyOS Documentation: https://docs.vyos.io/
- "MPLS L3VPN Tutorial" - various YouTube channels

### Practice
- Build topology step by step
- Configure one protocol at a time
- Verify before moving to next protocol
- Break things and troubleshoot!

## 🎯 Your Configuration Journey

### Week 1: Physical + IGP
- [ ] Create topology in EVE-NG
- [ ] Configure IP addresses on all interfaces
- [ ] Configure ISIS on all routers
- [ ] Verify ISIS neighbors
- [ ] Verify loopback reachability

### Week 2: MPLS
- [ ] Enable MPLS on all core/edge interfaces
- [ ] Configure LDP
- [ ] Verify LDP neighbors
- [ ] Verify label distribution
- [ ] Test MPLS path with traceroute

### Week 3: BGP
- [ ] Configure MP-BGP between core routers
- [ ] Create VRFs on edge routers
- [ ] Configure customer BGP sessions
- [ ] Verify VPN routing

### Week 4: Testing & Optimization
- [ ] End-to-end connectivity tests
- [ ] Failure scenarios (link down, router down)
- [ ] Traffic engineering experiments
- [ ] Performance testing

## 💡 Pro Tips

**1. Use Consistent Naming**
```
Interfaces: GigabitEthernet0/0/0/0 = Gi0/0/0/0
Descriptions: Always describe interface purpose
Loopbacks: Use for router IDs
```

**2. Document as You Go**
```
!! Added 2026-02-07: ISIS configuration
!! Reason: Core IGP
```

**3. Save Configs After Each Change**
```
# IOS-XR
commit
write memory

# IOS/CSR
write memory

# VyOS
commit
save
```

**4. Test Before Moving On**
- Don't configure everything at once
- Verify each step works
- If broken, fix before continuing

**5. Use Logical IP Addressing**
```
Core loopbacks: 10.0.0.1, 10.0.0.2
Edge loopbacks: 10.0.0.11, 10.0.0.12, 10.0.0.21, 10.0.0.22
Core links: 10.0.1.0/30, 10.0.1.4/30, etc.
Customer facing: 100.64.x.x/30
```

---

## 🚀 Ready to Start?

1. **Design first** - Draw the topology on paper
2. **IP address plan** - Create a spreadsheet
3. **Build in EVE-NG** - Add routers and links
4. **Configure step by step** - One protocol at a time
5. **Document everything** - Save configs as you go
6. **Test thoroughly** - Verify each step

**Remember**: Real network engineers configure routers like this every day. By doing this project, you're gaining genuine, marketable skills!

Good luck! 🌐
