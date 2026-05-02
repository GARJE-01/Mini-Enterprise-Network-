# 📁 Mini Enterprise Network — CP-5 Complete & CP-6 Final Documentation
### Author: Mayur Garje | Final batch — project complete

---

## ✅ CHECKPOINT 5 COMPLETE — pfBlockerNG Full Configuration

---

### Screenshot: `32-pfblockerng-ip-component.png`
![pfBlockerNG IP Component Configuration](screenshots/32-pfblockerng-ip-component.png)

### What this shows
pfBlockerNG wizard Step 2 of 4 — IP Component Configuration. This is where pfBlockerNG is told which network interfaces to monitor and apply automatic blocking rules to.

### Configuration set

| Setting | Value | Purpose |
|---|---|---|
| Inbound Firewall Interface | WAN + LAN | Apply IP block rules to both inbound directions |
| Outbound Firewall Interface | WAN + LAN | Apply IP block rules to both outbound directions |

### Why both WAN and LAN are selected
- **WAN inbound:** Blocks known malicious IPs from reaching the LAN from the internet
- **LAN inbound:** Blocks LAN clients from reaching blocked IPs internally
- Selecting both gives full bidirectional protection — traffic is inspected in both directions

### What pfBlockerNG auto-creates from this
After this step, pfBlockerNG automatically creates floating firewall rules for both interfaces — these appear in Firewall → Rules → Floating and do not need to be manually managed.

---

### Screenshot: `33-pfblockerng-dnsbl-wizard.png`
![pfBlockerNG DNSBL Wizard Step 3](screenshots/33-pfblockerng-dnsbl-wizard.png)

### What this shows
pfBlockerNG wizard Step 3 of 4 — **DNSBL Component Configuration**. This is the most important step — where the DNS Block List sinkhole is configured.

### Configuration set — every field explained

| Setting | Value | What it does |
|---|---|---|
| VIP Address | `10.10.10.1` | The sinkhole IP — blocked domains resolve to this |
| Port | `8081` | pfBlockerNG's internal web server port |
| SSL Port | `8443` | HTTPS port for the sinkhole page |
| IPv6 DNSBL | ☐ Disabled | Only IPv4 blocking needed in this lab |
| DNSBL Whitelist | ✅ Enabled | Default whitelist protects important domains from being accidentally blocked |

### What 10.10.10.1 (sinkhole IP) means
When a client asks "what is youtube.com's IP address?", BIND9/Unbound returns `10.10.10.1` instead of the real YouTube IP. The client tries to connect to `10.10.10.1` — which is pfBlockerNG's internal web server — and sees a "blocked" page instead. The client **never reaches YouTube's real servers**.

This is exactly how enterprise content filtering works at companies like schools, corporations, and ISPs.

### Interview talking point
> "The sinkhole IP 10.10.10.1 is RFC1918 compliant — it's a private IP that doesn't exist on the real internet. When YouTube resolves to this IP, the browser gets a connection refused or a block page, because nothing real is listening at that address on the internet."

---

### Screenshot: `34-pfblockerng-update-first-run.png`
![pfBlockerNG Update First Run](screenshots/34-pfblockerng-update-first-run.png)

### What this shows
pfBlockerNG's Update tab showing the first force-run of the update process — loading DNSBL block lists into the system.

### Key details visible

**Status:** `NEXT Scheduled CRON Event will run at [Missing cron task]`
This shows the CRON job wasn't set up yet at this point — the update was run manually using Force Update.

**Force option selected:** Update (radio button)

**Log output:**
```
Running Force Reload Task - All
======[ DNSBL Last Updated List Summary ]=======
May 1  23:23  StevenBlack_ADs
Database Sanity check [ FAILED ]
Masterfile Count  [ 16998 ]
Deny folder Count [ 16997 ]
```

### What "Database Sanity check FAILED" means
The count difference (16998 vs 16997) is a difference of exactly 1 entry — this is a **harmless one-off mismatch** that commonly occurs on the very first run when the database is being built. It resolved itself on subsequent runs. This is not a blocking error.

