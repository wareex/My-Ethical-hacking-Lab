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
| Attacker | Kali Linux 2025.4 | 192.168.106.134 | Hydra, Burp Suite, Hashcat, tcpdump |

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

### Step 3b: Harden MariaDB Against the CIS Benchmark

**Purpose:** Apply the Center for Internet Security (CIS) Benchmark for MariaDB — an internationally recognised, consensus-based security configuration standard used in healthcare environments and HIPAA-compatible frameworks. This transforms hardening from instinct-driven to standards-driven, and establishes a credible security baseline before any attack testing begins.

**RAT Mapping:** Establishes the capable guardian at the database layer — closing default configuration vulnerabilities that a motivated offender would exploit to gain unauthorised access.

#### Step 3b i: Verify MariaDB Version Currency (CIS Control 1.1)

CIS Control 1.1 requires the most recent, fully patched version of MariaDB. Unpatched engines contain publicly disclosed vulnerabilities that can bypass authentication or expose data.

```bash
mariadb --version
sudo apt update && sudo apt upgrade -y
```

#### Step 3b ii: Apply CIS Controls to MariaDB Configuration

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Locate the `[mysqld]` section and add or verify the following:

```ini
[mysqld]

# CIS Control 2.1 — Disable local_infile
# Prevents reading arbitrary OS files via SQL (e.g. LOAD DATA INFILE '/etc/passwd')
local_infile = 0

# CIS Control 3.1 — Bind database to localhost only
# Ensures MariaDB cannot be reached directly from the network
bind-address = 127.0.0.1

# CIS Control 6.1 — Enable error logging
log_error = /var/log/mysql/error.log

# CIS Control 6.2 — Set log_warnings level
# Captures connection errors and aborted clients
log_warnings = 2

# CIS Control 4.3 — Disable symbolic links
# Prevents symlink-based file system attacks
symbolic-links = 0
```

Save and exit (`Ctrl+X`, then `Y`, then `Enter`).

#### Step 3b iii: Restart MariaDB

```bash
sudo systemctl restart mariadb
sudo systemctl status mariadb
```

#### Step 3b iv: Verify CIS Controls Are Active

```bash
sudo mysql -u root -p
```

```sql
-- CIS 2.1: local_infile must be OFF
SHOW VARIABLES LIKE 'local_infile';

-- CIS 3.1: bind_address must be 127.0.0.1
SHOW VARIABLES LIKE 'bind_address';

-- CIS 6.1: log_error must show a path
SHOW VARIABLES LIKE 'log_error';

-- CIS 6.2: log_warnings must be 2
SHOW VARIABLES LIKE 'log_warnings';

-- CIS 4.3: symbolic links must be DISABLED
SHOW VARIABLES LIKE 'have_symlink';
```

#### Step 3b v: Verify Anonymous Users and Test Database Removal (CIS Controls 4.1 & 4.2)

```sql
-- CIS 4.1: No anonymous user accounts should exist
SELECT user, host FROM mysql.user WHERE user = '';

-- CIS 4.2: Test database must not exist
SHOW DATABASES LIKE 'test';
```

Both queries should return empty sets.

**CIS Benchmark Control Mapping Summary:**

