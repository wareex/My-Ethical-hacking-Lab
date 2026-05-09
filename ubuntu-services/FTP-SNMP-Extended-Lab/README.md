# Lab 03 — FTP & SNMP Extended Variant Lab

> **Environment:** Isolated VMware NAT lab | **Target:** Ubuntu Server 22.04 LTS | **Focus:** Extended SNMP enumeration, FTP bounce + resource exhaustion, Metasploit-assisted SNMP

---

## Overview

This lab is a variant of the FTP/SNMP attack lab, extending coverage with more advanced attack vectors including **FTP bounce attacks**, **connection slot exhaustion**, **Metasploit-assisted SNMP enumeration**, and **UDP amplification flooding**. It provides deeper coverage of SNMP reconnaissance and demonstrates more sophisticated FTP attack scenarios.

**Objectives:**
1. Build a 3-VM lab with Ubuntu Server as target
2. Deploy FTP (vsftpd) and SNMP (snmpd) services
3. Execute 3 advanced FTP attacks and 3 advanced SNMP attacks
4. Document countermeasures, limitations, and recommendations for each

---

## Lab Architecture

```
┌──────────────────────┐          NAT Network          ┌──────────────────────┐
│   Kali Linux         │◄─────────────────────────────►│   Ubuntu Server      │
│   Attacker           │       192.168.106.0/24        │   Target             │
│   192.168.106.134    │                               │   192.168.106.131    │
└──────────────────────┘                               └──────────────────────┘
                                                                ▲
                                                                │
                                                ┌──────────────────────┐
                                                │   Windows 8.1        │
                                                │   Client Validator   │
                                                │   192.168.106.133    │
                                                └──────────────────────┘
```

---

## Virtual Machines

| Role | OS | IP Address | Purpose |
|------|----|------------|---------|
| Attacker | Kali Linux 2025.x | 192.168.106.134 | Attack simulation |
| Target | Ubuntu Server 22.04.5 | 192.168.106.131 | Hosts vulnerable services |
| Client | Windows 8.1 x64 | 192.168.106.133 | Functional validation |

---

## Service Deployment (Ubuntu Server)

### FTP (vsftpd)
```bash
sudo apt update
sudo apt install vsftpd
sudo systemctl start vsftpd
sudo systemctl enable vsftpd
sudo adduser ftpuser
sudo systemctl status vsftpd
```

### SNMP (snmpd)
```bash
sudo apt install snmp snmpd
sudo nano /etc/snmp/snmpd.conf
# Add: rocommunity public
# Add: agentAddress udp:161
sudo systemctl restart snmpd
sudo systemctl status snmpd
```

---

## Attack Simulations

---

## Service 1: FTP Attacks (Advanced)

### Attack 1 — FTP Cleartext Credential Interception

**Objective:** Capture FTP credentials transmitted in cleartext using passive network monitoring.

FTP transmits authentication credentials (username and password) in plaintext over the network. Any attacker with access to the network path between client and server can passively capture these credentials without active exploitation.

**Method:** Network traffic capture during FTP authentication session.

**Credential Exposure Vector:**
- FTP `USER` and `PASS` commands sent unencrypted
- Capturable by any host on the same network segment (ARP poisoning not required on a hub or switched network with SPAN)

**Result:** Username and password transmitted in cleartext, confirmed capturable via packet analysis.

**Impact:**
- Credential theft without active interaction with the server
- Credentials can be reused for lateral movement
- No alert generated — passive attack leaves no server-side evidence

---

### Attack 2 — FTP Bounce Attack

**Objective:** Abuse the FTP PORT command to proxy scan internal hosts, using the FTP server as an intermediary.

**Vulnerability:** Active mode FTP allows the client to specify an arbitrary destination IP:port via the `PORT` command. A misconfigured server will then initiate a connection to that arbitrary target — enabling the attacker to proxy-scan internal systems through the FTP server.

**Attack Flow:**
```
Attacker → FTP Server (PORT internal_host:port) → Internal Host
```

**Target of scan:** Internal network hosts behind the FTP server

