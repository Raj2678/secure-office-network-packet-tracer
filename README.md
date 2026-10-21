# Secure Small-Office Network (Cisco Packet Tracer)

Build a segmented office network, lock it down, simulate a misbehaving guest,
and analyze the evidence the way a SOC analyst would.

**What you'll practice:** VLANs, inter-VLAN routing, DHCP, DNS/web services,
access control lists, SSH hardening, port security, centralized logging, testing
and documentation.

## Design

```
                       +------+
                       |  R1  |
                       +--+---+
                          | Gi0/0 (trunk: VLANs 10, 20, 30, 99)
                          | Gi0/1
                       +--+---+
                       | SW1  |
                       +-+-+-+-+
              Fa0/1 ----+ | | +---- Fa0/4
             SRV1 (20)    | |      PC3 (30, Guest)
                   Fa0/2 -+ +- Fa0/3
               PC1 (10, Staff)  PC2 (10, Staff)
```

| VLAN | Name | Subnet | Gateway | Notes |
|------|------|--------|---------|-------|
| 10 | STAFF | 192.168.10.0/24 | .1 | DHCP (addresses from .100) |
| 20 | SERVERS | 192.168.20.0/24 | .1 | SRV1 static: 192.168.20.10 |
| 30 | GUEST | 192.168.30.0/24 | .1 | DHCP (addresses from .100) |
| 99 | MGMT | 192.168.99.0/24 | .1 | SW1 static: 192.168.99.2 |

### Security policy

| From | To | Allowed |
|------|----|---------|
| Staff | Servers | Everything |
| Staff | Management | SSH and ping only |
| Staff | Guest | Nothing |
| Guest | Servers | Web (80) and DNS (53) only |
| Guest | Staff, Management | Nothing (denied and logged) |

Plus: SSH only (no Telnet), unused switch ports shut down, one device per PC
port, and R1 + SW1 send logs to SRV1.

## Phase 0: Install Packet Tracer

1. Create a free Cisco Networking Academy (NetAcad) account at netacad.com.
2. Enroll in the free "Getting Started with Cisco Packet Tracer" / "Introduction to Packet Tracer" course.
3. Download Packet Tracer from NetAcad only (not from other websites) and install it.
4. Open it and sign in with your NetAcad account.

## Phase 1: Build the topology

Add these devices (bottom-left device panel):

| Name | Device |
|------|--------|
| R1 | Routers > 2911 |
| SW1 | Switches > 2960 (24TT) |
| SRV1 | End Devices > Server |
| PC1, PC2, PC3 | End Devices > PC |

Connect with **Copper Straight-Through** cables:

| From | To |
|------|----|
| R1 Gi0/0 | SW1 Gi0/1 |
| SW1 Fa0/1 | SRV1 Fa0 |
| SW1 Fa0/2 | PC1 Fa0 |
| SW1 Fa0/3 | PC2 Fa0 |
| SW1 Fa0/4 | PC3 Fa0 |

Rename each device by clicking its label. Wait for the link lights to turn green
(or green/orange for a minute, then green).

## Phase 2: Base configuration

1. Click **SW1 > CLI**, press Enter, then paste `configs/01_SW1_vlans_and_ports.txt`
   (first type `enable` then `configure terminal`).
2. Click **R1 > CLI**. If asked about the initial configuration dialog, type `no`.
   Then `enable`, `configure terminal`, and paste `configs/02_R1_routing_and_dhcp.txt`.
3. **SRV1 > Desktop > IP Configuration** (Static):
   - IP address `192.168.20.10`, mask `255.255.255.0`
   - Gateway `192.168.20.1`, DNS server `192.168.20.10`
4. **SRV1 > Services**:
   - **HTTP**: make sure it is On
   - **DNS**: turn On, add a record: Name `www.lab.local`, Address `192.168.20.10`
   - **SYSLOG**: turn On
5. **PC1, PC2, PC3 > Desktop > IP Configuration**: choose **DHCP**.

