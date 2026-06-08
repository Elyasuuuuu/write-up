# Elyasuuuuu's Portfolio — Official Write-up

**Room:** A Great Portfolio (`agreatportfolio`)  
**Author:** Elyasuuuuu  
**Difficulty:** Easy  
**Type:** Web / Recon CTF (no VM)  
**Target:** [https://elyasuuuuu.fr/](https://elyasuuuuu.fr/)

---

## Introduction

This room simulates the opening phase of a web penetration test against a live link-in-bio portfolio. There is no AttackBox or target machine to deploy — everything runs in your browser against a real website.

Your goal is to recover **11 flags** in the format `ELY{...}` using standard recon techniques: viewing source code, checking common files, inspecting HTTP responses, using developer tools, and interacting with hidden features on the page.

Brute-forcing directories or hammering the site with automated fuzzing is unnecessary. Think like an analyst doing careful, manual enumeration first.

---

## Prerequisites

- A modern web browser (Firefox or Chrome recommended)
- Optional: `curl`, Burp Suite, or similar tools for HTTP inspection
- Basic familiarity with HTML, HTTP, and Linux shell commands

---

## Recon Methodology

A sensible order for this room:

1. **Passive inspection** — view source, read `robots.txt`, follow disallowed paths
2. **HTTP analysis** — check response headers, not just bodies
3. **Client-side review** — open DevTools and read what the frontend logs
4. **Interactive discovery** — find the hidden terminal and explore its filesystem
5. **UI tricks** — selection, clicks, and easter eggs in the page layout

Each TryHackMe task maps to one flag. Work through them in order if you are new to web recon.

---

## Walkthrough

### Task 1 — Briefing

Read the room description and confirm the target URL. This challenge is entirely browser-based.

**Target URL:** `https://elyasuuuuu.fr/`

---

### Task 2 — View Source

Every web assessment should start with understanding what the server delivers. Browsers only render part of the document — metadata and comments in the HTML often survive unnoticed.

**Steps:**

1. Navigate to the homepage.
2. Open **View Page Source** (right-click → *View Page Source*, or `Ctrl+U` / `Cmd+Option+U`).
3. Search the raw HTML for the flag format (`ELY{`).

Pay attention to `<meta>` tags and other non-visible elements. Developers occasionally embed version strings, build info, or jokes in markup that never appears on screen.

---

### Task 3 — Robots.txt

The `robots.txt` file tells search engine crawlers which paths to avoid. For pentesters, it is a free hint list.

**Steps:**

1. Request `/robots.txt` directly:
   ```bash
   curl -s https://elyasuuuuu.fr/robots.txt
   ```
2. Read the `Disallow` entries and any comments at the bottom of the file.

Note every path marked as off-limits — you will revisit several of them in later tasks.

---

### Task 4 — The Vault

`robots.txt` listed `/vault/` as disallowed. Disallowed does not mean protected; it only asks polite crawlers to stay away.

**Steps:**

1. Browse to `/vault/` or request the flag file inside it:
   ```bash
   curl -s https://elyasuuuuu.fr/vault/flag.txt
   ```

This is a classic content-discovery finding: a sensitive-looking directory referenced in `robots.txt` that remains publicly reachable.

---

### Task 5 — Header Hunter

API endpoints do not always return secrets in the JSON body. Response **headers** can carry metadata, tokens, or — in CTF rooms — flags.

**Steps:**

1. Request the status endpoint:
   ```bash
   curl -s https://elyasuuuuu.fr/api/v1/status.json
   ```
2. The body may include a hint but not the answer. Inspect the full HTTP response:
   ```bash
   curl -sI https://elyasuuuuu.fr/api/v1/status.json
   ```
3. Review every custom header in the response.

In Burp Suite: send the request to Repeater and read the **Response** headers panel. In browser DevTools: **Network** tab → select the request → **Headers**.

---

### Task 6 — Admin Panel

Another path from `robots.txt` was `/admin/`. Fake admin panels are a staple of web CTFs.

**Steps:**

1. Visit `https://elyasuuuuu.fr/admin/` in your browser.
2. Read the page content carefully.

No credentials are required — the flag is displayed on the page itself.

---

### Task 7 — Console Detective

Front-end JavaScript sometimes logs debug messages, anti-tamper warnings, or developer notes to the browser console. Users rarely open it; pentesters always should.

**Steps:**

1. Open the homepage.
2. Press **F12** (or `Ctrl+Shift+I` / `Cmd+Option+I`) to open Developer Tools.
3. Switch to the **Console** tab.
4. Look for styled log output or messages that reference the flag format.

Refresh the page if the console was opened after initial load.

---

### Task 8 — Shell Access

The portfolio hides an interactive terminal. It is not a real shell on the server — it is a simulated environment in the browser — but it behaves enough like Linux to be useful practice.

**Steps:**

1. Find the trigger on the page. The site owner's **display name** at the top of the profile is interactive — interact with it repeatedly in quick succession.
2. Alternatively, research classic keyboard easter eggs (covered in Task 10).
3. Once the terminal opens, explore available commands with `help`.
4. Read `flag.txt` with the appropriate shell command.

The terminal accepts common recon commands (`ls`, `cat`, `curl`, etc.) and responds with simulated output.

---

### Task 9 — Hidden in Plain Sight

A default `ls` listing hides dotfiles. On Unix systems, files and directories whose names begin with `.` are not shown unless you ask for them.

**Steps:**

1. Inside the terminal, list files including hidden entries:
   ```text
   ls -a
   ```
2. Identify suspicious dotfiles.
3. Read the hidden file with `cat`.

This mirrors a real post-exploitation habit: always check for `.bash_history`, `.ssh/`, `.env`, and other concealed artifacts.

---

### Task 10 — Konami Code

Retro games hid cheats behind a famous button sequence. This site respects that tradition.

**Steps:**

1. On the homepage (outside the terminal), enter the classic Konami sequence with arrow keys, followed by **B** and **A**.
   - On **QWERTY** keyboards: … **B**, **A**
   - On **AZERTY** keyboards: the final key is often **Q** instead of **A**
2. Alternatively, open the terminal (Task 8) and type the easter-egg command shown in `help`, then follow the on-screen instruction.
3. After unlocking, run the special `loot` command mentioned in the terminal's help output.

If `loot` says access denied, the unlock step was not completed successfully.

---

### Task 11 — Selection Master

Not all secrets are invisible to the DOM — some are merely **invisible to the eye**. CSS can set text colour to match the background, shrink opacity to near zero, or hide content off-screen while keeping it selectable.

**Steps:**

1. Scroll to the **footer** of the homepage.
2. Click and drag to select text in the copyright area, including below the visible line.
3. Copy the selection or read the highlighted characters.

You can also inspect the footer in DevTools → **Elements** and search for hidden spans or zero-opacity nodes.

---

### Task 12 — Click Forensics

The final flag rewards persistence. Some UI easter eggs trigger only after a specific number of interactions within a short time window.

**Steps:**

1. Return to the footer area.
2. Click repeatedly on the copyright block — quickly, without long pauses between clicks.
3. After enough consecutive clicks, the page should reveal the last flag visually.

If nothing happens, reset and try again with faster, uninterrupted clicks. The counter typically resets if you wait too long between taps.

---

## Summary

| Task | Technique | Key takeaway |
|------|-----------|--------------|
| 2 | View Source | HTML metadata is part of the attack surface |
| 3 | robots.txt | Crawler rules leak paths worth investigating |
| 4 | Vault | Disallowed ≠ protected |
| 5 | Headers | Always inspect full HTTP responses |
| 6 | Admin panel | Enumerate paths discovered during recon |
| 7 | Console | Front-end logs are underrated intel |
| 8 | Hidden terminal | UI elements can gate interactive features |
| 9 | Dotfiles | `ls -a` before assuming a directory is empty |
| 10 | Konami / loot | Client-side easter eggs may unlock new commands |
| 11 | Text selection | CSS hiding ≠ DOM removal |
| 12 | Click counter | Rate-limited UI triggers need patience |

---

## What You Learned

- **Web recon is layered.** Source code, robots files, and HTTP headers each reveal different slices of the same application.
- **The browser is a pentest tool.** DevTools console, element inspector, and network tab are as important as terminal utilities.
- **Assume less than you see.** Hidden files, invisible text, and locked commands are all common CTF (and real-world) patterns.
- **Follow the breadcrumbs.** Later tasks build on paths and techniques introduced earlier — the same flow applies on real engagements.

---

## Responsible Testing

This room runs against a **live production site**. Stick to the techniques described above. Do not run aggressive directory brute-forcing, denial-of-service attempts, or automated scanners at high rate against the target.

---

*Room by [Elyasuuuuu](https://elyasuuuuu.fr/) · Write-ups: [github.com/Elyasuuuuu/write-up](https://github.com/Elyasuuuuu/write-up)*
