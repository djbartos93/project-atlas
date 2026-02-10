# Cloud and On-Premises Interconnect Guide

## NOT REVIEWED YET

## Note

All IP addresses and interface names are generic in this guide. You should replace them with your own IP addresses and interface names. See IP_ADDRESSING_PHILOSOPHY.md for more information.

## Overview

This high-level guide explains how to interconnect your **EVE-NG lab network** with **real-world cloud services** (OCI, AWS, Azure) and **on-premises infrastructure**. It covers the architecture decisions, security considerations, and practical implementation strategies for creating a true hybrid cloud lab.

### What This Achieves

```
Real World:                         Lab Environment:
┌──────────────┐                   ┌──────────────────────┐
│ OCI/AWS Cloud│◄───VPN Tunnel────►│ EVE-NG (PastyNet)   │
│   K3s Cluster│                   │  ├─ Tier-1 ISPs     │
│   n8n        │                   │  ├─ Tier-2 ISPs     │
│   Services   │                   │  └─ Customer Sites  │
└──────────────┘                   └──────────────────────┘
       ▲                                     ▲
       │                                     │
       └──────────Transit through ISP lab───┘
```

**Key benefit**: Cloud services must traverse your entire simulated internet to reach on-prem resources, creating realistic traffic patterns and testing scenarios.

---

## Architecture Overview

### Three Connection Points

**1. Cloud → PastyNet ISP → Lab**
- Cloud workloads (K8s, n8n, etc.) connect via VPN tunnel to PastyNet
- Traffic transits through simulated ISPs
- Most realistic option

**2. EVE-NG → Physical Network → Cloud**
- EVE-NG VM bridges to your physical LAN
- Direct routing to cloud (bypasses ISP simulation)
- Simpler but less realistic

**3. Hybrid Approach** (Recommended)
- Critical services use direct connection
- Test traffic goes through ISP simulation
- Best of both worlds

---

## Option 1: Cloud Traffic Through ISP Simulation (Recommended)

### Goal
Force all cloud traffic to transit through your simulated Tier-1/Tier-2 ISPs.

### Architecture

```
Oracle Cloud (OCI)
 └─ K3s Cluster (10.20.0.0/16)
     └─ VPN Gateway VM (running VyOS)
         └─ WireGuard tunnel to PastyNet-Edge-1
             └─ BGP session (AS 65002 ↔ AS 65100)
                 └─ Traffic flows through:
                     PastyNet → IXP → Atlas ISP → On-prem

On-Premises
 └─ Proxmox Cluster
     └─ EVE-NG VM
         └─ PastyNet ISP (simulated)
             └─ Peer with cloud gateway
```

### Implementation Steps

**Step 1: Deploy Cloud Gateway**

See **08-CUSTOMER-SITE-CLOUD.md** for detailed setup:
- Deploy VyOS VM in OCI/AWS
- Configure WireGuard tunnel
- Establish BGP session
- Update cloud route tables

**Step 2: Configure PastyNet to Accept Cloud Customer**

Already covered in **06-PASTYNET-ISP-BUILD.md** (Phase 5-6):
- WireGuard tunnel configuration
- BGP peer setup
- Route policies

**Step 3: Bridge EVE-NG to Physical Network**

**On Proxmox host**:

```bash
# Create bridge for EVE-NG external connectivity
cat >> /etc/network/interfaces <<EOF

# Bridge for EVE-NG → Physical Network
auto vmbr1
iface vmbr1 inet static
    address 192.168.100.1/24
    bridge-ports eno1  # Your physical NIC
    bridge-stp off
    bridge-fd 0
EOF

systemctl restart networking
```

**In EVE-NG**:
- Add network interface to PastyNet routers
- Configure interface with public IP (from your home/lab network)
- This becomes WireGuard tunnel endpoint

**Step 4: Configure WireGuard Endpoint**

**On PastyNet-Edge-1** (inside EVE-NG):

