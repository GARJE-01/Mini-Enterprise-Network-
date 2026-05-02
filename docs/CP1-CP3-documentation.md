# 📁 Mini Enterprise Network — Detailed Documentation
### Author: Mayur Garje | Date: May 2026

---

## ✅ CHECKPOINT 1 — Network Topology (Packet Tracer)

### Screenshot: `01-packet-tracer-topology.png`

![Packet Tracer Topology](screenshots/01-packet-tracer-topology.png)

### What this shows
The full network topology built in Cisco Packet Tracer simulating a real enterprise network. This is the foundation of the entire project — every VLAN, every connection, and every routing decision starts here.

### Topology breakdown

| Device | Role | Connections |
|---|---|---|
| ISP Router | Simulates internet provider | Connected to R1 via Gig0/0 |
| R1 (Router 2911) | Core router — inter-VLAN routing + NAT | Gig0/1 to ISP, Gig0/0 to Switch |
| SW1 (Switch 2960) | Layer 2 switch — carries all VLANs | Gig0/1 from R1, Fa0/1–Fa0/4 to PCs |
| PC1 | Office VLAN 10 workstation | Fa0/1 on Switch |
| PC2 | Server VLAN 20 workstation | Fa0/2 on Switch |
| PC3 | Guest VLAN 30 workstation | Fa0/3 on Switch |
| PC4 | Management VLAN 40 workstation | Fa0/4 on Switch |

### VLAN Design

| VLAN ID | Name | Color in diagram | Subnet | Purpose |
|---|---|---|---|---|
| 10 | Office | Cyan | 192.168.10.0/24 | Employee workstations |
| 20 | Server | Orange | 192.168.20.0/24 | Internal servers |
| 30 | Guest | Pink | 192.168.30.0/24 | Guest internet-only access |
| 40 | Management | Green | 192.168.40.0/24 | Network device management |

### Key configurations applied

**Router R1 — Sub-interfaces (Router-on-a-stick):**
```
interface GigabitEthernet0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0

interface GigabitEthernet0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0

interface GigabitEthernet0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0

interface GigabitEthernet0/0.40
 encapsulation dot1Q 40
 ip address 192.168.40.1 255.255.255.0
```

**Switch SW1 — Trunk port to Router:**
```
interface GigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40

interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10

interface FastEthernet0/2
 switchport mode access
 switchport access vlan 20

interface FastEthernet0/3
 switchport mode access
 switchport access vlan 30

interface FastEthernet0/4
 switchport mode access
 switchport access vlan 40
```

**NAT/PAT — for internet simulation:**
```
ip nat inside source list 1 interface GigabitEthernet0/1 overload
access-list 1 permit 192.168.0.0 0.0.255.255
```

**ACL — Block Guest VLAN from reaching Server VLAN:**
```
ip access-list extended GUEST-RESTRICT
 deny ip 192.168.30.0 0.0.0.255 192.168.20.0 0.0.0.255
 permit ip any any
```

### Why this matters (interview talking point)
> "I designed a 4-VLAN topology using router-on-a-stick — a single physical link between router and switch carries all VLAN traffic tagged with 802.1Q. This is the standard approach enterprises use when they don't have a Layer 3 switch."

---

## ✅ CHECKPOINT 2 — Ubuntu Server (DNS + Samba Verified)

---

### Screenshot: `02-dns-ping-myserver-local.png`

![DNS Ping Test](screenshots/02-dns-ping-myserver-local.png)

### What this shows
A successful ping from the Windows host (`C:\Users\MAYUR`) to the domain name `myserver.local` — resolving correctly to `192.168.211.133`. This proves the BIND9 DNS server is working and resolving internal domain names.

### Why this is significant
- The PC is not pinging an IP address — it's pinging a **hostname**
- DNS is translating `myserver.local` → `192.168.211.133` automatically
- 4/4 packets received, 0% loss, average 0ms (VM on same host = near-zero latency)
- TTL=64 confirms the reply is coming from a Linux machine (Linux default TTL)

### What was configured to make this work

**BIND9 forward zone (`/etc/bind/db.mydomain`):**
```
myserver    IN  A   192.168.211.133
www         IN  A   192.168.211.133
```

**Windows CMD proof:**
```
C:\Users\MAYUR> ping myserver.local
Pinging myserver.local [192.168.211.133]
Reply from 192.168.211.133: bytes=32 time=1ms TTL=64
Packets: Sent=4, Received=4, Lost=0 (0% loss)
```

### Interview talking point
> "I verified DNS resolution end-to-end — from a Windows client pinging a Linux server by hostname, not by IP. This confirms the DNS server is authoritative for the internal zone and clients are using it correctly."

---