| CIS Control | Description | Setting | Value | Status |
|-------------|-------------|---------|-------|--------|
| 1.1 | Version currency | MariaDB patched | 10.6.x (latest) | ✔ Satisfied |
| 2.1 | Disable local_infile | local_infile | OFF | ✔ Satisfied |
| 3.1 | Bind to localhost | bind-address | 127.0.0.1 | ✔ Satisfied |
| 4.1 | Remove anonymous users | mysql.user | Empty set | ✔ Satisfied |
| 4.2 | Remove test database | SHOW DATABASES | Empty set | ✔ Satisfied |
| 4.3 | Disable symbolic links | symbolic-links | 0 / DISABLED | ✔ Satisfied |
| 6.1 | Enable error logging | log_error | /var/log/mysql/error.log | ✔ Satisfied |
| 6.2 | Set log warnings level | log_warnings | 2 | ✔ Satisfied |

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
    ServerName 192.168.106.131
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
Maximum Failed Login Attempts From IP Address:          0
Time to Reset Failed Login Attempts From IP:            0
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
new_login_session_management=1&languageChoice=1&authUser=admin&clearPass=wrongpassword
```

**Key findings:**
- Form action path → `/openemr/interface/login/login.php`
- Username field → `authUser`
- Password field → `clearPass`

---

### Step 10b: Demonstrate Network-Level Vulnerability — HTTP Traffic Sniffing

**Purpose:** Extend the Burp Suite finding by demonstrating a passive, harder-to-detect attack vector. Unlike Burp Suite — which requires the attacker to act as an explicit proxy — `tcpdump` operates passively on the network interface. It observes all traffic on the subnet without sending a single packet, modifying any connection, or generating any server-side log entry. Any attacker on the same network segment can execute this attack silently.

**What it proves:**
- HTTP traffic exposes credentials and patient data to any passive observer on the network segment
- The attack surface extends beyond the login endpoint to every data exchange, including patient records retrieved by authenticated users
- Application-layer controls (lockout, Fail2Ban) **cannot prevent or detect this attack** — no failed login attempts are generated; the attacker reads the correct password before it reaches the server
- A network-layer control (SSL/TLS) is required as a complementary mitigation — establishing the case for Mitigation 6

**Walkthrough (Kali machine):**

```bash
# Terminal 1 — Start passive capture against the OpenEMR server on port 80
sudo tcpdump -i eth0 -A \
  'tcp port 80 and host 192.168.106.131' \
  -w /tmp/http_baseline_capture.pcap
```

```bash
# Terminal 2 — Simulate a login to generate HTTP POST traffic
curl -X POST http://192.168.106.131/openemr/interface/login/login.php \
  -d 'authUser=NUA-admin-82&clearPass=wareezwareez&languageChoice=1'
```

Stop the capture (`Ctrl+C`), then read the file and filter for credentials:

```bash
sudo tcpdump -r /tmp/http_baseline_capture.pcap -A \
  | grep -E 'authUser|clearPass|patient'
```

**Result:**
```
authUser=NUA-admin-82&clearPass=wareezwareez&languageChoice=1
```

Credentials appear in completely readable plain text within the HTTP POST body. No decryption, no special tooling, no active connection manipulation was required.

**Baseline Finding:** HTTP traffic exposes credentials and patient data to any passive observer on the network segment. This vulnerability cannot be addressed by application-layer controls and requires encryption at the transport layer (Mitigation 6).

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

### Application Stack & Layered Defence Rationale

OpenEMR runs as a LAMP stack with four distinct layers, each carrying its own attack surface. No single mitigation is sufficient — each layer requires its own defence, and each mitigation is explicitly mapped to the layer it targets.

| Layer | Component | Version | Attack Surface |
|-------|-----------|---------|---------------|
| Operating System | Ubuntu Server | 22.04 LTS | Process access, filesystem, IP-level network |
| Web Server | Apache HTTP Server | 2.4.x | Traffic interception, unencrypted data in transit |
| Application | OpenEMR (PHP) | 8.0.0 / PHP 8.2 | Login endpoint, credential brute-force |
| Database | MariaDB | 10.6.x | PHI at rest, credential hash extraction |

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
  -o ~/hydra_m1_results.txt
```

*Result: Hydra stalled. No credential identified. Correct password at position 5 never reached within a single attack window.*

**Burp Suite Intruder Rerun Results (after lockout):**

| Request | Payload | Status Code | Length |
|---------|---------|-------------|--------|
| 0 | (baseline) | 200 | 913 |
| 1–15 | all passwords | 200 | 912–916 |

*All 15 requests — including `wareezwareez` — returned identical HTTP 200 responses. No HTTP 302 observed. No session cookie issued.*

