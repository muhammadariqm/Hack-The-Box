# <samp>HACK THE BOX</samp>
## `A R C H I V E S`

<div align="center">
  <img src="htb.jpg" alt="HTB Banner" width="100%" style="border-radius: 10px; box-shadow: 0 10px 30px rgba(0,0,0,0.5);">

  <br>

  [![HTB](https://img.shields.io/badge/HackTheBox-Writeups-9fe52d?style=for-the-badge&logo=hackthebox&logoColor=white)](https://app.hackthebox.com/)
  [![Status](https://img.shields.io/badge/Status-Active-success?style=for-the-badge)](./)
  [![Tools](https://img.shields.io/badge/Tools-Custom-blueviolet?style=for-the-badge)](./nexus/exploits)

</div>

---

### 🟢 <samp>S Y S T E M S _ O N L I N E</samp>

| MACHINE | STATUS | DIFFICULTY | WRITEUP | TAGS |
| :--- | :---: | :---: | :---: | :--- |
| **Archetype** | `PWNED` | ![Easy](https://img.shields.io/badge/-Easy-green) | [📂](./htb-archetype-writeup.md) | `SMB` `MSSQL` |
| **Cap** | `PWNED` | ![Easy](https://img.shields.io/badge/-Easy-green) | [📂](./htb-cap-writeup.md) | `IDOR` `PCAP` |
| **FlagCommand** | `PWNED` | ![Easy](https://img.shields.io/badge/-Easy-green) | [📂](./htb-flagcommand-writeup.md) | `Web` `API` |
| **Reactor** | `PWNED` | ![Medium](https://img.shields.io/badge/-Medium-orange) | [📂](./htb-reactor-writeup.md) | `K8s` `Registry` |
| **Support** | `PWNED` | ![Easy](https://img.shields.io/badge/-Easy-green) | [📂](./htb-support-writeup.md) | `AD` `LDAP` |
| **TwoMillion** | `PWNED` | ![Easy](https://img.shields.io/badge/-Easy-green) | [📂](./htb-twomillion-writeup.md) | `RCE` `API` |
| **Unified** | `PWNED` | ![Medium](https://img.shields.io/badge/-Medium-orange) | [📂](./unified/HTB-UNIFIED-WRITEUP.md) | `Log4j` `Mongo` |

---

### ⚡ <samp>E X P L O I T _ L I B S</samp>

#### `N E X U S`
- `[CVE-2026-38526]` **Krayin RCE** - Remote shell via file upload bypass.
- `[RECON-TOOL]` **AutoRecon** - Advanced network mapping suite.

#### `O R I O N`
- `[CVE-2025-32432]` **Craft RCE** - Specialized environment breakout.
- `[CVE-2026-24061]` **Telnet Escalation** - Privilege escalation vector.

---

### 💻 <samp>T E R M I N A L</samp>
```bash
# Initialize local workspace
$ git clone https://github.com/m-ariq/ctf-writeup.git
$ cd ctf-writeup

# Deploy Nexus Recon
$ chmod +x ./nexus/exploits/recon.sh
$ ./nexus/exploits/recon.sh -t 10.10.x.x
```

<br>

<div align="center">
  <sub><b>ROOT EVERYTHING</b></sub><br>
  <code>01001000 01010100 01000010</code>
</div>


