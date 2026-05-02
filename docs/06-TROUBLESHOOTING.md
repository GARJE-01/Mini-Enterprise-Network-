# 🔧 Troubleshooting Log

### Mini Enterprise Network — Mayur Garje

---

## Overview

This document records every error, problem, and unexpected behaviour
encountered during the entire project — from Ubuntu Server setup through
pfSense configuration and pfBlockerNG content filtering.

Real troubleshooting experience is one of the most valuable things a
network engineer can demonstrate. Anyone can follow a tutorial that works
perfectly. What separates a capable engineer is the ability to read an
error, understand its root cause, and apply the right fix.

Each entry follows this structure:

- **What happened** — the exact symptom or error
- **When it happened** — which part of the project
- **Root cause** — why it actually happened
- **Fix applied** — exactly what was done
- **Lesson learned** — what this teaches about the system

---

## Error Index

| #   | Error                                         | Component       | Severity |
| --- | --------------------------------------------- | --------------- | -------- |
| 1   | VMware copy-paste greyed out                  | VMware / Ubuntu | Low      |
| 2   | bind9.service alias refused                   | BIND9 DNS       | Medium   |
| 3   | DNS request timed out from Windows            | BIND9 DNS       | High     |
| 4   | IPv6 network unreachable in syslog            | BIND9 DNS       | Low      |
| 5   | DHCP not assigning IPs to clients             | ISC-DHCP        | High     |
| 6   | pfSense TTL expired routing error             | pfSense routing | High     |
| 7   | service pf restart failed                     | pfSense FreeBSD | Medium   |
| 8   | pfSense routing restart error messages        | pfSense routing | Medium   |
| 9   | pfBlockerNG DNSBL Source Definition error     | pfBlockerNG     | High     |
| 10  | YouTube not blocked after saving Custom_List  | pfBlockerNG     | High     |
| 11  | Database Sanity check FAILED                  | pfBlockerNG     | Low      |
| 12  | nslookup DNS timeout after adding block rules | DNS Enforcement | High     |

---

## Error 1 — VMware Copy-Paste Greyed Out

### Component

VMware Workstation Pro / Ubuntu Server VM

### What happened

After installing Ubuntu Server in VMware, copy-paste between the
Windows host and the Ubuntu VM was completely greyed out and non-functional.
Commands could not be pasted into the Ubuntu terminal from Windows —
every command had to be typed manually character by character.

### Symptom

```
VMware menu: VM → Settings → Options → Guest Isolation
Copy and Paste options: greyed out, cannot enable
```

### Root cause

VMware's copy-paste feature between host and guest requires
**VMware Tools** (or the open-source equivalent `open-vm-tools`)
to be installed inside the guest VM. Without VMware Tools,
the VM runs in isolation with no clipboard integration,
no automatic screen resolution adjustment, and no drag-and-drop.

Ubuntu Server installs with no VMware Tools by default — it is
not included in the base ISO.

### Fix applied

```bash
sudo apt install open-vm-tools open-vm-tools-desktop -y
sudo reboot
```

After reboot, copy-paste between Windows host and Ubuntu VM
worked immediately.

### Why two packages

- `open-vm-tools` — core VMware tools (clipboard, time sync, etc.)
- `open-vm-tools-desktop` — GUI extensions (not strictly needed
  for server but ensures full compatibility)

### Lesson learned

Always install VMware Tools immediately after creating any VM.
On Ubuntu: `open-vm-tools`. On CentOS/RHEL: `open-vm-tools`.
On Windows guest: VMware Tools installer from VM menu.
This is one of the first steps in any VMware deployment.

---

## Error 2 — bind9.service Alias Refused

### Component

BIND9 DNS Server — Ubuntu Server

### What happened

After installing BIND9 and trying to restart the service,
the standard command returned an error.

### Exact error

```bash
$ sudo systemctl restart bind9
Failed to restart bind9.service: Unit bind9.service is an
alias for named.service.
Refusing to operate on alias name: bind9.service
```