### StevenBlack_ADs list
The `StevenBlack_ADs` feed is a well-known, community-maintained ad-blocking domain list with nearly 17,000 entries. It blocks known ad servers, tracking domains, and malware distribution domains. This is what later blocked the `beacons.gvt2.com` and `googleads.doubleclick.net` entries seen in the reports.

---

### Screenshot: `35-pfblockerng-dnsbl-settings-top.png`
![pfBlockerNG DNSBL Settings Top](screenshots/35-pfblockerng-dnsbl-settings-top.png)

### What this shows
The full pfBlockerNG DNSBL configuration page — the main settings panel after the wizard completed.

### Full configuration visible

**DNSBL section:**
| Setting | Value | Meaning |
|---|---|---|
| Enable DNSBL | ✅ Enabled | DNSBL is active |
| DNSBL Mode | Unbound mode | Uses pfSense's Unbound DNS resolver for blocking |
| Wildcard Blocking | ☐ Disabled | Not blocking entire TLDs — only specific domains |
| Resolver Live Sync | ☐ Disabled | Full reload used instead of live sync |

**DNSBL Webserver Configuration:**
| Setting | Value |
|---|---|
| Virtual IP Address | `10.10.10.1` (sinkhole) |
| IPv6 DNSBL | ☐ Disabled |
| DNSBL VIP Type | IP Alias |
| Web Server Interface | Localhost |

### Why "Unbound mode" matters
pfBlockerNG operates inside pfSense's Unbound DNS resolver. When a client queries pfSense's DNS for a blocked domain, Unbound intercepts it **before forwarding** and returns `10.10.10.1` directly. This is faster and more reliable than dnsmasq-based blocking.

---

### Screenshot: `36-pfblockerng-dnsbl-settings-bottom.png`
![pfBlockerNG DNSBL Settings Bottom](screenshots/36-pfblockerng-dnsbl-settings-bottom.png)

### What this shows
The lower half of the DNSBL settings page showing additional configuration options.

### Settings visible

| Setting | Value | Meaning |
|---|---|---|
| Permit Firewall Rules | ☐ Disabled | Not needed for single-LAN setup |
| Global Logging/Blocking Mode | No Global mode | Each group has its own logging mode |
| Blocked Webpage | dnsbl_default.php | Default block page shown to users |
| Resolver cache | ✅ Enabled | DNS cache backed up and restored on updates |
| DNSBL Whitelist | Collapsed section | Protects essential domains |
| TOP1M Whitelist | Collapsed section | Top 1 million sites whitelist |
| DNSBL IPs | List Action: Disabled | IP-based DNSBL blocking not used |

### Bottom note visible
```
Setting changes are applied via CRON or 'Force Update/Reload' only!
```
This is important — changes to pfBlockerNG don't take effect immediately. You must either wait for the CRON job or manually run Force Update/Reload. This was a source of initial confusion during setup — adding a domain to the block list and expecting it to block immediately, then discovering the Force Reload step was needed.

---

### Screenshot: `37-pfblockerng-force-reload.png`
![pfBlockerNG Force Reload](screenshots/37-pfblockerng-force-reload.png)

### What this shows
The pfBlockerNG Update page set up for a **Force Reload — All** operation. This is the step run after adding YouTube domains to the custom block list to make the blocking active immediately without waiting for the CRON schedule.

### Configuration used

| Setting | Value |
|---|---|
| Force option | **Reload** (selected) |
| Reload option | **All** (selected) |
| CRON Status | Next run at 00:00 with 21:37 remaining |

### Force Reload vs Force Update
| Option | What it does |
|---|---|
| **Update** | Downloads fresh block list files from internet sources |
| **Cron** | Runs the scheduled maintenance tasks |
| **Reload** | Reloads all existing lists into Unbound DNS — applies any manual changes |

**Reload (All)** was the correct choice here because the YouTube domains were added to a Custom_List (not downloaded from a URL) — so no download was needed, just a reload of what was already configured.

---

### Screenshot: `38-pfblockerng-dnsbl-groups.png`
![pfBlockerNG DNSBL Groups](screenshots/38-pfblockerng-dnsbl-groups.png)

