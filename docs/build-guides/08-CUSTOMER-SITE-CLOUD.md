# Cloud Customer Site Configuration Guide

## NOT REVIEWED YET

## Note

All IP addresses and interface names are generic in this guide. You should replace them with your own IP addresses and interface names. See IP_ADDRESSING_PHILOSOPHY.md for more information.

## Overview

This guide walks through configuring a **cloud-based customer site** connected to PastyNet ISP via a secure VPN tunnel. This represents a modern **hybrid cloud** scenario where workloads run in public cloud (OCI/AWS/Azure) but need connectivity to on-premises resources through your ISP network.

### Customer Profile: "AcmeCorp Cloud Production"

```
Customer: AcmeCorp (AS 65002)
Service Type: Hybrid Cloud Transit
ISP: PastyNet (AS 65100)
Location: Oracle Cloud Infrastructure (OCI) or AWS
Connection: WireGuard VPN tunnel
Bandwidth: 1Gbps (tunnel throughput)
Equipment: 1x Cloud Gateway VM (VyOS or strongSwan)
```

### Why This Matters

**Real-world scenario**: Companies are moving to cloud but need secure, reliable connectivity between cloud and on-prem. This setup teaches:
- Cloud networking fundamentals
- VPN technologies (WireGuard vs IPsec)
- BGP over tunnels
- Hybrid cloud architecture
- Cloud security groups and routing

---

## Network Architecture

```
AcmeCorp Cloud Site (AS 65002)
│
├── Cloud-Gateway-1 (OCI/AWS VM running VyOS)
│   ├── Public IP: <cloud-provider-assigned>
│   ├── WireGuard tunnel to PastyNet
│   └── BGP session to PastyNet
│
├── Kubernetes Cluster (K3s on OCI/AWS)
│   └── Pod Network: 10.20.0.0/16
│
└── Other Services
    └── Internal Network: 10.21.0.0/16

Upstream Connection:
PastyNet-Edge-1 (AS 65100) ←─ WireGuard ─→ Cloud-Gateway-1 (AS 65002)
  172.16.0.1/30 (tunnel IP)                    172.16.0.2/30 (tunnel IP)
  100.64.20.1/30 (if direct connect)           100.64.20.2/30 (if direct connect)
```

### Design Decisions

**Why WireGuard instead of IPsec?**
- Simpler configuration (less complexity than IPsec)
- Better performance (modern cryptography)
- Smaller attack surface
- Easier to troubleshoot
- Works well in cloud environments (NAT-friendly)

**Why VPN tunnel instead of Direct Connect/FastConnect?**
- Lower cost (Direct Connect = $300+/month, VPN = free)
- Faster to provision (minutes vs weeks)
- Good enough for lab/learning purposes
- Can upgrade to Direct Connect later

**Platform**: VyOS VM in cloud (lightweight, supports WireGuard and BGP)

---

## Phase 1: Cloud Infrastructure Setup

### Goal
Prepare cloud environment for VPN gateway deployment.

### Step 1: Cloud Network Design (OCI Example)

**Create VCN (Virtual Cloud Network)**:
- CIDR: `10.20.0.0/16`
- Public subnet: `10.20.1.0/24` (for gateway VM)
- Private subnet: `10.20.10.0/24` (for K8s nodes)

**Security Groups**:

**Gateway VM security group**:
```
Ingress Rules:
- UDP 51820 from <PastyNet-public-IP>/32  # WireGuard
- TCP 22 from <your-management-IP>/32      # SSH management
- ICMP from 172.16.0.1/32                   # Tunnel ping

Egress Rules:
- ALL traffic to 0.0.0.0/0                  # Allow outbound
```

**Why restrict WireGuard?**: Only allow from known ISP endpoint for security

### Step 2: Deploy Gateway VM

**Instance specs**:
- Shape: VM.Standard.E4.Flex (2 OCPU, 8GB RAM) or AWS t3.medium
- OS: Ubuntu 22.04 LTS
- Storage: 50GB boot volume
- Network: Public subnet with public IP

