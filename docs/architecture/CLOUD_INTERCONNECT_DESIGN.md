# Cloud Interconnect Strategy - Project Atlas

**Challenge**: How to connect OCI/AWS/Azure/GCP back to your "Second ISP" in EVE-NG?

**Goal**: Make it realistic, teachable, and practical

---

## 🎯 Options Overview

### Option 1: VPN Tunnel (Recommended Start)

**Concept**: IPsec or WireGuard tunnel from cloud back to Second ISP

**Architecture**:
```
OCI VM (Cloud-CE router - VyOS)
   │
   │ WireGuard/IPsec tunnel over internet
   │ (encrypted)
   │
   ↓
Second ISP Border Router (Cloud-BR-1)
   │
   └─ Appears as "direct connection" to Second ISP
   │
   └─ BGP session over tunnel
```

**Implementation**:
```yaml
OCI Side:
  - Compute instance: VyOS router (ARM free tier)
  - Public IP: OCI provides
  - Install: WireGuard or strongSwan (IPsec)
  - BGP: via tunnel to Second ISP

Lab Side (EVE-NG):
  - Cloud-BR-1 router
  - Public IP: Your AT&T /29 (via passthrough)
  - WireGuard/IPsec endpoint
  - BGP peer with OCI
```

**Pros**:
- ✅ Free (uses OCI free tier)
- ✅ Simple to implement
- ✅ Realistic (this is how SD-WAN works)
- ✅ Encrypted (security best practice)
- ✅ Can run BGP over tunnel
- ✅ Full control

**Cons**:
- ⚠️ Not native cloud interconnect (but realistic!)
- ⚠️ Latency of internet path
- ⚠️ Bandwidth limited by your home upload speed

**Learning Value**:
- VPN technologies (IPsec/WireGuard)
- BGP over tunnels
- SD-WAN concepts
- Encryption and security

---

### Option 2: GRE + BGP (Traditional Service Provider)

**Concept**: GRE tunnel with BGP (what ISPs actually use)

**Architecture**:
```
OCI VM
   │
   │ GRE tunnel (unencrypted, but over internet)
   │ BGP runs inside GRE
   │
   ↓
Second ISP
```

**Why GRE?**
- This is what real ISPs use
- Simple, lightweight
- BGP works natively over GRE
- No encryption overhead (for lab, this is fine)


**Pros**:
- ✅ This is what ISPs actually do
- ✅ Simple and lightweight
- ✅ BGP works perfectly
- ✅ No encryption overhead

**Cons**:
- ⚠️ No encryption (but lab traffic is not sensitive)
- ⚠️ Still internet path

**Learning Value**:
- Real ISP interconnect methods
- GRE tunneling
- BGP fundamentals

---

### Option 3: Cloud Provider "Direct Connect" Simulation

**Concept**: Simulate OCI FastConnect / AWS Direct Connect / Azure ExpressRoute

**Reality Check**:
- Real Direct Connect costs $$$ (hundreds to thousands per month)
- Requires physical cross-connect at colocation facility
- Not practical for homelab

**But We Can Simulate It!**

**Architecture**:
```
OCI VCN (Virtual Cloud Network)
   │
   │ "FastConnect" (really a VPN tunnel)
   │ We PRETEND this is a dedicated circuit
   │
   ↓
Second ISP
```

**Implementation**:
```yaml
Technical Reality:
  - Use VPN or GRE tunnel
  - Name it "FastConnect-Sim"
  - Configure as if it's dedicated
  - Set higher BGP local preference
  - Document as "simulated direct connect"

Configuration:
  - Dedicated VLAN in lab (VLAN 700: Cloud-Interconnect)
  - Dedicated IP space (203.0.113.0/30)
  - BGP with higher preference than internet
  - QoS policies for prioritization
```

**Pros**:
- ✅ Teaches Direct Connect concepts
- ✅ More realistic than basic VPN
- ✅ Shows understanding of enterprise connectivity

**Cons**:
- ⚠️ Still a tunnel underneath
- ⚠️ Not actual dedicated circuit

**Learning Value**:
- Cloud interconnect services
- Dedicated vs internet connectivity
- BGP route preference
- Enterprise hybrid cloud


---

### Option 4: Multi-Cloud Gateway (Advanced)

**Concept**: Deploy multi-cloud routing instance

**Architecture**:
```
           ┌──────────────────┐
           │ Multi-Cloud GW   │
           │  (VyOS in OCI)   │
           │                  │
           │  Peers with:     │
           │  - OCI VCN       │
           │  - AWS VPC       │
           │  - Azure VNet    │
           │  - GCP VPC       │
           └────────┬─────────┘
                    │
              VPN to Lab
                    │
                    ↓
              Second ISP
```

**Use Case**:
- One tunnel from lab to OCI
- OCI instance routes to AWS, Azure, GCP
- Simulates "cloud backbone"

**Implementation**:
```yaml
OCI Multi-Cloud Router:
  Instance: ARM free tier VyOS
  Connections:
    - VPN to lab (primary)
    - VPN to AWS VPC
    - VPN to Azure VNet
    - VPN to GCP VPC (optional)
  Routing:
    - eBGP with each cloud
    - iBGP to lab
    - Acts as route reflector
```