### Root cause

On Ubuntu 22.04, BIND9 installs as a service named `named`
internally (the actual daemon name). The name `bind9` is
registered as a **systemd alias** — a pointer to `named.service`.

In newer versions of systemd (which Ubuntu 22.04 uses),
operating directly on alias names is restricted. systemd
requires using the real unit name for restart, stop, and
enable operations.

### Fix applied

Use `named` instead of `bind9` for all service commands:

```bash
# Wrong (alias — fails on Ubuntu 22.04)
sudo systemctl restart bind9

# Correct (real unit name — works)
sudo systemctl restart named
sudo systemctl enable named
sudo systemctl status named
```

### Verification

```bash
sudo systemctl status named
```

Expected:

```
● named.service - BIND Domain Name Server
     Loaded: loaded (/lib/systemd/system/named.service)
     Active: active (running)
```

### Lesson learned

Service names in Linux are not always the same as package names.
The package is `bind9` but the daemon is `named`. This is
a common point of confusion — always check `systemctl status`
to see the actual unit name if a service command fails.

Other examples of this pattern:

- Package `apache2` → daemon `apache2` (matches)
- Package `mysql-server` → daemon `mysql` (differs)
- Package `bind9` → daemon `named` (differs)

---

## Error 3 — DNS Request Timed Out From Windows

### Component

BIND9 DNS Server — Ubuntu Server

### What happened

After configuring BIND9 and trying to test DNS resolution
from the Windows host, `nslookup` consistently timed out
with no answer.

### Exact error

```
C:\Users\MAYUR> nslookup company.local 192.168.20.101
DNS request timed out.
    timeout was 2 seconds.
DNS request timed out.
    timeout was 2 seconds.
*** Request to 192.168.20.101 timed-out
```

### Root cause

BIND9 was configured to listen only on `127.0.0.1` (loopback)
by default. When `named.conf.options` does not specify a
`listen-on` directive, BIND9 typically listens on all interfaces —
but the default configuration installed by Ubuntu restricts it
to localhost only for security.

Windows clients sending DNS queries to `192.168.20.101:53`
were receiving no response because BIND9 was not listening
on that IP address.

The query reached the Ubuntu server's network interface but
was silently dropped before reaching BIND9.

### Diagnosis

```bash
# Check what ports BIND9 is listening on
sudo ss -ulnp | grep :53
```

Output before fix:

```
UNCONN  0  0  127.0.0.1:53  0.0.0.0:*  users:(("named",pid=1234))
```

Only `127.0.0.1` visible — not `192.168.20.101`.

### Fix applied

Added explicit `listen-on` directive to `/etc/bind/named.conf.options`:

```
options {
    directory "/var/cache/bind";
    allow-query { 192.168.0.0/16; localhost; };
    forwarders { 8.8.8.8; 8.8.4.4; };
    forward only;
    dnssec-validation auto;

    // Added this line:
    listen-on { 127.0.0.1; 192.168.20.101; };
};
```

Restarted BIND9:

```bash
sudo systemctl restart named
```

Verified listening interfaces after fix:

```bash
sudo ss -ulnp | grep :53
```

Output after fix:

```
UNCONN  0  0  127.0.0.1:53       0.0.0.0:*  users:(("named"))
UNCONN  0  0  192.168.20.101:53  0.0.0.0:*  users:(("named"))
```

Both addresses now listening. Windows DNS immediately worked.

### Lesson learned

Always verify what interface a service is listening on — not
just whether the service is running. A service can be `active
(running)` in systemctl but still not accessible from the
network if it is only bound to localhost.

The `ss -ulnp` command (show UDP listening ports with process)
is essential for diagnosing DNS, DHCP, and other UDP service
binding issues.

---

## Error 4 — IPv6 Network Unreachable in Syslog

### Component

BIND9 DNS Server — Ubuntu Server

### What happened