```bash
# External interface (connects to physical network)
set interfaces ethernet eth5 address '192.168.100.35/24'
set interfaces ethernet eth5 description 'External-to-Cloud'

# WireGuard tunnel to OCI
set interfaces wireguard wg0 address '172.16.0.1/30'
set interfaces wireguard wg0 port '51820'
set interfaces wireguard wg0 private-key '<private-key>'
set interfaces wireguard wg0 peer <OCI-public-key> endpoint '<OCI-public-IP>:51820'
set interfaces wireguard wg0 peer <OCI-public-key> allowed-ips '0.0.0.0/0'

# BGP over tunnel (already configured in guide 06)
set protocols bgp neighbor 172.16.0.2 remote-as '65002'
```

**Why this works**:
- PastyNet router has physical IP (192.168.100.35)
- Cloud can reach this IP over internet
- Tunnel terminates at simulated ISP router
- Traffic flows through entire simulated network

### Traffic Flow Example

**Cloud K8s pod calls on-prem database**:

```
1. K8s Pod (10.20.10.50) → Cloud VCN routing
2. Cloud-Gateway-1 (10.20.1.10) → WireGuard tunnel
3. PastyNet-Edge-1 (172.16.0.1) → receives packet
4. PastyNet → BGP routing → peer with Atlas at IXP
5. Atlas ISP → routes to on-prem customer
6. On-Prem Customer router (Acme-Edge-1)
7. Internal network (10.10.2.10 - database)
8. Response follows reverse path
```

**Total hops**: ~6-8 routers (realistic!)

### Benefits

- ✅ Fully realistic traffic patterns
- ✅ Tests entire ISP routing infrastructure
- ✅ BGP path selection in action
- ✅ Can simulate outages (disable Tier-1 peer)
- ✅ Traffic engineering demonstrations

### Drawbacks

- ❌ More complex setup
- ❌ Higher latency (more hops)
- ❌ Depends on EVE-NG being running
- ❌ Requires public IP or port forwarding

---

## Option 2: Direct Cloud Connectivity (Simpler)

### Goal
Connect cloud services directly to EVE-NG management network.

### Architecture

```
Oracle Cloud (OCI)
 └─ VyOS Gateway VM
     └─ VPN tunnel to Proxmox host (or physical firewall)
         └─ Direct routing to EVE-NG management network
             └─ Access to lab routers' management IPs
```

### Implementation

**Step 1: VPN from Cloud to Proxmox**

**Option A: WireGuard to Proxmox Host**:

```bash
# On Proxmox host
apt install wireguard -y

# Create tunnel
cat > /etc/wireguard/wg0.conf <<EOF
[Interface]
Address = 172.16.1.1/30
PrivateKey = <proxmox-private-key>
ListenPort = 51820

[Peer]
PublicKey = <oci-public-key>
Endpoint = <oci-public-ip>:51820
AllowedIPs = 10.20.0.0/16, 10.21.0.0/16
PersistentKeepalive = 25
EOF

# Start tunnel
wg-quick up wg0
systemctl enable wg-quick@wg0
```

**On OCI Gateway**:

```bash
# Same process but reverse IPs
[Interface]
Address = 172.16.1.2/30
...

[Peer]
PublicKey = <proxmox-public-key>
Endpoint = <your-home-ip>:51820  # May need port forward
AllowedIPs = 192.168.100.0/24
```

**Option B: OpenVPN** (if your router has built-in support):

```bash
# On home router/firewall with OpenVPN
# Configure OpenVPN server
# Give cloud gateway access to 192.168.100.0/24
```

**Step 2: Routing**

**On cloud side**:
```bash
# Route lab networks through tunnel
ip route add 192.168.100.0/24 via 172.16.1.1
```

**On Proxmox side**:
```bash
# Route cloud networks through tunnel
ip route add 10.20.0.0/16 via 172.16.1.2
```

### Traffic Flow Example

**Cloud service calls EVE-NG router management IP**:

