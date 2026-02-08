# Anchor Tenant Networks - Content Providers & Hyperscalers

## Overview

"Anchor tenants" are strategically important networks that represent major internet players - CDNs, hyperscalers, and large content providers. These add realism to your ISP simulation by providing:
- Real-world peering relationships
- Diverse traffic patterns
- Traffic engineering scenarios
- Multi-path routing practice

---

## Network Architecture

### Updated Topology with Anchor Tenants

```
                        INTERNET (Simulated)
                               |
            +------------------+------------------+
            |                                     |
    ┌───────────────┐                    ┌───────────────┐
    │  LEVEL 3      │                    │   COGENT      │
    │  AS 3356      │                    │   AS 174      │
    │  (4 routers)  │                    │  (4 routers)  │
    └───────┬───────┘                    └───────┬───────┘
            │                                    │
            │         ┌──────────────────┐       │
            └─────────┤  Internet        ├───────┘
                      │  Exchange Point  │
                      │  (2 Route Servers)│
            ┌─────────┤  AS 64999        ├───────┐
            │         └──────────────────┘       │
            │                 │                  │
            │                 │ (IXP Peering)    │
            │                 │                  │
    ┌───────┴────────┐  ┌─────┴────────┐  ┌──────┴────────┐
    │ CDN Network    │  │ Content Net  │  │ Eyeball ISP   │
    │ AS 65300       │  │ AS 65302     │  │ AS 65303      │
    │ (Cloudflare-   │  │ (Meta-like)  │  │ (Comcast-like)│
    │  like)         │  │              │  │               │
    └────────┬───────┘  └──────────────┘  └───────────────┘
             │
             │ (PNI)
             │
    ┌────────┴──────────────────────────────────┐
    │                                           │
┌───┴──────────┐                        ┌───────┴──────┐
│ Atlas Lab ISP│                        │  Second ISP  │
│  AS 65000    │                        │   AS 65100   │
│  (On-Prem)   │                        │   (Cloud)    │
└──────┬───────┘                        └───────┬──────┘
       │                                        │
       │  ┌─────────────────────────────────┐   │
       │  │   Hyperscaler Network           │   │
       │  │   AS 65301 (AWS-like)           │   │
       │  │   Direct Connect to Second ISP  │   │
       └──┤   Transit from Level3           ├───┘
          └─────────────────────────────────┘
                      │
              ┌───────┴────────┐
              │   Multi-Site   │
              │    Customer    │
              └────────────────┘
```

---

## Anchor Tenant #1: CDN Network (Cloudflare-like)

### Profile
- **ASN**: AS 65300
- **Type**: Global Content Delivery Network
- **Platform**: VyOS (2GB RAM, 2 vCPU)
- **Purpose**: Anycast DNS, CDN edge, DDoS protection

### Peering Strategy

**Multi-Homed Transit**:
- Level 3 (AS 3356) - Primary
- Cogent (AS 174) - Secondary
- Prefer shorter AS path to Level 3

**IXP Peering** (Settlement-Free):
- Both Tier-1s at IXP
- Atlas Lab ISP (PNI - Private Network Interconnect)
- Content Provider network (peer-peer)

**Traffic Engineering**:
- Anycast: Same /24 announced from multiple locations
- Use communities to control path advertisement
- AS-path prepending to influence inbound traffic

### IP Allocation

```yaml
Loopback: 10.255.30.1/32
Management: 192.168.100.230/24

Peering Networks:
  Level3_Transit: 203.0.113.32/30 (GigE to L3-Border-1)
  Cogent_Transit: 203.0.113.36/30 (GigE to Cogent-Border-1)
  IXP_Level3: 198.51.100.60/24 (at IXP-RS1)
  IXP_Cogent: 198.51.100.60/24 (same anycast IP)
  Atlas_PNI: 10.0.200.0/30 (direct to Atlas-Core-1)

Announced Prefixes:
  1.1.1.0/24          # DNS service (anycast)
  1.0.0.0/24          # WARP VPN
  104.16.0.0/13       # CDN edge ranges
  172.64.0.0/13       # Additional CDN
  188.114.96.0/20     # European CDN
```

### Traffic Patterns to Simulate
- **DNS queries**: High volume, low bandwidth (~100 Mbps)
- **CDN traffic**: Moderate volume, high bandwidth (~1 Gbps)
- **DDoS mitigation**: Sudden traffic spikes, blackhole communities

---

## Anchor Tenant #2: Hyperscaler Network (AWS-like)

### Profile
- **ASN**: AS 65301
- **Type**: Public Cloud Provider
- **Platform**: VyOS (2GB RAM, 2 vCPU)
- **Purpose**: Simulate AWS with Direct Connect

### Peering Strategy

**Transit Only**:
- Level 3 (AS 3356) - Primary internet transit
- NO Cogent (simulates selective provider choice)

**Direct Connect**:
- Private connection to Second ISP (AS 65100)
- Simulates AWS Direct Connect or Azure ExpressRoute
- Preferred path for on-prem customer traffic

**No IXP Peering**: Hyperscalers typically don't peer at public IXPs for production traffic

### IP Allocation

