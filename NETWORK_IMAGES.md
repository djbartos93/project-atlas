# NETWORK DEVICE IMAGES 

**TL;DR Recommendation:** Start with **Option B (Hybrid Free)** - use VyOS and Arista vEOS. Can upgrade to Cisco later if desired for resume value.

---

## OPTION A: CISCO CML PERSONAL (Commercial)

### What You Get
- **Product:** Cisco Modeling Labs Personal Edition
- **Cost:** $199/year
- **Link:** https://learningnetworkstore.cisco.com/cisco-modeling-labs-personal

### Included Images (All Legal)
| Platform | Filename | Version | Purpose |
|----------|----------|---------|---------|
| IOS-XRv | `iosxrv-k9-demo-7.3.1.qcow2` | 7.3.1 | Core routers (MPLS) |
| CSR1000v | `csr1000v-universalk9.17.03.01.qcow2` | 17.3+ | Edge routers (BGP) |
| IOSv | `vios-adventerprisek9-m.vmdk.SPA.159-3.M6` | 15.9 | Layer 3 switches |
| IOSvL2 | `vios_l2-adventerprisek9-m.ssa.high_iron_20200929` | Latest | Layer 2 switches |
| NX-OSv | `nexus9300v.10.3.1.F.qcow2` | 10.3+ | Data center |
| IOS-XRv 9000 | `iosxrv9000-fullk9-x-7.3.1.qcow2` | 7.3.1 | High-end routing |

### Pros
✅ All images legally licensed
✅ "Real" Cisco experience for resume
✅ Official support from Cisco
✅ Regular updates included
✅ Can use on resume/interviews confidently
✅ Most enterprise-relevant

### Cons
❌ Costs $199/year
❌ Requires credit card
❌ License checks (must be online)
❌ Limited to personal use

### EVE-NG Integration
- Images work directly in EVE-NG
- Just need to convert to qcow2 if needed
- Follow EVE-NG image naming conventions

### Best For
- Resume building for enterprise roles
- Learning Cisco-specific features
- If you have budget
- If targeting Cisco shops

---

## OPTION B: HYBRID FREE (Recommended)

### Core Platform: VyOS (Open Source)

**What:** Professional routing platform based on Debian Linux
**Cost:** FREE (open source)
**Download:** https://vyos.net/get/snapshots/

#### VyOS Capabilities
- ✅ BGP (full-featured, all AFIs)
- ✅ OSPF / ISIS
- ✅ MPLS (LDP, RSVP-TE)
- ✅ VRF (VRF-Lite and MPLS L3VPN)
- ✅ Policy-based routing
- ✅ QoS
- ✅ Firewall / NAT
- ✅ VPN (IPsec, WireGuard, OpenVPN)
- ✅ VRRP / high availability

**Exact Image:**
- Filename: `vyos-1.4-rolling-202401*.iso`
- Version: 1.4 rolling (latest)
- Size: ~500MB
- Format: ISO (convert to qcow2)

**EVE-NG Setup:**
```bash
# Directory structure
/opt/unetlab/addons/qemu/vyos-1.4/
└── virtioa.qcow2

# Resource requirements
CPU: 2 vCPU
RAM: 2GB (can work with 1GB)
Disk: 8GB
```

**Boot Time:** 2-3 minutes
**Default Login:** vyos / vyos

#### VyOS in Your Topology
- ✅ Core routers (Core-CHI, Core-NYC)
- ✅ Edge routers (all BGW-* routers)
- ✅ Customer edge (Lab-CE)
- ✅ Can do everything Cisco can for this project

### Secondary Platform: Arista vEOS (Free Registration)

**What:** Arista's virtual switch/router platform
**Cost:** FREE (requires free account)
**Download:** https://www.arista.com/en/support/software-download

#### vEOS Capabilities
- ✅ BGP (excellent implementation)
- ✅ OSPF / ISIS
- ✅ Layer 2/3 switching
- ✅ VXLAN / EVPN
- ✅ Excellent automation support