```
1. Cloud K8s pod (10.20.10.50)
2. Cloud-Gateway-1 → VPN tunnel
3. Proxmox host (172.16.1.1)
4. Bridge to EVE-NG management network
5. EVE-NG router (192.168.100.30 - Atlas-Core-1)
```

**Result**: Direct access to lab routers, bypasses ISP simulation

### Benefits

- ✅ Simple to configure
- ✅ Low latency
- ✅ Doesn't depend on EVE-NG internal routing
- ✅ Good for management access

### Drawbacks

- ❌ Not realistic (no ISP transit)
- ❌ Doesn't test BGP routing
- ❌ Limited learning value for ISP concepts

---

## Option 3: Hybrid Approach (Best of Both)

### Goal
Use both direct and simulated paths for different purposes.

### Architecture

```
Cloud Services:
 ├─ Critical Management Traffic → Direct VPN to Proxmox
 │   (SSH, monitoring, backups)
 │
 └─ Application Traffic → Through ISP simulation
     (K8s services, n8n workflows, testing)
```

### Implementation

**Step 1: Two VPN Tunnels**

**Tunnel 1: Management (Direct)**
- Cloud → Proxmox host
- Used for: SSH, monitoring, Netbox, backups
- Route: `192.168.100.0/24` (management network)

**Tunnel 2: Application (Simulated)**
- Cloud → PastyNet in EVE-NG
- Used for: K8s inter-cluster, n8n workflows, testing
- Route: `10.10.0.0/16` (on-prem customer networks)

**Step 2: Policy-Based Routing on Cloud Gateway**

```bash
# On Cloud-Gateway-1 (VyOS)

# Mark management traffic
set firewall ipv4 name MARK-MGMT rule 10 action 'accept'
set firewall ipv4 name MARK-MGMT rule 10 destination address '192.168.100.0/24'
set firewall ipv4 name MARK-MGMT rule 10 set mark '100'

# Routing tables
set protocols static table 100 route 0.0.0.0/0 next-hop 172.16.1.1  # Direct tunnel
set protocols static route 0.0.0.0/0 next-hop 172.16.0.1             # Simulated tunnel

# Apply policy routing
set interfaces ethernet eth1 firewall in name 'MARK-MGMT'
set policy route TABLE-100 rule 10 mark '100'
set policy route TABLE-100 rule 10 set table '100'
```

**Result**:
- Management traffic (to 192.168.100.0/24) uses direct tunnel
- Everything else uses simulated ISP path

### Benefits

- ✅ Reliable management access
- ✅ Realistic application traffic flow
- ✅ Best of both worlds
- ✅ Flexibility for testing

### Complexity

- ⚠️ Requires careful routing configuration
- ⚠️ Two tunnels to maintain
- ⚠️ Policy routing can be tricky

---

## Security Considerations

### Protecting Your Lab

**Firewall Rules** (on PastyNet or cloud gateway):

```bash
# Only allow specific cloud IPs through tunnel
set firewall name CLOUD-TO-LAB rule 10 action 'accept'
set firewall name CLOUD-TO-LAB rule 10 source address '10.20.0.0/16'
set firewall name CLOUD-TO-LAB rule 10 destination address '10.10.0.0/16'

set firewall name CLOUD-TO-LAB rule 999 action 'drop'
set firewall name CLOUD-TO-LAB rule 999 log 'enable'
```

**Cloud Security Groups**:

```
Inbound Rules (Cloud Gateway VM):
- UDP 51820 from lab-public-IP ONLY (WireGuard)
- TCP 22 from your-mgmt-IP ONLY (SSH)
- DROP all other inbound

Outbound Rules:
- Allow to lab networks (10.10.0.0/16, 192.168.100.0/24)
- Allow to internet (for software updates)
```

**Never expose**:
- EVE-NG management interface to internet
- Lab router telnet/SSH to internet
- Simulated ISP routers to real internet

---

## NAT Considerations