**Analysis:** Account lockout eliminated the distinguishing HTTP 302 signal entirely. Even the correct credential produced an identical response to all incorrect ones — operationally impossible for the attacker to identify success. Under RAT: first capable guardian introduced.

**Residual Vulnerability:** Lockout operates at the application layer only. It locks the account but not the attacker's machine — Kali IP (`192.168.106.134`) continues to communicate freely. No alert generated, no forensic record created. Network traffic and PHI on disk are completely unaffected. These exposures justify M2, M5, and M6.

---

### Mitigation 2 — Fail2Ban (System-Level IP Banning)

**Purpose:** Operate at the network/OS level to ban the attacker's IP entirely after threshold is reached — blocking all requests regardless of username.

**Why this extends beyond account lockout:** Account lockout blocks the account. Fail2Ban blocks the attacker's machine — even multi-username attacks from the same IP are stopped.

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

**Add the following jail block:**
```ini
[openemr]
enabled  = true
port     = http,https
filter   = openemr
logpath  = /var/log/apache2/access.log
maxretry = 5
findtime = 60
bantime  = 600
```

#### Step C: Restart Fail2Ban

```bash
sudo systemctl restart fail2ban

# Verify jails are active
sudo fail2ban-client status
sudo fail2ban-client status openemr
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
target    prot opt source          destination
REJECT    tcp  --  192.168.106.134  anywhere   reject-with icmp-port-unreachable
```

**Analysis:** After 5 POST attempts, Fail2Ban inserted the attacker's IP into the `f2b-openemr` iptables chain. All subsequent requests from Kali were dropped at the firewall level — the Apache process never received them. Under RAT: second capable guardian — network-level barrier.

**Residual Vulnerability:** Fail2Ban is agnostic to username — a material improvement over lockout alone. However, neither M1 nor M2 protects against direct OS-level access to MariaDB data files or against passive network sniffing. These require database-layer and transport-layer mitigations.

---

### Mitigation 3 — bcrypt Password Hashing (Database Layer)

**Purpose:** Ensure that even if an attacker bypasses authentication and accesses the database directly — through SQL injection, a misconfigured root account, or physical media access — stolen password hashes cannot be reversed within a practical timeframe.

**RAT Mapping:** Neutralises the **suitability of the target** at the credential storage layer — the motivated offender reaches the data but it carries no usable value without the key.

#### Step A: Access MySQL Database

```bash
mysql -u root
```

```sql
SHOW DATABASES;
USE openemr;
SHOW TABLES;
-- Look for: users_secure

SELECT username, password FROM users_secure;
```

**Extracted hash:**

| Username | Password Hash |
|----------|--------------|
| NUA-admin-82 | `$2y$10$nu1Vrjx6D2pe0us8CTgV7u1smkP5P5yz8boRcUnjcZJSF63J2xZSy` |

The `$2y$10$` prefix identifies this as bcrypt with a cost factor of 10.

#### Step B: Prepare Hash for Hashcat

```bash
# Replace $2y$ with $2a$ (Hashcat requirement)
# Final hash: $2a$10$nu1Vrjx6D2pe0us8CTgV7u1smkP5P5yz8boRcUnjcZJSF63J2xZSy

nano hash.txt
# Paste the converted hash, save and exit
```

#### Step C: Run Hashcat

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

**Analysis:** bcrypt cost factor 10 reduces hash throughput by many orders of magnitude. Even with the correct password in the wordlist, cracking is computationally infeasible within any meaningful timeframe.

**Residual Vulnerability:** bcrypt protects only the password column in `users_secure`. It does not protect `patient_data`, `form_encounter`, `prescriptions`, or any other PHI table. An attacker with OS-level access to MariaDB data files can read every patient record without encountering bcrypt at all. This justifies Mitigation 5.

---

### Mitigation 4 — Real-Time IDS Script (Custom Monitoring)

**Purpose:** Detect ongoing brute-force attacks in real time and generate forensic alerts — independent of whether upstream controls block the attack.

**RAT Mapping:** Most visible expression of capable guardianship — removes the attacker's anonymity and creates an evidentiary trail for incident response.

