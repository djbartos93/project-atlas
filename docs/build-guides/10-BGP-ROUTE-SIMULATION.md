# BGP Route Simulation with ExaBGP - Implementation Guide

## NOT REVIEWED YET

## Note

All IP addresses and interface names are generic in this guide. You should replace them with your own IP addresses and interface names. See IP_ADDRESSING_PHILOSOPHY.md for more information.

## Overview

This guide walks through deploying **ExaBGP route injectors** to simulate a realistic internet BGP table. Instead of running hundreds of real routers, you'll inject 50,000-100,000 routes from a single lightweight VM, making your ISP lab feel like it's connected to the real internet.

### What is ExaBGP?

**ExaBGP** is a Python-based BGP speaker designed for:
- Route injection and manipulation
- BGP testing and simulation
- Flow spec (DDoS mitigation)
- Programmatic BGP control

**In your lab**: ExaBGP will inject thousands of "fake" routes to make your BGP table look realistic, while keeping resource usage minimal.

---

## Quick Reference

### What You'll Build

```
ExaBGP Injector VM → peers with → Tier-1 ISPs
                                   ↓
                         Announces 50k-100k routes
                                   ↓
                    Routes propagate through your network
                                   ↓
                       Your BGP table looks real!
```

**Resources needed**:
- 1 VM: 2GB RAM, 1 vCPU, 20GB disk
- Python 3.8+
- ExaBGP 4.2+
- Route list file (pre-generated)

**Time to deploy**: 30-60 minutes

---

## Architecture Decision: Fake vs Real Routes

### Understanding the Split

**Question from user**: "If we inject fake routes, how will K8s services reach real destinations?"

**Answer**: You'll use TWO separate IP ranges:

**1. Fake Routes (ExaBGP)** - For BGP table realism:
- Ranges: `45.10.0.0/16`, `103.20.0.0/16`, `185.50.0.0/16`, etc.
- Purpose: Make BGP table large (50k routes)
- Destination: Null-routed (traffic goes nowhere)
- Used for: Learning, testing route selection, BGP convergence

**2. Real Routes (Anchor Tenants)** - For actual traffic:
- Ranges: `185.10.0.0/16` (CDN), `52.10.0.0/16` (AWS), etc.
- Purpose: Route actual traffic to services
- Destination: Real VMs that can respond
- Used for: K8s connectivity, testing, actual services

### Traffic Flow Examples

**Example 1: Testing BGP convergence (fake route)**
```
curl 45.10.1.1  →  Routed by BGP  →  Null-routed  →  No response (expected)
```

**Example 2: Accessing real service (real route)**
```
K8s pod calls 185.10.1.1  →  Routed by BGP  →  CDN anchor tenant  →  Responds!
```

### Best Practice

- **ExaBGP announces**: `0.0.0.0/0` (default) + 50k fake prefixes
- **Anchor tenants announce**: Real prefixes where services exist
- **Your services use**: Anchor tenant prefixes only

**Result**: Huge realistic BGP table + actual working connectivity

---

## Phase 1: Generate Route List

### Goal
Create a file with 50,000-100,000 realistic prefixes to announce.

### Option 1: Use Real BGP Table Data (Recommended)

**Download from RouteViews**:

```bash
# Get latest BGP table snapshot
wget http://archive.routeviews.org/bgpdata/$(date +%Y.%m)/RIBS/rib.$(date +%Y%m%d).0000.bz2

# Extract and parse (requires bgpdump)
sudo apt install bgpdump -y
bgpdump -m rib.*.bz2 | awk '{print $6}' | sort -u > real-routes.txt

# Take first 50k routes
head -50000 real-routes.txt > bgp-routes-50k.txt
```

**Why real routes?**: They have realistic distribution of prefix sizes (/24s, /20s, /16s)

### Option 2: Generate Synthetic Routes (Faster)

**Create generation script** (`generate-routes.py`):