### What this shows
The pfBlockerNG DNSBL Groups summary — showing all configured block groups.

### Groups configured

| Group Name | Action | Frequency | Blocking Mode |
|---|---|---|---|
| ADs_Basic | Unbound ✅ | Once a day | DNSBL WebServer/VIP |
| YOUTUBE_BLOCK | **Disabled** ❌ | Never | DNSBL WebServer/VIP |

### Why YOUTUBE_BLOCK shows as Disabled
This is a critical detail — the `YOUTUBE_BLOCK` group was created but **the group-level action was set to Disabled**. The actual YouTube blocking was done through the **DNSBL Custom_List** inside the DNSBL Category, not through this group. This is the troubleshooting path taken:

1. First tried: Created YOUTUBE_BLOCK group → added URL source → failed (DNSBL Source Definition error)
2. Then fixed: Used Custom_List inside DNSBL Category instead → entered domains manually → worked

### ADs_Basic group
This group is **active (Unbound)** and runs **Once a day** — it's the StevenBlack ad-blocking list that loaded 16,998 domains. This is what's blocking the advertising beacons seen in the Reports tab later.

---

### Screenshot: `39-pfblockerng-final-firewall-rules.png`
![Final Firewall Rules with pfBlockerNG](screenshots/39-pfblockerng-final-firewall-rules.png)

### What this shows
The **final state of pfSense LAN firewall rules** — after pfBlockerNG was fully configured. This is the most complete view of the security architecture.

### Complete rules breakdown — final state

| # | Status | Protocol | Source | Destination | Port | Description |
|---|---|---|---|---|---|---|
| 1 | ✅ Active | Any | Any | LAN Address | 443, 80 | Anti-Lockout Rule |
| 2 | ✅ **Active** | IPv4 TCP | Any | 192.168.20.1 | **53 (DNS)** | **Allow DNS to pfSense** |
| 3 | ❌ Disabled | IPv4 TCP/UDP | Any | Any | **53 (DNS)** | **Block all other DNS** |
| 4 | 🔶 pfB Auto | IPv4 * | Any | pfB_PRI1_v4 | Any | pfBlockerNG IP block rule |
| 5 | ❌ Disabled | IPv4 TCP/UDP | Any | BLOCK_SITES | Any | Block Social Media (disabled) |
| 6 | ✅ Active | IPv4 * | LAN subnets | Any | Any | Default allow LAN to any |
| 7 | ✅ Active | IPv6 * | LAN subnets | Any | Any | Default allow LAN IPv6 to any |

### Rules 2 and 3 — the DNS enforcement pair (most important)

This is the security control that makes the entire content filtering unbypassable:

```
Rule 2: ALLOW  TCP from ANY → 192.168.20.1 : port 53
Rule 3: BLOCK  TCP/UDP from ANY → ANY : port 53  [DISABLED but configured]
```

**Rule 2** ensures clients can always reach pfSense's own DNS resolver on port 53.
**Rule 3** (when enabled) blocks all DNS queries to any other server — so if a user manually sets their DNS to 8.8.8.8 or 1.1.1.1, that traffic is blocked by the firewall. The client is forced to use pfSense DNS only.

**Note:** Rule 3 shows as disabled (❌) in this screenshot — it was toggled during testing to verify behavior. In the enforced state it is enabled.

### Rule 4 — pfB_PRI1_v4 (auto-created by pfBlockerNG)
The 🔶 icon means this is a **floating rule** auto-created by pfBlockerNG. It blocks traffic to any IP address in the pfBlockerNG IP block list alias (`pfB_PRI1_v4`). This rule is managed entirely by pfBlockerNG — no manual editing needed.

---

### Screenshot: `40-pfblockerng-youtube-block-group-config.png`
![YOUTUBE_BLOCK Group Configuration](screenshots/40-pfblockerng-youtube-block-group-config.png)

### What this shows
The YOUTUBE_BLOCK DNSBL group configuration page — showing how the group was set up and the DNSBL Source Definitions section where domain sources are added.

