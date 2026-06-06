# TryHackMe - The Return of the Yeti

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Hard  
**Category:** Forensics, WiFi, RDP  
**Room:** [The Return of the Yeti](https://tryhackme.com/room/adv3nt0fdbopsjcap)  
**Type:** PCAP analysis (no deployable machine)

## Introduction

Advent of Cyber '23 — Side Quest 1. Van Spy captured WiFi traffic from an AntarctiCrafts intern's network. You receive `VanSpy.pcapng` and must reconstruct the attacker's actions.

**Objectives:**
- Identify the WiFi network name and password
- Name the suspicious tool used by the attacker
- Recover the CyberPolice case number and `yetikey1.txt` content from the RDP session

---

## Reconnaissance

Download `VanSpy.pcapng.zip` from the room and extract it.

**Tools:**
- **Wireshark** — PCAP analysis
- **aircrack-ng** or **hashcat** — WPA cracking
- **tshark** — format conversion
- **PyRDP** — RDP session replay

```bash
# Convert if aircrack-ng rejects .pcapng
tshark -F pcap -r VanSpy.pcapng -w VanSpy.pcap
```

Open the capture and inspect **802.11** traffic (beacon frames, EAPOL handshake).

---

## Enumeration

### Question 1 — WiFi network name

Beacon frames advertise the SSID in cleartext.

Wireshark filter:

```
wlan.ssid
```

Or use **Wireless → WLAN Traffic** to list networks.

Submit the SSID on TryHackMe.

---

### Question 2 — WiFi password

The network uses **WPA2-PSK (AES)**. The PCAP contains a complete **4-way handshake**, enabling offline dictionary attack.

**Method A — aircrack-ng**

```bash
tshark -F pcap -r VanSpy.pcapng -w VanSpy.pcap

# Note BSSID from Wireless → WLAN Traffic
aircrack-ng -w /usr/share/wordlists/rockyou.txt -b <BSSID> VanSpy.pcap
```

**Method B — hashcat**

Extract handshake with [cap2hashcat](https://hashcat.net/cap2hashcat/):

```bash
hashcat -m 22000 handshake.hc22000 /usr/share/wordlists/rockyou.txt
```

**Decrypt WiFi traffic in Wireshark:**

```
Edit → Preferences → Protocols → IEEE 802.11 → Decryption Keys → Edit
```

- Type: `wpa-pwd`
- Key: `<password>:<SSID>`

802.11 frames become readable TCP/TLS/RDP traffic.

---

### Question 3 — Attacker's suspicious tool

Room hint: *"Van Spy had planted a backdoor"*. Classic backdoor port → **4444**.

Wireshark filter:

```
tcp.port == 4444
```

**Follow TCP Stream** reveals a PowerShell session on `INTERN-PC`:

1. Backdoor `psh4444.exe` on Desktop
2. Download of a credential-theft tool from GitHub
3. Certificate export:

```powershell
cmd /c mimikatz.exe privilege::debug token::elevate crypto::capi "crypto::certificates /systemstore:LOCAL_MACHINE /store:\`"Remote Desktop\`" /export" exit
```

4. Base64 exfiltration of the `.pfx` file

Here **mimikatz** exports the **Remote Desktop** certificate from LOCAL_MACHINE — not for NTLM dumping, but to **decrypt the TLS-wrapped RDP session** in the PCAP.

---

## Exploitation

### Questions 4 & 5 — RDP session analysis

After WiFi decryption, an RDP session to `10.1.1.1` appears, encrypted with TLS.

#### Step 1 — Recover the RDP certificate

From the TCP stream on port 4444, copy the Base64 `.pfx` string and decode:

```powershell
[IO.File]::WriteAllBytes("cert.pfx", [Convert]::FromBase64String("MIIJuQ..."))
```

Try the tool name as the PFX password.

#### Step 2 — Configure Wireshark TLS decryption

```
Edit → Preferences → Protocols → TLS → RSA keys list
```

- IP: `10.1.1.1`, port: `3389`, protocol: `rdp`
- Key file: `cert.pfx`, password: `<from analysis>`

#### Step 3 — Export RDP PDUs

```
File → Export PDUs to File → OSI Layer 7
```

Save as `rdp-export.pcap`.

#### Step 4 — Replay with PyRDP

```bash
pip install pyrdp
pyrdp-convert -f replay rdp-export.pcap
pyrdp-player exported.pyrdp
```

Watch the session: Elf McSkidy connects, files a CyberPolice report, and copies `yetikey1.txt` to the clipboard.

- **Q4:** Case number visible in the complaint form on screen
- **Q5:** `yetikey1.txt` content shown in PyRDP clipboard panel during `Get-Content`

---

## Flags

| Question | How to obtain |
|----------|---------------|
| WiFi SSID | 802.11 beacon frames |
| WiFi password | WPA handshake + rockyou |
| Suspicious tool | TCP stream on port 4444 |
| Case number | PyRDP replay of RDP session |
| yetikey1.txt | PyRDP clipboard during file read |

*Answers intentionally omitted. Complete the room on [TryHackMe](https://tryhackme.com/room/adv3nt0fdbopsjcap).*

---

## Conclusion

### What I learned

| Step | Skill |
|------|-------|
| 802.11 beacons | Read SSIDs from WiFi captures |
| WPA handshake | Offline PSK cracking with wordlists |
| wpa-pwd (Wireshark) | Decrypt 802.11 payloads |
| TCP stream follow | Reconstruct backdoor activity |
| mimikatz crypto::certificates | Export RDP certs for TLS decryption |
| Wireshark TLS keys | Decrypt encrypted RDP |
| PyRDP | Replay RDP sessions as video |

A dense forensics side quest covering WiFi, credential theft tooling, and encrypted remote desktop analysis in one PCAP.