```python
#!/usr/bin/env python3
import ipaddress
import random

def generate_routes(count=50000):
    """Generate realistic-looking BGP prefixes"""
    routes = set()

    # Prefix length distribution (realistic)
    prefix_lengths = {
        24: 0.60,  # 60% are /24s
        23: 0.15,  # 15% are /23s
        22: 0.10,  # 10% are /22s
        20: 0.08,  # 8% are /20s
        16: 0.05,  # 5% are /16s
        19: 0.02,  # 2% are /19s
    }

    # IP ranges to use (avoid reserved ranges)
    usable_ranges = [
        "45.0.0.0/8",      # APNIC
        "103.0.0.0/8",     # APNIC
        "185.0.0.0/8",     # RIPE
        "191.0.0.0/8",     # LACNIC
        "201.0.0.0/8",     # LACNIC
    ]

    while len(routes) < count:
        # Pick random range
        base_range = random.choice(usable_ranges)
        network = ipaddress.ip_network(base_range)

        # Pick random prefix length
        prefix_len = random.choices(
            list(prefix_lengths.keys()),
            weights=list(prefix_lengths.values())
        )[0]

        # Generate random subnet
        try:
            random_ip = ipaddress.ip_address(
                random.randint(
                    int(network.network_address),
                    int(network.broadcast_address)
                )
            )
            route = ipaddress.ip_network(f"{random_ip}/{prefix_len}", strict=False)
            routes.add(str(route))
        except ValueError:
            continue

    return sorted(routes)

if __name__ == "__main__":
    print("Generating 50,000 routes...")
    routes = generate_routes(50000)

    with open("bgp-routes-50k.txt", "w") as f:
        for route in routes:
            f.write(f"{route}\n")

    print(f"Generated {len(routes)} routes to bgp-routes-50k.txt")

    # Statistics
    prefix_counts = {}
    for route in routes:
        prefix_len = route.split('/')[1]
        prefix_counts[prefix_len] = prefix_counts.get(prefix_len, 0) + 1

    print("\nPrefix distribution:")
    for prefix, count in sorted(prefix_counts.items()):
        print(f"  /{prefix}: {count} ({count/len(routes)*100:.1f}%)")
```

**Run it**:

```bash
python3 generate-routes.py
```

**Output**: `bgp-routes-50k.txt` with 50,000 realistic routes

---

## Phase 2: Deploy ExaBGP VM

### Goal
Set up a VM running ExaBGP to inject routes.

### Step 1: Create VM

**VM Specs**:
- OS: Ubuntu 22.04 LTS
- RAM: 2GB (ExaBGP is memory-efficient)
- vCPU: 1 core
- Disk: 20GB
- Network: Connect to Level 3 or Cogent (or both)

**Why connect to Tier-1?**: They propagate to everyone else, so one injection point = full table everywhere

### Step 2: Install ExaBGP

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Python and dependencies
sudo apt install python3 python3-pip git -y

# Install ExaBGP
sudo pip3 install exabgp

# Verify installation
exabgp --version
```

**Expected**: `ExaBGP 4.2.21` (or newer)

### Step 3: Configure Network Interface

**Edit `/etc/netplan/01-netcfg.yaml`** (or `/etc/network/interfaces`):

```yaml
network:
  version: 2
  ethernets:
    eth0:
      addresses:
        - 192.168.100.70/24  # Management
      gateway4: 192.168.100.1
      nameservers:
        addresses: [8.8.8.8]

    eth1:
      addresses:
        - 203.0.113.50/30  # Peer with Level3-Border-1
```

**Apply**:

```bash
sudo netplan apply
ping 203.0.113.49  # Ping Level 3
```

---

## Phase 3: ExaBGP Configuration

### Goal
Configure ExaBGP to announce routes to Tier-1 ISP.

### Understanding ExaBGP Config

**ExaBGP config has 3 sections**:
1. **Neighbor**: Who we peer with (Level 3, Cogent)
2. **Process**: How to generate routes (script or static)
3. **Flow**: Announce/withdraw routes dynamically

### Step 1: Create ExaBGP Configuration

**Create `/etc/exabgp/exabgp.conf`**:

```ini
# ExaBGP Configuration for Route Injection

# Process to announce routes from file
process announce-routes {
    run /usr/local/bin/exabgp-announce.py;
    encoder json;
}

# BGP neighbor configuration
neighbor 203.0.113.49 {
    description "Level3-Border-1";
    router-id 10.99.0.1;
    local-address 203.0.113.50;
    local-as 65999;
    peer-as 3356;

    family {
        ipv4 unicast;
    }

    # Announce routes from process
    api {
        processes [ announce-routes ];
    }
}

