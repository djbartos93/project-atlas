# BGP Simulation Strategy for Project Atlas

## Overview

This document outlines strategies to create realistic, dynamic BGP behavior without requiring dozens of additional ISP routers. The goal is to simulate real-world internet conditions including route changes, flapping, traffic engineering, and multi-path routing.

---

## Problem Statement

A static BGP lab with 20 routers and fixed routes doesn't represent real internet behavior:
- No route churn or flapping
- No alternative paths appearing/disappearing
- No simulation of traffic engineering changes
- Limited BGP table size (~50 routes vs real-world 900k+)
- No representation of major content networks (Cloudflare, Google, Meta, etc.)

**Solution**: Combine lightweight route injectors with strategic "anchor tenant" networks to create dynamic, realistic BGP behavior.

---

## Strategy 1: ExaBGP Route Injector (Primary Recommendation)

### What is ExaBGP?
Lightweight Python-based BGP speaker that can inject/withdraw thousands of routes programmatically without requiring full router VMs.

### Resource Requirements
- **RAM**: 512MB per instance
- **CPU**: Minimal (0.5 vCPU)
- **OS**: Linux VM or container

### Deployment Architecture

```
IXP Route Server 1 (RS1)
    |
    |-- ExaBGP Injector #1 (AS65500)
    |     Announces: 50,000 IPv4 prefixes
    |     Represents: Asia-Pacific ISPs
    |
    |-- ExaBGP Injector #2 (AS65501)
    |     Announces: 30,000 IPv4 prefixes
    |     Represents: European ISPs
    |
    |-- ExaBGP Injector #3 (AS65502)
          Announces: 20,000 IPv4 prefixes
          Represents: Latin American ISPs
```

### Configuration Example

**ExaBGP config** (`/etc/exabgp/exabgp.conf`):
```python
neighbor 198.51.100.251 {
    router-id 10.255.1.1;
    local-address 198.51.100.10;
    local-as 65500;
    peer-as 64999;  # IXP Route Server

    static {
        # Announce 50k prefixes
        route 45.0.0.0/8 next-hop 198.51.100.10;
        route 46.0.0.0/8 next-hop 198.51.100.10;
        # ... (generate with script)
    }
}
```

**Python script for dynamic announcements**:
```python
#!/usr/bin/env python3
from exabgp.configuration.environment import environment
from random import randint, choice
import time

# Simulate route flapping
def flap_routes():
    prefixes = ["45.10.0.0/16", "45.20.0.0/16", "45.30.0.0/16"]

    while True:
        # Randomly withdraw route
        prefix = choice(prefixes)
        print(f"withdraw route {prefix}")
        time.sleep(randint(60, 300))  # 1-5 minutes

        # Re-announce
        print(f"announce route {prefix} next-hop 198.51.100.10")
        time.sleep(randint(600, 1800))  # 10-30 minutes

if __name__ == '__main__':
    flap_routes()
```

### Benefits
- **Massive route injection**: 100k+ routes with 1.5GB RAM total
- **Dynamic behavior**: Script route announcements/withdrawals
- **Low overhead**: No full OS per "ISP"
- **Realistic scale**: Approaches real internet BGP table size

### Limitations
- No transit routing (just route announcement)
- Can't simulate actual packet forwarding through these "ISPs"
- Limited protocol testing (OSPF, MPLS, etc. not available)

---

## Strategy 2: Content Network "Anchor Tenants"

### Concept
Add 3-4 strategic routers representing major internet players. These aren't full ISPs but specialized networks with realistic peering relationships.

### Recommended Anchor Tenants

#### 1. CDN Network (Cloudflare-like) - AS65300

**Purpose**: Simulate global content delivery network
**Platform**: VyOS (2GB RAM)
**Peering Strategy**:
- Multi-homed to both Tier-1s (Level3 + Cogent)
- Direct peering at IXP (settlement-free)
- Direct peering with your Atlas ISP (PNI)

