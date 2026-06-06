# TryHackMe - Juicy Details

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Easy  
**Category:** Forensics, Log Analysis  
**Room:** [Juicy Details](https://tryhackme.com/room/juicydetails)  
**Type:** Log investigation (no deployable machine)

## Introduction

> *A popular juice shop has been breached! Analyze the logs to see what had happened...*

This room simulates a **digital forensics** investigation. You receive server logs from a compromised OWASP Juice Shop instance and must reconstruct the attacker's full kill chain — from scanning to shell access.

**Objectives:**
- Introduction — confirm flag format and download logs
- Task 2: Reconnaissance — identify tools, vulnerable endpoints, and attack vectors
- Task 3: Stolen Data — determine what was exfiltrated and how the attacker gained access

---

## Introduction — Are you ready?

You were hired as an analyst for a major juice shop chain. An attacker breached the network. Your job:

1. Identify **techniques and tools** used
2. Find **vulnerable endpoints**
3. Determine **what sensitive data** was stolen

Download **`logs.zip`** from the room, extract it, then type **`I am ready!`** on TryHackMe to begin.

| Question | Answer |
|----------|--------|
| Are you ready? | `I am ready!` |

Three log files matter:

| File | Purpose |
|------|---------|
| `access.log` | Apache/web requests — primary source |
| `auth.log` | SSH and system authentication |
| `vsftpd.log` | FTP transfers |

```bash
unzip logs.zip
cd logs/
```

---

## Task 2 — Reconnaissance

Most answers live in **`access.log`**. Filter by User-Agent strings — automated tools leave distinctive signatures at the end of each line.

### Identify attacker tools (in order of appearance)

Legitimate browser traffic (Firefox, Chrome) should be excluded. Tool markers appear in User-Agent headers:

```bash
grep -v Firefox access.log | grep -v Chrome | less
```

Or search each tool explicitly and note **chronological order** in the log:

```bash
grep -i nmap access.log | head -1
grep -i hydra access.log | head -1
grep -i sqlmap access.log | head -1
grep -i curl access.log | head -1
grep -i feroxbuster access.log | head -1
```

Typical markers: `(Nmap)`, `(Hydra)`, `(http://sqlmap.org)`, `curl/`, `feroxbuster`.

**Order:** scanning → brute-force → SQLi → manual HTTP → directory enumeration.

---

### Brute-force endpoint

Hydra targets login forms. Find all Hydra requests and note the URI:

```bash
grep -i hydra access.log | head -5
```

Look for `POST` requests to the login API path.

---

### SQL injection endpoint and parameter

sqlmap entries reveal both the vulnerable route and query parameter:

```bash
grep -i sqlmap access.log | head -3
```

The request URL follows the pattern `/rest/products/search?q=...` — the injectable parameter is visible in the query string.

---

### File retrieval endpoint

After exploitation, the attacker enumerates directories with **feroxbuster** looking for file download paths:

```bash
grep -i feroxbuster access.log | tail -20
```

FTP-related paths appear late in the attack timeline. Cross-reference with `vsftpd.log` for confirmation.

---

## Task 3 — Stolen Data

### Email scraping — website section

Search for repeated GET requests that could expose user emails:

```bash
grep -i reviews access.log | head -10
```

Juice Shop exposes reviewer data via product review API routes (`/rest/products/<id>/reviews`). The room asks for the **section name**, not the URL path.

---

### Brute-force success and timestamp

Hydra runs with multiple threads (default 16). A successful login returns **HTTP 200** instead of 401:

```bash
grep -i hydra access.log | grep " 200 "
```

The timestamp is in Apache combined log format: `[11/Apr/2021:09:xx:xx +0000]`.

Answer format: `Yay, <timestamp>` or `Nay` if no 200 found.

---

### SQL injection — exfiltrated user fields

Inspect sqlmap UNION SELECT payloads and successful (200) responses:

```bash
grep -i sqlmap access.log | grep -i union | tail -5
grep -i select access.log | grep " 200 " | tail -5
```

The attacker dumps columns from the `Users` table — identify which **user information fields** were retrieved (exclude internal IDs).

---

### Files downloaded via FTP

Switch to **`vsftpd.log`**:

```bash
grep -i download vsftpd.log
```

Two backup files were retrieved. List both filenames (comma-separated, order as in logs).

---

### FTP service and account

From the same FTP log, identify:
- Which **service** was used (protocol name)
- Which **account** authenticated (anonymous FTP is common on misconfigured servers)

```bash
grep -i "anonymous\|login" vsftpd.log
```

---

### Shell access — service and username

Check **`auth.log`** for successful remote login:

```bash
grep -i accepted auth.log
```

Look for `Accepted password for <user> from <attacker_ip> ... ssh2`.

---

## Answers Reference

| Task | Question | How to obtain |
|------|----------|---------------|
| Intro | Are you ready? | Type `I am ready!` |
| 2 | Tools (ordered) | User-Agent order in `access.log` |
| 2 | Brute-force endpoint | `grep Hydra access.log` |
| 2 | SQLi endpoint | `grep sqlmap access.log` |
| 2 | SQLi parameter | Query string in sqlmap URLs |
| 2 | File retrieval endpoint | feroxbuster + `/ftp` in logs |
| 3 | Email scrape section | `/reviews` API pattern |
| 3 | Brute-force success | Hydra + HTTP 200 timestamp |
| 3 | SQLi data fields | UNION SELECT on Users table |
| 3 | Downloaded files | `vsftpd.log` DOWNLOAD lines |
| 3 | FTP service/account | `vsftpd.log` login |
| 3 | Shell service/user | `auth.log` Accepted |

*Specific answers intentionally omitted. Complete the room on [TryHackMe](https://tryhackme.com/room/juicydetails).*

---

## Conclusion

### What I learned

| Skill | Takeaway |
|-------|----------|
| Log correlation | `access.log` + `auth.log` + `vsftpd.log` tell one story |
| User-Agent forensics | Tools sign their requests — nmap, Hydra, sqlmap, feroxbuster |
| Attack timeline | Log order reveals kill chain stages |
| HTTP status codes | 401 flood → single 200 = successful brute-force |
| SQLi in logs | sqlmap URLs expose endpoint, parameter, and dumped columns |
| FTP forensics | `vsftpd.log` records anonymous logins and file transfers |
| SSH forensics | `auth.log` confirms post-exploitation shell access |

Juice Shop is a deliberately vulnerable app — this room shows how a real breach looks in plain-text logs, and why centralized logging matters.
