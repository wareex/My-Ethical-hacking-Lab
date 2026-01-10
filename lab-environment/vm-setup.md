# Virtual Machine Setup

## Virtualization Platform
- VMware Workstation Pro 17
- Host Operating System: Windows 10/11

---

## Virtual Machines Deployed

### 1. Kali Linux (Attacker)
- Version: Kali Linux 2025.x
- Purpose: Penetration testing and attack execution
- Network Adapter: NAT
- Tools Used: Nmap, Hydra, tcpdump, hping3, Metasploit

### 2. Ubuntu Server (Target)
- Version: Ubuntu Server 22.04 LTS
- Purpose: Hosting vulnerable services
- Network Adapter: NAT
- Services:
  - FTP (vsftpd)
  - SNMP (snmpd)

### 3. Windows 8.1 (Client)
- Purpose: Functional validation of services
- Network Adapter: NAT
- Used to confirm service availability before and after attacks

### 4. Metasploitable 2
- Purpose: Intentionally vulnerable training VM
- Network Adapter: NAT / Host-Only
- Contains known vulnerable services

---

## IP Addressing
All machines were assigned IPs within the same private subnet.

| Machine | IP Address |
|------|------|
| Kali | 192.168.106.134 |
| Ubuntu Server | 192.168.106.131 |
| Windows 8.1 | 192.168.106.133 |
| Metasploitable 2 | 192.168.106.135 |

---

## Connectivity Validation
ICMP ping tests were conducted between all machines to confirm network reachability
before service deployment and attack execution.

Screenshots:
- VMware overview
- Network adapter configuration
- IP address confirmation
