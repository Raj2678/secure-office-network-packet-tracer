# Secure Small-Office Network: Design, Hardening and Incident Investigation



## 1. Objective
what network you built and what security goals it meets

## 1. Objective

I built a segmented office network in Cisco Packet Tracer to practise the
access control and hardening skills a SOC analyst uses day to day. The network
splits staff, servers, guests and management traffic into separate VLANs,
restricts what each group can reach, and sends device logs to a central
server so activity can be reviewed.

## 2. Topology


| VLAN | Name | Subnet | Purpose |
|------|------|--------|---------|
| 10 | STAFF | 192.168.10.0/24 | Employee PCs |
| 20 | SERVERS | 192.168.20.0/24 | Web, DNS, syslog |
| 30 | GUEST | 192.168.30.0/24 | Untrusted guest devices |
| 99 | MGMT | 192.168.99.0/24 | Device management |

![VLAN configuration](screenshots/01-vlan-brief.png)

## 3. Security controls implemented

| Control | Evidence | What it does |
|---------|----------|---------------|
| VLAN segmentation | `01-vlan-brief.png` | Separates staff, servers, guest and management traffic at Layer 2 |
| Access control lists | `02-acl-staff-guest.png`, `03/04-acl-applied-*.png` | Enforces least-privilege: staff reach servers and management via SSH only; guests reach only the web server |
| SSH-only remote access | `05/06-ssh-enabled-*.png`, `09-telnet-refused.png` | Disables Telnet (cleartext passwords) in favour of encrypted SSH |
| Port security | `10-port-security-status.png`, `11-port-security-addresses.png` | Locks each access port to one known device (sticky MAC), rejects anything else |
| Centralised logging | `15-syslog-fresh-event.png` | Sends router and switch log events to a central syslog server for investigation |

## 4. Connectivity and access tests

| # | Test | Expected | Result | Evidence |
|---|------|----------|--------|----------|
| 1 | Staff PC reaches the web server | Loads |pass | |
| 2 | Guest PC reaches the web server | Loads | pass| |
| 3 | Guest PC pings the server | Fails |Pass | |
| 4 | Guest PC pings a staff PC | Fails |pass | |
| 5 | Staff SSH to R1 | Succeeds | | `7-ssh-success-staff.png` |
| 6 | Guest SSH to R1 | Fails | | `8-ssh-blocked-guest.png` |
| 7 | Telnet to R1 | Refused | | `9-telnet-refused.png` |

## 5. Incident 1: Unauthorized device on an access port

**Summary:** An unknown device was connected to a staff access port (Fa0/3) to
test whether port security would prevent an unauthorized device from joining
the network.

**What I did:**
1. Disconnected the legitimate device from Fa0/3.
2. Connected a new, unrecognized device to the same port.
3. Attempted to request an address and ping the internal server.

**Evidence:**
- `12-rogue-device-blocked.png` — the unauthorized device could not reach the server
- `13-port-security-violation.png` — `show port-security interface fastEthernet0/3` showing a recorded violation

**Analysis:** The device received no connectivity when I tested it, the ping to
the server got no response at all. The switch port only trusted the MAC
address of the original staff PC, so when a different device connected,
port security restricted its traffic. This matters in a real office because
someone could plug a personal device into an unused wall socket to try to
access internal systems, locking each port to a known device prevents that
without needing a password or firewall rule at all.

**Verdict:** Blocked. The unauthorized device could not reach any part of the
network.

**Recommendation:** Apply port security to every access port by default, not
only the ones currently in use, since an empty port is itself an attack
surface. In production, 802.1X network access control would provide the same
protection with centrally managed credentials instead of static MAC addresses.

## 6. Incident 2: Credential brute-force attempt (SSH)

**Summary:** A guest-network device attempted to log in to the router's
management interface via SSH, simulating an attacker trying to guess
administrator credentials.

**What I did:**
1. From the guest VLAN, attempted `ssh -l admin 192.168.99.1`.
2. Compared this against a legitimate login attempt from the staff VLAN.

**Evidence:**
- `08-ssh-blocked-guest.png` — guest attempt blocked before reaching a login prompt
- `07-ssh-success-staff.png` — legitimate staff login succeeding for comparison
- `09-telnet-refused.png` — unencrypted Telnet access refused entirely

**Analysis:** [Write 2-3 sentences: why was the guest blocked at the network
layer rather than just a wrong password? Why does disabling Telnet matter even
if SSH is also password-protected?]

**Verdict:** Blocked. The guest ACL prevented any management-plane access
attempt, regardless of credentials.

**Recommendation:** Keep management interfaces unreachable from untrusted
VLANs as the first line of defence, rather than relying on passwords alone.
In a production SIEM, repeated denied attempts like this would also trigger
an alert for investigation.

## 7. Limitations

- Packet Tracer is a simulator: not every real-world log message or attacker
  behaviour is reproduced exactly as it would be on real hardware.
- ACLs here are stateless and basic; a real network would use a stateful
  firewall in addition to VLAN/ACL segmentation.
- Testing was limited to the controls listed above; a rogue DHCP server test
  was attempted but produced inconsistent simulator behaviour (duplicate
  address conflicts) and was not included as formal evidence.