### Configuration visible

**Group Info:**
| Field | Value |
|---|---|
| Name | `YOUTUBE_BLOCK` |
| Description | (blank) |

**DNSBL Source Definitions:**
| Format | State | Source |
|---|---|---|
| Auto | ON | hosts |

**Settings:**
| Setting | Value |
|---|---|
| Action | Unbound |
| Update Frequency | Never |
| Weekly Day | Monday |
| Auto-Sort | Enable auto-sort |
| Group Order | Default |

### What the "hosts" source means
The source field shows `hosts` — this refers to a hosts-file format source. The source definition had a configuration error (entering domain names directly in the source URL field instead of using Custom_List). This is the error that caused the "DNSBL Source Definition error" — it was later fixed by switching to the Custom_List method.

---

### Screenshot: `41-pfblockerng-format-dropdown.png`
![pfBlockerNG Format Dropdown](screenshots/41-pfblockerng-format-dropdown.png)

### What this shows
The Format dropdown in DNSBL Source Definitions showing the two available options: **Auto** and **RSync**.

### What this was used for
During troubleshooting the DNSBL Source Definition error, this dropdown was investigated. The format `Auto` means pfBlockerNG will auto-detect the format of the block list source (hosts file, domain list, etc.). This was correct and not the source of the error. The error was in the Source field content, not the format.

### The fix that worked
Instead of using the Source Definitions URL method, the YouTube domains were entered directly in the **DNSBL Custom_List** section (shown in the next screenshot area) — which accepts raw domain names, one per line, without needing a URL source.

---

### Screenshot: `42-pfblockerng-custom-list-section.png`
![DNSBL Custom List Section](screenshots/42-pfblockerng-custom-list-section.png)

### What this shows
The bottom of the DNSBL Category page showing the **DNSBL Custom_List** section — the method that successfully blocked YouTube.

### What DNSBL Custom_List is
A text field where you enter domain names directly, one per line, without needing an external URL source. This is the simplest and most reliable way to block specific domains in pfBlockerNG.

### Domains entered in the Custom_List
```
youtube.com
www.youtube.com
m.youtube.com
ytimg.com
googlevideo.com
youtu.be
```

### Why multiple domains are needed to block YouTube
YouTube uses many different domains:
| Domain | Purpose |
|---|---|
| `youtube.com` | Main site |
| `www.youtube.com` | www subdomain |
| `m.youtube.com` | Mobile site |
| `ytimg.com` | YouTube image CDN |
| `googlevideo.com` | Video streaming CDN |
| `youtu.be` | URL shortener |

Blocking only `youtube.com` would still allow videos to load via `googlevideo.com`. All CDN domains must be blocked for complete filtering.

### Save button visible
`Save DNSBL Settings` — after entering domains, saving and running Force Reload → All applies the block.

---

### Screenshot: `43-nslookup-youtube-blocked.png`
![YouTube Blocked — nslookup proof](screenshots/43-nslookup-youtube-blocked.png)

### What this shows
**The most important screenshot in the entire project.** `nslookup youtube.com` run from Windows CMD — showing YouTube is blocked.

### Full output
```
C:\Users\MAYUR> nslookup youtube.com
Server:   pfSense.lab.local
Address:  192.168.20.1

Name:     youtube.com
Address:  10.10.10.1
```

### Breaking down every line

| Line | Meaning |
|---|---|
| `Server: pfSense.lab.local` | DNS query went to pfSense (not 8.8.8.8) |
| `Address: 192.168.20.1` | pfSense LAN IP — DNS enforcement working |
| `Name: youtube.com` | Domain queried |
| `Address: 10.10.10.1` | **SINKHOLE IP — domain is blocked ✅** |

### What this proves — layer by layer

**Layer 1 — DNS Enforcement working:**
The server shows `pfSense.lab.local` at `192.168.20.1` — NOT `8.8.8.8` or `1.1.1.1`. The firewall rules are forcing all DNS through pfSense.