**Route Announcements**:
```
1.1.1.0/24        # DNS service (anycast)
104.16.0.0/12     # CDN ranges
172.64.0.0/13     # Additional CDN
Total: ~200 prefixes
```

**Anycast Simulation**: Same prefixes announced from multiple locations with identical metrics

#### 2. Hyperscaler Network (AWS-like) - AS65301

**Purpose**: Simulate major cloud provider
**Platform**: VyOS (2GB RAM)
**Peering Strategy**:
- Transit from Level3 only (no Cogent)
- Direct connection to Second ISP (simulates AWS Direct Connect)

**Route Announcements**:
```
52.0.0.0/11       # EC2 ranges (us-east-1)
54.0.0.0/11       # EC2 ranges (us-west-2)
3.0.0.0/15        # EC2 ranges (newer)
Total: ~8,000 prefixes (use ExaBGP to inject bulk)
```

**Traffic Engineering**:
- Prefer Direct Connect path for your on-prem traffic
- Use internet path as backup
- Simulate Direct Connect "failover" by withdrawing private BGP session

#### 3. Major Content Provider (Meta-like) - AS65302

**Purpose**: Simulate large content network with selective peering
**Platform**: VyOS (1GB RAM)
**Peering Strategy**:
- IXP peering ONLY (no paid transit)
- Settlement-free with Tier-1s
- Selective announcement (only to peers matching policy)

**Route Announcements**:
```
31.13.0.0/16      # Facebook ranges
157.240.0.0/16    # Instagram ranges
Total: ~1,000 prefixes
```

**BGP Communities for Traffic Engineering**:
```bash
# Don't advertise to certain peers
set policy community-list CDN-ONLY rule 10 action permit
set policy community-list CDN-ONLY rule 10 regex '65302:100'

# Prepend to influence inbound path
set policy route-map PEER-OUT rule 10 set as-path prepend '65302 65302 65302'
```

#### 4. Regional Eyeball Network (Comcast-like) - AS65303

**Purpose**: Simulate residential ISP with large customer base
**Platform**: VyOS (1GB RAM)
**Peering Strategy**:
- Transit from Cogent
- Direct peering with Atlas ISP (settlement-free, "hot potato")
- Large announced blocks (aggregates)

**Route Announcements**:
```
68.0.0.0/10       # Residential broadband
96.0.0.0/11       # Business services
Total: ~15,000 prefixes
```

---

## Strategy 3: Route Flapping & Churn Simulation

### Automated BGP Events

Create scheduled scripts to simulate real-world instability:

**Daily Flapping Script** (`/opt/bgp-sim/daily-flap.sh`):
```bash
#!/bin/bash
# Simulates route flapping events

ROUTES=("45.10.0.0/16" "45.20.0.0/16" "45.30.0.0/16")

while true; do
    # Pick random route
    ROUTE=${ROUTES[$RANDOM % ${#ROUTES[@]}]}

    # Withdraw
    echo "announce route $ROUTE next-hop 198.51.100.10 withdrawn" | \
        socat - UNIX-CONNECT:/var/run/exabgp.sock

    # Wait 2-10 minutes
    sleep $((120 + RANDOM % 480))

    # Re-announce
    echo "announce route $ROUTE next-hop 198.51.100.10" | \
        socat - UNIX-CONNECT:/var/run/exabgp.sock

    # Wait 30-90 minutes before next flap
    sleep $((1800 + RANDOM % 3600))
done
```

**AS Path Change Simulation**:
```python
# Periodically change AS path to simulate rerouting
import random
import time

paths = [
    [65500, 65510, 65520],  # Normal path
    [65500, 65511, 65520],  # Alternate path
    [65500, 65510, 65512, 65520],  # Longer path (traffic engineering)
]

while True:
    path = random.choice(paths)
    announce_route("45.10.0.0/16", as_path=path)
    time.sleep(random.randint(3600, 7200))  # Change every 1-2 hours
```

