# Anchor Tenant Networks - Implementation Guide

## NOT REVIEWED YET

## Note

All IP addresses and interface names are generic in this guide. You should replace them with your own IP addresses and interface names. See IP_ADDRESSING_PHILOSOPHY.md for more information.

## Overview

This guide provides practical implementation steps for deploying **anchor tenant networks** - the "big players" that make your ISP lab feel realistic. These are simplified representations of CDNs, hyperscalers, content providers, and eyeball ISPs that peer with your network.

### What Are Anchor Tenants?

**Real-world context**: Major internet players like Cloudflare, Google, Meta, Netflix don't just buy transit - they peer extensively at IXPs and sometimes directly with ISPs. Adding these to your lab makes BGP routing more realistic.

**In your lab**: Anchor tenants are lightweight VMs (1-2GB RAM each) running FRRouting or VyOS that:
- Announce realistic prefix ranges
- Peer at the IXP
- Simulate different traffic patterns
- Make your BGP table look like a real ISP

---

## Quick Reference: 4 Anchor Tenant Types

| Type | Example | AS Number | Prefixes | Where They Peer |
|------|---------|-----------|----------|-----------------|
| **CDN** | Cloudflare-like | 65510 | ~10 (/24s) | IXP + Direct to both Tier-1s |
| **Hyperscaler** | AWS-like | 65520 | ~20 (/16-/24s) | IXP only |
| **Content** | Netflix-like | 65530 | ~5 (/24s) | IXP + Direct to Atlas ISP |
| **Eyeball ISP** | Comcast-like | 65540 | ~50 (/16-/20s) | IXP only |

**Total resources**: ~6GB RAM, 4 VMs

---

## Deployment Architecture

```
Project Atlas Topology with Anchor Tenants:

                    ┌─────────────────┐
                    │   ExaBGP Route  │
                    │    Injector     │ (Phase 2: adds 50k routes)
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
    ┌───▼────┐          ┌───▼────┐          ┌───▼────┐
    │ Level3 │          │Cogent │          │  IXP   │
    │AS 3356 │◄────────►│AS 174  │          │ AS64999│
    └───┬────┘          └───┬────┘          └───┬────┘
        │                    │                    │
        │  ┌─────────────────┴──────────┬─────────┤
        │  │                            │         │
        │  │ ┌──────────────────────────┘         │
        │  │ │  ┌───────────────────────────────┐ │
        │  │ │  │                               │ │
    ┌───▼──▼─▼──▼───┐              ┌────────────▼─▼────┐
    │   Atlas ISP   │              │   PastyNet ISP    │
    │   AS 65000    │              │    AS 65100       │
    └───────────────┘              └───────────────────┘

    Anchor Tenants (peer at IXP and/or directly):
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │ CDN          │  │ Hyperscaler  │  │ Content      │  │ Eyeball ISP  │
    │ AS 65510     │  │ AS 65520     │  │ AS 65530     │  │ AS 65540     │
    │ (Cloudflare) │  │ (AWS)        │  │ (Netflix)    │  │ (Comcast)    │
    └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘
         ▲▲▲               ▲                 ▲▲                 ▲
         │││               │                 ││                 │
         ││└───────IXP─────┘                 ││                 │
         ││                                  ││                 │
         │└──Level3──────────────────────────┘│                 │
         └───Cogent────────Atlas ISP──────────┘────────IXP──────┘
```

---

## Phase 1: CDN Anchor Tenant (Cloudflare-like)

### Goal
Deploy a CDN network that peers aggressively everywhere.

### Why This Matters

**Real CDNs** (Cloudflare, Akamai, Fastly):
- Peer at 200+ IXPs worldwide
- Often offer settlement-free peering
- Announce same prefixes everywhere (anycast)
- Handle 10-20% of all internet traffic

**What you'll learn**:
- Anycast concepts
- Aggressive peering strategies
- Multi-location route announcement

### Step 1: Deploy CDN VM

**VM Specs**:
- Platform: FRRouting on Ubuntu (lightweight)
- RAM: 1GB
- vCPU: 1 core
- Storage: 10GB
- Network: Connects to IXP + both Tier-1s