After starting BIND9, the syslog showed a stream of error messages
about IPv6 nameservers being unreachable.

### Exact error (from syslog / terminal)

```
May 01 16:47:35 enterpriseserver named[2402]: network unreachable
resolving './NS/IN': 2001:500:1::53#53
May 01 16:47:35 enterpriseserver named[2402]: network unreachable
resolving './NS/IN': 2001:503:c27::2:3#53
May 01 16:47:35 enterpriseserver named[2402]: network unreachable
resolving './NS/IN': 2001:7fd::1#53
```

![24-ubuntu-dig-localhost](../screenshots/23.png)

_(Syslog entries visible at top of terminal before the dig command)_

### Root cause

BIND9's `forward only` option causes it to forward external
queries to 8.8.8.8 (IPv4). However, BIND9 also attempts to
query the DNS root nameservers for DNSSEC validation — and
many root nameservers have IPv6 addresses.

Since the Ubuntu VM had no IPv6 connectivity configured,
BIND9 could not reach these IPv6 root nameserver addresses.
The error is generated each time BIND9 tries and fails to
reach an IPv6 address.

### Whether this is a real problem

**No.** These messages are informational only. IPv4 DNS
resolution worked perfectly throughout. The `forward only`
directive meant all resolution was done via 8.8.8.8 (IPv4),
which succeeded. The IPv6 root server attempts were a
secondary process that failed silently without affecting
DNS functionality.

### Fix options

**Option A (applied — easiest):**
No fix needed. Monitor to confirm IPv4 DNS works. Ignore
IPv6 syslog noise.

**Option B (cleaner — suppresses the messages):**
Add `filter-aaaa-on-v4 yes;` to named.conf.options to
prevent BIND9 from trying IPv6 at all.

**Option C (proper fix for production):**
Configure IPv6 on the VM and ensure IPv6 routing is available.

### Lesson learned

Syslog errors do not always mean something is broken.
Reading the error message carefully — `network unreachable
resolving` for IPv6 addresses — immediately indicates this
is an IPv6 connectivity issue, not a DNS configuration
failure. Always test the actual functionality (DNS resolution)
rather than treating every log line as a critical failure.

---

## Error 5 — DHCP Not Assigning IPs to Clients

### Component

ISC-DHCP Server — Ubuntu Server

### What happened

After installing and starting `isc-dhcp-server`, client devices
on the network were not receiving automatic IP addresses.
The DHCP server status showed active but no leases were being
created.

### Symptom

```bash
sudo systemctl status isc-dhcp-server
# Shows: active (running)

# But on client:
ipconfig
# Shows: 169.254.x.x (APIPA — no DHCP response received)
```

### Root cause

The ISC-DHCP server was not configured to listen on a specific
network interface. Without this setting, DHCP tries to listen
on all interfaces including the loopback (`lo`) interface.
On Ubuntu 22.04, this causes the service to either fail
silently or listen on the wrong interface.

The DHCP discover broadcast from clients was arriving on
`ens33` but DHCP was not listening there.

### Diagnosis

```bash
# Check the defaults file
cat /etc/default/isc-dhcp-server
```

Output before fix:

```
INTERFACESv4=""
```

Empty — no interface specified.

### Fix applied

Edit the defaults file to specify the correct interface:

```bash
sudo nano /etc/default/isc-dhcp-server
```

Changed:

```
INTERFACESv4="ens33"
```

Restarted the service:

```bash
sudo systemctl restart isc-dhcp-server
sudo systemctl status isc-dhcp-server
```

### Verification

```bash
# Watch for DHCP activity
sudo tail -f /var/log/syslog | grep dhcp

# Check leases file
cat /var/lib/dhcp/dhcpd.leases
```

After fix, lease entries appeared immediately when a client
requested an IP.

### Lesson learned

Always specify the listening interface explicitly for DHCP.
It is not enough to configure the subnet in `dhcpd.conf` —
the daemon also needs to know which network interface to
watch for DHCP broadcast packets.