# Optional: Second peer to Cogent for redundancy
# neighbor 203.0.113.53 {
#     description "Cogent-Border-1";
#     router-id 10.99.0.1;
#     local-address 203.0.113.54;
#     local-as 65999;
#     peer-as 174;
#     family {
#         ipv4 unicast;
#     }
#     api {
#         processes [ announce-routes ];
#     }
# }
```

**Key settings**:
- `local-as 65999`: Private AS for simulation (won't conflict)
- `router-id`: Unique identifier
- `process`: Points to script that generates announcements

### Step 2: Create Route Announcement Script

**Create `/usr/local/bin/exabgp-announce.py`**:

```python
#!/usr/bin/env python3
"""
ExaBGP route announcement script
Reads routes from file and announces via BGP
"""

import sys
import time
import os

ROUTE_FILE = "/etc/exabgp/routes.txt"
NEXT_HOP = "203.0.113.50"  # Our IP (ExaBGP VM)

def announce_routes(route_file):
    """Read routes from file and announce them"""
    if not os.path.exists(route_file):
        sys.stderr.write(f"Route file {route_file} not found!\n")
        sys.exit(1)

    with open(route_file, 'r') as f:
        routes = [line.strip() for line in f if line.strip()]

    sys.stderr.write(f"Loaded {len(routes)} routes from {route_file}\n")

    # Announce each route in ExaBGP JSON format
    for route in routes:
        announcement = {
            "exabgp": "4.0.1",
            "time": time.time(),
            "neighbor": {
                "ip": "203.0.113.49",  # Level3 peer IP
                "address": {
                    "local": "203.0.113.50",
                    "peer": "203.0.113.49"
                }
            },
            "type": "update",
            "update": {
                "announce": {
                    "ipv4 unicast": {
                        NEXT_HOP: [
                            {
                                "nlri": route
                            }
                        ]
                    }
                }
            }
        }

        # Send to stdout (ExaBGP reads from stdout)
        print(f"announce route {route} next-hop {NEXT_HOP}")
        sys.stdout.flush()

        # Small delay to avoid overwhelming BGP (optional)
        # time.sleep(0.001)  # 1ms delay = 1000 routes/sec

    sys.stderr.write(f"Announced all {len(routes)} routes\n")

    # Keep process alive
    while True:
        time.sleep(60)

if __name__ == "__main__":
    announce_routes(ROUTE_FILE)
```

**Make executable**:

```bash
sudo chmod +x /usr/local/bin/exabgp-announce.py
```

### Step 3: Copy Routes File

```bash
# Copy your generated routes
sudo cp bgp-routes-50k.txt /etc/exabgp/routes.txt

# Verify
wc -l /etc/exabgp/routes.txt
```

---

## Phase 4: Start ExaBGP

### Goal
Launch ExaBGP and verify routes are being announced.

### Step 1: Test Configuration

```bash
# Test config syntax
exabgp --test /etc/exabgp/exabgp.conf
```

**Expected**: No errors

### Step 2: Start ExaBGP

**Option 1: Foreground (for testing)**:

```bash
exabgp /etc/exabgp/exabgp.conf
```

**Watch output**:
```
2026-02-09 02:00:00 | INFO     | Loaded 50000 routes from /etc/exabgp/routes.txt
2026-02-09 02:00:01 | INFO     | Neighbor 203.0.113.49 connected
2026-02-09 02:00:02 | INFO     | Announced all 50000 routes
```

**Option 2: Systemd service (for production)**:

**Create `/etc/systemd/system/exabgp.service`**:

```ini
[Unit]
Description=ExaBGP BGP Route Injector
After=network.target

[Service]
Type=simple
User=exabgp
Group=exabgp
ExecStart=/usr/local/bin/exabgp /etc/exabgp/exabgp.conf
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**Create user and start**:

```bash
# Create dedicated user
sudo useradd -r -s /bin/false exabgp
sudo chown -R exabgp:exabgp /etc/exabgp

# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable exabgp
sudo systemctl start exabgp

# Check status
sudo systemctl status exabgp
sudo journalctl -u exabgp -f
```

---

## Phase 5: Verify Route Injection

### Goal
Confirm routes are appearing in your ISP routers.

### Step 1: Check BGP Session

**On Level3-Border-1** (Juniper):

```junos
show bgp summary
show bgp neighbor 203.0.113.50
```