**Install FRRouting**:

```bash
# On Ubuntu 22.04
curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -
echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) frr-stable | sudo tee -a /etc/apt/sources.list.d/frr.list
sudo apt update && sudo apt install frr frr-pythontools -y

# Enable BGP daemon
sudo sed -i 's/bgpd=no/bgpd=yes/' /etc/frr/daemons
sudo systemctl restart frr
```

### Step 2: Configure Interfaces

```bash
# /etc/network/interfaces (or Netplan on Ubuntu)

auto eth0
iface eth0 inet static
  address 192.168.100.60/24
  gateway 192.168.100.1

# IXP peering interface
auto eth1
iface eth1 inet static
  address 198.51.100.60/24

# Direct peer to Level3
auto eth2
iface eth2 inet static
  address 203.0.113.33/30

# Direct peer to Cogent
auto eth3
iface eth3 inet static
  address 203.0.113.37/30

# Loopback for router-ID
auto lo:1
iface lo:1 inet static
  address 10.50.0.1/32
```

### Step 3: FRRouting BGP Configuration

**Edit `/etc/frr/frr.conf`**:

```
hostname CDN-Anchor
log file /var/log/frr/frr.log
!
router bgp 65510
 bgp router-id 10.50.0.1
 no bgp default ipv4-unicast

 ! IXP peering
 neighbor 198.51.100.251 remote-as 64999
 neighbor 198.51.100.251 description IXP-RS1

 ! Level3 direct peering
 neighbor 203.0.113.34 remote-as 3356
 neighbor 203.0.113.34 description Level3-Direct

 ! Cogent direct peering
 neighbor 203.0.113.38 remote-as 174
 neighbor 203.0.113.38 description Cogent-Direct

 ! Enable IPv4 unicast
 address-family ipv4 unicast
  neighbor 198.51.100.251 activate
  neighbor 203.0.113.34 activate
  neighbor 203.0.113.38 activate

  ! Announce CDN prefixes (anycast ranges)
  network 185.10.1.0/24
  network 185.10.2.0/24
  network 185.10.3.0/24
  network 185.10.4.0/24
  network 185.10.5.0/24

  ! Route policy (announce same prefixes to everyone)
  neighbor 198.51.100.251 route-map EXPORT-CDN out
  neighbor 203.0.113.34 route-map EXPORT-CDN out
  neighbor 203.0.113.38 route-map EXPORT-CDN out
 exit-address-family
!
! Create static routes for announced prefixes (null route to prevent loops)
ip route 185.10.1.0/24 Null0
ip route 185.10.2.0/24 Null0
ip route 185.10.3.0/24 Null0
ip route 185.10.4.0/24 Null0
ip route 185.10.5.0/24 Null0
!
! Route maps
route-map EXPORT-CDN permit 10
 match ip address prefix-list CDN-PREFIXES
!
ip prefix-list CDN-PREFIXES seq 10 permit 185.10.0.0/16 le 24
!
line vty
!
```

**Why null routes?**: Prevents routing loops if someone sends traffic to these IPs

### Step 4: Apply Configuration

```bash
sudo systemctl restart frr
sudo vtysh -c "show bgp summary"
sudo vtysh -c "show ip route bgp"
```

**Expected**: BGP sessions established to IXP route server and both Tier-1s

---

## Phase 2: Hyperscaler Anchor Tenant (AWS-like)

### Goal
Deploy a large cloud provider that only peers at IXP (no direct peering).

### Why This Matters

**Real hyperscalers** (AWS, Azure, GCP):
- Massive AS numbers with thousands of prefixes
- Selective peering (IXP only, no direct peering with small ISPs)
- Announce large blocks (but broken into /24s for traffic engineering)

### Step 1: Deploy Hyperscaler VM

**Same process as CDN**, but:
- Only needs 1 network interface to IXP
- Announces more prefixes (20-30)
- No direct peering connections

### Step 2: FRRouting Configuration