This same principle applies to many other services:

- BIND9: `listen-on` in named.conf.options
- DHCP: `INTERFACESv4` in /etc/default/isc-dhcp-server
- OpenVPN: `local` directive in server.conf

---

## Error 6 — pfSense TTL Expired Routing Error

### Component

pfSense — WAN/LAN routing and NAT

### What happened

After completing the pfSense setup wizard and configuring
the LAN, internet access from LAN clients was failing.
Ping tests from the pfSense console to internet addresses
returned TTL expired errors.

### Exact error

```bash
# From pfSense console (Option 8 → Shell)
ping 8.8.8.8

PING 8.8.8.8: 56 data bytes
Request timeout for icmp_seq 0
36 bytes from 192.168.211.2: Time to live exceeded
Request timeout for icmp_seq 1
36 bytes from 192.168.211.2: Time to live exceeded
```

### Root cause

A **routing loop** existed between pfSense and the VMware NAT
gateway. This happened because:

1. pfSense default gateway was set to `Automatic`
2. pfSense was trying to reach internet via an incorrect path
3. Packets were bouncing between pfSense and VMware NAT
   gateway (192.168.211.2) without escaping to the internet
4. TTL decremented to 0 before reaching destination

The `Time to live exceeded` response from `192.168.211.2`
confirms the packet was looping back to the VMware NAT
gateway rather than being forwarded to the internet.

### Diagnosis steps

Step 1: Restart routing service from console:

```bash
service routing restart
```

Checked output for default route:

```
default    192.168.211.2    -fib 0    done
```

Default route to 192.168.211.2 was being set — correct.

Step 2: Checked gateway configuration in web GUI:

```
System → Routing → Gateways
```

Default gateway IPv4 was set to `Automatic` — pfSense
was not consistently using WAN_DHCP.

### Fix applied

Explicitly set default gateway to WAN_DHCP:

```
System → Routing → Gateways
Default gateway IPv4: WAN_DHCP (192.168.211.2)
→ Save → Apply Changes
```

![17-pfsense-gateway-applied](../screenshots/16.png)

_(Shows gateway saved with WAN_DHCP explicitly set as default)_

After saving, internet ping from pfSense console succeeded:

```
ping 8.8.8.8
64 bytes from 8.8.8.8: icmp_seq=0 ttl=117 time=50.334 ms
64 bytes from 8.8.8.8: icmp_seq=1 ttl=117 time=60.662 ms
6 packets transmitted, 6 received, 0.0% packet loss
```

![21-pfsense-console-ping-internet](../screenshots/20.png)

_(Shows successful ping to 8.8.8.8 from pfSense console)_

### Lesson learned

When a firewall shows TTL expired errors and the reply comes
from the adjacent gateway (not from the destination), a
routing loop is almost always the cause. The fix is to
verify and explicitly set the default gateway rather than
relying on automatic selection.

In production environments, this symptom also appears when:

- Default routes are missing from the routing table
- Two firewalls are both set as default gateways for each other
- NAT is misconfigured and packets return to sender

---

## Error 7 — service pf restart Failed

### Component

pfSense — FreeBSD packet filter service

### What happened

During troubleshooting, an attempt was made to restart the
pfSense packet filter (firewall engine) from the console shell
to force firewall rules to reload.

### Exact error

```bash
[2.8.1-RELEASE][root@pfSense.lab.local]/root: service pf restart

Cannot 'restart' pf. Set pf_enable to YES in /etc/rc.conf
or use 'onerestart' instead.
```

![19-pfsense-console-pf-restart](../screenshots/18.png)

_(Shows the exact error message in pfSense console)_

### Root cause

pfSense is based on **FreeBSD**, not Linux. FreeBSD's `rc`
service management system has stricter rules about restarting
critical system services while they are running.