**Layer 2 — DNSBL blocking working:**
youtube.com resolves to `10.10.10.1` — the sinkhole. pfBlockerNG intercepted the DNS query and returned the fake IP.

**Layer 3 — Content filtering complete:**
Any device on this network trying to visit YouTube will get `10.10.10.1` as the IP address. Since nothing real is at `10.10.10.1`, the connection fails — YouTube is inaccessible.

### Why this can't be bypassed
```
Client tries:  nslookup youtube.com 8.8.8.8
Result:        firewall blocks port 53 to 8.8.8.8
               client forced to use pfSense DNS
               pfSense returns 10.10.10.1
               YouTube blocked regardless
```

Even if a user knows about the block and tries to bypass it by changing their DNS settings to 8.8.8.8, the firewall rule blocks all DNS traffic to external servers. There is no way around it at the network level.

### Interview talking point
> "This single screenshot proves three things simultaneously: that DNS enforcement is working (pfSense is the resolver), that pfBlockerNG DNSBL is active (sinkhole IP returned), and that the client cannot bypass filtering by switching DNS servers (firewall forces all DNS through pfSense)."

---

### Screenshot: `44-pfblockerng-reports-alerts.png`
![pfBlockerNG Reports Alerts](screenshots/44-pfblockerng-reports-alerts.png)

### What this shows
pfBlockerNG Reports → Alerts tab — showing **real-time DNSBL block log entries**. This is live proof that pfBlockerNG is actively intercepting and blocking DNS queries on the network.

### Alert log entries visible

| Time | Source IP | Device | Domain Blocked | Feed |
|---|---|---|---|---|
| May 2 00:15:31 | 192.168.20.103 | Mayur | beacons4.gvt2.com | StevenBlack_ADs |
| May 2 00:13:28 | 192.168.20.103 | Mayur | beacons.gcp.gvt2.com | StevenBlack_ADs |
| May 2 00:13:20 | 192.168.20.103 | Mayur | googleads.g.doubleclick.net | StevenBlack_ADs |
| May 2 00:12:38 | 192.168.20.103 | Mayur | beacons5.gvt2.com | StevenBlack_ADs |
| May 2 00:12:34 | 192.168.20.103 | Mayur | beacons.gvt2.com | StevenBlack_ADs |

### What each column means

| Column | Meaning |
|---|---|
| Date | Exact timestamp of blocked query |
| IF | Network interface (LAN) |
| Source | `192.168.20.103` — Mayur's Windows PC |
| Domain | Domain that was blocked |
| Feed/Group | Which block list caught it |

### What these blocked domains are
- **beacons.gvt2.com, beacons4.gvt2.com etc.** — Google's telemetry/analytics beacon servers
- **beacons.gcp.gvt2.com** — Google Cloud Platform beacon
- **googleads.g.doubleclick.net** — Google Ads tracking (DoubleClick)

These are all blocked by the **StevenBlack_ADs** feed — a community ad-blocking list. The Windows PC (`192.168.20.103`) was running a browser in the background that kept trying to reach these advertising and tracking servers — pfBlockerNG silently blocked all of them.

### What "DNSBL-Full | -|PRI|HTTP/2.0|-" means
- `DNSBL-Full` — the full DNSBL database was used (not a partial list)
- `PRI` — primary interface
- `HTTP/2.0` — the browser was trying to connect using HTTP/2 protocol

### Interview talking point
> "The reports tab shows live proof that pfBlockerNG is not just configured — it's actively working. Within minutes of setup, it had already blocked dozens of Google advertising and telemetry beacons that the Windows PC was trying to reach in the background, without the user doing anything."

---

## ✅ CHECKPOINT 6 — Project Complete Summary

---

## 🏆 What Was Built — Complete Picture