**Exact Image:**
- Filename: `vEOS-lab-4.28.0F.vmdk` or `vEOS64-lab-4.28.0F.qcow2`
- Version: 4.28+ (latest stable)
- Size: ~1GB
- Format: VMDK or qcow2

**EVE-NG Setup:**
```bash
/opt/unetlab/addons/qemu/veos-4.28.0F/
└── virtioa.qcow2

CPU: 2 vCPU
RAM: 2GB
Disk: 10GB
```

**Account Creation:**
1. Go to https://www.arista.com/
2. Click "Support" → "Software Download"
3. Create free account
4. Download vEOS-lab image

#### vEOS in Your Topology
- ✅ IXP route server (optional)
- ✅ Data center switching (optional)
- ✅ Alternative to VyOS for variety

### Hybrid Free Topology

```
Tier-1 Upstreams: VyOS (2x)
        ↓
Core MPLS: VyOS (2x)
        ↓
Edge BGP: VyOS (4x)
        ↓
Customer Edge: VyOS (1x)

Optional:
IXP Route Server: Arista vEOS
```

### Pros
✅ $0 cost
✅ Fully capable (can do everything Cisco can)
✅ Open source = learn internals
✅ VyOS has excellent documentation
✅ Arista on resume (enterprise credibility)
✅ Easy to source images
✅ No licensing headaches

### Cons
⚠️ Less "brand name" than Cisco
⚠️ VyOS CLI different from Cisco (more Linux-like)
⚠️ Won't learn IOS-specific commands

### Best For
- Budget-conscious
- Want to focus on concepts not vendor syntax
- Open source preference
- Still impressive on resume
- Can always add Cisco later

---

## OPTION C: PURE OPEN SOURCE

### Platforms Available

#### 1. VyOS (Primary)
- Use for everything
- See Option B details above

#### 2. FRRouting (FRR)
**What:** Advanced routing protocol suite for Linux
**Cost:** FREE
**Download:** https://frrouting.org/

**Setup:** Install FRR on Ubuntu/Debian VMs
```bash
# Create Ubuntu VM in EVE-NG
# Install FRR
curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -
echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) frr-stable | sudo tee -a /etc/apt/sources.list.d/frr.list
sudo apt update && sudo apt install frr frr-pythontools
```

**Capabilities:**
- BGP, OSPF, ISIS, RIP, EIGRP
- MPLS (LDP, RSVP-TE)
- PIM (multicast)
- BFD
- Static routes, policy routing

#### 3. OpenBSD + bgpd
**What:** BSD-based router
**Cost:** FREE

#### 4. pfSense/OPNsense
**What:** FreeBSD-based firewall/router
**Cost:** FREE
**Good for:** Edge routers, firewalling

### Pure Open Source Topology

```
All routers: VyOS or FRRouting on Ubuntu
```

### Pros
✅ $0 cost
✅ Learn Linux networking deeply
✅ No vendor lock-in
✅ Complete control

### Cons
❌ No "big name" vendor experience
❌ More manual setup
❌ Less enterprise resume value

---

## COMPARISON TABLE

| Feature | Cisco CML | Hybrid Free | Pure OSS |
|---------|-----------|-------------|----------|
| **Cost/Year** | $199 | $0 | $0 |
| **Resume Value** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Capabilities** | Full | Full | Full |
| **Learning Curve** | Medium | Medium | Medium |
| **Enterprise Relevance** | High | Medium-High | Medium |
| **Ease of Setup** | Easy | Easy | Medium |
| **Support** | Official | Community | Community |
| **Legal Concerns** | None | None | None |

---

## DECISION MATRIX

### Choose Cisco CML Personal If:
- [ ] You have $199 budget
- [ ] Targeting Cisco-focused roles
- [ ] Want "official" experience
- [ ] Resume needs enterprise vendor names
- [ ] Prefer supported software

### Choose Hybrid Free If:
- [x] Budget conscious ($0)
- [x] Want full capability
- [x] Learn concepts not syntax
- [x] Arista experience valuable
- [x] Can explain open source choice in interviews