The `pf` (packet filter) is a core kernel-level component.
FreeBSD's `rc` system does not allow `restart` on `pf` because
stopping the packet filter even momentarily could drop all
active network connections — including the management session.

Instead FreeBSD requires:

- `onerestart` — graceful reload without dropping connections
- Or controlling pf through the application layer (pfSense GUI)

### Fix applied

Used the pfSense web GUI Apply Changes button instead:

```
Firewall → Rules → Apply Changes
```

pfSense internally handles the pf reload correctly —
using `pfctl -f /tmp/rules.debug` which reloads rules
atomically without interrupting existing connections.

### Alternative (correct CLI approach)

```bash
pfctl -f /tmp/rules.debug
```

This reloads the compiled firewall rules without a full
service stop/start.

### Lesson learned

FreeBSD and Linux handle service management differently.
On Linux, `service iptables restart` is standard.
On FreeBSD/pfSense, core services like `pf` must be
reloaded rather than restarted to avoid dropping connections.

When working with pfSense via CLI, prefer the web GUI for
firewall rule changes — pfSense handles the pf reload
correctly from the GUI, whereas manual CLI commands can
cause unexpected behaviour.

---

## Error 8 — Routing Restart Error Messages

### Component

pfSense — routing service console output

### What happened

When running `service routing restart` from the pfSense
console shell, the output contained many lines that appeared
to be errors but the routing actually worked correctly
afterward.

### Exact output

```bash
[2.8.1-RELEASE][root@pfSense.lab.local]/root: service routing restart

route: message indicates error: Address already in use
delete host 127.0.0.1: gateway lo0 fib 0: gateway uses the same route
default    192.168.211.2    -fib 0    done
route: message indicates error: Address already in use
delete host ::1: gateway lo0 fib 0: gateway uses the same route
route: route has not been found
delete net fe80::: gateway ::1 fib 0: not in table
route: route has not been found
add host 127.0.0.1: gateway lo0 fib 0: route already in table
route: message indicates error: File exists
add host ::1: gateway lo0 fib 0: route already in table
```

![18-pfsense-console-routing-restart](../screenshots/17.png)

_(Shows full routing restart output)_

### Root cause

These messages are **normal output** when restarting routing
on a live, running system. Here is what each message actually means:

| Message                       | Real meaning                            | Problem?   |
| ----------------------------- | --------------------------------------- | ---------- |
| `Address already in use`      | Route entry exists before restart began | No         |
| `gateway uses the same route` | Loopback route already registered       | No         |
| `default 192.168.211.2 done`  | Default route set successfully          | ✅ Success |
| `route has not been found`    | IPv6 routes not yet populated           | Minor      |
| `route already in table`      | Route being re-added after delete       | No         |
| `File exists`                 | Route file already present              | No         |

The critical success line is:

```
default    192.168.211.2    -fib 0    done
```

This confirms the default route to the WAN gateway was
set correctly. Everything else is cleanup noise from the
restart process deleting and recreating routes on a
live system.

### Fix applied

None needed. Reading the output carefully revealed the
routing restart was successful. The key lesson was not
to panic at error-looking messages without understanding
what they mean.

### Lesson learned

Not every line containing the word "error" is actually
an error. System administrators must learn to distinguish
between:

- **Fatal errors** — service cannot start, routes cannot be set
- **Warning messages** — something unexpected but handled
- **Informational noise** — normal output that uses error terminology

The correct approach: look for the success line in the output
(`done`, `OK`, `applied`) rather than counting error-looking
messages.

---

## Error 9 — pfBlockerNG DNSBL Source Definition Error

### Component

pfBlockerNG — DNSBL Group configuration

### What happened

While configuring the YOUTUBE_BLOCK DNSBL group, an attempt
was made to add YouTube as a block source by entering the
domain name directly in the Source field.

### Exact error

```
DNSBL Source Definition error (invalid URL)
```

### What was attempted (wrong approach)

```
DNSBL Source Definitions:
Format: Auto
State:  ON
Source: youtube.com    ← entered here directly
```

