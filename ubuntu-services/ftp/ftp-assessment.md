# FTP Service Penetration Test (vsftpd)

## 1. Scope and Authorization
- Target: Ubuntu Server (192.168.106.131)
- Service: FTP (vsftpd)
- Environment: Isolated lab
- Authorization: Self-owned infrastructure

---

## 2. Pre-Attack Functional Validation

Before testing, the FTP service was validated from a legitimate client.

**Client:** Windows 8.1  
**Command:**
ftp 192.168.106.131


Result:
- Successful authentication
- Directory listing accessible

![FTP Windows Validation](images/ftp-windows-validation.png)

This confirms the service was functioning normally prior to attack activity.

---

## 3. Attack 1 – FTP Service Enumeration

### Objective
Identify exposed services and versions.

### Methodology
Service enumeration was performed using Nmap.

**Command (Kali):**
nmap -sV -p 21 192.168.106.131


### Result
- FTP service detected
- vsftpd version disclosed

![Nmap Enumeration](images/nmap-enum.png)

### Impact
Service version disclosure enables targeted exploitation.

---

## 4. Attack 2 – Credential Strength Testing (Brute Force)

### Objective
Evaluate resistance to password-guessing attacks.

### Methodology
A controlled wordlist containing a limited number of passwords was used.

**Command:**
hydra -l ftpuser -P shortlists.txt ftp://192.168.106.131


### Evidence
![Hydra Attack](images/hydra-attack.png)

### Log Validation (Ubuntu)
sudo journalctl -u vsftpd


![vsftpd Logs](images/vsftpd-logs.png)

### Impact
- No rate limiting
- All attempts logged but not blocked

---

## 5. Attack 3 – Availability Testing (DoS Simulation)

### Objective
Assess FTP service resilience to high-volume traffic.

### Methodology
A short SYN flood was conducted.

**Command:**
sudo hping3 -S -p 21 --flood 192.168.106.131


### Functional Validation
During the attack, Windows client connections failed.

![DoS Validation](images/dos-validation.png)

### Impact
- Service unavailable
- Legitimate users denied access

---

## 6. Overall Impact Summary

| Property | Impact |
|------|------|
| Confidentiality | High |
| Integrity | Low |
| Availability | High |

---

## 7. Mitigation Recommendations
- Replace FTP with SFTP
- Enforce strong authentication
- Implement rate limiting
- Apply firewall restrictions