**Regulatory significance:** In healthcare environments, detection triggers HIPAA breach notification assessment and mandatory incident response workflows under GDPR Article 33.

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
- Alert persisted in `/var/log/openemr_ids_alerts.log` as forensic record — survives system reboot

**Residual Vulnerability:** The IDS provides detection and forensic visibility but does not encrypt database files or protect traffic in transit. PHI remains readable from raw `.ibd` files on disk, and credentials remain visible in HTTP traffic. These require Mitigations 5 and 6.

---

### Mitigation 5 — Encryption at Rest (MariaDB InnoDB Tablespace Encryption)

**Purpose:** Ensure that even if an attacker bypasses all authentication controls and gains direct OS-level or root-level access to the database files — through a stolen backup, misconfigured account, or physical media access — PHI remains inaccessible and unreadable.

**RAT Mapping:** Removes the **suitability of the target** at the physical storage layer. Even when the motivated offender reaches the database files, the absence of the decryption key neutralises the value of what they have accessed.

**Implementation:** MariaDB InnoDB Tablespace Encryption using the `file_key_management` plugin. This is transparent — authorised SQL access decrypts data on the fly, while raw `.ibd` files on disk are AES-256 encrypted.

### Verification: Logical View vs. Physical View

#### A. The Logical View (Authorised SQL Access)

Log into the database and query a patient record to confirm encryption is transparent and does not break application functionality:

```bash
sudo mysql -u root -p
```

```sql
USE openemr;
SELECT fname, lname, phone_home FROM patient_data LIMIT 1;
```

**Result:** Returns readable patient data — `Davie | Lashma | +44129394944`. The database engine decrypts transparently for authorised users. This proves the system is operational — the capable guardian functions correctly for legitimate access.

#### B. The Physical View (Unauthorised Disk Access)

Step outside the database engine and attempt to read the raw data file directly from disk — simulating an attacker who has stolen a backup or gained root access:

```bash
# Attempt to find patient data in the raw tablespace file
sudo strings /var/lib/mysql/openemr/patient_data.ibd | grep "Davie"
sudo strings /var/lib/mysql/openemr/patient_data.ibd | grep -i "1994"

# Attempt to search for any readable text in the file
sudo strings -n 3 /var/lib/mysql/openemr/patient_data.ibd | head -80
```

**Result:** Commands return no results — only a stream of binary noise. No patient names, dates of birth, diagnoses, or contact records are recoverable from the raw file.

**Comparative Evidence:**

| Metric | Authorised View (DB Engine) | Unauthorised View (Raw Disk) |
|--------|----------------------------|------------------------------|
| Tool Used | mysql client / SQL | strings / hexdump |
| Access Level | Logical (Database User) | Physical (OS / Root User) |
| Data Visibility | **Plaintext** (decrypted on-the-fly) | **Ciphertext** (AES-256 encrypted) |
| Mitigation Status | Operational | **Effective** |

**Analysis:** Unlike the baseline state where these commands would reveal PHI in plain text, the encryption at rest has transformed patient data into high-entropy ciphertext. Physical media theft, unauthorised root access, or backup file exfiltration all yield the same result: an unreadable ciphertext stream that cannot be leveraged without the key at `/etc/mysql/encryption/keyfile`.

**Residual Vulnerability:** InnoDB tablespace encryption closes the disk-level PHI access vector conclusively. However, it does not protect data in motion. Every HTTP request between the browser and the server — including authenticated patient record retrievals — still travels in plain text. This justifies Mitigation 6.

---

### Mitigation 6 — Encryption in Transit (SSL/TLS on Apache)

**Purpose:** Install an SSL/TLS certificate and enforce HTTPS so that all traffic between the client and server is encrypted. A passive sniffer captures only TLS-encrypted binary data — no credentials, no session tokens, no patient records are visible.

**RAT Mapping:** Removes the **opportunity for network-level interception**. The motivated offender can observe traffic but cannot decode it — the suitable target (readable PHI in transit) no longer exists.