### Root cause

The **Source field** in DNSBL Source Definitions expects a
**URL** pointing to a downloadable plain-text domain list file.
Examples of valid sources:

```
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts
https://someblockingsite.com/list.txt
```

Entering `youtube.com` causes pfBlockerNG to:

1. Try to download a file from `http://youtube.com`
2. Receive an HTML page (YouTube's homepage) instead of a text list
3. Try to parse HTML as a domain list → fail → error

### Fix applied

**Step 1:** Disabled the invalid source row in DNSBL Source
Definitions (clicked the State dropdown → OFF → deleted row).

**Step 2:** Navigated to a different section:

```
Firewall → pfBlockerNG → DNSBL → DNSBL Category
→ Scroll to bottom
→ DNSBL Custom_List section (collapsed — click + to expand)
```

**Step 3:** Entered YouTube domains directly in Custom_List
(one per line):

```
youtube.com
www.youtube.com
m.youtube.com
ytimg.com
googlevideo.com
youtu.be
```

**Step 4:** Saved and ran Force Reload → All.

### Why Custom_List works but Source field does not

| Method             | Input expected                  | Works for YouTube? |
| ------------------ | ------------------------------- | ------------------ |
| Source Definitions | URL to a downloadable list file | ❌ No URL exists   |
| Custom_List        | Raw domain names, one per line  | ✅ Yes             |

### Lesson learned

In pfBlockerNG, there are two completely different ways to
add blocked domains:

1. **Feed-based (Source Definitions):** For large community
   lists maintained online — download URL is provided.
2. **Manual (Custom_List):** For specific domains you want to
   block yourself — enter domain names directly.

YouTube blocking is a manual custom addition — it belongs in
Custom_List, not in Source Definitions.

---

## Error 10 — YouTube Not Blocked After Saving Custom_List

### Component

pfBlockerNG — DNSBL update timing

### What happened

After adding YouTube domains to Custom_List and clicking
Save DNSBL Settings, `nslookup youtube.com` from Windows
still returned the real YouTube IP address — not the
sinkhole 10.10.10.1.

### Symptom

```
C:\Users\MAYUR> nslookup youtube.com
Server:  pfSense.lab.local
Address: 192.168.20.1

Name:    youtube.com
Address: 142.250.x.x   ← still real YouTube IP, not blocked
```

### Root cause

pfBlockerNG does **not** apply changes immediately when
settings are saved. The settings are written to pfBlockerNG's
database but Unbound DNS resolver is not updated until either:

1. The scheduled CRON job runs (midnight by default)
2. A manual Force Update/Reload is triggered

Saving settings in pfBlockerNG only writes the configuration.
The actual enforcement in Unbound requires a separate reload step.

This is documented in pfBlockerNG with the notice:

```
Setting changes are applied via CRON or 'Force Update/Reload' only!
```

### Fix applied

Navigated to Update tab:

```
Firewall → pfBlockerNG → Update
```

Selected:

```
Force option:  ● Reload
Reload option: ● All
→ Click Run
```

![37-pfblockerng-force-reload](../screenshots/37.png)

_(Shows Update tab with Reload → All selected)_

Waited for the reload to complete (approximately 30 seconds).

Tested immediately after:

```
C:\Users\MAYUR> nslookup youtube.com
Server:  pfSense.lab.local
Address: 192.168.20.1

Name:    youtube.com
Address: 10.10.10.1   ← sinkhole IP — blocked ✅
```

![43-nslookup-youtube-blocked](../screenshots/43.png)

_(Shows YouTube resolving to 10.10.10.1)_

### Why Reload not Update was chosen

| Force Option | What it does                             | Needed here?                        |
| ------------ | ---------------------------------------- | ----------------------------------- |
| Update       | Downloads fresh list files from internet | No — Custom_List needs no download  |
| Cron         | Runs scheduled tasks                     | No — immediate action needed        |
| **Reload**   | Reloads existing config into Unbound     | **Yes — applies saved Custom_List** |

### Lesson learned

pfBlockerNG operates in a save-then-apply model, not a
save-and-apply model. Every configuration change requires
a Force Reload to take effect. This is by design — it
prevents partially-complete configurations from being
applied mid-change.

In production environments this is actually a feature:
changes can be staged and reviewed before being applied,
and rollback is possible before Force Reload.

---

## Error 11 — Database Sanity Check FAILED

### Component

pfBlockerNG — first update run

### What happened

On the very first Force Update after pfBlockerNG wizard
completion, the update log showed a FAILED sanity check.

### Exact log output

```
======[ DNSBL Last Updated List Summary ]========
May 1  23:23  StevenBlack_ADs
Database Sanity check [ FAILED ] ** These two counts should match! **
Masterfile Count  [ 16998 ]
Deny folder Count [ 16997 ]
Duplication sanity check (Pass=No IPs reported)
```

### Root cause

A count mismatch of exactly 1 between the masterfile
(16998 entries) and deny folder (16997 entries) during
the first database build.

This occurs because pfBlockerNG writes domain entries
asynchronously during the first build. The sanity check
count runs while the last entry is still being written —
creating a 1-entry discrepancy.

This is a **known first-run behaviour** documented in
pfBlockerNG community forums and GitHub issues. It does
not affect functionality.

### Whether this caused any real problem

No. The DNSBL was fully functional immediately after this
first run. The StevenBlack_ADs list began blocking
advertising domains within minutes (visible in Reports →
Alerts tab).

### Fix applied

None needed. The error resolved itself on the next
scheduled update run when the full 16998 entries were
properly counted in both locations.

### Lesson learned

First-run errors in complex software systems are common —
database initialisation, cache builds, and file synchronisation
all have race conditions that appear only on the first
execution. Always test actual functionality rather than
treating first-run log anomalies as blocking issues.

---

## Error 12 — nslookup DNS Timeout After Adding Block Rules

### Component

pfSense Firewall Rules — DNS enforcement

### What happened

After adding a DNS block rule (Rule 3 — block all DNS to
any destination on port 53), all DNS resolution on the
network stopped working completely — including DNS to
pfSense itself.

### Symptom

```
C:\Users\MAYUR> nslookup google.com
DNS request timed out.
    timeout was 2 seconds.
DNS request timed out.
    timeout was 2 seconds.
*** Can't find server name for address 192.168.20.1: Timed out
*** Default servers are not available
```

### Root cause

Rule 3 (block port 53 to any destination) was added **before**
Rule 2 (allow port 53 to pfSense). Since pfSense evaluates
rules top to bottom, the block rule matched ALL DNS traffic —
including queries intended for pfSense at 192.168.20.1:53.

```
Rule order (wrong):
1. Anti-lockout
2. BLOCK any → any:53    ← catches DNS to pfSense too
3. ALLOW any → 192.168.20.1:53  ← never reached
4. Default allow

Result: ALL DNS blocked including pfSense DNS
```

### Fix applied

Reordered the rules so the allow rule comes before the
block rule:

```
Correct rule order:
1. Anti-lockout
2. ALLOW any → 192.168.20.1:53   ← allows pfSense DNS first
3. BLOCK any → any:53             ← blocks everything else
4. pfBlockerNG auto-rule
5. Default allow
```

In pfSense, rules are reordered by dragging them in the
Firewall → Rules → LAN table.

After reordering and applying:

```
C:\Users\MAYUR> nslookup google.com
Server:  pfSense.lab.local
Address: 192.168.20.1

Name:   google.com
Address: 142.250.70.110   ← resolves correctly ✅
```

### Lesson learned

In firewall rule design, **order is everything**. The most
specific rules (allow DNS to pfSense) must always come before
the more general rules (block all DNS). This principle applies
to every firewall platform:

- pfSense: drag to reorder in GUI
- Cisco ASA: `access-list` entries evaluated top-down
- iptables: `-I` to insert at position, `-A` to append
- Palo Alto: rule order in Security Policies

A useful mental model: write rules from most specific to
most general, top to bottom. Catch specific allowed traffic
first, then block the rest.

---

## Troubleshooting Commands Reference

A consolidated list of the most useful diagnostic commands
used throughout the project:

### Ubuntu Server

```bash
# Check service status
sudo systemctl status named
sudo systemctl status isc-dhcp-server
sudo systemctl status apache2
sudo systemctl status smbd

# Check what ports a service is listening on
sudo ss -ulnp | grep :53    # UDP port 53 (DNS)
sudo ss -tlnp | grep :445   # TCP port 445 (Samba)
sudo ss -ulnp | grep :67    # UDP port 67 (DHCP)
sudo ss -tlnp | grep :80    # TCP port 80 (HTTP)

# Test DNS resolution from server itself
dig @localhost company.local
dig @192.168.20.101 company.local
nslookup company.local 127.0.0.1

# Check DHCP leases
cat /var/lib/dhcp/dhcpd.leases

# Watch live DHCP activity
sudo tail -f /var/log/syslog | grep dhcp

# Check BIND9 config syntax before restarting
sudo named-checkconf
sudo named-checkzone company.local /etc/bind/zones/db.company.local

# Check network interface and IP
ip a show ens33
ip route show

# Test web server
curl -I http://localhost
curl http://localhost

# Check Samba share
sudo smbclient -L localhost -U%
```

### pfSense Console (Shell — Option 8)

```bash
# Test internet connectivity
ping 8.8.8.8
ping -c 4 google.com

# View routing table
netstat -rn

# View active firewall rules
pfctl -sr

# View NAT rules
pfctl -sn

# Reload firewall rules (correct method)
pfctl -f /tmp/rules.debug

# Restart routing (with expected noise in output)
service routing restart

# Check interface status
ifconfig em0
ifconfig em1

# View DHCP leases pfSense is tracking
cat /var/dhcpd/var/db/dhcpd.leases
```

### Windows CMD (Testing from Client)

```bash
# Check current IP and DNS server
ipconfig /all

# Flush DNS cache (important before testing changes)
ipconfig /flushdns

# Release and renew DHCP lease
ipconfig /release
ipconfig /renew

# Test DNS resolution
nslookup company.local
nslookup youtube.com
nslookup google.com

# Test specific DNS server
nslookup google.com 192.168.20.1
nslookup google.com 8.8.8.8

# Test connectivity
ping 192.168.20.1      (pfSense)
ping 192.168.20.101    (Ubuntu server)
ping myserver.local    (DNS resolution test)

# Trace route to diagnose routing
tracert 8.8.8.8
```

---

## Summary — Key Lessons From This Project

| Lesson                                                        | Context         |
| ------------------------------------------------------------- | --------------- |
| Install VMware Tools immediately                              | Every new VM    |
| Use real unit names not package names for systemctl           | bind9 → named   |
| Always verify listening interfaces, not just service status   | BIND9, DHCP     |
| Syslog noise ≠ broken service — test actual functionality     | IPv6 warnings   |
| Specify interface explicitly for DHCP                         | INTERFACESv4    |
| Explicitly set default gateway — don't rely on Automatic      | pfSense routing |
| FreeBSD service management differs from Linux                 | pf restart      |
| Read routing output for the success line not the error count  | routing restart |
| Source Definitions needs URL — Custom_List needs domain names | pfBlockerNG     |
| Save ≠ Apply in pfBlockerNG — Force Reload required           | pfBlockerNG     |
| First-run anomalies are common — test functionality           | Sanity check    |
| Specific rules before general rules — order matters           | DNS enforcement |

---

_Document: 06-TROUBLESHOOTING.md_
_Project: Mini Enterprise Network_
_Author: Mayur Garje_
_Date: May 2026_