### Screenshot: `03-dns-nslookup-google.png`

![DNS External Resolution](screenshots/03-dns-nslookup-google.png)

### What this shows
`nslookup google.com` run from Windows, using the Ubuntu DNS server (`192.168.211.133`) as the resolver. Google.com resolves successfully — proving DNS **forwarding** to external servers (8.8.8.8) is working.

### Key details visible
- **Server:** Unknown (this is normal — the server has no PTR record for itself yet)
- **Address:** `192.168.211.133` — confirming our Ubuntu DNS server is being queried
- **Result:** `google.com` resolves to `142.250.70.110` (real Google IP) ✅
- IPv6 address also returned (`2404:6800:...`) — confirms full DNS resolution

### What this proves
The DNS server does two jobs simultaneously:
1. Resolves **internal** names (myserver.local → 192.168.211.133)
2. Forwards **external** names (google.com → real internet IP via 8.8.8.8)

This is exactly how enterprise DNS works — internal zone authority + external forwarding.

### Configuration that enables forwarding (`/etc/bind/named.conf.options`):
```
forwarders {
    8.8.8.8;
    8.8.4.4;
};
forward only;
```

---

### Screenshot: `04-samba-windows-access.png`

![Samba File Access from Windows](screenshots/04-samba-windows-access.png)

### What this shows
Windows File Explorer accessing the Samba network share at `\\192.168.211.133\CompanyShare`. A file named `test` (Text Document, 1KB, created 01-05-2026 at 17:02) is visible inside the shared folder — proving file sharing between Linux and Windows is fully working.

### Key details visible
- **Path in address bar:** `Network > 192.168.211.133 > CompanyShare`
- **Share name:** `CompanyShare`
- **File:** `test.txt` — created on the Linux server, visible from Windows
- **Date:** 01-05-2026 17:02 — this is a real file created during testing

### Samba configuration that made this work (`/etc/samba/smb.conf`):
```ini
[CompanyShare]
   path = /srv/samba/shared
   browseable = yes
   writable = yes
   guest ok = yes
   read only = no
```

### Access from Windows:
```
In File Explorer address bar:
\\192.168.211.133\CompanyShare

Or via Run dialog (Win+R):
\\192.168.211.133
```

### Interview talking point
> "I configured a Samba file server on Linux that Windows clients can access natively through File Explorer — no additional software needed on the Windows side. This is the same technology companies use for shared network drives."

---

### Screenshot: `05-samba-linux-server-side.png`

![Samba Server Side](screenshots/05-samba-linux-server-side.png)

### What this shows
The Ubuntu server terminal showing the Samba shared directory and its contents from the Linux side — confirming the same `test.txt` file that Windows saw is physically stored at `/srv/samba/shared/`.

### Commands visible and what they mean

```bash
# Check directory permissions
ls -ld /srv/samba/shared
# Output: drwxrwxrwx 2 root root 4096 May 1 11:14 /srv/samba/shared
# drwxrwxrwx = full read/write/execute for all users (777)

# List contents of share
ls /srv/samba/shared
# Output: test.txt
```

### What `drwxrwxrwx` means
- `d` = directory
- `rwx` = owner (root) has read, write, execute
- `rwx` = group has read, write, execute  
- `rwx` = everyone has read, write, execute
- This is what allows Windows clients to read and write files

### Hostname visible
`netadmin@enterpriseserver` — showing the server hostname was properly set to `enterpriseserver` and the admin user is `netadmin`.

---

## 🔖 CHECKPOINT 2 COMPLETE

**What is documented so far:**
- ✅ CP-1: Packet Tracer topology — 4 VLANs, routing, trunking, NAT, ACLs
- ✅ CP-2: Ubuntu DNS verified (internal + external resolution)
- ✅ CP-2: Samba file server verified (Linux side + Windows client side)

**Screenshots documented:** 5 of 16 in this batch

---

## ✅ CHECKPOINT 3 — pfSense Installation & Setup

---

### Screenshot: `06-pfsense-login.png`

![pfSense Login](screenshots/06-pfsense-login.png)

### What this shows
The pfSense web GUI login page — accessed from the Windows host browser after pfSense VM was installed and configured. This confirms pfSense is reachable and the web interface is running.

### Key details
- **URL accessed:** `https://192.168.20.1` (pfSense LAN IP)
- **Username:** `admin`
- **Product:** pfSense Community Edition (free, open source)
- **Maintained by:** Netgate

### What pfSense is
pfSense is a full enterprise-grade firewall operating system based on FreeBSD. It is used by small businesses, ISPs, and even some large enterprises as a cost-effective alternative to Cisco ASA or Fortinet firewalls. Running pfSense in this project is equivalent to managing a real firewall.

---