#### Baseline Reconfirmation Before SSL

```bash
# Terminal 1 — Capture HTTP traffic
sudo -v
sudo tcpdump -i eth0 -U -w /tmp/pre_ssl_capture.pcap \
  'tcp port 80 and host 192.168.106.131' &

# Terminal 2 — Trigger a login request
curl -L -X POST http://192.168.106.131/openemr/interface/login/login.php \
  -d 'authUser=NUA-admin-82&clearPass=wareezwareez&languageChoice=1'

# Terminal 1 — Stop capture and inspect
sleep 2 && sudo kill $!
tcpdump -r /tmp/pre_ssl_capture.pcap -A | grep -iE 'authUser|clearPass'
```

**Result:** `authUser=NUA-admin-82&clearPass=wareezwareez` visible in plain text — vulnerability confirmed before proceeding.

#### Step A: Enable Apache SSL Module

```bash
sudo a2enmod ssl
sudo a2enmod headers
sudo systemctl restart apache2
apache2ctl -M | grep -E 'ssl|headers'
```

#### Step B: Generate Self-Signed SSL Certificate

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/openemr-selfsigned.key \
  -out /etc/ssl/certs/openemr-selfsigned.crt \
  -subj "/C=NG/ST=Research/L=Lab/O=PMS-Research/CN=192.168.106.131"

# Verify certificate details
sudo openssl x509 -in /etc/ssl/certs/openemr-selfsigned.crt \
  -text -noout | grep -E 'Subject:|Not After|Public-Key'
```

#### Step C: Reconfigure Apache Virtual Host to Enforce HTTPS

```bash
sudo nano /etc/apache2/sites-available/openemr.conf
```

```apache
# Block 1: Redirect all HTTP traffic to HTTPS
<VirtualHost *:80>
    ServerName 192.168.106.131
    Redirect permanent / https://192.168.106.131/
</VirtualHost>

# Block 2: Serve OpenEMR over TLS only
<VirtualHost *:443>
    ServerName 192.168.106.131
    DocumentRoot /var/www/html/openemr
    SSLEngine on
    SSLCertificateFile     /etc/ssl/certs/openemr-selfsigned.crt
    SSLCertificateKeyFile  /etc/ssl/private/openemr-selfsigned.key
    SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
    SSLCipherSuite HIGH:!aNULL:!MD5
    Header always set Strict-Transport-Security 'max-age=63072000'
    <Directory /var/www/html/openemr>
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    ErrorLog  ${APACHE_LOG_DIR}/openemr_error.log
    CustomLog ${APACHE_LOG_DIR}/openemr_access.log combined
</VirtualHost>
```

#### Step D: Reload Apache and Verify Redirect

```bash
sudo systemctl reload apache2
sudo systemctl status apache2

# Confirm HTTP redirects to HTTPS
sudo curl -v http://192.168.106.131/openemr 2>&1 | grep -E 'HTTP/|Location'
```

**Expected:** `HTTP/1.1 301 Moved Permanently` with `Location: https://192.168.106.131/`

#### Step E: Rerun Network Sniff — Post-SSL Verification

```bash
# Terminal 1 — Capture HTTPS traffic on port 443
sudo tcpdump -i eth0 -A 'tcp port 443 and host 192.168.106.131' \
  -w /tmp/post_ssl_capture.pcap &

# Terminal 2 — Trigger HTTPS login
curl -k -s -X POST https://192.168.106.131/openemr/interface/login/login.php \
  -d 'authUser=NUA-admin-82&clearPass=wareezwareez&languageChoice=1'

# Terminal 1 — Stop and read
sudo kill %1
sudo tcpdump -r /tmp/post_ssl_capture.pcap -A | head -40
```

**Result:** Only TLS-encrypted binary data captured. No credentials, no patient records visible.

**Before / After Comparison:**