```yaml
Loopback: 10.255.31.1/32

Peering Networks:
  Level3_Transit: 203.0.113.40/30
  Second_ISP_DirectConnect: 10.1.200.0/30

Announced Prefixes:
  # Simulate large cloud provider prefix space
  # Use ExaBGP to announce bulk routes
  52.0.0.0/11         # us-east-1 simulation
  54.0.0.0/11         # us-west-2 simulation
  3.0.0.0/15          # newer ranges
  # Total: ~8,000 prefixes via ExaBGP
```

### ExaBGP for Route Injection

Since hyperscalers announce thousands of prefixes, use ExaBGP:

```python
# /etc/exabgp/hyperscaler-routes.py
#!/usr/bin/env python3

prefixes = [
    '52.0.0.0/11',
    '54.0.0.0/11',
    '3.0.0.0/15',
    # ... generate full list
]

for prefix in prefixes:
    print(f'announce route {prefix} next-hop 10.255.31.1')
```

---

## Anchor Tenant #3: Content Provider (Meta-like)

### Profile
- **ASN**: AS 65302
- **Type**: Large Content/Social Media Network
- **Platform**: VyOS (1GB RAM, 1 vCPU)
- **Purpose**: Selective peering, no paid transit

### Peering Strategy

**IXP Peering ONLY**:
- Level 3 at IXP (settlement-free)
- Cogent at IXP (settlement-free)
- Atlas Lab ISP at IXP
- NO paid transit providers

**Peering Policy**:
- Open peering policy at IXPs
- Selective announcement (only peer routes, not full table)
- Use communities to tag routes

### IP Allocation

```yaml
Loopback: 10.255.32.1/32

Peering Networks:
  IXP: 198.51.100.62/24

Announced Prefixes:
  31.13.0.0/16        # Main application ranges
  157.240.0.0/16      # CDN/edge
  185.60.216.0/22     # European presence
  # Total: ~1,000 prefixes
```


---

## Anchor Tenant #4: Regional Eyeball ISP (Comcast-like)

### Profile
- **ASN**: AS 65303
- **Type**: Large Residential/Business ISP
- **Platform**: VyOS (1GB RAM, 1 vCPU)
- **Purpose**: Residential customer simulation, hot potato routing

### Peering Strategy

**Paid Transit**:
- Cogent (AS 174) - Primary
- Level 3 (AS 3356) - Backup

**Settlement-Free Peering**:
- Atlas Lab ISP at IXP (local traffic exchange)
- "Hot potato" routing: hand off traffic as soon as possible

### IP Allocation

```yaml
Loopback: 10.255.33.1/32

Peering Networks:
  Cogent_Transit: 203.0.113.44/30
  IXP: 198.51.100.63/24

Announced Prefixes:
  68.0.0.0/10         # Residential broadband
  96.0.0.0/11         # Business services
  50.128.0.0/9        # Additional ranges
  # Total: ~15,000 prefixes (use ExaBGP)
```


---

## Resource Summary

### Total Anchor Tenant Resources

| Network | Type | RAM | vCPU | Count | Total RAM |
|---------|------|-----|------|-------|-----------|
| CDN (AS 65300) | VyOS | 2GB | 2 | 1 | 2GB |
| Hyperscaler (AS 65301) | VyOS | 2GB | 2 | 1 | 2GB |
| Content (AS 65302) | VyOS | 1GB | 1 | 1 | 1GB |
| Eyeball (AS 65303) | VyOS | 1GB | 1 | 1 | 1GB |
| **TOTAL** | | | | | **6GB** |

### Full Topology Resource Usage

| Component | RAM | Description |
|-----------|-----|-------------|
| Original 20 routers | 68GB | Core ISP topology |
| Anchor tenants | 6GB | 4 content/CDN networks |
| ExaBGP injectors (3x) | 1.5GB | Route injection |
| EVE-NG overhead | ~20GB | Hypervisor |
| **TOTAL** | **~95.5GB** | Still under 120GB allocation |

---

## Traffic Engineering Scenarios

### Scenario 1: CDN Traffic Optimization

**Goal**: Route CDN traffic through PNI instead of transit

```bash
# On Atlas-Core-1
set policy route-map FROM-CDN-PNI rule 10 action permit
set policy route-map FROM-CDN-PNI rule 10 set local-preference 200

# Result: CDN traffic enters via PNI, exits via PNI
# Saves transit costs, lower latency
```

### Scenario 2: Direct Connect Failover

**Goal**: Simulate Direct Connect failure, fallback to internet

```bash
# On Hyperscaler-Edge
# Shutdown Direct Connect interface
set interfaces ethernet eth1 disable

# BGP automatically reroutes via Level3 transit
# Verify with: show ip route 10.10.0.0/16
# Should now point to Level3 next-hop
```

### Scenario 3: DDoS Mitigation

**Goal**: Use blackhole communities to drop attack traffic

```bash
# On CDN-Edge-1 (under attack)
# Signal upstream to blackhole traffic to 1.1.1.100
set protocols bgp address-family ipv4-unicast network 1.1.1.100/32
set protocols bgp address-family ipv4-unicast network 1.1.1.100/32 route-map ADD-BLACKHOLE

set policy route-map ADD-BLACKHOLE rule 10 action permit
set policy route-map ADD-BLACKHOLE rule 10 set community '65000:666'  # Blackhole community
```
## VERSION HISTORY

- v1.0 (2026-02-07): Initial resource compilation
- v1.1 (2026-02-08): Cleaned up and updated original AI documentation