**Assign Elastic/Reserved IP** (so it doesn't change on reboot)

**Initial setup**:
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install VyOS (if using VyOS ISO) OR install WireGuard + FRRouting
sudo apt install wireguard-tools frr -y
```

**Alternative: Use VyOS cloud image**:
- Download VyOS cloud image (qcow2 or AMI)
- Deploy as custom image in OCI/AWS
- Much cleaner than installing WireGuard on Ubuntu

---

## Phase 2: WireGuard Tunnel Configuration

### Goal
Establish secure encrypted tunnel between cloud and PastyNet ISP.

### Understanding WireGuard

**How it works**:
1. Each side generates public/private keypair
2. Exchange public keys (like SSH)
3. Configure endpoints and allowed IPs
4. Tunnel automatically established when traffic flows

**Benefits over IPsec**:
- No complex phase 1/phase 2 negotiation
- Roaming support (survives IP changes)
- Always-on (no "tunnel down" states)

### Step 1: Generate WireGuard Keys

**On Cloud-Gateway-1**:

```bash
# Generate keypair
wg genkey | sudo tee /etc/wireguard/private.key
sudo chmod 600 /etc/wireguard/private.key
sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key

# View your public key (share with PastyNet)
cat /etc/wireguard/public.key
```

**On PastyNet-Edge-1** (ISP side):
```bash
# Same process
wg genkey | sudo tee /etc/wireguard/private.key
sudo cat /etc/wireguard/private.key | wg pubkey | sudo tee /etc/wireguard/public.key
```

### Step 2: Configure WireGuard Interface (VyOS)

**On Cloud-Gateway-1** (VyOS):

```bash
set interfaces wireguard wg0 address '172.16.0.2/30'
set interfaces wireguard wg0 description 'Tunnel-to-PastyNet'
set interfaces wireguard wg0 port '51820'
set interfaces wireguard wg0 private-key '<your-private-key-here>'

# Peer configuration (PastyNet side)
set interfaces wireguard wg0 peer <PastyNet-public-key>
set interfaces wireguard wg0 peer <PastyNet-public-key> endpoint '<PastyNet-public-IP>:51820'
set interfaces wireguard wg0 peer <PastyNet-public-key> allowed-ips '0.0.0.0/0'
set interfaces wireguard wg0 peer <PastyNet-public-key> persistent-keepalive '25'
```

**Key settings explained**:
- `address '172.16.0.2/30'`: Tunnel IP (not routable on internet)
- `allowed-ips '0.0.0.0/0'`: Accept all traffic through tunnel
- `persistent-keepalive '25'`: Send keepalive every 25 seconds (keeps NAT alive)

**On PastyNet-Edge-1** (ISP side, already shown in guide 06):

```bash
set interfaces wireguard wg0 address '172.16.0.1/30'
set interfaces wireguard wg0 peer <Cloud-public-key>
set interfaces wireguard wg0 peer <Cloud-public-key> endpoint '<Cloud-public-IP>:51820'
set interfaces wireguard wg0 peer <Cloud-public-key> allowed-ips '0.0.0.0/0'
```

**Note**: ISP doesn't need keepalive (they're not behind NAT)

### Step 3: Verify Tunnel

```bash
# Check WireGuard status
sudo wg show

# Ping through tunnel
ping 172.16.0.1 source-address 172.16.0.2

# Check interface
show interfaces wireguard
```

**Expected output**:
```
interface: wg0
  public key: <your-public-key>
  private key: (hidden)
  listening port: 51820

peer: <PastyNet-public-key>
  endpoint: <PastyNet-IP>:51820
  allowed ips: 0.0.0.0/0
  latest handshake: 45 seconds ago
  transfer: 2.5 KiB received, 3.1 KiB sent
```

**Troubleshooting**:
- No handshake? Check security groups (UDP 51820 blocked?)
- Ping fails? Check allowed-ips configuration
- Endpoint wrong? Verify public IPs on both sides

---

## Phase 3: BGP Over Tunnel

### Goal
Establish BGP session through WireGuard tunnel to exchange routes.

### Why BGP Over Tunnel?

**Alternatives**:
- Static routes (simple but not dynamic)
- BGP over public internet (not secure)
- Cloud-native routing (vendor lock-in)

**BGP over VPN**:
- Dynamic routing (automatic failover)
- Secure (encrypted tunnel)
- Standard protocol (no vendor lock-in)
- Scales to multiple sites

### Step 1: Basic BGP Configuration

**On Cloud-Gateway-1**:

```bash
set protocols bgp system-as '65002'
set protocols bgp parameters router-id '10.21.0.1'  # Use loopback or internal IP

# BGP neighbor to PastyNet (over tunnel)
set protocols bgp neighbor 172.16.0.1 remote-as '65100'
set protocols bgp neighbor 172.16.0.1 description 'PastyNet-via-Tunnel'
set protocols bgp neighbor 172.16.0.1 address-family ipv4-unicast
set protocols bgp neighbor 172.16.0.1 address-family ipv4-unicast route-map export 'TO-PASTYNET'
set protocols bgp neighbor 172.16.0.1 address-family ipv4-unicast route-map import 'FROM-PASTYNET'
```

### Step 2: Announce Cloud Networks

**What to announce**:
- K8s pod network: `10.20.0.0/16`
- Cloud internal network: `10.21.0.0/16`

```bash
# Create prefix list of cloud networks
set policy prefix-list CLOUD-NETWORKS rule 10 action 'permit'
set policy prefix-list CLOUD-NETWORKS rule 10 prefix '10.20.0.0/16'

set policy prefix-list CLOUD-NETWORKS rule 20 action 'permit'
set policy prefix-list CLOUD-NETWORKS rule 20 prefix '10.21.0.0/16'

# Export route-map
set policy route-map TO-PASTYNET rule 10 action 'permit'
set policy route-map TO-PASTYNET rule 10 match ip address prefix-list 'CLOUD-NETWORKS'

set policy route-map TO-PASTYNET rule 999 action 'deny'
```

**Why prefix list?**: Only announce your networks, not learned routes (prevents routing loops)

### Step 3: Accept Routes from ISP

**Simple approach** (accept default route):

```bash
# Accept default route from ISP
set policy prefix-list DEFAULT-ROUTE rule 10 action 'permit'
set policy prefix-list DEFAULT-ROUTE rule 10 prefix '0.0.0.0/0'

set policy route-map FROM-PASTYNET rule 10 action 'permit'
set policy route-map FROM-PASTYNET rule 10 match ip address prefix-list 'DEFAULT-ROUTE'
```

**Advanced approach** (accept specific on-prem networks):

```bash
# Accept on-prem networks (for hybrid cloud connectivity)
set policy prefix-list ONPREM-NETWORKS rule 10 action 'permit'
set policy prefix-list ONPREM-NETWORKS rule 10 prefix '10.10.0.0/16'  # On-prem site

set policy route-map FROM-PASTYNET rule 10 action 'permit'
set policy route-map FROM-PASTYNET rule 10 match ip address prefix-list 'ONPREM-NETWORKS'

# Also accept default route
set policy route-map FROM-PASTYNET rule 20 action 'permit'
set policy route-map FROM-PASTYNET rule 20 match ip address prefix-list 'DEFAULT-ROUTE'
```

### Verification

```bash
show bgp summary
show ip route bgp
ping 8.8.8.8 source-address 10.21.0.1
```

**Expected**:
- BGP session: `Established` with 172.16.0.1
- Routes learned: Default route (0.0.0.0/0) from PastyNet
- Routes announced: Your cloud networks (10.20.0.0/16, 10.21.0.0/16)

---

## Phase 4: Cloud Network Integration

### Goal
Connect K8s cluster and other cloud services to the VPN gateway.

### Step 1: Cloud Route Tables (OCI Example)

**VCN Route Table** (for private subnet where K8s runs):

```
Destination         Target                      Description
0.0.0.0/0          Internet Gateway            Default internet
10.10.0.0/16       Cloud-Gateway-1 (10.20.1.10) On-prem network via VPN
```

**How to configure** (OCI Console):
1. Navigate to VCN → Route Tables
2. Edit route table for private subnet
3. Add route: `10.10.0.0/16` → Private IP of Cloud-Gateway-1
4. Save changes

**AWS equivalent**:
```bash
# Using AWS CLI
aws ec2 create-route \
  --route-table-id rtb-xxxxx \
  --destination-cidr-block 10.10.0.0/16 \
  --instance-id i-xxxxx  # Cloud-Gateway-1 instance
```

### Step 2: Disable Source/Destination Check (AWS)

**AWS only** (OCI doesn't need this):

```bash
aws ec2 modify-instance-attribute \
  --instance-id i-xxxxx \
  --no-source-dest-check
```

**Why?**: Allows gateway VM to forward traffic for other IPs

### Step 3: K8s Integration

**For K3s cluster** (most common in cloud):

```yaml
# K3s doesn't need special config - just uses host routing

# Verify K8s nodes can reach on-prem
kubectl run -it --rm debug --image=busybox --restart=Never -- ping 10.10.1.1
```

**For managed Kubernetes** (OKE, EKS):
- Pods automatically use VCN routing
- No special configuration needed
- Traffic to 10.10.0.0/16 goes via Cloud-Gateway-1

---

## Phase 5: High Availability (Optional)

### Goal
Add redundancy to prevent single point of failure.

### Option 1: Dual Gateway VMs

**Architecture**:
```
PastyNet-Edge-1 ←─ WireGuard ─→ Cloud-Gateway-1 (primary)
PastyNet-Edge-2 ←─ WireGuard ─→ Cloud-Gateway-2 (backup)
```

**Configuration**:
```bash
# On Cloud-Gateway-2
set interfaces wireguard wg0 address '172.16.0.6/30'  # Different tunnel IP
set interfaces wireguard wg0 peer <PastyNet-Edge-2-public-key>
set interfaces wireguard wg0 peer <PastyNet-Edge-2-public-key> endpoint '<PastyNet-Edge-2-IP>:51820'

# BGP with lower preference (backup path)
set protocols bgp neighbor 172.16.0.5 remote-as '65100'
set protocols bgp neighbor 172.16.0.5 description 'PastyNet-Backup'
```

**Route table**:
- Primary: Cloud-Gateway-1 (higher BGP local-pref)
- Backup: Cloud-Gateway-2 (lower local-pref)

**Failover**:
- If Gateway-1 fails, BGP withdraws routes
- Gateway-2 routes become active
- ~1-3 minute failover time

### Option 2: VRRP (Virtual Router Redundancy)

**Single tunnel IP shared between two VMs**:
- Cloud-Gateway-1 and Gateway-2 share virtual IP
- VRRP elects active gateway
- Faster failover (<10 seconds)
- More complex configuration

---

## Phase 6: Security Hardening

### Goal
Protect cloud gateway from attacks.

### Step 1: Firewall Rules

**On Cloud-Gateway-1** (VyOS firewall):

```bash
# Define zones
set zone-policy zone TUNNEL interface 'wg0'
set zone-policy zone CLOUD interface 'eth1'  # Cloud internal network
set zone-policy zone LOCAL local-zone

# Allow tunnel to cloud
set firewall name TUNNEL-TO-CLOUD rule 10 action 'accept'
set firewall name TUNNEL-TO-CLOUD rule 10 state established 'enable'
set firewall name TUNNEL-TO-CLOUD rule 10 state related 'enable'

# Allow specific services from cloud to tunnel
set firewall name CLOUD-TO-TUNNEL rule 10 action 'accept'
set firewall name CLOUD-TO-TUNNEL rule 10 destination address '10.10.0.0/16'  # On-prem only

# Protect router itself
set firewall name TUNNEL-TO-LOCAL rule 10 action 'accept'
set firewall name TUNNEL-TO-LOCAL rule 10 protocol 'tcp'
set firewall name TUNNEL-TO-LOCAL rule 10 destination port '179'  # BGP

set firewall name TUNNEL-TO-LOCAL rule 20 action 'accept'
set firewall name TUNNEL-TO-LOCAL rule 20 protocol 'icmp'

set firewall name TUNNEL-TO-LOCAL rule 999 action 'drop'
set firewall name TUNNEL-TO-LOCAL rule 999 log 'enable'

# Apply zones
set zone-policy zone CLOUD from TUNNEL firewall name 'TUNNEL-TO-CLOUD'
set zone-policy zone TUNNEL from CLOUD firewall name 'CLOUD-TO-TUNNEL'
set zone-policy zone LOCAL from TUNNEL firewall name 'TUNNEL-TO-LOCAL'
```

### Step 2: Cloud Security Groups

**Tighten security group on gateway VM**:

```
Ingress:
- UDP 51820 from <PastyNet-public-IP>/32 ONLY
- TCP 22 from <management-IP>/32 ONLY
- NO other inbound traffic

Egress:
- Allow all (or restrict to specific destinations)
```

**K8s worker security group**:

```
Ingress:
- Allow from Cloud-Gateway-1 private IP
- Allow pod-to-pod (within VCN)
- NO direct internet access

Egress:
- Through Cloud-Gateway-1 (via route table)
```

### Step 3: Monitoring and Alerting

**WireGuard tunnel monitoring**:

```bash
#!/bin/bash
# /usr/local/bin/check-wireguard.sh

# Check if tunnel is up
if ! wg show wg0 | grep -q "latest handshake"; then
  echo "WireGuard tunnel down!" | mail -s "ALERT: Tunnel Down" admin@example.com
  exit 1
fi

# Check tunnel age (should be < 3 minutes)
HANDSHAKE_AGE=$(wg show wg0 latest-handshakes | awk '{print $2}')
CURRENT_TIME=$(date +%s)
AGE=$((CURRENT_TIME - HANDSHAKE_AGE))

if [ $AGE -gt 180 ]; then
  echo "WireGuard tunnel stale! Last handshake: ${AGE}s ago"
  exit 1
fi
```

**Add to cron**:
```bash
*/5 * * * * /usr/local/bin/check-wireguard.sh
```

---

## Common Issues and Troubleshooting

### Issue 1: Tunnel Not Establishing

**Symptoms**: No "latest handshake" in `wg show`

**Check**:
```bash
sudo wg show wg0
ping 172.16.0.1
```

**Common causes**:
- Security group blocking UDP 51820
- Wrong public keys exchanged
- Endpoint IP incorrect
- Firewall on cloud VM blocking WireGuard
- NAT not allowing UDP through

**Fix**:
```bash
# Verify security group
# OCI Console → Instances → Cloud-Gateway-1 → Security Lists

# Test UDP connectivity (from PastyNet side)
nc -u <cloud-public-IP> 51820

# Check local firewall
sudo ufw status
sudo ufw allow 51820/udp
```

### Issue 2: Tunnel Up But BGP Not Establishing

**Check**:
```bash
show bgp summary
ping 172.16.0.1
telnet 172.16.0.1 179
```

**Common causes**:
- Firewall blocking TCP 179 over tunnel
- Wrong AS numbers
- BGP not configured on other side

### Issue 3: K8s Pods Can't Reach On-Prem

**Check**:
```bash
# From K8s pod
kubectl run -it --rm debug --image=busybox --restart=Never -- traceroute 10.10.1.1

# Check route table
show ip route 10.10.1.1
```

**Common causes**:
- VCN route table not configured
- Cloud-Gateway-1 not advertising routes via BGP
- Security group blocking traffic from K8s subnet

---

## Configuration Summary

### Complete Cloud-Gateway-1 Overview

```bash
# System
set system host-name Cloud-Gateway-1

# Interfaces
set interfaces ethernet eth0 address '10.20.1.10/24'  # Cloud internal
set interfaces wireguard wg0 address '172.16.0.2/30'  # Tunnel to PastyNet
set interfaces loopback lo address '10.21.0.1/32'

# WireGuard
set interfaces wireguard wg0 port '51820'
set interfaces wireguard wg0 private-key '<private-key>'
set interfaces wireguard wg0 peer <PastyNet-pub-key> endpoint '<PastyNet-IP>:51820'
set interfaces wireguard wg0 peer <PastyNet-pub-key> allowed-ips '0.0.0.0/0'
set interfaces wireguard wg0 peer <PastyNet-pub-key> persistent-keepalive '25'

# BGP
set protocols bgp system-as '65002'
set protocols bgp parameters router-id '10.21.0.1'
set protocols bgp neighbor 172.16.0.1 remote-as '65100'
set protocols bgp neighbor 172.16.0.1 description 'PastyNet-ISP'

# Announce cloud networks
set protocols bgp address-family ipv4-unicast network '10.20.0.0/16'
set protocols bgp address-family ipv4-unicast network '10.21.0.0/16'
```

---

## Best Practices

1. **Use reserved/elastic IPs**: Don't let cloud public IPs change
2. **Enable keepalives**: Essential for NAT traversal (cloud side)
3. **Monitor tunnel health**: Alert on handshake failures
4. **Restrict security groups**: Only allow WireGuard from ISP IP
5. **Use BGP for routing**: Don't rely on static routes
6. **Plan IP addressing**: Avoid overlaps between cloud and on-prem
7. **Test failover**: Know what happens when tunnel fails
8. **Document everything**: Keys, IPs, configurations

---

## Traffic Flow Examples

### Example 1: K8s Pod to On-Prem Database

```
K8s Pod (10.20.10.50) → Node (10.20.10.5) → VCN Route (to Gateway) →
Cloud-Gateway-1 (10.20.1.10) → WireGuard tunnel → PastyNet-Edge-1 →
Atlas ISP → On-Prem Customer (10.10.2.10 - database)
```

### Example 2: On-Prem User to Cloud API

```
On-Prem User (10.10.1.100) → Acme-Edge-1 → Atlas ISP → PastyNet-Edge-1 →
WireGuard tunnel → Cloud-Gateway-1 → K8s Service (10.20.10.80)
```

### Example 3: Cloud VM to Internet

```
Cloud VM (10.20.10.30) → Internet Gateway (direct) OR
Cloud VM (10.20.10.30) → Cloud-Gateway-1 → PastyNet → Internet (if forced through VPN)
```

---

## Next Steps

1. Deploy gateway VM in cloud
2. Configure WireGuard tunnel
3. Establish BGP session
4. Update cloud route tables
5. Test connectivity from K8s pods
6. Add monitoring and alerting
7. (Optional) Implement HA with second gateway

---

## Integration with Project Atlas

**This cloud site connects to**:
- PastyNet-Edge-1 (AS 65100) via WireGuard tunnel
- Receives routes to on-prem networks
- Announces cloud networks (10.20.0.0/16, 10.21.0.0/16)

**In your lab topology**:
```
Cloud K8s → Cloud-Gateway-1 ←─ Tunnel ─→ PastyNet → Atlas ISP → On-Prem
```

**Multi-site routing**:
```
Cloud Site (AS 65002) ←→ PastyNet (AS 65100) ←→ IXP ←→ Atlas ISP (AS 65000) ←→ On-Prem Site (AS 65001)
```

**Full path example**:
```
Cloud Pod → PastyNet → IXP → Atlas ISP → On-Prem Customer
(10.20.10.50)                             (10.10.1.100)
```

---

*This guide demonstrates modern hybrid cloud connectivity using VPN tunnels and BGP. It's the cloud counterpart to the on-premises customer site, showing how multi-site enterprises connect their infrastructure.*

## VERSION HISTORY

- v1.0 (2026-02-09): Initial cloud customer site configuration guide with WireGuard/BGP