| Metric | Before M6 (HTTP) | After M6 (HTTPS) |
|--------|-----------------|------------------|
| Credentials visible in packet capture | YES — plain text | NO — TLS encrypted |
| Patient data visible in capture | YES | NO — TLS encrypted |
| HTTP still accessible | YES | NO — 301 redirect enforced |
| Passive sniffer effective | YES | NO — binary noise only |
| Protocol in use | HTTP/1.1 (plain text) | TLS 1.2/1.3 only |
| HSTS header enforced | NO | YES — max-age=63072000 |
| Login endpoint attacks blocked | YES (M1+M2) | YES (all previous active) |

**Residual Vulnerability:** Encryption in transit closes the final PHI exposure vector that operates independently of authentication controls. Combined with M5, PHI is now protected both at rest and in transit. However, all six mitigations share one remaining limitation: they cannot prevent access by an attacker who already possesses valid credentials through phishing, social engineering, or insider threat. This justifies Mitigation 7.

---

### Mitigation 7 — Multi-Factor Authentication (OpenEMR Native TOTP)

**Purpose:** Require a second factor — a time-based one-time password (TOTP) generated by an authenticator app on the registered device — in addition to username and password. Even if credentials are compromised by any means, the attacker cannot authenticate without simultaneously possessing the physical device.

**RAT Mapping:** Severs the link between credential knowledge and system access. The threat model shifts from **remote automated exploitation** to **targeted physical attack** — a qualitatively different and substantially more demanding category of threat.

#### Baseline Evidence Before MFA — Credentials Alone Produce Immediate Access

```bash
curl -k -v -X POST http://192.168.106.131/openemr/interface/login/login.php \
  -d 'authUser=NUA-admin-82&clearPass=wareezwareez&languageChoice=1' \
  2>&1 | grep -E 'HTTP/|Location|Set-Cookie'
```

**Result:** HTTP 302 redirect and session cookie issued immediately — no second factor required.

#### Step A: Enrol the Administrator Account

In a browser, navigate to:

```
Administration (right menu) → MFA Management
```

- Enable MFA for this account: ✔ (checked)
- Click **Register**
- OpenEMR generates a QR code for the account

Open Google Authenticator or Authy on a mobile device:
1. Tap `+` → Scan QR code
2. The app begins generating a rolling 6-digit code refreshing every 30 seconds

#### Step B: Verify MFA is Enforced at Login

Navigate to: `https://192.168.106.131/openemr`

- Username: `NUA-admin-82`
- Password: `wareezwareez`
- Click Login

**Result:** OpenEMR presents a TOTP prompt — password alone is no longer sufficient to establish a session.

#### Step C: Demonstrate That Hydra Cannot Bypass MFA

```bash
hydra -l NUA-admin-82 \
  -P ~/research_wordlist.txt \
  192.168.106.131 \
  http-post-form \
  "/openemr/interface/login/login.php:authUser=^USER^&clearPass=^PASS^&languageChoice=1:Invalid" \
  -V -t 5 -w 2
```

**Result:** Hydra cannot supply the TOTP token. Even correctly identifying the password produces no authenticated session — the attack is structurally defeated.

**Before / After Comparison:**

| Metric | Before M7 | After M7 |
|--------|-----------|---------|
| Correct credentials produce session | YES — immediately | NO — TOTP prompt shown |
| Session cookie issued on password alone | YES | NO |
| Hydra establishes authenticated session | YES (baseline) | NO — cannot supply TOTP |
| Attacker with stolen credentials succeeds | YES | NO — device also required |
| Attack type required to bypass | Remote automated | Physical device theft |
| Threat model category | Remote unauthenticated | Targeted physical attack |
| All previous mitigations still active | N/A | YES — cumulative stack |

**Analysis:** MFA completes the authentication stack. Credential compromise through brute force, wordlist attack, database breach, phishing, or social engineering is now insufficient for system access. The attacker must additionally compromise the physical registered device — moving the threat model into targeted physical attack territory. Under RAT: the capable guardian now operates at the credential layer itself, and all three enabling conditions are simultaneously neutralised.

---

## Final Results Summary

### Master Comparative Results Table