---

## Strategy 4: BGP Communities & Traffic Engineering

### Implement Realistic Communities

**Atlas ISP Communities** (AS65000):
```
65000:100  = Learned from customer
65000:200  = Learned from peer
65000:300  = Learned from transit

65000:1000 = Do not advertise to peers
65000:2000 = Do not advertise to customers
65000:3000 = Prepend 1x to peers
65000:3001 = Prepend 2x to peers
65000:3002 = Prepend 3x to peers
```

**Use Case Example**:
```bash
# Customer wants to prefer Atlas ISP for inbound traffic
# Tag routes with community to prepend on other ISPs
set policy route-map CUSTOMER-IN rule 10 match community PREPEND-OTHERS
set policy route-map CUSTOMER-IN rule 10 set community '65000:3002'
```

---

## Recommended Deployment Plan

### Phase 1: Foundation (Week 1)
1. Build core 20-router topology
2. Establish base BGP peering
3. Verify basic reachability

### Phase 2: Route Injection (Week 2)
1. Deploy ExaBGP instance #1 at IXP
2. Inject 10,000 test routes
3. Monitor BGP table size and convergence time
4. Scale to 50,000 routes

### Phase 3: Anchor Tenants (Week 3)
1. Add CDN network (AS65300)
2. Configure multi-homing and anycast
3. Add Hyperscaler (AS65301) with Direct Connect
4. Test traffic engineering and path preferences

### Phase 4: Dynamic Behavior (Week 4)
1. Implement route flapping scripts
2. Add AS path manipulation
3. Configure BGP communities across all routers
4. Create traffic pattern automation

### Phase 5: Advanced Scenarios (Week 5+)
1. Simulate DDoS mitigation (blackhole communities)
2. Test BGP hijacking detection
3. Implement RTBH (Remotely Triggered Black Hole)
4. Create route leak scenarios for troubleshooting practice

---

## Resource Summary

### With ExaBGP + 4 Anchor Tenants

| Component | RAM | CPU | Count | Total RAM |
|-----------|-----|-----|-------|-----------|
| Core 20 routers | 68GB | 40 vCPU | 1 set | 68GB |
| ExaBGP Injectors | 512MB | 0.5 vCPU | 3 | 1.5GB |
| CDN (AS65300) | 2GB | 2 vCPU | 1 | 2GB |
| Hyperscaler (AS65301) | 2GB | 2 vCPU | 1 | 2GB |
| Content (AS65302) | 1GB | 1 vCPU | 1 | 1GB |
| Eyeball (AS65303) | 1GB | 1 vCPU | 1 | 1GB |
| **TOTAL** | | | | **~75.5GB** |

Still well within your 120GB EVE-NG allocation!

---

## Verification & Testing

### BGP Table Size Check
```bash
# On any router
show ip bgp summary
# Should show 50k-100k+ prefixes

show ip bgp | count
# Verify route count
```

### Route Diversity Check
```bash
# Check multiple paths to same destination
show ip bgp 1.1.1.0/24
# Should show paths through Level3, Cogent, Atlas ISP

show ip route 1.1.1.0 longer-prefixes
# Verify best path selection
```

### Flapping Detection
```bash
# Check BGP dampening
show ip bgp dampening flap-statistics
# Should show prefixes with flap count

show ip bgp dampening dampened-paths
# Show suppressed routes
```

---

## Additional Resources

- **ExaBGP Documentation**: https://github.com/Exa-Networks/exabgp
- **GoBGP**: https://osrg.github.io/gobgp/
- **BGP Communities**: RFC 1997, RFC 8092
- **Route Flap Dampening**: RFC 2439
- **BGP Best Practices**: MANRS (Mutually Agreed Norms for Routing Security)

---

## VERSION HISTORY

- v1.0 (2026-02-07): Initial resource compilation
- v1.1 (2026-02-08): Cleaned up and updated original AI documentation