### Screenshot: `07-pfsense-setup-wizard.png`

![pfSense Setup Wizard](screenshots/07-pfsense-setup-wizard.png)

### What this shows
The pfSense initial setup wizard — the first screen seen after logging in for the first time. The yellow warning at the top is a security notice that the default password is still set (best practice is to change it immediately).

### Setup wizard steps completed
The wizard walked through 9 steps:
1. Welcome screen (this screenshot)
2. Netgate registration
3. **General Information** — hostname, domain, DNS servers
4. Time server configuration
5. **WAN Interface** — set to DHCP (gets IP from VMware NAT)
6. **LAN Interface** — set to 192.168.20.1/24
7. Set admin password
8. Reload configuration
9. **Wizard completed** ✅

---

### Screenshot: `08-pfsense-general-info.png`

![pfSense General Info](screenshots/08-pfsense-general-info.png)

### What this shows
pfSense General Information configuration screen from the setup wizard — where the firewall's identity and DNS settings are defined.

### Configuration set here

| Setting | Value | Reason |
|---|---|---|
| Hostname | `pfSense` | Name of this firewall |
| Domain | `lab.local` | Internal domain suffix |
| Primary DNS | `8.8.8.8` | Google DNS for external resolution |
| Secondary DNS | `1.1.1.1` | Cloudflare DNS as backup |
| Override DNS | ✅ Checked | Allows ISP/DHCP to update DNS if needed |

### Important note about DNS here
At this stage, pfSense is pointing to 8.8.8.8 and 1.1.1.1 for its own DNS. Later in the project, DNS enforcement firewall rules were added to **prevent clients from using these directly** — they must go through pfSense only.

---

### Screenshot: `09-pfsense-lan-config.png`

![pfSense LAN Configuration](screenshots/09-pfsense-lan-config.png)

### What this shows
The LAN interface configuration screen — where pfSense's internal IP address is set. This IP becomes the **default gateway** for all devices on the LAN.

### Configuration set

| Setting | Value |
|---|---|
| LAN IP Address | `192.168.20.1` |
| Subnet Mask | `/24` (255.255.255.0) |

### Why 192.168.20.1?
This matches VLAN 20 (Servers) subnet from the Packet Tracer design — keeping the project consistent. All LAN clients will use `192.168.20.1` as their default gateway and DNS server.

---

### Screenshot: `10-pfsense-wizard-complete.png`

![pfSense Wizard Complete](screenshots/10-pfsense-wizard-complete.png)

### What this shows
pfSense setup wizard completed successfully — Step 9 of 9. The green progress bar at the top confirms all 9 configuration steps were completed. pfSense is now fully configured and operational as a firewall.

### What was achieved at this point
- ✅ Firewall is online
- ✅ WAN interface connected to internet (via VMware NAT)
- ✅ LAN interface configured at 192.168.20.1
- ✅ Basic firewall rules automatically created by wizard
- ✅ NAT automatically configured (LAN → WAN)

---

### Screenshot: `11-pfsense-dashboard-sysinfo.png`

![pfSense Dashboard System Info](screenshots/11-pfsense-dashboard-sysinfo.png)

### What this shows
The pfSense dashboard — the main control panel showing full system information of the running firewall.

### Key system details visible

| Field | Value | What it means |
|---|---|---|
| Name | `pfSense.lab.local` | Firewall hostname + domain |
| User | `admin@192.168.20.10` | Logged in from Ubuntu server IP |
| System | VMware Virtual Machine | Confirms running in VMware |
| Version | `2.8.1-RELEASE (amd64)` | Latest stable pfSense version |
| Built | Mon Dec 15 2025 | Recent build |
| CPU | Intel Core i5-11400H @ 2.70GHz | Host PC's CPU passed through |
| Uptime | 1 Hour 1 Minute 3 Seconds | Freshly configured |
| Date/Time | Fri May 1 19:54:27 IST 2026 | Project date confirmed |
| DNS Servers | 127.0.0.1, ::1 | Using its own resolver (correct) |

### Interface status (top right)
- **WAN** ↑ `192.168.211.134` — connected to internet ✅
- **LAN** ↑ `192.168.20.1` — serving internal network ✅

Both interfaces are UP (green arrows) — firewall is fully operational.

---

### Screenshot: `12-pfsense-dashboard-interfaces.png`

![pfSense Dashboard Interfaces](screenshots/12-pfsense-dashboard-interfaces.png)

### What this shows
The lower half of the pfSense dashboard showing resource usage and interface summary.

### Resource usage at time of screenshot

| Resource | Usage | Notes |
|---|---|---|
| CPU | 1% | Near-idle — firewall is efficient |
| Memory | 41% of 967 MiB | ~400MB used — healthy |
| SWAP | 0% of 1024 MiB | No swap needed — plenty of RAM |
| Disk | 899MB used of 16GB (6%) | Plenty of space |

