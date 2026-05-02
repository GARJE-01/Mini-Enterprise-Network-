# 📁 Mini Enterprise Network — CP-4 & CP-5 Documentation
### Author: Mayur Garje | Continuing from CP-3

---

## ✅ CHECKPOINT 4 — pfSense Routing, Connectivity & DNS Configuration

---

### Screenshot: `17-pfsense-gateway-applied.png`
![Gateway Applied](screenshots/17-pfsense-gateway-applied.png)

### What this shows
Gateway configuration saved and applied successfully — green banner confirms "The changes have been applied successfully." This screenshot also shows a critical difference from the previous gateway screenshots: **Default gateway IPv4 is now explicitly set to WAN_DHCP** (not Automatic).

### What changed and why it matters

| Setting | Before | After |
|---|---|---|
| Default gateway IPv4 | Automatic | **WAN_DHCP** (explicitly set) |
| Gateway IP | 192.168.211.2 | 192.168.211.2 |

Setting this explicitly ensures pfSense always routes internet-bound traffic through the correct WAN gateway — important when troubleshooting routing issues. This was one of the fixes applied during the TTL expired error troubleshooting.

---

### Screenshot: `18-pfsense-console-routing-restart.png`
![pfSense Console Routing Restart](screenshots/18-pfsense-console-routing-restart.png)

### What this shows
The pfSense console (accessed directly on the VM, not via web GUI) showing the output of `service routing restart` — run from the shell (Option 8). This was a **troubleshooting step** taken to fix a routing issue.

### Error messages visible and what they mean

```
route: message indicates error: Address already in use
delete host 127.0.0.1: gateway lo0 fib 0: gateway uses the same route
default    192.168.211.2    -fib 0    done
```

**These are NOT critical failures** — they are normal output when restarting routing on a live system. Here's what each means:

| Message | Meaning | Is it a problem? |
|---|---|---|
| `Address already in use` | Route already exists before restart | No — expected |
| `gateway uses the same route` | Loopback route already registered | No — expected |
| `default 192.168.211.2 done` | Default route set successfully | ✅ Success |
| `route has not been found` | IPv6 routes not yet populated | Minor — IPv6 not configured |
| `route already in table` | Route being re-added after restart | No — expected |

### Why this was done
During initial pfSense setup, internet traffic wasn't routing correctly. Restarting the routing service forced pfSense to re-apply all routing table entries — this fixed the connectivity issue.

### Interview talking point
> "When pfSense showed TTL expired errors, I accessed the console directly and restarted the routing service. Reading the output, I identified that the default route to 192.168.211.2 was being set correctly — the other messages were normal restart noise, not actual failures."

---

### Screenshot: `19-pfsense-console-pf-restart.png`
![pfSense Console PF Restart Error](screenshots/19-pfsense-console-pf-restart.png)

### What this shows
Another console troubleshooting screenshot — this time showing an attempt to restart the `pf` (packet filter / firewall) service and the error it produced.

### The error:
```
service pf restart
Cannot 'restart' pf. Set pf_enable to YES in /etc/rc.conf
or use 'onerestart' instead.
```

### What this means and how it was handled
In pfSense/FreeBSD, the `pf` firewall service cannot be restarted with a standard `restart` command — it requires `onerestart` or must be controlled through the web GUI. This is a FreeBSD-specific behavior, not a bug.

**This is a great troubleshooting story for interviews:**
> "I tried to restart the firewall service via CLI and got an error. I learned that pfSense's pf service works differently from Linux — it uses FreeBSD's rc system which requires `onerestart` instead of `restart`. I used the web GUI's Apply Changes button instead, which handled it correctly."

### What was done instead
Changes were applied through the pfSense web GUI (Firewall → Rules → Apply Changes) which correctly handles pf reloading internally.

---

### Screenshot: `20-pfsense-console-bootup.png`
![pfSense Console Bootup](screenshots/20-pfsense-console-bootup.png)

### What this shows
The pfSense VM console after a reboot — showing the complete boot sequence and the pfSense management menu. This is the **pfSense console interface** (accessed directly on the VM without a browser).

### Key information visible