| Mitigation | Layer | Hydra Blocked | Burp Blocked | Sniffing Blocked | DB File Blocked | Cred Theft Blocked | Forensic Record |
|-----------|-------|:---:|:---:|:---:|:---:|:---:|:---:|
| None — Baseline | — | ❌ | ❌ | ❌ | ❌ | ❌ | None |
| M1: Account Lockout | Application | ✅ | ✅ | ❌ | ❌ | ❌ | None |
| M2: Fail2Ban | Network/OS | ✅ | ✅ | ❌ | ❌ | ❌ | Partial |
| M3: bcrypt | DB — Creds | ⚠️ | ⚠️ | ❌ | ❌ | ⚠️ | Partial |
| M4: IDS Monitor | Logging/OS | ✅ | ✅ | ❌ | ❌ | ❌ | ✅ |
| M5: Encryption at Rest | DB — PHI | ✅ | ✅ | ❌ | ✅ | ❌ | ✅ |
| M6: SSL/TLS (HTTPS) | Web Server | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| M7: MFA (TOTP) | Application | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

> ⚠️ = Partial protection only

**Key findings:**

- **M1 + M2** neutralised Hydra and Burp Suite Intruder but are entirely blind to attacks that bypass the login endpoint — direct database file access and network sniffing both succeed without triggering either control
- **M3** made password hash cracking computationally infeasible but leaves all PHI tables completely unencrypted at the disk level
- **M4** transformed the system from one that silently resists attacks to one that actively documents them — satisfying HIPAA and GDPR detection and audit trail requirements
- **M5** closed the disk-level PHI access vector conclusively — the same `strings` command that previously returned patient names and diagnoses returned only binary noise after encryption was applied
- **M6** closed the network interception vector — the same `tcpdump` capture that previously returned credentials in plain text returned only encrypted binary data after HTTPS was enforced
- **M7** addressed the final residual vulnerability — valid credentials alone are now insufficient; the threat model shifts from remote automated exploitation to targeted physical attack

**Combined defence posture:** The baseline system was compromised in under **1 second**. The fully hardened system simultaneously **prevents** (lockout), **blocks** (Fail2Ban), **neutralises offline** (bcrypt), **detects and records** (IDS), **encrypts on disk** (InnoDB TDE), **encrypts in transit** (SSL/TLS), and **requires physical possession** (MFA) — removing all three enabling conditions of Routine Activity Theory at once.

---

## Tools Reference

| Tool | Version | Purpose |
|------|---------|---------|
| Hydra | v9.5 | HTTP POST brute-force attack |
| Burp Suite Community | v2025.7.4 | Traffic interception, Intruder attack module |
| tcpdump | — | Passive network packet capture |
| Hashcat | v6.2.6 | bcrypt hash cracking attempt (mode 3200) |
| Fail2Ban | — | Automated IP banning via iptables |
| Custom IDS Script | — | Real-time brute-force detection + forensic logging |
| OpenSSL | — | Self-signed RSA 2048-bit TLS certificate |
| Google Authenticator | — | TOTP second-factor enrolment |
| OpenEMR | 8.0.0 | Target: Electronic Health Record web application |
| MariaDB | 10.6.x | Database — bcrypt extraction, InnoDB TDE |

---

## Research Framework

**Theoretical basis:** Routine Activity Theory (Cohen & Felson, 1979) applied to cybersecurity

> "A criminal act requires the convergence in space and time of a motivated offender, a suitable target, and the absence of a capable guardian."

This study demonstrates empirically that each mitigation layer corresponds directly to introducing a capable guardian or removing a target condition — progressively eliminating the prerequisites for a successful attack until all three are simultaneously neutralised.

---

> ⚠️ **Ethical Disclaimer:** This research was conducted entirely within an isolated virtual laboratory environment using synthetic data. No real patient data was accessed, processed, or stored at any point. All attacks were performed against locally controlled machines. This project is intended solely for academic and educational purposes. Unauthorised penetration testing against real systems is illegal.