### Interface confirmation
- **WAN:** `192.168.211.134` — public-facing IP (VMware NAT)
- **LAN:** `192.168.20.1` — internal network gateway

### DNS Servers shown: `127.0.0.1, ::1, 8.8.8.8, 1.1.1.1`
pfSense is using its own local resolver (127.0.0.1) as primary — then forwarding to Google/Cloudflare for external names.

---

### Screenshot: `13-pfsense-dhcp-config.png`

![pfSense DHCP Configuration](screenshots/13-pfsense-dhcp-config.png)

### What this shows
pfSense DHCP Server configuration for the LAN interface — showing the address pool that automatically assigns IPs to all connected devices.

### DHCP Configuration set

| Setting | Value |
|---|---|
| DHCP Backend | ISC DHCP |
| Enable | ✅ Enabled on LAN interface |
| Subnet | 192.168.20.0/24 |
| Subnet Range | 192.168.20.1 – 192.168.20.254 |
| Address Pool | **192.168.20.100 – 192.168.20.199** |

### What this means practically
Any device connecting to the LAN will automatically receive:
- An IP address between `.100` and `.199`
- Default gateway: `192.168.20.1` (pfSense)
- DNS server: `192.168.20.1` (pfSense — enforced)

The range `.100` to `.199` leaves `.1` to `.99` free for static IP assignments (servers, printers, network devices).

---

### Screenshot: `14-pfsense-dhcp-applied.png`

![pfSense DHCP Applied Successfully](screenshots/14-pfsense-dhcp-applied.png)

### What this shows
Confirmation that DHCP server configuration was saved and applied successfully — the green banner reads "The changes have been applied successfully."

### Notable warning visible
The yellow warning reads: *"ISC DHCP has reached end-of-life and will be removed in a future version of pfSense."*

This is an informational notice — ISC DHCP still works fully in pfSense 2.8.1. Future versions will use Kea DHCP as the replacement. This is the kind of real-world detail that shows you're working with actual enterprise software — not a textbook.

---

### Screenshot: `15-pfsense-gateways.png` & `16-pfsense-gateways-2.png`

![pfSense Gateways](screenshots/15-pfsense-gateways.png)

### What this shows
The pfSense routing table — showing configured gateways for WAN connectivity.

### Gateways configured

| Name | Interface | Gateway IP | Monitor IP | Description |
|---|---|---|---|---|
| WAN_DHCP | WAN | 192.168.211.2 | 192.168.211.2 | VMware NAT gateway (IPv4) |
| WAN_DHCP6 | WAN | dynamic | dynamic | IPv6 gateway (auto) |

### What 192.168.211.2 is
This is the **VMware NAT gateway** — the virtual router VMware creates for NAT-mode VMs. All internet-bound traffic from pfSense exits through this IP. This simulates how a real router connects to an ISP's gateway.

### Default gateway behavior
Both IPv4 and IPv6 are set to **Automatic** — pfSense selects the best gateway automatically. The WAN_DHCP gateway (192.168.211.2) is marked as the default ✅ (globe icon).

### Interview talking point
> "pfSense learned its WAN gateway automatically via DHCP from VMware NAT — just like a real firewall would learn its upstream gateway from an ISP. The LAN side I configured manually at 192.168.20.1 to match our internal network design."

---

## 🔖 CHECKPOINT 3 COMPLETE

**Progress so far:**
- ✅ CP-1: Packet Tracer — 4 VLANs, routing, trunking, NAT, ACLs
- ✅ CP-2: Ubuntu DNS (internal + external) + Samba (Linux + Windows verified)
- ✅ CP-3: pfSense install, wizard, dashboard, DHCP, gateways

**Screenshots documented in this batch:** 16 of 16 ✅

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
✅ CP-2: Ubuntu DNS + Samba documented (5 screenshots)
✅ CP-3: pfSense setup, dashboard, DHCP, gateways (11 screenshots)

NEXT BATCH NEEDED (send these screenshots):
- pfSense DNS Resolver settings
- pfSense firewall rules (DNS enforcement)
- pfBlockerNG installation
- pfBlockerNG DNSBL custom list
- YouTube blocked nslookup result (nslookup youtube.com → 10.10.10.1)
- Apache webpage in browser
- Any remaining Ubuntu terminal screenshots

SCREENSHOTS REMAINING: approximately 28 more
DOCUMENTS PRODUCED SO FAR:
1. README.md (base)
2. CP1-CP2-documentation.md (this file)

Continue from CP-4: pfSense DNS Enforcement + pfBlockerNG
```