```
pfSense 2.8.1-RELEASE amd64 20251215-1731
Bootup complete
FreeBSD/amd64 (pfSense.lab.local) (ttyv0)
VMware Virtual Machine — Netgate Device ID: 271423a5b86835b6613a

*** Welcome to pfSense 2.8.1-RELEASE (amd64) on pfSense ***

WAN (wan) → em0 → v4/DHCP4: 192.168.1.101/24
LAN (lan) → em1 → v4: 192.168.20.1/24
```

### Interface assignment at boot
- **WAN (em0):** `192.168.1.101/24` — received via DHCP from VMware NAT
- **LAN (em1):** `192.168.20.1/24` — static, manually configured

### pfSense console menu options visible
This is the full management menu available without a browser:

| Option | Function |
|---|---|
| 1 | Assign Interfaces |
| 2 | Set interface(s) IP address |
| 3 | Reset admin account and password |
| 4 | Reset to factory defaults |
| 5 | Reboot system |
| 8 | **Shell** (used for troubleshooting) |
| 14 | Enable Secure Shell (sshd) |

Option 8 (Shell) is what was used to run the routing and pf restart commands in the previous screenshots.

---

### Screenshot: `21-pfsense-console-ping-internet.png`
![pfSense Console Ping Internet](screenshots/21-pfsense-console-ping-internet.png)

### What this shows
A successful ping to `8.8.8.8` (Google DNS) run directly from the pfSense shell — proving the firewall itself has internet connectivity through the WAN interface.

### Output details
```
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=117 time=50.334 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=60.662 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=117 time=10.263 ms
6 packets transmitted, 6 received, 0.0% packet loss
round-trip min/avg/max/stddev = 10.263/36.448/74.662/26.400 ms
```

### What TTL=117 tells us
TTL starts at 128 (Windows default) or 64 (Linux). Google's servers respond with TTL=128. By the time it reaches pfSense, TTL=117 — meaning the packet passed through approximately 11 hops to reach Google. This confirms real internet routing is happening, not a local loop.

### Why this verification matters
Before configuring DNS enforcement and firewall rules, it was critical to confirm:
1. ✅ pfSense WAN has internet access
2. ✅ NAT is working (private IP reaching public internet)
3. ✅ Routing table is correct

This ping confirmed all three.

---

### Screenshot: `22-pfsense-diagnostics-ping.png`
![pfSense Diagnostics Ping via GUI](screenshots/22-pfsense-diagnostics-ping.png)

### What this shows
The pfSense web GUI Diagnostics → Ping tool — pinging `8.8.8.8` from the WAN interface. This is a second method of verifying internet connectivity, done from the web browser instead of the console.

### Results visible
```
PING 8.8.8.8 (8.8.8.8) from 192.168.1.101: 56 data bytes
64 bytes from 8.8.8.8: icmp_seq=0 ttl=117 time=14.994 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=47.221 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=117 time=7.594 ms
3 packets transmitted, 3 received, 0.0% packet loss
```

### Configuration used
- **Hostname:** 8.8.8.8
- **IP Protocol:** IPv4
- **Source address:** WAN
- **Pings:** 3

**Source IP `192.168.1.101`** — this is pfSense's WAN IP at this point (before the final LAN reconfiguration). Confirms NAT is working — pfSense uses this as the source IP when reaching the internet.

---

### Screenshot: `23-apache-via-ip.png`
![Apache Web Server via IP](screenshots/23-apache-via-ip.png)

### What this shows
The Apache web server page accessed from a browser using the **IP address** `192.168.20.101`. The page displays: **"Welcome to My Network lab"** with subtitle **"Hosted on Ubuntu Server behind pfSense Firewall"**.

### Key details
- **URL:** `http://192.168.20.101` (direct IP access)
- **Status:** ⚠️ "Not secure" — HTTP only (no SSL, expected for a lab)
- **Server:** Ubuntu Server VM running Apache2
- **Behind:** pfSense firewall — traffic flows through the firewall to reach this server

### What "behind pfSense Firewall" means
The Ubuntu server is on the LAN side (`192.168.20.101`). To access it from the Windows host, traffic goes:

```
Windows Browser → pfSense LAN (192.168.20.1) → Ubuntu Server (192.168.20.101)
```

pfSense's firewall rules allow this traffic. This confirms the server is reachable through the firewall — a real enterprise deployment pattern.

---

### Screenshot: `24-ubuntu-dig-localhost.png`
![Ubuntu dig localhost DNS test](screenshots/24-ubuntu-dig-localhost.png)

### What this shows
The Ubuntu server terminal showing a `dig` command testing DNS resolution — this is the **most detailed DNS verification** in the entire project.

### Full dig output analysis
```bash
dig @localhost company.local
```

**Answer received:**
```
;; ANSWER SECTION:
company.local.    604800    IN    A    192.168.20.101
;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(localhost) (UDP)
;; WHEN: Fri May 01 16:48:34 UTC 2026
```

### Breaking down every field

| Field | Value | Meaning |
|---|---|---|
| `company.local.` | Domain queried | Internal zone name |
| `604800` | TTL in seconds | Records cached for 7 days |
| `IN A` | Record type | IPv4 address record |
| `192.168.20.101` | Result | Ubuntu server's IP ✅ |
| `Query time: 0 msec` | Speed | Instant — local DNS |
| `SERVER: 127.0.0.1#53` | DNS server used | BIND9 on localhost |
| `UDP` | Protocol | Standard DNS protocol |

### Warning visible — why it's harmless
```
WARNING: .local is reserved for Multicast DNS
```
This is a known dig warning when querying `.local` domains. It does not affect functionality — it's just informational. The answer is still returned correctly.

### Syslog entries at top of screenshot
```
May 01 16:47:35 enterpriseserver named[2402]: network unreachable resolving './NS/IN'
```
These are BIND9 startup messages showing it tried to reach IPv6 root nameservers but couldn't (no IPv6 configured). IPv4 DNS worked perfectly — these entries are normal in an IPv4-only lab.

---

### Screenshot: `25-ubuntu-ping-pfsense-and-dig.png`
![Ubuntu ping pfSense and dig test](screenshots/25-ubuntu-ping-pfsense-and-dig.png)

### What this shows
Two critical verifications run from the Ubuntu server terminal:

**Test 1 — Ping pfSense LAN:**
```bash
ping 192.168.20.1
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=0.810 ms
5 packets transmitted, 5 received, 0% packet loss
```
✅ Ubuntu server can reach pfSense gateway — routing is correct

**Test 2 — dig DNS from Ubuntu server's own DNS:**
```bash
dig @192.168.20.101 company.local

;; ANSWER SECTION:
company.local.    604800    IN    A    192.168.20.101
;; SERVER: 192.168.20.101#53(192.168.20.101) (UDP)
```
✅ DNS resolves correctly when queried by IP address (not just localhost)

### Why both tests matter
- Ping confirms Layer 3 connectivity between Ubuntu and pfSense
- dig by IP confirms BIND9 is listening on the network interface (not just loopback)
- If dig only worked via `localhost` but not via `192.168.20.101`, Windows clients couldn't use it — this test rules that out

---

### Screenshot: `26-apache-via-domain.png`
![Apache via Domain Name](screenshots/26-apache-via-domain.png)

### What this shows
The Apache web server accessed using the **domain name** `company.local` — not the IP address. This is the most complete DNS proof in the project.

### Why this is the most important Apache screenshot

| Access method | What it proves |
|---|---|
| `http://192.168.20.101` | Apache is running |
| `http://company.local` | **DNS + Apache both working together** |

When the browser resolves `company.local`, it:
1. Asks the DNS server (192.168.20.101) to resolve `company.local`
2. Gets back `192.168.20.101`
3. Connects to Apache on that IP
4. Displays the page

The entire DNS → HTTP chain is working end-to-end.

### URL bar details
- **Address:** `company.local`
- **Security:** "Not secure" (HTTP, no SSL — expected for lab)
- **Page content:** "Welcome to My Network lab — Hosted on Ubuntu Server behind pfSense Firewall"

---

### Screenshot: `27-pfsense-dns-forwarder.png`
![pfSense DNS Forwarder Settings](screenshots/27-pfsense-dns-forwarder.png)