## Phase 3: Test connectivity (before any ACLs)

On PC1 open **Desktop > Command Prompt**:

```
ipconfig
ping 192.168.20.10
ping 192.168.99.1
```

Also ping PC3's address (find it with `ipconfig` on PC3), and open
**Web Browser** on any PC and visit `http://192.168.20.10` and `http://www.lab.local`.

Everything should work. Fix problems now, because ACLs will make troubleshooting harder.
Useful commands: `show vlan brief`, `show ip interface brief`, `show ip dhcp binding` (on R1).

## Phase 4: Apply the ACLs

Paste `configs/03_R1_acls.txt` into R1. Then run the tests in the table below.

| # | Test | Expected |
|---|------|----------|
| 1 | PC1: web browser to `http://www.lab.local` | Loads |
| 2 | PC1: `ping 192.168.20.10` | Succeeds |
| 3 | PC1: ping PC3 | Fails |
| 4 | PC3: web browser to `http://192.168.20.10` | Loads |
| 5 | PC3: `ping 192.168.20.10` | Fails |
| 6 | PC3: ping PC1 | Fails |

Check the evidence on R1:

```
show access-lists
show ip interface gigabitEthernet0/0.30
```

The `deny ip any any log` line shows a match count that rises with each blocked attempt.

## Phase 5: Hardening and logging

1. Edit the placeholder passwords in `configs/04_R1_hardening_and_logging.txt` and
   `configs/05_SW1_hardening_logging_portsecurity.txt` (use lab-only passwords).
2. Paste file 04 into R1 and file 05 into SW1.
3. Make each PC send one ping so port security learns its address.
4. Test SSH from PC1 (Command Prompt):
   ```
   ssh -l admin 192.168.99.1
   ```
   It should ask for the password. From PC3 the same command should fail (blocked by the ACL).
5. Telnet should no longer work: `telnet 192.168.99.1` fails.

## Phase 6: Simulate and investigate

1. On PC3 (guest), repeat the blocked actions several times: ping the server,
   ping a staff PC, `ssh -l admin 192.168.99.1`.
2. On R1 run `show access-lists` and note how the deny counter grew.
3. Open **SRV1 > Services > SYSLOG** and look at the messages from R1 and SW1.
4. **Rogue device test:** delete PC2's cable, add a new PC, and connect it to SW1 Fa0/3.
   It should be blocked by port security. Check with `show port-security interface fa0/3`
   and the syslog server.
5. Write up what you found as an incident-style analysis (see `REPORT_TEMPLATE.md`).

Note: Packet Tracer's logging is simplified. If a particular log message does not
appear in the syslog server, use `show access-lists` counters and the interface
up/down messages (shut and unshut an interface) as your evidence, and say so in
your limitations section.

## Phase 7: Save and document

1. Save as `secure-office.pkt` (File > Save).
2. Take screenshots: topology, `show vlan brief`, `show access-lists`, the syslog
   server, the test results.
3. Fill in `REPORT_TEMPLATE.md`.
4. Put the `.pkt` file, configs, screenshots, and report on GitHub.

## Troubleshooting

| Problem | Try |
|---------|-----|
| Link light stays red | Wrong cable type or port, or the port is shut down |
| PC gets no IP | `show ip interface brief` on R1 (Gi0/0 and subinterfaces up?), `show vlan brief` on SW1 |
| Staff cannot reach servers | Check SRV1 gateway is 192.168.20.1 and the trunk allows VLAN 20 |
| Command rejected | Packet Tracer versions differ slightly. Copy the exact error and look it up |
| Everything stopped after ACLs | `show access-lists` to see which line is matching, and check DHCP is permitted |

## Limitations

- Packet Tracer simulates devices; it cannot run Python tools like `portscan.py`.
  The PCs only have simple commands (ping, ipconfig, ssh, telnet, tracert).
- ACLs here are stateless and basic.
- No real attack traffic is generated.