**Expected**:
```
Peer: 203.0.113.50 AS 65999
  Type: External    State: Established
  Flags: <ImportEval Export Eval>
  Last State: OpenConfirm   Last Event: RecvKeepAlive
  Last Error: None
  Options: <Preference LocalAddress PeerAS Refresh>
  Holdtime: 180 Preference: 170 Local AS: 3356 Local System AS: 0
  Number of flaps: 0
  Peer ID: 10.99.0.1      Local ID: 10.3356.0.11
  Active Holdtime: 180
  Keepalive Interval: 60         Group index: 0    Peer index: 4
  I/O Session Thread: bgpio-0 State: Enabled
  BFD: disabled, down
  NLRI for restart configured on peer: inet-unicast
  NLRI advertised by peer: inet-unicast
  NLRI for this session: inet-unicast
  Peer supports Refresh capability (2)
  Table inet.0 Bit: 20000
    RIB State: BGP restart is complete
    Send state: in sync
    Active prefixes:              50000
    Received prefixes:            50000
    Accepted prefixes:            50000
    Suppressed due to damping:    0
    Advertised prefixes:          0
  Last traffic (seconds): Received 12    Sent 8     Checked 12
  Input messages:  Total 102    Updates 2        Refreshes 0    Octets 1234567
  Output messages: Total 4      Updates 0        Refreshes 0    Octets 234
```

**Key metric**: `Accepted prefixes: 50000` ✓

### Step 2: Check Route Table

```junos
show route summary
show route protocol bgp | count
```

**Sample output**:
```
inet.0: 52000 destinations, 52000 routes (52000 active, 0 holddown, 0 hidden)
```

**Before ExaBGP**: ~100-200 routes (just your lab)
**After ExaBGP**: ~50,000 routes ✓

### Step 3: Verify Route Propagation

**On Atlas ISP (downstream from Level 3)**:

```bash
show bgp summary
show ip route bgp | count
```

**Expected**: Routes from ExaBGP visible on Atlas ISP (via Level 3)

**Check specific route**:

```bash
show ip route 45.10.1.0/24
```

**Sample output**:
```
B    45.10.1.0/24 [20/0] via 100.64.0.2, eth4, 00:05:23
                          Level3-Transit
```

**Success**: ExaBGP route made it through Tier-1 to Tier-2! ✓

---

## Phase 6: Advanced Features

### Goal
Add dynamic route manipulation and flapping.

### Feature 1: Route Flapping

**Create flap script** (`/usr/local/bin/exabgp-flap.py`):

```python
#!/usr/bin/env python3
"""Route flapping simulator for ExaBGP"""

import sys
import time
import random

FLAP_ROUTES = [
    "45.10.1.0/24",
    "45.10.2.0/24",
    "103.20.5.0/24",
]

NEXT_HOP = "203.0.113.50"

def flap_routes():
    """Announce and withdraw routes randomly"""
    while True:
        route = random.choice(FLAP_ROUTES)

        # Randomly announce or withdraw
        if random.random() < 0.5:
            print(f"announce route {route} next-hop {NEXT_HOP}")
            sys.stderr.write(f"[FLAP] Announced {route}\n")
        else:
            print(f"withdraw route {route}")
            sys.stderr.write(f"[FLAP] Withdrew {route}\n")

        sys.stdout.flush()

        # Random delay between flaps (30-120 seconds)
        time.sleep(random.randint(30, 120))

if __name__ == "__main__":
    flap_routes()
```

**Add to ExaBGP config**:

```ini
process flap-routes {
    run /usr/local/bin/exabgp-flap.py;
}
```

**Why flap?**: Teaches BGP dampening, route convergence, stability

### Feature 2: AS-Path Prepending

**Modify announcement to make routes less attractive**:

```python
# In exabgp-announce.py, add AS-path prepending
announcement = f"announce route {route} next-hop {NEXT_HOP} as-path [65999 65999 65999]"
```

**Result**: Route has longer AS-path, less preferred

### Feature 3: Community Tagging

```python
announcement = f"announce route {route} next-hop {NEXT_HOP} community [65999:100]"
```

**Use case**: Tag simulated routes so you can filter them later

---

## Resource Usage and Scaling

### Memory Usage

**ExaBGP memory usage by route count**:

| Routes | RAM Usage | CPU Usage |
|--------|-----------|-----------|
| 10,000 | ~200MB | <5% |
| 50,000 | ~500MB | ~10% |
| 100,000 | ~1GB | ~15% |
| 500,000 | ~3GB | ~25% |

**Rule of thumb**: ~10KB per route in memory

### Tuning for Large Tables

**For 100k+ routes**, modify script:

```python
# Batch announcements instead of one-by-one
BATCH_SIZE = 100

for i in range(0, len(routes), BATCH_SIZE):
    batch = routes[i:i+BATCH_SIZE]
    for route in batch:
        print(f"announce route {route} next-hop {NEXT_HOP}")
    sys.stdout.flush()
    time.sleep(0.1)  # Small delay between batches
```

**Why batch?**: Reduces BGP update overhead

---

## Monitoring and Troubleshooting

### Check ExaBGP Status

```bash
# Service status
sudo systemctl status exabgp

# Live logs
sudo journalctl -u exabgp -f

# Check process
ps aux | grep exabgp
```

### Common Issues

**Issue 1: BGP Session Not Establishing**

**Symptoms**: ExaBGP can't connect to peer

**Check**:
```bash
ping 203.0.113.49  # Can reach peer?
sudo tcpdump -i eth1 port 179  # BGP traffic?
```

**Common causes**:
- Firewall blocking TCP 179
- Wrong peer IP in config
- Peer not configured on ISP side
- AS number mismatch

**Issue 2: Routes Not Being Accepted**

**Check on ISP side**:
```junos
show bgp neighbor 203.0.113.50 statistics
```

**Look for**:
- `Rejected prefixes`: Import policy blocking routes
- `Route limit exceeded`: Prefix limit hit

**Fix**: Adjust import policy on ISP router

**Issue 3: High Memory Usage**

**Check**:
```bash
free -h
ps aux | grep exabgp
```

**Solutions**:
- Reduce route count
- Use prefix aggregation (announce /16s instead of many /24s)
- Add more RAM to VM

---

## Integration with Anchor Tenants

### Combining ExaBGP + Anchor Tenants

**Strategy**:
1. **ExaBGP** announces: Background routes (45.0.0.0/8, 103.0.0.0/8, etc.)
2. **Anchor tenants** announce: Interactive routes (185.10.0.0/16 - CDN, etc.)

**Result**:
- Huge BGP table (50k routes)
- Some routes actually work (anchor tenants)

**Traffic patterns**:
```
Ping 45.10.1.1 (ExaBGP)      → No response (expected)
Ping 185.10.1.1 (CDN anchor) → Response! (real VM)
```

---

## Configuration Summary

### Complete ExaBGP Setup Checklist

- [ ] VM deployed (2GB RAM, 1 vCPU)
- [ ] Network connected to Tier-1 ISP
- [ ] ExaBGP installed
- [ ] Route list generated (50k routes)
- [ ] `/etc/exabgp/exabgp.conf` created
- [ ] `/usr/local/bin/exabgp-announce.py` created
- [ ] Routes file at `/etc/exabgp/routes.txt`
- [ ] Systemd service configured
- [ ] ExaBGP running
- [ ] BGP session established
- [ ] Routes visible on Tier-1
- [ ] Routes propagated to Atlas/PastyNet
- [ ] Monitoring configured

---

## Best Practices

1. **Start small**: Test with 1,000 routes first, then scale up
2. **Use realistic prefixes**: Download from RouteViews for accuracy
3. **Separate fake from real**: ExaBGP for scale, anchors for traffic
4. **Monitor memory**: Watch for OOM kills on large tables
5. **Enable logging**: Debug issues with `journalctl -u exabgp -f`
6. **Test failover**: Stop ExaBGP, verify routes withdraw
7. **Document what's fake**: Tag ExaBGP routes with communities

---

## Next Steps

1. Generate route list (50k routes)
2. Deploy ExaBGP VM
3. Configure BGP session to Level 3
4. Start route injection
5. Verify on Tier-1 and Tier-2 ISPs
6. (Optional) Add route flapping
7. (Optional) Scale to 100k routes
8. Combine with anchor tenants for full realism

---

*This guide demonstrates industrial-scale BGP simulation without requiring 50,000 real routers. Combined with anchor tenants (previous guide), you have a complete realistic internet simulation.*

## VERSION HISTORY

- v1.0 (2026-02-09): Initial ExaBGP route injection implementation guide
