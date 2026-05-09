# Lab 02 — FTP & SNMP Network Service Attacks

> **Environment:** Isolated VMware NAT lab | **Target:** Ubuntu Server 22.04 LTS | **Services:** vsftpd (FTP), snmpd (SNMP)

---

## Overview

This lab deploys two real-world services — FTP and SNMP — on a live Ubuntu Server and performs structured penetration testing against each. The lab demonstrates how common enterprise protocol misconfigurations lead to authentication compromise, information disclosure, and service disruption.

**Objectives:**
1. Build a 3-VM isolated lab (Ubuntu Server, Windows 8.1, Kali Linux)
2. Deploy and configure FTP (vsftpd) and SNMP (snmpd) on Ubuntu Server
3. Perform 3 attacks per service (6 total)
4. Validate service availability from Windows client before and after attacks
5. Document evidence and propose countermeasures

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

| Role | OS | Download | IP Address | Purpose |
|------|----|----------|------------|---------|
| Server/Target | Ubuntu Server 22.04.5 LTS | [ubuntu-22.04.5-desktop-amd64.iso](https://releases.ubuntu.com/jammy/ubuntu-22.04.5-desktop-amd64.iso) | 192.168.106.131 | Hosts vulnerable services |
| Client | Windows 8.1 x64 | [Win8.1_English_x64.iso](https://ia803407.us.archive.org/26/items/win-8.1-english-x-64_202105/Win8.1_English_x64.iso) | 192.168.106.133 | Legitimate user simulation |
| Attacker | Kali Linux 2025.x | [kali-linux-2025.4-installer-amd64.iso](https://cdimage.kali.org/kali-2025.4/kali-linux-2025.4-installer-amd64.iso) | 192.168.106.134 | Attack simulation |

**Network Adapter:** All VMs set to VMware NAT — isolated from external internet, all VMs share the same subnet.

---

## Network Configuration

### IP Address Verification

```bash
# Ubuntu Server (Target)
ifconfig
# IPv4: 192.168.106.131

# Kali Linux (Attacker)
ifconfig
# IPv4: 192.168.106.134

# Windows 8.1 (Client)
ipconfig
# IPv4: 192.168.106.133 | Gateway: 192.168.106.2
```

### Connectivity Verification — Ping Tests

```bash
# From Kali → Ubuntu
ping 192.168.106.131   # ✓ Successful

# From Kali → Windows
ping 192.168.106.2     # ✓ Successful

# From Ubuntu → Kali
ping 192.168.106.134   # ✓ Successful

# From Windows → Ubuntu
ping 192.168.106.131   # ✓ Successful
```

---

## Service Deployment (Ubuntu Server)

### Service 1 — FTP (vsftpd)

FTP was installed using the vsftpd daemon. A local user account was created to simulate legitimate access.

```bash
# Install FTP service
sudo apt update
sudo apt install vsftpd

# Start and enable
sudo systemctl start vsftpd
sudo systemctl enable vsftpd

# Create test user
sudo adduser ftpuser

# Verify service status
sudo systemctl status vsftpd
```

### Service 2 — SNMP (snmpd)

SNMP was installed using the snmpd daemon with a read-only community string (`public`) for monitoring.

```bash
# Install SNMP packages
sudo apt install snmp snmpd

# Edit configuration
sudo nano /etc/snmp/snmpd.conf
```

**Required configuration entries:**
```
rocommunity public
agentAddress udp:161
```

```bash
# Restart and verify
sudo systemctl restart snmpd
sudo systemctl status snmpd
```

---

## Functional Validation (Windows Client — Pre-Attack)

### FTP Validation

```cmd
ftp 192.168.106.131
# Username: majorwz
# Password: [password]
ls
```
*Result: Successful authentication, directory listing accessible.*

### SNMP Validation
```cmd
ping 192.168.106.131
```
*Result: Network reachable, SNMP service active and responsive.*

---

## Attack Simulations

---

## Service 1: FTP Attacks

### Attack 1 — FTP Service Enumeration (Nmap)

**Objective:** Identify the FTP service version and configuration details.

**Command (Kali):**
```bash
nmap -sV -p 21 192.168.106.131
```

**What this does:**
- Scans port 21
- Identifies service name and version string
- Confirms the attack surface

**Result:** vsftpd version disclosed. Version information enables targeted CVE research.

**Confirm on Ubuntu:**
```bash
vsftpd -V
```

**Analysis:** Service version exposure allows attackers to look up known exploits and target weaknesses specific to the identified version.

---

### Attack 2 — FTP Brute-Force (Hydra)

**Objective:** Attempt credential compromise via automated password guessing.

A custom wordlist (`shortlists.txt`) containing 21 passwords was created to simulate a controlled dictionary attack.

**Command (Kali):**
```bash
hydra -l ftpuser -P /home/majorwz/Desktop/shortlists.txt ftp://192.168.106.131
```

**What this does:**
- Iterates through all 21 passwords in the wordlist
- Tests each against the FTP authentication endpoint
- Reports any successful credential match

**Log Evidence (Ubuntu):**
```bash
sudo journalctl -u vsftpd --since "10 minutes ago"
```

**Log Analysis Table:**

| Attack End Time (Kali) | vsftpd Log Timestamp (Ubuntu) | Attempts (Kali) | Attempts on Ubuntu Log | Auth Result | Attacker IP |
|------------------------|-------------------------------|-----------------|------------------------|-------------|-------------|
| 2025-12-18 14:08:26 | Dec 18 13:08:20 | 21 | 21 | All failed | 192.168.106.134 |

**Analysis:** Brute-force attacks exploit weak password policies and the absence of login rate limiting. All 21 attempts were logged, confirming the attack was detected in server logs but not blocked.

---

### Attack 3 — FTP Denial-of-Service (hping3 SYN Flood)

**Objective:** Disrupt FTP service availability using a TCP SYN flood.

**Command (Kali):**
```bash
sudo hping3 -S 192.168.106.131 -p 21 --flood
# Run briefly — a few seconds to minutes only
```

**What this does:**
- Sends a continuous stream of TCP SYN packets to port 21
- Exhausts server connection resources
- Prevents legitimate users from connecting

**Evidence:**
- Windows 8.1 FTP connection returned: *"connection closed by remote host"*
- FTP service became unavailable during flood
- Service restored after attack was stopped

**Flood Measurement Table:**

| Flood Started | Flood Stopped | Total Duration | Packets Transmitted |
|---------------|---------------|----------------|---------------------|
| 2:05 PM | 2:12 PM | 7 minutes | 19,169,153 packets |

**Analysis:** The absence of traffic filtering and rate limiting on port 21 allows a single attacker to completely disrupt service availability, affecting all legitimate users.

---

## Service 2: SNMP Attacks

### Attack 4 — SNMP Information Disclosure (snmpwalk)

**Objective:** Extract sensitive system information using the default community string.

**Command (Kali):**
```bash
snmpwalk -v2c -c public 192.168.106.131
```

**What this does:**
- Queries SNMP OID tree using default `public` community string
- Extracts system details, network interfaces, running processes, installed packages

**Data Exposed:**
- Ubuntu live server hostname and OS details
- Client server IP addresses
- List of installed software packages
- SNMP service command log entries
- Network interface configurations

**Analysis:** Default SNMP community strings are publicly known. Any attacker with network access can enumerate the entire system without any credentials.

---

### Attack 5 — SNMP Community String Brute-Force (Nmap)

**Objective:** Identify weak or non-standard SNMP community strings.

**Command (Kali):**
```bash
nmap -sU -p 161 --script snmp-brute 192.168.106.131
```

**Result:** Community string `public` discovered and confirmed.

**Analysis:** Default community strings remain unchanged in many deployments. This script confirms the string in seconds, demonstrating that legacy authentication is trivially defeated.

---

### Attack 6 — SNMP System Metadata Enumeration (Nmap scripts)

**Objective:** Enumerate SNMP-exposed system metadata and device information.

**Command (Kali):**
```bash
nmap -sU -p 161 --script snmp-info 192.168.106.131
```

**Data Exposed:**
- System description and OID
- Device uptime
- Contact and location fields
- Network interface metadata

**Analysis:** SNMP enumeration allows attackers to map internal infrastructure, identify device types, and plan further targeted attacks against the network.

---

## Evidence Summary

| Attack | Tool | Target Port | Key Finding | Risk |
|--------|------|-------------|-------------|------|
| FTP Enumeration | Nmap | TCP 21 | vsftpd version disclosed | Medium |
| FTP Brute-Force | Hydra | TCP 21 | All 21 attempts logged, rate not limited | High |
| FTP SYN Flood | hping3 | TCP 21 | Service disrupted — 19.1M packets in 7 min | High |
| SNMP Info Disclosure | snmpwalk | UDP 161 | Full system info via default community string | High |
| SNMP String Brute-Force | Nmap | UDP 161 | Community string `public` discovered | High |
| SNMP Metadata Enum | Nmap | UDP 161 | System OID, uptime, interface data exposed | Medium |

---

## Countermeasures & Mitigations

### FTP Hardening

**Authentication:**
- Enforce strong, unique passwords for all FTP accounts
- Implement account lockout after failed attempts (e.g., `pam_tally2`)
- Consider replacing FTP entirely with SFTP (SSH-based, encrypted)

**DoS Prevention:**
- Enable rate limiting and connection throttling on vsftpd
- Configure max connections: `max_clients=10`, `max_per_ip=2` in `/etc/vsftpd.conf`
- Deploy firewall rules to limit SYN packet rates:
  ```bash
  iptables -A INPUT -p tcp --syn --dport 21 -m limit --limit 1/s -j ACCEPT
  ```

**Monitoring:**
- Enable vsftpd logging: `xferlog_enable=YES`
- Monitor logs with: `journalctl -u vsftpd -f`

### SNMP Hardening

**Protocol:**
- Upgrade from SNMPv2c to SNMPv3 (authentication + encryption)
- Disable SNMPv1/v2c entirely if not required

**Community Strings:**
- Change default `public` community string immediately
- Use complex, randomly generated strings

**Access Control:**
- Restrict SNMP queries to trusted management IPs only:
  ```
  rocommunity <strong_string> 192.168.106.0/24
  ```
- Disable write access (`rwcommunity`) unless required

**If SNMP is not needed:** `sudo systemctl disable snmpd --now`