### Port Forwarding for Home Labs

**If you're behind NAT** (home internet):

**On your home router**:
```
Port Forward Rule:
  External Port: 51820 (UDP)
  Internal IP: 192.168.100.1 (Proxmox host or EVE-NG external IP)
  Internal Port: 51820 (UDP)
  Protocol: UDP
```

**Dynamic DNS** (if your IP changes):

```bash
# Use a DDNS service (NoIP, DuckDNS, etc.)
# Update WireGuard endpoint to use hostname
set interfaces wireguard wg0 peer <cloud-key> endpoint 'yourlab.ddns.net:51820'
```

### Avoiding NAT Complexity

**Alternative**: Use cloud as the WireGuard server:
- Cloud has static public IP (always reachable)
- Lab initiates tunnel (works through NAT)
- More reliable for home labs

**Config change**:
```bash
# On Proxmox/PastyNet (client)
[Peer]
Endpoint = <cloud-static-ip>:51820
PersistentKeepalive = 25  # Keep NAT mapping alive
```

---

## Multi-Site Scenarios

### Connecting Multiple Cloud Regions

**Scenario**: OCI in Phoenix + AWS in Virginia

```
OCI Phoenix (10.20.0.0/16)
 └─ VPN to PastyNet (AS 65002)

AWS Virginia (10.30.0.0/16)
 └─ VPN to Atlas ISP (AS 65003)

Traffic between clouds:
OCI → PastyNet → IXP → Atlas → AWS
```

**Why route between clouds through your lab?**
- Test multi-region connectivity
- Demonstrate BGP routing between sites
- Simulate real-world hybrid cloud

**BGP configuration**:

```bash
# PastyNet announces OCI routes to IXP
set policy route-map TO-IXP rule 10 action 'permit'
set policy route-map TO-IXP rule 10 match ip address prefix-list 'OCI-NETWORKS'

# Atlas announces AWS routes to IXP
set policy route-map TO-IXP rule 10 action 'permit'
set policy route-map TO-IXP rule 10 match ip address prefix-list 'AWS-NETWORKS'

# Result: OCI ↔ AWS traffic via IXP peering
```

---

## Monitoring and Observability

### Tracking Cloud-to-Lab Traffic

**NetFlow/sFlow** (on PastyNet routers):

```bash
# Enable NetFlow on WireGuard interface
set system flow-accounting interface 'wg0'
set system flow-accounting netflow server 192.168.100.100 port 2055
```

**Grafana Dashboard**:
- Traffic volume by source/destination
- Latency between cloud and lab
- Tunnel up/down events

**Alerts**:
```
Alert: Tunnel Down
  Condition: No WireGuard handshake in 5 minutes
  Action: Email + PagerDuty

Alert: High Latency
  Condition: Ping >100ms for 10 minutes
  Action: Log for investigation
```

---

## Testing Scenarios

### Use Cases for Cloud-Lab Interconnect

**1. Multi-Cluster Kubernetes**
- K3s in cloud + K3s on-prem
- Services call each other across VPN
- Tests: Service mesh, latency, failure scenarios

**2. Hybrid Database Replication**
- PostgreSQL primary in cloud
- Replica in on-prem lab
- Traffic flows through ISP simulation

**3. n8n Workflows**
- n8n in cloud triggers on-prem automation
- Tests: API calls across VPN, error handling

**4. Backup and DR**
- Backup cloud data to on-prem storage
- Tests: Bandwidth, encryption, restore

**5. BGP Failover Testing**
- Disable Tier-1 peer
- Watch cloud traffic reroute
- Measure convergence time

---

## Troubleshooting

### Common Issues

**Issue 1: Tunnel Won't Establish**

**Check**:
```bash
# On both sides
sudo wg show
ping <tunnel-peer-ip>

# Check firewall
sudo iptables -L -n | grep 51820
```

**Common causes**:
- Port forwarding not configured
- Security group blocking UDP 51820
- Wrong public keys
- Clock skew (WireGuard requires accurate time)