**Result:** FTP server relayed connection attempts to specified internal hosts, enabling indirect port scanning of internal infrastructure that is otherwise unreachable from the attacker's position.

**Impact:**
- Internal network scanning via trusted intermediary
- Firewall bypass using the FTP server's trusted network position
- Potential for data exfiltration through the bounce channel

---

### Attack 3 — FTP Connection Slot Saturation (Resource Exhaustion)

**Objective:** Exhaust the FTP server's connection pool to prevent legitimate access.

Unlike a SYN flood (which targets the OS TCP stack), this attack targets the application-layer connection limit of vsftpd itself by opening and holding the maximum number of permitted connections.

**Method:** Open and hold multiple simultaneous FTP connections to saturate the server's `max_clients` limit.

**Expected Behaviour:**
- vsftpd's connection queue fills
- New connection attempts are refused with: `421 Service not available, closing control connection`
- Legitimate users cannot connect

**Functional Validation (Windows 8.1):**  
After attack initiation, FTP connection from Windows client was refused — service completely unavailable.

**Comparison to SYN Flood:**

| Vector | Mechanism | Detectability | Layer |
|--------|-----------|---------------|-------|
| SYN Flood | TCP stack exhaustion | Easier — high packet volume | Network |
| Connection Slot Saturation | App-layer queue exhaustion | Harder — low packet rate, looks like slow clients | Application |

**Analysis:** Connection slot exhaustion attacks are harder to detect because they involve lower packet rates and appear as slow but legitimate clients. They bypass network-layer flood detection while still achieving full service disruption.

---

## Service 2: SNMP Attacks (Advanced)

### Attack 4 — SNMP Process & Software Enumeration via Metasploit

**Objective:** Extract deep system metadata — running processes, installed software, network interfaces — using the Metasploit SNMP scanner module.

**Commands (Kali):**
```bash
msfconsole

use auxiliary/scanner/snmp/snmp_enum
set RHOSTS 192.168.106.131
set COMMUNITY public
run
```

**Data Extracted:**

*a. System & Network Information:*
- Hostname, OS, uptime, contact, location

*b. Network Interface & IP Details:*
- Interface names, MAC addresses, IP assignments
- All listening ports

*c. Storage Information:*
- Physical memory, virtual memory, memory buffer, cached memory

*d. Device & Software Components:*
- Full list of installed software packages
- Package versions and paths

*e. Running Processes:*
- Complete process list (PID, name, path, arguments)

**Correlation Verification (Ubuntu):**
```bash
uname -a     # Matches OS info from SNMP output
ps aux       # Matches running process list from SNMP
```
*All extracted data matched Ubuntu system state exactly — 100% accuracy.*

**Analysis:** Framework-assisted SNMP enumeration provides an attacker with a near-complete picture of the target system in a single command. This dramatically accelerates reconnaissance and enables precise chained exploitation targeting specific vulnerable software versions.

---

### Attack 5 — SNMP Service Disruption via UDP Amplification Flood

**Objective:** Disrupt SNMP availability using high-rate malformed UDP packets with randomised source addresses.

**Command (Kali):**
```bash
sudo hping3 --udp -p 161 --flood --rand-source 192.168.106.131
```

**What this does:**
- Sends UDP packets to port 161 at maximum rate
- `--rand-source` randomises source IP addresses, bypassing simple IP-based blocking
- Overwhelms SNMP daemon's ability to process requests

**Service Failure Validation:**  
While flooding was active, an SNMP walk was attempted from a second terminal:
```bash
snmpwalk -v2c -c public 192.168.106.131
```
**Output:** `Timeout: No Response from 192.168.106.131` — service completely unresponsive.

**Server Network Statistics (Ubuntu):**
```bash
sudo netstat -su
# Confirmed: UDP packet drops, ICMP messages, extension headers
```

**SNMP Daemon Log (Ubuntu):**
```bash
sudo journalctl -u snmpd
# Log 1: [Jan 05, 20:41:52] — flood timing confirmed
# Log 2: SNMP Daemon unresponsive state entries
```

**Result:**
- SNMP daemon completely unresponsive during attack
- Packet drops confirmed in kernel network statistics
- Server logs confirmed the disruption timeline

