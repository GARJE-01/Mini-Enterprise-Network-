# 🗺️ Network Topology
### Mini Enterprise Network — Mayur Garje

---

## Overview

This project is built in two parts that work together to simulate a complete enterprise network:

- **Part A** runs inside **Cisco Packet Tracer** — a network simulator used to design and configure VLANs, routing, ACLs, and NAT using real Cisco IOS commands.
- **Part B** runs inside **VMware Workstation Pro** on a Windows 11 PC — real virtual machines running actual server and firewall software on a live network.

Both parts are intentionally separate. Packet Tracer cannot communicate with real VMs — it is a sealed simulation environment. The value of Part A is demonstrating Cisco CLI skills (CCNA exam topics). The value of Part B is demonstrating real server and firewall deployment skills.

---

## Part A — Packet Tracer Topology

### Screenshot
> 📸 **Insert image: `screenshots/0.png`**
> *(Full topology view showing ISP, R1, SW1, and all 4 VLAN PCs)*

### Topology Diagram

```
                    [ Internet / ISP ]
                          |
                     Gig0/0 (WAN)
                          |
                    [ R1 — Router 2911 ]
                     Inter-VLAN routing
                     NAT/PAT to internet
                     ACL enforcement
                          |
                     Gig0/0 (LAN trunk)
                          |
                    [ SW1 — Switch 2960 ]
                     802.1Q trunk port
                     VLAN access ports
                     STP loop prevention
                    /    |    |    \
                  Fa0/1 Fa0/2 Fa0/3 Fa0/4
                   |     |     |     |
                [PC1] [PC2] [PC3] [PC4]
               VLAN10 VLAN20 VLAN30 VLAN40
               Office Server  Guest  Mgmt
```

### Devices Used

| Device | Model | Role |
|---|---|---|
| ISP | Generic Router | Simulates internet provider |
| R1 | Cisco 2911 | Core router — routing, NAT, ACL |
| SW1 | Cisco 2960 | Layer 2 switch — VLAN switching |
| PC1 | Generic PC | Office VLAN workstation |
| PC2 | Generic PC | Server VLAN workstation |
| PC3 | Generic PC | Guest VLAN workstation |
| PC4 | Generic PC | Management VLAN workstation |

### Physical Connections

| From | Port | To | Port | Cable Type |
|---|---|---|---|---|
| ISP | Gig0/0 | R1 | Gig0/1 | Serial/crossover |
| R1 | Gig0/0 | SW1 | Gig0/1 | Straight-through (trunk) |
| SW1 | Fa0/1 | PC1 | Fa0 | Straight-through |
| SW1 | Fa0/2 | PC2 | Fa0 | Straight-through |
| SW1 | Fa0/3 | PC3 | Fa0 | Straight-through |
| SW1 | Fa0/4 | PC4 | Fa0 | Straight-through |

---

## Part A — VLAN Design

### VLAN Table

| VLAN ID | Name | Subnet | Usable Range | Gateway | Purpose |
|---|---|---|---|---|---|
| 10 | Office | 192.168.10.0/24 | .1 – .254 | 192.168.10.1 | Employee PCs and workstations |
| 20 | Server | 192.168.20.0/24 | .1 – .254 | 192.168.20.1 | Internal servers — DNS, DHCP, Web, Files |
| 30 | Guest | 192.168.30.0/24 | .1 – .254 | 192.168.30.1 | Guest Wi-Fi — internet only, no internal access |
| 40 | Management | 192.168.40.0/24 | .1 – .254 | 192.168.40.1 | Network device access and monitoring |

### Why Each VLAN Exists

**VLAN 10 — Office**
All employee workstations sit in this VLAN. By isolating employees into their own VLAN, their broadcast traffic does not flood the server or guest segments. Employees can reach the internet and the server VLAN but are managed separately.

**VLAN 20 — Servers**
All internal servers (DNS, DHCP, Apache, Samba) live here. Placing servers in a dedicated VLAN means their traffic is isolated from end users. Access to this VLAN can be tightly controlled — in this project, Guest VLAN (30) is blocked from reaching it.

**VLAN 30 — Guest**
Guest devices (phones, visitor laptops) are placed here. This VLAN has internet access only — ACL rules block it from reaching the Server VLAN or Office VLAN. If a guest device has malware, it cannot reach internal resources.

**VLAN 40 — Management**
Reserved for managing network devices — switch, router, and monitoring tools. In a production network only network engineers can access this VLAN, keeping device management traffic completely separate from user traffic.

### IP Address Assignment Table (Packet Tracer)