### What this shows
pfSense DNS Forwarder (dnsmasq) configuration — the service that handles DNS queries from LAN clients and forwards them upstream.

### Configuration visible

| Setting | Value | Purpose |
|---|---|---|
| Enable | ✅ Enabled | DNS forwarder is active |
| DHCP Registration | ☐ Disabled | Don't auto-register DHCP hostnames |
| Static DHCP | ☐ Disabled | Don't register static DHCP mappings |
| Prefer DHCP | ☐ Disabled | Use manual records first |
| Ignore System DNS | ☐ Disabled | Use system DNS servers (8.8.8.8, 1.1.1.1) |
| Query DNS sequentially | ☐ Disabled | Query all DNS servers in parallel |

### How this fits into the full DNS chain
```
LAN Client → pfSense DNS Forwarder (192.168.20.1:53)
           → Forwards to Ubuntu BIND9 (for company.local)
           → Forwards to 8.8.8.8 (for external domains)
```

---

### Screenshot: `28-pfsense-dns-forwarding-mode.png`
![pfSense DNS Forwarding Mode Enabled](screenshots/28-pfsense-dns-forwarding-mode.png)

### What this shows
The DNS Resolver settings showing **DNS Query Forwarding enabled** — the critical setting that makes pfSense forward DNS queries to upstream servers instead of resolving them directly.

### Setting visible
```
DNS Query Forwarding: ✅ Enable Forwarding Mode
```

**What this does:** When a client asks pfSense to resolve `google.com`, pfSense forwards the query to `8.8.8.8` instead of going to the root nameservers directly. For `company.local`, it uses the domain override to forward to Ubuntu BIND9 at `192.168.20.101`.

### Why this setting was needed
Without forwarding mode enabled, pfSense's DNS resolver would try to resolve everything itself by querying root nameservers — which sometimes fails in NAT environments. Enabling forwarding made external DNS resolution reliable.

---

### Screenshot: `29-pfsense-firewall-rules.png`
![pfSense Firewall Rules](screenshots/29-pfsense-firewall-rules.png)

### What this shows
The pfSense LAN firewall rules — showing all active rules controlling traffic on the internal network. This is the **security enforcement layer** of the entire project.

### Rules table — full breakdown

| # | State | Protocol | Source | Destination | Port | Description | Status |
|---|---|---|---|---|---|---|---|
| 1 | ✅ Active | * (any) | * | LAN Address | 443, 80 | Anti-Lockout Rule | 🔒 Built-in |
| 2 | ❌ Disabled | IPv4 TCP/UDP | * | BLOCK_SITES | * | Block Social Media | 🔴 Disabled |
| 3 | ✅ Active | IPv4 * | LAN subnets | * | * | Default allow LAN to any | ✅ Active |
| 4 | ✅ Active | IPv6 * | LAN subnets | * | * | Default allow LAN IPv6 to any | ✅ Active |

### Rule-by-rule explanation

**Rule 1 — Anti-Lockout Rule (built-in, cannot delete)**
Ensures you can always reach the pfSense web GUI from the LAN. If all other rules accidentally block traffic, this rule guarantees access to ports 443 and 80 on the LAN address. This is pfSense's safety net.

**Rule 2 — Block Social Media (currently disabled, shown with ❌)**
This rule references an alias called `BLOCK_SITES`. The alias contains the IP addresses or domains of sites to block. It was created but **disabled** at this point — DNS-based blocking via pfBlockerNG was used instead, which is more effective (blocks at DNS level before any connection is made).

**Rule 3 — Default allow LAN to any (active)**
pfSense's default rule allowing all LAN devices to reach any destination. This is what gives LAN clients internet access. Traffic flows: LAN client → pfSense → WAN → Internet.

**Rule 4 — Default allow LAN IPv6**
Same as Rule 3 but for IPv6 traffic.

### Interview talking point
> "I created a firewall rule alias called BLOCK_SITES for IP-based blocking, but discovered DNS-level blocking with pfBlockerNG is more effective — it stops the connection before it even starts, rather than blocking packets mid-connection. So I disabled the IP rule and relied on DNSBL instead."

---

## ✅ CHECKPOINT 5 — pfBlockerNG Installation & Content Filtering