**Issue 2: Tunnel Up But No Traffic**

**Check**:
```bash
# Routing
ip route
traceroute 10.10.1.1

# BGP (if using BGP over tunnel)
show bgp summary
show ip route bgp
```

**Common causes**:
- Routes not configured
- BGP not established
- Firewall blocking traffic over tunnel

**Issue 3: Intermittent Connectivity**

**Check**:
```bash
# Handshake freshness
sudo wg show wg0 latest-handshakes

# MTU issues
ping -M do -s 1400 <peer-ip>
```

**Common causes**:
- NAT timeout (add PersistentKeepalive)
- MTU mismatch (reduce MTU to 1400)
- Flapping route in BGP

---

## Configuration Examples

### Complete Cloud-to-Lab Setup

**Cloud Gateway (OCI)**:
```bash
# WireGuard to PastyNet
set interfaces wireguard wg0 address '172.16.0.2/30'
set interfaces wireguard wg0 peer <pastynet-key> endpoint '203.0.113.35:51820'
set interfaces wireguard wg0 peer <pastynet-key> allowed-ips '0.0.0.0/0'
set interfaces wireguard wg0 peer <pastynet-key> persistent-keepalive '25'

# BGP
set protocols bgp neighbor 172.16.0.1 remote-as '65100'

# Announce cloud networks
set protocols bgp address-family ipv4-unicast network '10.20.0.0/16'
```

**PastyNet-Edge-1 (EVE-NG)**:
```bash
# External interface (to physical network)
set interfaces ethernet eth5 address '192.168.100.35/24'

# WireGuard
set interfaces wireguard wg0 address '172.16.0.1/30'
set interfaces wireguard wg0 port '51820'
set interfaces wireguard wg0 peer <cloud-key> allowed-ips '0.0.0.0/0'

# BGP
set protocols bgp neighbor 172.16.0.2 remote-as '65002'

# Announce on-prem networks via BGP to cloud
set protocols bgp neighbor 172.16.0.2 address-family ipv4-unicast route-map export 'TO-CLOUD'
```

---

## Best Practices

1. **Use separate AS numbers**: Cloud = AS 65002, don't reuse ISP ASNs
2. **Implement route filters**: Don't announce internet routes to cloud
3. **Monitor tunnel health**: Alert on handshake failures
4. **Test failover regularly**: What happens when tunnel fails?
5. **Document IP addressing**: Avoid overlaps between cloud and lab
6. **Use persistent keepalive**: Essential for NAT traversal
7. **Consider MTU**: Set to 1420 for WireGuard (1500 - 80 byte overhead)
8. **Encrypt everything**: Even in lab, practice good security

---

## Next Steps

1. Decide on architecture (Option 1, 2, or 3)
2. Deploy cloud gateway VM
3. Configure WireGuard tunnels
4. Set up BGP (if using Option 1)
5. Update route tables
6. Test connectivity
7. Add monitoring
8. Document your setup

---

## Integration with Project Atlas

**Complete topology with cloud**:

```
Oracle Cloud (AS 65002) ←─ VPN ─→ PastyNet (AS 65100)
                                      ↓
                                    IXP (AS 64999)
                                      ↓
                          ┌───────────┴───────────┐
                          ↓                       ↓
                    Level 3 (AS 3356)      Cogent (AS 174)
                          ↓                       ↓
                          └───────────┬───────────┘
                                      ↓
                            Atlas ISP (AS 65000)
                                      ↓
                          On-Prem Customer (AS 65001)
```

**Traffic flows**:
- Cloud → On-Prem: Through entire simulated internet!
- Realistic latency, routing, failover scenarios

---

*This guide provides the foundation for connecting your lab to the real world. Choose the architecture that fits your learning goals and infrastructure constraints.*

## VERSION HISTORY

- v1.0 (2026-02-09): Initial cloud/on-prem interconnect guide