```
hostname Hyperscaler-Anchor
!
router bgp 65520
 bgp router-id 10.51.0.1
 no bgp default ipv4-unicast

 ! IXP peering only
 neighbor 198.51.100.251 remote-as 64999
 neighbor 198.51.100.251 description IXP-RS1

 address-family ipv4 unicast
  neighbor 198.51.100.251 activate

  ! Announce hyperscaler prefixes (simulating AWS ranges)
  network 52.10.0.0/16
  network 52.11.0.0/16
  network 52.12.0.0/16
  network 34.200.0.0/16
  network 34.201.0.0/16

  ! Also announce some /24s for TE
  network 52.10.1.0/24
  network 52.10.2.0/24
  network 52.11.1.0/24

  neighbor 198.51.100.251 route-map EXPORT-HYPERSCALER out
 exit-address-family
!
! Null routes
ip route 52.10.0.0/16 Null0
ip route 52.11.0.0/16 Null0
ip route 52.12.0.0/16 Null0
ip route 34.200.0.0/16 Null0
ip route 34.201.0.0/16 Null0
!
route-map EXPORT-HYPERSCALER permit 10
!
```

**Key difference from CDN**: Only peers at IXP, no direct connections

---

## Phase 3: Content Provider (Netflix-like)

### Goal
Deploy a content delivery network that peers selectively.

### Why This Matters

**Real content providers** (Netflix, YouTube, Twitch):
- Peer at IXPs
- Sometimes direct peering with large ISPs
- Announce relatively few prefixes
- High bandwidth consumption

### FRRouting Configuration

```
hostname Content-Anchor
!
router bgp 65530
 bgp router-id 10.52.0.1
 no bgp default ipv4-unicast

 ! IXP peering
 neighbor 198.51.100.251 remote-as 64999
 neighbor 198.51.100.251 description IXP-RS1

 ! Direct peering with Atlas ISP (common for content)
 neighbor 203.0.113.41 remote-as 65000
 neighbor 203.0.113.41 description Atlas-ISP-Direct

 address-family ipv4 unicast
  neighbor 198.51.100.251 activate
  neighbor 203.0.113.41 activate

  ! Content provider prefixes
  network 199.50.1.0/24
  network 199.50.2.0/24
  network 199.50.3.0/24

  neighbor 198.51.100.251 route-map EXPORT-CONTENT out
  neighbor 203.0.113.41 route-map EXPORT-CONTENT out

  ! Higher local-pref for direct peering (prefer over IXP)
  neighbor 203.0.113.41 route-map PREFER-DIRECT in
 exit-address-family
!
ip route 199.50.1.0/24 Null0
ip route 199.50.2.0/24 Null0
ip route 199.50.3.0/24 Null0
!
route-map EXPORT-CONTENT permit 10
!
route-map PREFER-DIRECT permit 10
 set local-preference 150
!
```

**Why direct peer with Atlas?**: Content providers often peer directly with ISPs to reduce latency

---

## Phase 4: Eyeball ISP (Comcast-like)

### Goal
Deploy a large residential ISP with many prefixes.

### Why This Matters

**Real eyeball ISPs** (Comcast, AT&T, Verizon):
- Huge customer base (millions of subscribers)
- Many IP blocks for different regions
- Conservative peering (IXP mainly)
- Lots of inbound traffic, less outbound

### FRRouting Configuration

```
hostname Eyeball-ISP-Anchor
!
router bgp 65540
 bgp router-id 10.53.0.1
 no bgp default ipv4-unicast

 ! IXP peering only
 neighbor 198.51.100.251 remote-as 64999
 neighbor 198.51.100.251 description IXP-RS1

 address-family ipv4 unicast
  neighbor 198.51.100.251 activate

  ! Residential ISP blocks (many regions)
  network 71.10.0.0/16
  network 71.11.0.0/16
  network 71.12.0.0/16
  network 96.20.0.0/16
  network 96.21.0.0/16
  network 173.50.0.0/16

  ! Also some /20s for different cities
  network 71.10.0.0/20
  network 71.10.16.0/20
  network 71.10.32.0/20

  neighbor 198.51.100.251 route-map EXPORT-EYEBALL out
 exit-address-family
!
! Null routes for all announced prefixes
ip route 71.10.0.0/16 Null0
ip route 71.11.0.0/16 Null0
ip route 71.12.0.0/16 Null0
ip route 96.20.0.0/16 Null0
ip route 96.21.0.0/16 Null0
ip route 173.50.0.0/16 Null0
!
route-map EXPORT-EYEBALL permit 10
!
```