| Device | VLAN | IP Address | Subnet Mask | Default Gateway |
|---|---|---|---|---|
| R1 — sub-interface VLAN 10 | 10 | 192.168.10.1 | 255.255.255.0 | — |
| R1 — sub-interface VLAN 20 | 20 | 192.168.20.1 | 255.255.255.0 | — |
| R1 — sub-interface VLAN 30 | 30 | 192.168.30.1 | 255.255.255.0 | — |
| R1 — sub-interface VLAN 40 | 40 | 192.168.40.1 | 255.255.255.0 | — |
| PC1 | 10 | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 |
| PC2 | 20 | 192.168.20.10 | 255.255.255.0 | 192.168.20.1 |
| PC3 | 30 | 192.168.30.10 | 255.255.255.0 | 192.168.30.1 |
| PC4 | 40 | 192.168.40.10 | 255.255.255.0 | 192.168.40.1 |

---

## Part A — Traffic Flow

### Inter-VLAN Routing (Router-on-a-Stick)

A single physical cable connects SW1 to R1. This cable carries traffic for all 4 VLANs simultaneously, each tagged with an 802.1Q VLAN ID. On the router side, 4 logical sub-interfaces are configured — one per VLAN. This design is called **router-on-a-stick**.

```
PC1 (VLAN 10) wants to reach PC2 (VLAN 20):

PC1 → SW1 (tagged VLAN 10) → R1 sub-interface Gig0/0.10
    → R1 routing table → R1 sub-interface Gig0/0.20
    → SW1 (tagged VLAN 20) → PC2
```

Without this router, PCs in different VLANs cannot communicate — they are in completely separate broadcast domains.

### NAT/PAT — Internet Access

```
PC1 (192.168.10.10) → R1 NAT → ISP Router
Private IP translated to public IP → Internet
Reply comes back → R1 NAT → returns to PC1
```

NAT (Network Address Translation) hides all internal private IPs behind a single public IP. PAT (Port Address Translation) allows multiple devices to share one public IP by tracking connections using port numbers.

### ACL — Guest VLAN Blocked from Servers

```
PC3 (VLAN 30 Guest) tries to reach PC2 (VLAN 20 Server):
→ R1 checks ACL: source 192.168.30.x → destination 192.168.20.x
→ ACL rule: DENY
→ Packet dropped — PC3 cannot reach servers

PC3 tries to reach internet:
→ R1 checks ACL: source 192.168.30.x → destination any
→ ACL rule: PERMIT
→ Traffic forwarded to ISP → internet accessible
```

---

## Part B — VMware Topology

### Network Diagram

```
┌─────────────────────────────────────────────────┐
│              Windows 11 Host PC                  │
│              Intel Core i5-11400H               │
│              8GB RAM                            │
│                                                 │
│  ┌──────────────────┐  ┌─────────────────────┐  │
│  │  pfSense VM      │  │  Ubuntu Server VM   │  │
│  │  FreeBSD         │  │  Ubuntu 22.04 LTS   │  │
│  │  1GB RAM         │  │  2GB RAM            │  │
│  │  16GB disk       │  │  20GB disk          │  │
│  │                  │  │                     │  │
│  │  WAN: DHCP       │  │  IP: 192.168.20.101 │  │
│  │  192.168.211.134 │  │  (static)           │  │
│  │                  │  │                     │  │
│  │  LAN: static     │  │  Services:          │  │
│  │  192.168.20.1    │  │  - BIND9 DNS        │  │
│  │                  │  │  - ISC-DHCP         │  │
│  │  Services:       │  │  - Apache2          │  │
│  │  - DHCP server   │  │  - Samba            │  │
│  │  - DNS resolver  │  │                     │  │
│  │  - Firewall      │  └──────────┬──────────┘  │
│  │  - pfBlockerNG   │             │              │
│  └────────┬─────────┘             │              │
│           │                       │              │
│           └──────────┬────────────┘              │
│                      │                           │
│              VMware Virtual Switch               │
│              192.168.20.0/24 (LAN)               │
│                      │                           │
│              VMware NAT Network                  │
│              192.168.211.0/24 (WAN)              │
│                      │                           │
└──────────────────────┼──────────────────────────┘
                       │
               [ Home Router / Internet ]
```

### VMware Network Mode Explained

| VM | Adapter | Network | IP | Internet Access |
|---|---|---|---|---|
| pfSense WAN | NAT | 192.168.211.0/24 | 192.168.211.134 (DHCP) | Yes — through VMware NAT |
| pfSense LAN | Host-only/Custom | 192.168.20.0/24 | 192.168.20.1 (static) | Via WAN interface |
| Ubuntu Server | Custom | 192.168.20.0/24 | 192.168.20.101 (static) | Via pfSense LAN |

### Part B — IP Addressing Table

