# 🛡️ Ethical Hacking & Penetration Testing Portfolio

> **All activities documented here were conducted in isolated, controlled virtual lab environments for educational and research purposes only. No real systems were targeted.**

---

## About

This portfolio documents hands-on penetration testing labs, vulnerability assessments, and mitigation research conducted as part of an ethical hacking programme. Each lab follows a structured methodology: environment setup → service configuration → functional validation → attack simulation → evidence collection → countermeasures.

**Core Competencies Demonstrated:**
- Virtual lab design and network segmentation
- Service configuration and vulnerability identification
- Penetration testing with industry-standard tools
- Attack documentation with log analysis
- Mitigation strategy design and implementation
- Research-grade testing methodology

---

## Lab Index

| # | Lab | Target | Services | Tools | Focus |
|---|-----|--------|----------|-------|-------|
| 01 | [Metasploitable 2 Exploitation](./01-Metasploitable2-Exploitation/) | Metasploitable 2 | SMB, Telnet, MySQL, HTTP, NFS | Nmap, Metasploit, Nikto | Multi-service exploitation |
| 02 | [FTP & SNMP Network Service Attacks](./02-FTP-SNMP-Ubuntu-Lab/) | Ubuntu Server | FTP (vsftpd), SNMP (snmpd) | Hydra, hping3, snmpwalk, Nmap | Auth attacks + DoS |
| 03 | [FTP & SNMP Extended Variant Lab](./03-FTP-SNMP-Extended-Lab/) | Ubuntu Server | FTP (vsftpd), SNMP (snmpd) | Hydra, hping3, Metasploit, Nmap | Deeper SNMP enumeration |
| 04 | [OpenEMR Brute-Force Attack & Mitigation Research](./04-OpenEMR-Brute-Force-Research/) | OpenEMR 8.0.0 | Web App Login (HTTP) | Hydra, Burp Suite, Hashcat, Fail2Ban | Research: attack → layered defence |

---

## Lab Environment Stack

All labs were built on **VMware Workstation Pro 17** using NAT-isolated networks.

| Role | OS | IP Address |
|------|----|------------|
| Attacker | Kali Linux 2025.x | 192.168.106.134 |
| Target (Ubuntu) | Ubuntu Server 22.04 LTS | 192.168.106.131 |
| Target (Legacy) | Metasploitable 2 | 192.168.106.135 |
| Client | Windows 8.1 x64 | 192.168.106.133 |

---

## Tools Used

`Nmap` `Metasploit Framework` `Hydra` `Burp Suite` `hping3` `snmpwalk` `Nikto` `Hashcat` `Fail2Ban` `MySQL Client` `Telnet` `showmount`

---

## Methodology

Each lab follows this structured approach:

```
1. Lab Setup          → VM provisioning, network configuration, IP assignment
2. Service Config     → Install, configure, and harden/weaken target services
3. Validation         → Confirm service availability from Windows client (baseline)
4. Attack Simulation  → Execute attacks with full command documentation
5. Evidence           → Screenshots, logs, terminal output, log analysis tables
6. Countermeasures    → Mitigation strategies, limitations, and recommendations
```

---

## Legal Disclaimer

All penetration testing documented in this portfolio was performed exclusively within self-contained virtual machines on a private host system. No external networks, third-party systems, or real-world infrastructure were targeted at any point. All techniques are presented for educational purposes in accordance with ethical hacking principles.