**Key characteristic**: Announces customer IP ranges, doesn't announce content/CDN prefixes

---

## Verification and Testing

### Goal
Verify anchor tenants are working correctly.

### Step 1: Check BGP Sessions

**On each anchor tenant**:

```bash
sudo vtysh -c "show bgp summary"
```

**Expected**:
- All neighbors in `Established` state
- Prefixes being sent/received

### Step 2: Verify Route Propagation

**On Atlas ISP or Tier-1**:

```bash
show bgp ipv4 unicast summary
show ip route bgp | include 185.10   # CDN routes
show ip route bgp | include 52.      # Hyperscaler routes
show ip route bgp | include 199.50   # Content routes
show ip route bgp | include 71.      # Eyeball ISP routes
```

**Expected**: Routes from all anchor tenants visible

### Step 3: Test Traffic Engineering

**On Atlas ISP, check best path selection**:

```bash
show bgp ipv4 unicast 185.10.1.0/24
```

**Sample output**:
```
BGP routing table entry for 185.10.1.0/24
Paths: (3 available, best #2)
  Not advertised to any peer
  65510
    198.51.100.60 from 198.51.100.251 (10.50.0.1)
      Origin IGP, localpref 100, valid, external
  65510
    203.0.113.33 from 203.0.113.34 (10.50.0.1)
      Origin IGP, localpref 100, valid, external, best
  65510
    100.64.0.6 from 10.0.0.2 (10.0.0.2)
      Origin IGP, localpref 50, valid, internal
```

**Analysis**:
- Best path is via direct peering (203.0.113.33)
- IXP path also available
- Transit path has lowest local-pref (50)

### Step 4: Simulate Route Changes

**On CDN anchor, withdraw a prefix**:

```bash
sudo vtysh
conf t
router bgp 65510
address-family ipv4 unicast
no network 185.10.1.0/24
exit
exit
write memory
```

**On Atlas ISP, verify withdrawal**:

```bash
show bgp ipv4 unicast 185.10.1.0/24
# Should show route withdrawn or alternate path selected
```

---

## Advanced: Route Flapping Simulation

### Goal
Simulate realistic BGP churn (routes appearing/disappearing).

### Create Flapping Script

**On CDN anchor** (`/usr/local/bin/flap-routes.sh`):

```bash
#!/bin/bash

# Routes to flap
ROUTES=("185.10.1.0/24" "185.10.2.0/24")

while true; do
  # Withdraw routes
  for ROUTE in "${ROUTES[@]}"; do
    sudo vtysh -c "conf t" -c "router bgp 65510" -c "address-family ipv4 unicast" \
      -c "no network $ROUTE" -c "exit" -c "exit" -c "write memory"
    echo "$(date): Withdrew $ROUTE"
  done

  sleep 120  # Down for 2 minutes

  # Re-announce routes
  for ROUTE in "${ROUTES[@]}"; do
    sudo vtysh -c "conf t" -c "router bgp 65510" -c "address-family ipv4 unicast" \
      -c "network $ROUTE" -c "exit" -c "exit" -c "write memory"
    echo "$(date): Announced $ROUTE"
  done

  sleep 300  # Up for 5 minutes
done
```

**Run in screen**:

```bash
sudo apt install screen -y
screen -S route-flap
sudo /usr/local/bin/flap-routes.sh
# Ctrl+A, D to detach
```

**Why this matters**: Real internet has constant route changes. This makes your lab more realistic.

---

## Resource Summary

### Complete Anchor Tenant Deployment

