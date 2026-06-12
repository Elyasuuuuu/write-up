# TryHackMe - Sequel Dump

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect hands-on analysis of the room PCAP.*

**Difficulty:** Hard  
**Category:** Forensics, SQL Injection  
**Room:** [Sequel Dump](https://tryhackme.com/room/hfb1sequeldump) (Premium)  
**Description:** Decipher captured traffic from an automated SQL injection attack and reconstruct the exfiltrated database dump.

## Introduction

A wave of suspicious HTTP requests hit a database-backed search application. Analysts suspect **sqlmap** performed a **blind boolean SQL injection**, leaking data character by character. A PCAP (`challenge.pcapng`) is your only evidence.

This is a **pure forensics** room: you do not exploit the live app. You reverse the attack from the wire and recover what was stolen — including the flag hidden in the last dumped row.

**Objectives:**
- Connect to the correct TryHackMe VPN (Premium lab network)
- Transfer `challenge.pcapng` off the room VM
- Identify sqlmap blind SQLi patterns in HTTP traffic
- Map true/false responses to binary-search thresholds
- Rebuild `profile_db.profiles` and extract the flag

---

## Prerequisites

| Tool | Purpose |
|------|---------|
| Wireshark / tshark | PCAP inspection and export |
| Python 3 + Scapy | Automated request/response pairing |
| TryHackMe Premium VPN | Reach `10.129.x.x` lab machines |

```bash
sudo apt install -y tshark python3-scapy
```

---

## VPN and machine access

The room VM lives on the **Premium lab network** (`10.129.x.x`). The KoTH VPN (`ElyasuuuuuKOTH.ovpn`) routes other subnets (`10.10.x`, `10.101.x`, …) but **not** `10.129.x` — ping will fail until you switch profiles.

Use the **Premium** OpenVPN config from your TryHackMe account (e.g. `eu-west-3-<user>-premium.ovpn`):

```bash
# Stop KoTH VPN if active
sudo kill $(cat /tmp/koth-vpn.pid 2>/dev/null)

# Start Premium VPN
sudo openvpn --config ~/Bureau/eu-west-3-<user>-premium.ovpn --disable-dco --daemon \
  --log /tmp/thm-premium-vpn.log --writepid /tmp/thm-premium-vpn.pid
```

Verify routing and reachability:

```bash
ip route | grep 10.128        # expect 10.128.0.0/12 via tun0
ping -c 2 <TARGET_IP>
```

Deploy the machine from the room page and note the assigned IP.

---

## Stage 1 — Obtain the PCAP

The capture is ~9 MB and contains tens of thousands of HTTP packets. Analysing it inside the browser VM is painful — host it locally and download to your machine.

### On the room VM (terminal)

```bash
cd ~/Desktop
python3 -m http.server 8080
```

Leave this running. You should see `challenge.pcapng` in the directory listing.

### On your analysis machine (VPN connected)

```bash
mkdir -p sequel-dump && cd sequel-dump
curl -O http://<TARGET_IP>:8080/challenge.pcapng
ls -lh challenge.pcapng    # ~8–9 MB
```

Or browse to `http://<TARGET_IP>:8080/` and click the file.

> **Troubleshooting:** If port 8080 is closed, the HTTP server is not running on the VM yet. Port 80 on these boxes is typically WebSockify (noVNC), not a file server.

---

## Stage 2 — Initial PCAP triage

Quick HTTP sample:

```bash
tshark -r challenge.pcapng -Y "http" -c 20
```

Filter sqlmap traffic:

```bash
tshark -r challenge.pcapng -Y 'http.user_agent contains "sqlmap"' -c 5
```

### Indicators of compromise

| Field | Value |
|-------|-------|
| User-Agent | `sqlmap/1.8.x#stable (https://sqlmap.org)` |
| Method / path | `GET /search_app/search.php?query=...` |
| Injection | Boolean blind via `ORD(MID(...))>N` |
| Database | `profile_db` |
| Table | `profiles` |
| Dumped columns | `id`, `name`, `description` |

### Example decoded request

```
GET /search_app/search.php?query=1 AND ORD(MID((
  SELECT IFNULL(CAST(`description` AS NCHAR),0x20)
  FROM profile_db.`profiles`
  ORDER BY id LIMIT 0,1
),54,1))>99 HTTP/1.1
```

Meaning: *"Is ASCII character at position 54 of row 0's `description` greater than 99?"*

### Payload anatomy

| Fragment | Role |
|----------|------|
| `LIMIT X,1` | Row offset (`X` = 0…6 for seven profiles) |
| `MID(..., POS, 1)` | Character position within the field |
| `>N` | Binary-search threshold (ASCII decimal) |
| `CAST(\`column\` AS NCHAR)` | Which column is being exfiltrated |

---

## Stage 3 — True vs false responses

The app never prints SQL output. sqlmap infers truth from **two distinct HTTP response bodies**:

| Condition | Body | `Content-Length` (this capture) |
|-----------|------|----------------------------------|
| **True** | Normal search hit (cryptography-themed blurb) | `156` |
| **False** | `No results found` | `41` |

> Other writeups report 425 vs 262 bytes — framing differs by capture. **Always calibrate on your PCAP**: follow one request/response pair in Wireshark and note which body means true.

### Binary search walkthrough (one character)

For position 41, row 6, `description` field, sqlmap sends:

```
...)>64   → 156 bytes (true)   → char > 64
...)>96   → 156 bytes (true)   → char > 96
...)>112  → 156 bytes (true)   → char > 112
...)>120  → 156 bytes (true)   → char > 120
...)>124  → 156 bytes (true)   → char > 124
...)>126  → 41 bytes  (false)  → char ≤ 126
...)>125  → 41 bytes  (false)  → char ≤ 125
```

Result: ASCII **125** → `}`. Repeat for every position until the string ends.

---

## Stage 4 — Manual recovery (Wireshark)

Educational steps for a single character:

1. Filter: `http.request.uri contains "ORD(MID"` and `http.request.uri contains "LIMIT 6,1"`.
2. Pick one `MID` position (e.g. `,41,1)`).
3. For each `>N` threshold, check the paired response body/length.
4. Derive `chr(last_true_threshold)` (or `last_true + 1` depending on convention).
5. Concatenate positions in order.

At full PCAP scale (~tens of thousands of requests), manual recovery is impractical — automate.

---

## Stage 5 — Automated reconstruction

### Option A — Python + Scapy (recommended)

Script logic:
1. Pair each `HTTPRequest` with its `HTTPResponse` on the same TCP stream.
2. Parse `query=` for `MID` position, `LIMIT` offset, column name, and `>threshold`.
3. Mark success when response body does **not** contain `No results found`.
4. Per (row, column, position), track `[min, max]` ASCII range from binary search.
5. Emit `chr(max_threshold)` and print the reconstructed table.

```bash
cd sequel-dump
# place challenge.pcapng here; script: blind_sqli_table_reconstruction.py
python3 blind_sqli_table_reconstruction.py
```

Reference implementation: [CarsonShaffer/THM-Sequel-Dump](https://github.com/CarsonShaffer/THM-Sequel-Dump).

### Option B — tshark export + parser

```bash
tshark -r challenge.pcapng \
  -Y 'http.request.uri contains "ORD%28MID%28%28SELECT" and http.response.code == 200' \
  -T fields -e frame.number -e http.content_length -e http.request.uri \
  -E separator=, -E quote=d \
  | python3 -c 'import sys, urllib.parse; [print(urllib.parse.unquote(line), end="") for line in sys.stdin]' \
  > queries.csv
```

Build a parser that reads `queries.csv`, groups by row/column/position, and applies the same binary-search merge logic.

---

## Stage 6 — Reconstructed data

The script rebuilds **seven rows** from `profile_db.profiles`:

| Row | id | name | description (summary) |
|-----|----|------|-------------------------|
| 0 | 1 | Void | Cryptography expert, encryptions and fortress vulnerabilities |
| 1 | 2 | Zero-Day | Exploit hunter, penetration testing and reverse engineering |
| 2 | 3 | Phantom | OSINT specialist, dark web tracking |
| 3 | 4 | Root | Network infiltrator, firewall bypass |
| 4 | 5 | Specter | Forensic analyst, digital crime scenes |
| 5 | 6 | Sentinel | AI security specialist, countermeasures against evolving threats |
| 6 | 7 | supersecrethiddenuser | Contains the incident flag (`Here's the flag: THM{...}`) |

The room answer is the `THM{...}` token from **row 6's `description` field**.

---

## Attack chain summary

```
Premium VPN → deploy VM
  └─> python3 -m http.server 8080 on ~/Desktop
        └─> download challenge.pcapng (~9 MB)
              └─> filter sqlmap GET /search_app/search.php
                    └─> map 156-byte (true) vs 41-byte (false) responses
                          └─> binary-search ORD(MID()) per char per row
                                └─> rebuild profile_db.profiles (7 rows)
                                      └─> flag in row 6 description
```

---

## Key takeaways

1. **Blind SQLi is loud on the wire** — sqlmap generates thousands of near-identical probes with a distinctive User-Agent and `ORD(MID())` pattern.
2. **A two-state oracle is enough** — differing response bodies/lengths leak entire columns without ever showing query results.
3. **Binary search compresses the leak** — ~7 requests per character instead of up to 95 linear probes.
4. **Forensics needs automation** — Wireshark teaches the mechanics; Scapy/tshark scripts finish the job in seconds.
5. **VPN profile matters** — Premium lab IPs (`10.129.x`) are unreachable on the KoTH VPN.

---

## References

- [OWASP — Blind SQL Injection](https://owasp.org/www-community/attacks/Blind_SQL_Injection)
- [sqlmap project](https://sqlmap.org/)
- [Carson Shaffer — Hackfinity Battle Forensics Writeup](https://blog.carsonshaffer.me/tryhackme-hackfinity-battle-forensics-writeup-6584c41c244f)
- [THM-Sequel-Dump reconstruction script](https://github.com/CarsonShaffer/THM-Sequel-Dump)