**Analysis:** UDP is connectionless by design, making precise rate control difficult. Random-source flooding also evades IP-based firewall rules. SNMP services are inherently vulnerable to UDP flooding — creating monitoring blind spots during attacks.

---

## Countermeasures, Limitations & Recommendations

---

### Attack 1: FTP Cleartext Credential Interception

**Countermeasures:**
- Replace FTP with SFTP (SSH File Transfer Protocol) or FTPS (FTP over TLS)
- Enforce encrypted authentication for all file transfers
- Implement network segmentation to limit attacker access to authentication traffic
- Deploy IDS to alert on plaintext authentication protocol usage

**Limitations:**
- FTPS requires digital certificate management — increases administrative complexity
- Legacy systems may not support SFTP/FTPS

**Recommendations:**
- Fully decommission FTP wherever possible
- Enforce SSH-based SFTP with key-based authentication
- Conduct regular network traffic audits to detect unencrypted protocol usage

---

### Attack 2: FTP Bounce Attack

**Countermeasures:**
- Disable the FTP PORT command: set `port_enable=NO` in `/etc/vsftpd.conf`
- Enforce passive mode (PASV) exclusively
- Block outbound connections from the FTP server to internal hosts via firewall rules

**Limitations:**
- Enforcing passive mode may reduce compatibility with older FTP clients
- Configuration errors may still allow limited exploitation
- Does not protect against attacks initiated from within the internal network

**Recommendations:**
- Strictly enforce passive FTP mode at all times
- Restrict FTP server egress traffic to only essential destinations
- Review vsftpd configuration files regularly

---

### Attack 3: FTP Connection Slot Saturation

**Countermeasures:**
- Configure connection limits in vsftpd:
  ```
  max_clients=10
  max_per_ip=2
  ```
- Enable connection throttling and rate limiting
- Deploy firewall-based connection tracking using `iptables` or `nftables`

**Limitations:**
- Strict limits may unintentionally block high-volume legitimate users
- Distributed attacks from multiple IPs can bypass per-IP restrictions
- Does not prevent slow application-layer attacks that consume resources over time

**Recommendations:**
- Combine FTP connection limits with network-level rate limiting
- Deploy HIPS (Host-based Intrusion Prevention System)
- Monitor active connections in real time with automated blocking for suspicious patterns

---

### Attack 4: SNMP Metasploit Enumeration

**Countermeasures:**
- Disable SNMP entirely if not operationally required
- Upgrade SNMPv2c to SNMPv3 (authentication + encryption)
- Restrict SNMP access to trusted IP addresses only
- Change all default community strings immediately

**Limitations:**
- SNMPv3 requires cryptographic key management — increases administrative workload
- Some legacy monitoring tools may not fully support SNMPv3
- Misconfigurations may still result in unintended disclosure

**Recommendations:**
- Enforce SNMPv3 with both authentication and encryption
- Apply ACLs to restrict SNMP queries to authorised systems
- Conduct regular SNMP configuration audits
- Actively monitor SNMP logs for enumeration attempts

---

### Attack 5: SNMP UDP Flood

**Countermeasures:**
- Apply firewall-level UDP rate limiting
- Disable SNMP write access to minimise potential damage
- Configure system-level packet filtering
- Deploy monitoring tools to detect abnormal UDP traffic patterns

**Limitations:**
- UDP is connectionless — precise traffic control is inherently difficult
- High-volume floods can overwhelm servers and network infrastructure simultaneously
- Aggressive filtering may also affect legitimate SNMP monitoring traffic

**Recommendations:**
- Restrict SNMP services to internal management networks only
- Implement firewall-based UDP flood protection
- Deploy network-based IDS/IPS
- Disable SNMP entirely if not strictly required

---

## Tools Reference

| Tool | Purpose |
|------|---------|
| `Hydra` | FTP credential brute-force |
| `hping3` | SYN flood, UDP flood, connection testing |
| `snmpwalk` | SNMP OID tree enumeration |
| `Nmap` | Port scanning, service enumeration, SNMP scripts |
| `Metasploit (snmp_enum)` | Framework-assisted deep SNMP enumeration |