| Anchor Tenant | Platform | RAM | CPU | NICs | Peers |
|---------------|----------|-----|-----|------|-------|
| CDN | FRR | 1GB | 1 | 4 | IXP + L3 + Cogent |
| Hyperscaler | FRR | 1GB | 1 | 2 | IXP only |
| Content | FRR | 1GB | 1 | 3 | IXP + Atlas |
| Eyeball ISP | FRR | 2GB | 1 | 2 | IXP only |
| **Total** | | **5-6GB** | **4** | **11** | |

**Additional overhead**: ~500MB for FRRouting processes

---

## Monitoring and Observability

### Goal
Track anchor tenant health and BGP activity.

### Enable SNMP on Anchor Tenants

```bash
sudo apt install snmpd -y

# Edit /etc/snmp/snmpd.conf
agentAddress udp:161
rocommunity public 192.168.100.0/24

sudo systemctl restart snmpd
```

**Query from monitoring system**:

```bash
snmpwalk -v2c -c public 192.168.100.60 1.3.6.1.2.1.15  # BGP MIB
```

### BGP Session Monitoring

**Create monitoring script**:

```bash
#!/bin/bash
# /usr/local/bin/monitor-bgp.sh

ANCHORS=("192.168.100.60:CDN" "192.168.100.61:Hyperscaler"
         "192.168.100.62:Content" "192.168.100.63:Eyeball")

for ANCHOR in "${ANCHORS[@]}"; do
  IP="${ANCHOR%%:*}"
  NAME="${ANCHOR##*:}"

  # SSH and check BGP
  SESSIONS=$(ssh -o StrictHostKeyChecking=no $IP "sudo vtysh -c 'show bgp summary' | grep -c Established")

  echo "$NAME ($IP): $SESSIONS established sessions"
done
```

---

## Integration with Project Atlas

### Where Anchor Tenants Fit

**IXP peering fabric** (`198.51.100.0/24`):
- IXP-RS1: `198.51.100.251`
- Level3-BR1: `198.51.100.51`
- Cogent-BR1: `198.51.100.52`
- Atlas-Core-1: `198.51.100.53`
- PastyNet-Edge-1: `198.51.100.59`
- **CDN**: `198.51.100.60`
- **Hyperscaler**: `198.51.100.61`
- **Content**: `198.51.100.62`
- **Eyeball ISP**: `198.51.100.63`

**Direct peering connections**:
- CDN ↔ Level3: `203.0.113.32/30`
- CDN ↔ Cogent: `203.0.113.36/30`
- Content ↔ Atlas: `203.0.113.40/30`

---

## Troubleshooting

### Common Issues

**Issue 1: BGP Session Won't Establish**

Check:
```bash
sudo vtysh -c "show bgp neighbors 198.51.100.251"
ping 198.51.100.251
```

Common causes:
- Network connectivity broken
- Wrong AS number
- IXP route server not configured for this peer

**Issue 2: Routes Not Appearing in ISP**

Check:
```bash
# On anchor tenant
sudo vtysh -c "show ip bgp neighbors 198.51.100.251 advertised-routes"

# On ISP
show bgp ipv4 unicast neighbors 198.51.100.60 routes
```

Common causes:
- Route-map blocking announcements
- Prefix not in routing table (forgot null route)
- Route server filtering prefixes

---

## Best Practices

1. **Start with CDN**: Easiest to deploy, peers everywhere
2. **Use realistic AS numbers**: Don't use your production ASNs!
3. **Null route announced prefixes**: Prevents routing loops
4. **Document peering relationships**: Know who peers where
5. **Monitor session state**: Alert on BGP flaps
6. **Use route-maps**: Control what you announce/accept
7. **Test route withdrawal**: Know what happens when paths fail

---

## Next Steps

1. Deploy CDN anchor tenant first
2. Verify BGP peering at IXP
3. Add direct peering to Tier-1s
4. Deploy remaining anchor tenants
5. Monitor BGP table growth
6. (Optional) Enable route flapping
7. Integrate with BGP simulation (ExaBGP) for full table

---

*This guide demonstrates how to add realistic "internet backbone" presence to your lab. Combined with ExaBGP route injection (next guide), you'll have a truly realistic ISP simulation.*

## VERSION HISTORY

- v1.0 (2026-02-09): Initial anchor tenant implementation guide