---

### Screenshot: `30-pfsense-package-manager-pfblockerng.png`
![pfBlockerNG in Package Manager](screenshots/30-pfsense-package-manager-pfblockerng.png)

### What this shows
pfSense Package Manager showing the search results for "pfblockerng" — the step where pfBlockerNG was found and installed.

### Two versions available

| Package | Version | Description |
|---|---|---|
| **pfBlockerNG** | 3.2.8 | Stable release — this is what was installed |
| pfBlockerNG-devel | 3.2.14_1 | Development/beta version |

### What pfBlockerNG does (from the package description)
- **IP blocking:** Manage IPv4/v6 deny/permit lists
- **GeoIP blocking:** Block entire countries using MaxMind database
- **DNSBL:** Domain Name Block List — blocks domains at DNS level
- **Reputation filtering:** Advanced threat intelligence integration

### Why pfBlockerNG was chosen over simple firewall rules
Simple firewall rules block by IP address — but YouTube has thousands of IPs that change constantly. pfBlockerNG's DNSBL blocks at the **DNS layer** — when any device asks "what is youtube.com's IP?", pfBlockerNG intercepts and returns a fake IP (sinkhole) instead. The device never even gets the real IP to connect to.

### Installation path
```
System → Package Manager → Available Packages → Search "pfblockerng" → Install
```

---

### Screenshot: `31-pfblockerng-setup-wizard.png`
![pfBlockerNG Setup Wizard](screenshots/31-pfblockerng-setup-wizard.png)

### What this shows
The pfBlockerNG setup wizard — the first screen after installation. pfBlockerNG has its own wizard separate from pfSense's main wizard.

### Wizard details
- **Title:** Wizard / pfBlockerNG Setup
- **Purpose:** Configure an entry-level IP and DNSBL configuration
- **Note:** The wizard can be skipped for manual configuration

### What the wizard configures
1. Enable pfBlockerNG
2. Select interfaces for IP blocking
3. Configure DNSBL (DNS Block List)
4. Set up feed sources
5. Apply initial configuration

The wizard was used to set up the initial DNSBL configuration, then custom domain entries were added manually for YouTube and related domains.

---

## 🔖 CHECKPOINT 5 PARTIAL — More pfBlockerNG screenshots needed

**What's documented so far in this batch:** 16 screenshots across CP-4 and CP-5 start

**What's needed in the next batch:**
- pfBlockerNG DNSBL custom list with YouTube domains entered
- pfBlockerNG update/reload screen
- nslookup youtube.com showing 10.10.10.1 (the blocked result)
- pfBlockerNG reports/logs showing blocked queries
- Any remaining Ubuntu or pfSense screenshots

---

## 🔁 CHECKPOINT RESUME CARD
*Save this — paste it in a new chat to continue*

```
I am documenting my Mini Enterprise Network CCNA project with Claude.

AUTHOR: Mayur Garje
PROJECT: Mini Enterprise Network with Server Infrastructure

CHECKPOINTS COMPLETED:
✅ CP-0: Base README written
✅ CP-1: Packet Tracer topology documented
✅ CP-2: Ubuntu DNS + Samba documented
✅ CP-3: pfSense install, dashboard, DHCP, gateways
✅ CP-4: pfSense routing, console troubleshooting, connectivity,
         Apache (IP + domain), DNS dig tests, DNS forwarder,
         firewall rules (BLOCK_SITES alias + default rules)
🔄 CP-5: pfBlockerNG started — package install + wizard documented

SCREENSHOTS USED SO FAR: 31 of ~44
SCREENSHOTS REMAINING: ~13

NEXT BATCH NEEDED:
- pfBlockerNG DNSBL custom list (YouTube domains)
- pfBlockerNG update/reload confirmation
- nslookup youtube.com → 10.10.10.1 (blocked proof)
- pfBlockerNG reports/logs
- Any remaining screenshots

DOCUMENTS PRODUCED:
1. README.md (base)
2. CP1-CP3-documentation.md
3. CP4-CP5-documentation.md (this file)

Continue from: CP-5 pfBlockerNG DNSBL + YouTube block proof
```