| Device | Interface | IP Address | Subnet Mask | Gateway | Role |
|---|---|---|---|---|---|
| pfSense | WAN (em0) | 192.168.211.134 | 255.255.255.0 | 192.168.211.2 | Internet-facing interface |
| pfSense | LAN (em1) | 192.168.20.1 | 255.255.255.0 | — | LAN gateway + DNS + DHCP |
| Ubuntu Server | ens33 | 192.168.20.101 | 255.255.255.0 | 192.168.20.1 | All server services |
| Windows Host | (physical) | 192.168.20.103 | 255.255.255.0 | 192.168.20.1 | Test client (DHCP from pfSense) |
| VMware NAT | — | 192.168.211.2 | 255.255.255.0 | — | pfSense WAN gateway |

---

## Part B — Traffic Flow

### Client Browsing the Internet

```
Windows PC (192.168.20.103)
  → DNS query to pfSense (192.168.20.1:53)
  → pfSense DNS resolver forwards to 8.8.8.8
  → Returns IP of website
  → Browser connects to website IP
  → pfSense NAT: 192.168.20.103 → 192.168.211.134
  → VMware NAT: 192.168.211.134 → home router
  → Internet
```

### Client Accessing Company Intranet

```
Windows PC browser → http://company.local
  → DNS query to pfSense (192.168.20.1:53)
  → pfSense domain override: company.local → 192.168.20.101
  → BIND9 on Ubuntu returns 192.168.20.101
  → Browser connects to 192.168.20.101
  → Apache2 on Ubuntu serves the intranet page
  → Page displayed in browser ✅
```

### Client Trying to Access YouTube (Blocked)

```
Windows PC browser → youtube.com
  → DNS query to pfSense (192.168.20.1:53)
  → pfBlockerNG intercepts query
  → Returns sinkhole IP: 10.10.10.1
  → Browser tries to connect to 10.10.10.1
  → No real server at 10.10.10.1
  → Connection fails — YouTube inaccessible ✅
```

### Client Trying to Bypass DNS (Blocked by Firewall)

```
Windows PC manually sets DNS to 8.8.8.8
  → DNS query goes to 8.8.8.8:53
  → pfSense firewall rule: BLOCK any → any:53
  → Packet dropped
  → Client cannot reach 8.8.8.8 for DNS
  → Falls back to pfSense DNS
  → pfBlockerNG still blocks YouTube ✅
```

---

## How Part A and Part B Relate

| Concept | Part A (Packet Tracer) | Part B (VMware) |
|---|---|---|
| VLAN segmentation | 4 VLANs designed and configured | Single LAN (192.168.20.0/24) |
| Routing | Router-on-a-stick (R1) | pfSense handles routing |
| DHCP | Configured on router | pfSense DHCP server |
| DNS | Simulated in PT | Real BIND9 + pfSense resolver |
| Firewall | ACL rules on router | pfSense firewall rules |
| NAT | Configured on R1 | pfSense NAT |
| Content filtering | Not in PT | pfBlockerNG DNSBL |
| Web server | Not in PT | Apache2 on Ubuntu |
| File server | Not in PT | Samba on Ubuntu |

Part A demonstrates the **network design and Cisco CLI skills** required for CCNA. Part B demonstrates the **real-world server and security deployment skills** required as a network engineer.

---

## Key Design Decisions

**Why router-on-a-stick and not a Layer 3 switch?**
A Layer 3 switch would be more efficient for inter-VLAN routing in a real enterprise. Router-on-a-stick was chosen because it is explicitly covered in the CCNA exam and demonstrates a thorough understanding of how 802.1Q trunking and sub-interfaces work.

**Why pfSense instead of a Cisco ASA?**
pfSense is free and open source. It runs the same firewall concepts (NAT, ACL rules, stateful inspection) as enterprise firewalls but at no cost — making it ideal for a home lab. The underlying concepts transfer directly to Cisco ASA, Fortinet, and Palo Alto.

**Why BIND9 on Ubuntu when pfSense has a built-in DNS resolver?**
Running BIND9 separately demonstrates understanding of how enterprise DNS actually works — with a dedicated authoritative DNS server separate from the firewall. pfSense's DNS resolver was configured to forward `company.local` queries to BIND9, mirroring real enterprise DNS architecture.

**Why a single LAN in Part B instead of VLANs?**
VLANs in VMware require either a managed virtual switch (VMware vSphere feature, not in Workstation) or physical hardware. The VLAN segmentation concept is demonstrated fully in Part A. Part B focuses on demonstrating that real services work — not re-demonstrating VLAN concepts that are already covered.

---

*Document: NETWORK-TOPOLOGY.md*
*Project: Mini Enterprise Network*
*Author: Mayur Garje*
*Date: May 2026*