**Pros**:
- ✅ Teaches multi-cloud networking
- ✅ One tunnel from lab (efficient)
- ✅ Realistic enterprise pattern

**Cons**:
- ⚠️ More complex
- ⚠️ Multiple cloud accounts needed
- ⚠️ Potential egress charges

**Learning Value**: **MAXIMUM**
- Multi-cloud networking
- BGP route reflectors
- Cloud VPN gateways
- Hybrid cloud architecture

---

## 🎯 Recommended Approach

### Phase 1: Start Simple (Week 1)

**Use Option 1: WireGuard VPN**

```yaml
Week 1 Implementation:
  OCI:
    - Deploy ARM instance (free tier)
    - Install VyOS
    - Configure WireGuard
    - Setup BGP

  Lab:
    - Cloud-BR-1 gets public IP (passthrough)
    - Configure WireGuard peer
    - Setup BGP session

  Testing:
    - Ping over tunnel
    - BGP routes exchanged
    - Traffic flows cloud ↔ lab
```

**Result**: Working cloud connectivity in Week 1!

### Phase 2: Make It Realistic (Week 2-3)

**Enhance with "FastConnect" Simulation**

```yaml
Enhancements:
  - Rename tunnel: "OCI-FastConnect-Sim"
  - Dedicated VLAN (700)
  - High BGP local preference (200)
  - QoS marking
  - SLA monitoring
  - Document as simulated Direct Connect

Documentation:
  "This tunnel simulates OCI FastConnect, a dedicated private
   connection between on-premises and OCI. In production, this
   would be a physical cross-connect at a colocation facility."
```

### Phase 3: Multi-Cloud (Week 4-5)

**Add AWS/Azure/GCP**

```yaml
Multi-Cloud Setup:
  1. Deploy gateway instance in OCI
  2. Add VPN to AWS VPC
  3. Add VPN to Azure VNet
  4. Configure BGP mesh
  5. Lab sees all clouds via one tunnel
```

---

## 💰 Cost Analysis

### Option 1: WireGuard VPN
```
OCI Free Tier: $0 (ARM instance)
Bandwidth: $0 (within free tier limits)
Your Internet: Existing connection
Total: $0/month
```

### Option 2: GRE Tunnel
```
Same as Option 1: $0/month
```

### Option 3: "FastConnect" Simulation
```
Same as Option 1: $0/month
(just configured differently)
```

### Option 4: Multi-Cloud Gateway
```
OCI Free Tier: $0
AWS Free Tier: $0 (first year)
Azure: ~$0-5/month (small VM)
GCP: ~$0-5/month (small VM)
Bandwidth: ~$5-10/month (inter-cloud)
Total: ~$5-15/month
```
---

### Verification

```bash
# Check tunnel is up
show interfaces wireguard

# Check BGP session
show bgp summary

# Check routes
show ip bgp
show ip route

# Test connectivity
ping 10.20.0.1 source-address 192.168.100.1
```

---

## 📊 Traffic Flow Example

**Scenario**: On-prem K8s pod wants to reach OCI K8s pod

```
On-Prem Pod (10.244.1.5)
   ↓
On-Prem CE Router (192.168.100.254)
   ↓ BGP to Atlas Lab ISP
Atlas Lab ISP
   ↓ via IXP peering
Second ISP (Cloud-BR-1)
   ↓ WireGuard tunnel
OCI Cloud-CE Router
   ↓
OCI K8s Pod (10.21.1.5)
```

**Total hops**:
- 4 BGP hops (realistic!)
- 1 encrypted tunnel
- Crosses "ISP" boundary
- Routes via IXP (free peering!)

**This is exactly how real multi-site enterprise networks work!**

---

## 🎓 Learning Outcomes

By implementing this, you'll learn:

**VPN Technologies**:
- WireGuard configuration
- IPsec fundamentals
- Tunnel keepalives
- MTU considerations

**BGP Over Tunnels**:
- eBGP sessions over VPN
- Route advertisement
- Path selection
- Multihoming

**Cloud Networking**:
- OCI VCN configuration
- Security lists and NACLs
- Public IP assignment
- Instance deployment

**Hybrid Cloud**:
- On-prem to cloud connectivity
- Route preference
- Failover scenarios
- Traffic engineering



---

## ✅ Recommendation Summary

**Start Here**:
1. Deploy WireGuard tunnel OCI ↔ Lab
2. Configure BGP over tunnel
3. Test connectivity

**Enhance**:
4. Rename as "FastConnect-Sim"
5. Add QoS and monitoring
6. Document as simulated direct connect

**Expand**:
7. Add AWS/Azure/GCP
8. Multi-cloud gateway in OCI
9. Full hybrid cloud architecture

**Cost**: $0 to start, ~$5-15/month for full multi-cloud

## VERSION HISTORY

- v1.0 (2026-02-07): Initial resource compilation
- v1.1 (2026-02-08): Cleaned up and updated original AI documentation