# TryHackMe - Event Horizon

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Hard  
**Category:** Digital Forensics, Malware Analysis, Network Traffic Analysis  
**Room:** [Event Horizon](https://tryhackme.com/room/eventhorizonroom)  
**Description:** Unearth the secrets beyond the Event Horizon.

## Introduction

*"Unearth the secrets beyond the Event Horizon."* A DFIR challenge providing a PCAP and a PowerShell memory dump. Trace a phishing attack from SMTP delivery through Covenant C2 implant deployment, decrypt encrypted C2 traffic, and recover credentials and a hidden flag from a screenshot exfiltrated by the attacker.

**Files provided:**
- `traffic.pcapng` — full network capture
- `powershell.DMP` — minidump of the infected PowerShell process

**MD5 of zip:** `a1eda8f91365c322ebc8ce9b178248bc`

**Objectives:**
- Recover email credentials and phishing content from SMTP/POP traffic
- Identify the malicious download command and reconstruct the Covenant stager
- Extract the initial AES setup key from the .NET binary
- Decrypt C2 traffic and recover the Administrator NTLM hash
- Decode exfiltrated image data for the final flag

---

## Phase 1 — Email Credential Recovery

### SMTP brute-force and successful login

Filter Wireshark for `smtp` or `pop`. Around the successful SMTP authentication (packet ~4665), follow the TCP stream.

Credentials are **Base64-encoded** in the AUTH LOGIN exchange:

```bash
echo "dG9tLmRvbUBldmVudGhvcml6b24udGht" | base64 -d
echo "cGFzc3dvcmQ=" | base64 -d
```

Alternatively, filter POP3 for successful logins:

```
pop && frame contains "+OK Mailbox"
```

Follow the matching TCP stream for the email:password pair.

---

## Phase 2 — Phishing Email Analysis

In the same SMTP TCP stream, extract the multipart MIME message:

- **Plain-text body** — social engineering text urging the victim to run an attached script as Administrator
- **Attachment** — `eventhorizon.ps1` (Base64-encoded PowerShell)

Decode the attachment:

```bash
echo "<BASE64_FROM_ATTACHMENT>" | base64 -d > eventhorizon.ps1
```

The script is a distraction (black hole mass calculations). The **last line** contains the download command for the second-stage payload `radius.ps1`.

---

## Phase 3 — Malware Reconstruction

### Locate radius.ps1 in PCAP

Find the HTTP GET for `radius.ps1` (packet ~4722):

```
http://10.0.2.45/radius.ps1
```

Follow the TCP stream or export the HTTP object.

### Decompress the .NET payload

`radius.ps1` contains a Base64 blob compressed with `DeflateStream`. Extract and decompress:

```python
import base64, zlib

blob = "<BASE64_FROM_RADIUS_PS1>"
compressed = base64.b64decode(blob)
decompressed = zlib.decompress(compressed, -15)

with open("payload.bin", "wb") as f:
    f.write(decompressed)
```

Verify with `file payload.bin` — expect `PE32` / `.NET` executable (MZ header).

Submit the MD5 to VirusTotal — detections reference **Covenant C2**.

---

## Phase 4 — Initial AES Setup Key

Decompile `payload.bin` with **ILSpy** (VS Code extension) or **dnSpy**.

Locate the `GruntStager` class → `ExecuteStager` method. The **AESSetupKey** is a Base64-encoded string just before the stager initiates its encrypted connection to the C2 server.

This is the answer to the "initial AES key" question — the key embedded in the stage-0 binary, not the session key derived later.

---

## Phase 5 — Covenant C2 Traffic Decryption

Clone [CovenantDecryptor](https://github.com/naacbin/CovenantDecryptor):

```bash
git clone https://github.com/naacbin/CovenantDecryptor.git
cd CovenantDecryptor && pip install -r requirements.txt
```

### Extract C2 traffic from PCAP

Find the Covenant HTTP stream (GET to `test.html`, POST data starting ~packet 4742). Export or extract Base64/URL-encoded C2 data:

```bash
tshark -r traffic.pcapng -Y "http.request.method == POST" \
  -T fields -e http.file_data > post_data.txt
```

Or follow the HTTP stream in Wireshark and grep for C2 parameters:

```bash
grep -oP 'i=[a-f0-9]+&data=[A-Za-z0-9+/=]+&session=[\w-]+|eyJ[A-Za-z0-9+/=]+' traffic.txt > c2_data.txt
```

### Step 1 — Extract RSA modulus

```bash
python3 decrypt_covenant_traffic.py modulus \
  -i c2_data.txt -k "<AESSetupKey>" -t base64
```

### Step 2 — Recover RSA private key from memory dump

```bash
python3 extract_privatekey.py \
  -i ../powershell.DMP -m "<MODULUS>" -o ./keys/
```

### Step 3 — Recover session AES key

```bash
python3 decrypt_covenant_traffic.py key \
  -i c2_data.txt -k "<AESSetupKey>" -t base64 \
  -r keys/privkey1.pem -s 1
```

If stage-0 response parsing fails, extract only the server response (packet ~4745) to a separate file and retry without `-s 1`.

### Step 4 — Decrypt all C2 messages

```bash
python3 decrypt_covenant_traffic.py decrypt \
  -i c2_data.txt -k "<SESSION_KEY>" -t hex -s 2
```

The decrypted output reveals:
- System enumeration (hostname, domain, OS)
- **mimikatz** execution and privilege escalation to SYSTEM
- **Administrator NTLM hash** in the credential dump
- SAM database contents

---

## Phase 6 — Final Flag (Screenshot Exfiltration)

In the decrypted C2 output, response messages ~15–16 contain a large Base64 blob. Save it to a file, strip non-Base64 characters, and decode:

```bash
base64 -d flag.b64 > flag.bin
file flag.bin
# PNG image data
```

Open the PNG — it is a desktop screenshot containing the final flag text.

---

## Answers Summary

| Question | How to obtain |
|----------|---------------|
| Email credentials | Base64-decode SMTP AUTH LOGIN or POP3 successful login |
| Email body | Plain-text section of the SMTP MIME message |
| Malicious download command | Last line of decoded `eventhorizon.ps1` |
| Initial AES key | `AESSetupKey` in GruntStager via ILSpy/dnSpy |
| Administrator NTLM hash | Decrypted Covenant C2 traffic (mimikatz output) |
| Flag | Decode Base64 image from decrypted C2 response messages |

*Flag values intentionally omitted. Complete the room on [TryHackMe](https://tryhackme.com/room/eventhorizonroom).*

---

## Conclusion

### What I learned

| Concept | Lesson |
|---------|--------|
| SMTP forensics | AUTH LOGIN credentials are Base64, not encrypted |
| PowerShell staging | Attackers hide downloaders at the end of benign-looking scripts |
| Covenant C2 | Three-stage handshake: RSA key exchange → session key → encrypted commands |
| .NET decompilation | Stager binaries embed crypto keys in plaintext |
| Memory forensics | Minidumps retain RSA primes for private key reconstruction |
| Data exfiltration | C2 channels can carry screenshots encoded as Base64 blobs |

---

## References

- [CovenantDecryptor](https://github.com/naacbin/CovenantDecryptor)
- [Covenant GruntHTTPStager source](https://github.com/cobbr/Covenant/blob/master/Covenant/Data/Grunt/GruntHTTP/GruntHTTPStager.cs)
- [0xb0b Writeup](https://0xb0b.gitbook.io/writeups/tryhackme/2025/event-horizon)
- [ILSpy VS Code Extension](https://marketplace.visualstudio.com/items?itemName=icsharpcode.ilspy-vscode)