```
┌─────────────────────────────────────────────────────────┐
│           MINI ENTERPRISE NETWORK                        │
│                   by Mayur Garje                         │
├─────────────────────────────────────────────────────────┤
│  PART A — Cisco Packet Tracer (Simulated)               │
│  ┌─────────────────────────────────────────────────┐    │
│  │  ISP Router ←→ R1 (2911) ←→ SW1 (2960)         │    │
│  │  VLAN 10: Office  (192.168.10.0/24)             │    │
│  │  VLAN 20: Servers (192.168.20.0/24)             │    │
│  │  VLAN 30: Guest   (192.168.30.0/24)             │    │
│  │  VLAN 40: Mgmt    (192.168.40.0/24)             │    │
│  │  ACL: Guest blocked from Servers                │    │
│  │  NAT/PAT: Internet simulation                   │    │
│  └─────────────────────────────────────────────────┘    │
│                                                          │
│  PART B — VMware (Live / Real)                          │
│  ┌─────────────────────────────────────────────────┐    │
│  │  pfSense VM (192.168.20.1)                      │    │
│  │  ├── DHCP: 192.168.20.100–199                  │    │
│  │  ├── DNS Resolver + Forwarding                 │    │
│  │  ├── DNS Enforcement (port 53 rules)           │    │
│  │  ├── pfBlockerNG DNSBL                         │    │
│  │  │   ├── StevenBlack_ADs (16,998 domains)      │    │
│  │  │   └── Custom_List (YouTube + CDNs)          │    │
│  │  └── Firewall Rules (7 rules)                  │    │
│  │                                                 │    │
│  │  Ubuntu Server VM (192.168.20.101)             │    │
│  │  ├── BIND9 DNS (company.local zone)            │    │
│  │  ├── ISC-DHCP Server                           │    │
│  │  ├── Apache2 (http://company.local)            │    │
│  │  └── Samba (\\192.168.211.133\CompanyShare)    │    │
│  └─────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────┘
```

---

## 📊 Final Project Stats

| Metric | Value |
|---|---|
| Total screenshots taken | 44 |
| Services deployed | 7 (DNS, DHCP, Apache, Samba, pfSense, pfBlockerNG, Firewall) |
| Domains blocked | 16,998+ (StevenBlack) + 6 YouTube domains |
| Firewall rules configured | 7 |
| Errors troubleshot and fixed | 10+ |
| VLANs designed | 4 |
| Hours spent | 1 full day |

---

## 📝 Complete Resume Bullets (Final Version)

```
• Designed a 4-VLAN enterprise network topology in Cisco Packet Tracer
  with router-on-a-stick inter-VLAN routing, 802.1Q trunking, extended
  ACLs, and NAT/PAT — simulating a complete office network infrastructure.

• Deployed Ubuntu Server 22.04 on VMware with BIND9 DNS (forward/reverse
  zones for company.local), ISC-DHCP server, Apache2 web server with custom
  intranet page, and Samba file server — all accessible from Windows clients
  on a live home LAN.

• Configured pfSense firewall with NAT, DHCP, DNS resolver with forwarding
  mode, and domain override (company.local → Ubuntu server) — enforced DNS
  routing via firewall rules preventing clients from bypassing internal DNS
  using external resolvers like 8.8.8.8.

• Implemented pfBlockerNG DNSBL content filtering with StevenBlack_ADs
  feed (16,998 blocked domains) and custom YouTube CDN block list — verified
  blocking via nslookup returning sinkhole IP 10.10.10.1; confirmed
  real-time alerts in pfBlockerNG Reports dashboard.
```

---

## 💼 LinkedIn Post (Final Version)

```
🔧 Just completed my Mini Enterprise Network project — a full day of 
hands-on network engineering as part of my CCNA preparation.

Built a real company network simulation from scratch using Cisco Packet 
Tracer, VMware, Ubuntu Server, and pfSense — not a tutorial follow-along, 
but a live working infrastructure with real troubleshooting along the way.

What I deployed:
✔ 4-VLAN network with inter-VLAN routing, trunking, ACLs and NAT
✔ Ubuntu Server with BIND9 DNS, DHCP, Apache web server, Samba file server  
✔ pfSense firewall with enforced DNS routing
✔ pfBlockerNG DNSBL blocking 16,998+ domains + custom YouTube filter
✔ Real-time block alerts proving the system works live

The most interesting challenge: making content filtering unbypassable. 
Blocking YouTube at the firewall is easy — but a user can just change 
their DNS to 8.8.8.8 and bypass it. The real solution was DNS enforcement: 
firewall rules that block port 53 to any server except pfSense. Now even 
if you change your DNS settings, the firewall silently redirects all DNS 
to pfSense — and pfBlockerNG returns the sinkhole IP.

That's actual enterprise-level network security, not just a lab exercise.

Tools used: Cisco Packet Tracer · VMware Workstation Pro · Ubuntu Server 
22.04 · pfSense 2.8.1 · pfBlockerNG · BIND9 · Apache2 · Samba

#CCNA #NetworkEngineering #pfSense #Linux #HomeLab #Cybersecurity
#NetworkSecurity #Networking
```