### Choose Pure Open Source If:
- [ ] Zero budget (not even $0)
- [ ] Deeply interested in Linux networking
- [ ] Focusing on cloud/devops roles
- [ ] Don't need vendor-specific experience

---

## RECOMMENDED DECISION: HYBRID FREE

**My Recommendation for Your Situation:**

Start with **Hybrid Free (Option B)** using VyOS + Arista vEOS because:

1. **$0 Cost** - No budget concerns
2. **Full Capability** - Can do everything Cisco can
3. **Quick Start** - Images available immediately
4. **Resume Value** - Arista is enterprise-credible
5. **Upgrade Path** - Can add Cisco CML later if desired

**Later Addition:**
1. Purchase CML Personal ($199)
2. Replace some VyOS routers with Cisco images
3. Now you have BOTH open source AND Cisco experience

---

## IMAGE ACQUISITION GUIDE

### For VyOS (Free)

```bash
# Download latest rolling release
wget https://vyos.net/get/snapshots/vyos-1.4-rolling-$(date +%Y%m%d)-amd64.iso

# Create qcow2 disk
qemu-img create -f qcow2 vyos.qcow2 8G

# Install VyOS to qcow2
# (Boot ISO in VM, install to disk)

# Or download pre-built qcow2 from community
```

**Upload to EVE-NG:**
```bash
# Create directory
ssh root@192.168.100.50
mkdir -p /opt/unetlab/addons/qemu/vyos-1.4/

# Upload from workstation
scp vyos.qcow2 root@192.168.100.50:/opt/unetlab/addons/qemu/vyos-1.4/virtioa.qcow2

# Fix permissions
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

### For Arista vEOS (Free Account)

```bash
# 1. Create account at arista.com
# 2. Navigate to Software Download
# 3. Search "vEOS-lab"
# 4. Download vEOS-lab-4.28.0F.vmdk (or latest)
# 5. Convert if needed:
qemu-img convert -f vmdk -O qcow2 vEOS-lab-4.28.0F.vmdk veos.qcow2

# 6. Upload to EVE-NG
mkdir -p /opt/unetlab/addons/qemu/veos-4.28.0F/
scp veos.qcow2 root@192.168.100.50:/opt/unetlab/addons/qemu/veos-4.28.0F/virtioa.qcow2
```

### For Cisco CML (Paid)

```bash
# 1. Purchase at learningnetworkstore.cisco.com
# 2. Download CML Personal
# 3. Extract images from CML installation
# 4. Images are in qcow2 format
# 5. Upload to EVE-NG following naming conventions
```

---

## TESTING YOUR IMAGES

### Verify Image in EVE-NG

1. **Create Test Lab**
   - In EVE-NG web UI
   - Name: "Image-Test"

2. **Add Node**
   - Select template (VyOS, vEOS, etc.)
   - Name: test-router
   - CPU: 2, RAM: 2048

3. **Start Node**
   - Right-click → Start
   - Wait 2-5 minutes

4. **Connect Console**
   - Right-click → Console
   - Should see boot messages
   - Should get login prompt

5. **Test Basic Commands**
   ```bash
   # VyOS
   configure
   set interfaces ethernet eth0 address 10.0.0.1/24
   commit
   
   # Arista
   enable
   configure
   interface Ethernet1
   ip address 10.0.0.1/24
   ```

6. **If Successful**
   - Shut down test node
   - Delete test lab
   - Ready for production topology!

---

## CONVERSION BETWEEN FORMATS

### VMDK to qcow2
```bash
qemu-img convert -f vmdk -O qcow2 input.vmdk output.qcow2
```

### OVA to qcow2
```bash
# Extract OVA
tar -xvf input.ova

# Convert VMDK to qcow2
qemu-img convert -f vmdk -O qcow2 extracted.vmdk output.qcow2
```

### ISO to qcow2 (with installation)
```bash
# Create empty disk
qemu-img create -f qcow2 disk.qcow2 8G

# Boot ISO and install
# (Do this in EVE-NG or local VM)
```