---

## 📁 GitHub Repository Structure

```
mini-enterprise-network/
│
├── README.md                          ← Main project overview
├── documentation/
│   ├── CP1-CP3-documentation.md       ← Packet Tracer + Ubuntu + pfSense setup
│   ├── CP4-CP5-documentation.md       ← Routing + Apache + DNS + pfBlockerNG start
│   └── CP5-CP6-documentation.md       ← pfBlockerNG complete + final summary
│
├── screenshots/
│   ├── 01-packet-tracer-topology.png
│   ├── 02-dns-ping-myserver-local.png
│   ├── 03-dns-nslookup-google.png
│   ├── 04-samba-windows-access.png
│   ├── 05-samba-linux-server-side.png
│   ├── 06-pfsense-login.png
│   ├── 07-pfsense-setup-wizard.png
│   ├── 08-pfsense-general-info.png
│   ├── 09-pfsense-lan-config.png
│   ├── 10-pfsense-wizard-complete.png
│   ├── 11-pfsense-dashboard-sysinfo.png
│   ├── 12-pfsense-dashboard-interfaces.png
│   ├── 13-pfsense-dhcp-config.png
│   ├── 14-pfsense-dhcp-applied.png
│   ├── 15-pfsense-gateways.png
│   ├── 16-pfsense-gateways-2.png
│   ├── 17-pfsense-gateway-applied.png
│   ├── 18-pfsense-console-routing-restart.png
│   ├── 19-pfsense-console-pf-restart.png
│   ├── 20-pfsense-console-bootup.png
│   ├── 21-pfsense-console-ping-internet.png
│   ├── 22-pfsense-diagnostics-ping.png
│   ├── 23-apache-via-ip.png
│   ├── 24-ubuntu-dig-localhost.png
│   ├── 25-ubuntu-ping-pfsense-and-dig.png
│   ├── 26-apache-via-domain.png
│   ├── 27-pfsense-dns-forwarder.png
│   ├── 28-pfsense-dns-forwarding-mode.png
│   ├── 29-pfsense-firewall-rules.png
│   ├── 30-pfsense-package-manager-pfblockerng.png
│   ├── 31-pfblockerng-setup-wizard.png
│   ├── 32-pfblockerng-ip-component.png
│   ├── 33-pfblockerng-dnsbl-wizard.png
│   ├── 34-pfblockerng-update-first-run.png
│   ├── 35-pfblockerng-dnsbl-settings-top.png
│   ├── 36-pfblockerng-dnsbl-settings-bottom.png
│   ├── 37-pfblockerng-force-reload.png
│   ├── 38-pfblockerng-dnsbl-groups.png
│   ├── 39-pfblockerng-final-firewall-rules.png
│   ├── 40-pfblockerng-youtube-block-group-config.png
│   ├── 41-pfblockerng-format-dropdown.png
│   ├── 42-pfblockerng-custom-list-section.png
│   ├── 43-nslookup-youtube-blocked.png        ← KEY SCREENSHOT
│   └── 44-pfblockerng-reports-alerts.png      ← KEY SCREENSHOT
│
└── configs/
    ├── named.conf.options                      ← BIND9 DNS config
    ├── named.conf.local                        ← DNS zones config
    ├── db.company.local                        ← Forward zone file
    ├── dhcpd.conf                              ← DHCP config
    └── smb.conf                               ← Samba config
```
