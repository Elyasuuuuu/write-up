# TryHackMe - Cache Me Outside

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Easy  
**Category:** OSINT  
**Room:** [Cache Me Outside](https://tryhackme.com/room/cachemeoutside)  
**Description:** Can you find this ex hacker turned outdoorsman?

## Introduction

*"Can you find this ex hacker turned outdoorsman?"* A beginner-friendly OSINT room built around a fictional retired hacker who left digital breadcrumbs across fitness apps, GitHub, email, and social media. You start with a single conversation screenshot and reconstruct his identity step by step.

**Objectives:**
- Identify the target's full name
- Recover an accidentally exposed email address
- Obtain a phone number through email interaction
- Geolocate the city from a social media photo
- Pinpoint a specific tram station from post context

---

## Starting Point — The Screenshot

The room provides a leaked conversation screenshot. Buried in the image is a link to a **Komoot** profile — a fitness/outdoor activity platform.

Open the profile URL and note:
- The target's **full name**
- His bio (ex-hacker, outdoor enthusiast)
- A link to his **GitHub** account (`jiml33t`)

---

## Step 1 — Full Name (Komoot)

The Komoot profile displays the retired hacker's real name publicly. Submit it as **Question 1**.

---

## Step 2 — Exposed Email (Git Commit Metadata)

Follow the GitHub link from Komoot:

```
https://github.com/jiml33t/jiml33t
```

The profile page does not show an email, but Git stores author metadata inside commits.

### Method A — Clone and inspect logs

```bash
git clone https://github.com/jiml33t/jiml33t.git
cd jiml33t
git log
```

### Method B — View the commit patch directly

```
https://github.com/jiml33t/jiml33t/commit/<COMMIT_HASH>.patch
```

The patch header contains the author email in the `From:` field — accidentally committed and never scrubbed.

Submit the email as **Question 2**.

---

## Step 3 — Phone Number (Active OSINT)

### Dead end — passive OSINT tools

Standard username searches (Sherlock, Holehe, etc.) on `jiml33t` do not surface a phone number. This step requires **active** OSINT.

### Breakthrough — email auto-reply

Send an email to the address recovered in Step 2. The target has an automatic out-of-office / auto-reply configured that includes personal contact details.

The auto-reply body contains a Romanian phone number. Submit it as **Question 3** (include country code).

> **Note:** Some writeups mention a Threads hint suggesting email interaction — the auto-reply is the intended path.

---

## Step 4 — City (Social Media Geolocation)

Search the GitHub username `jiml33t` across social platforms. You should find linked accounts on **Instagram** and **Threads**.

### Image analysis

A recent post contains a photo with a visible business sign: **irigatii.ro** (an irrigation company).

### Geolocation workflow

1. Reverse image search (Google Lens) on the post photo
2. Search `irigatii.ro` on Google Maps — the company has a physical location
3. Use **Street View** to confirm the environment matches the photo
4. Cross-reference with post captions mentioning outdoor activity near the area

The city is in **Romania**. Submit the city name as **Question 4**.

---

## Step 5 — Tram Station (Correlation OSINT)

A Threads/Komoot post from **7 May 2026** mentions:
- Taking a tram
- Shopping at a French supermarket (Auchan)
- Being near the geolocated area from Step 4

### Investigation process

1. Open Google Maps around the `irigatii.ro` location in the identified city
2. Find nearby **tram lines** and **terminus stations**
3. Cross-reference with the Auchan supermarket proximity
4. Read the post text carefully for station name hints

A targeted search helps narrow it down:

```
tram station near IRIGATII.RO billboard Calea Buziașului Timișoara Romania
```

Submit the exact tram station name as **Question 5**.

---

## Investigation Chain

```
Screenshot → Komoot profile → full name + GitHub
    ↓
Git commit metadata → exposed email
    ↓
Email auto-reply → phone number
    ↓
Social media (Threads/IG) → photo with irigatii.ro sign
    ↓
Google Lens + Maps → city (Timișoara)
    ↓
Post context + Maps → tram station (7 May 2026)
```

---

## Answers

| Question | How to obtain |
|----------|---------------|
| Q1 — Full name | Komoot profile |
| Q2 — Email address | Git commit patch / `git log` on `jiml33t/jiml33t` |
| Q3 — Phone number | Auto-reply after emailing the exposed address |
| Q4 — City | Geolocate `irigatii.ro` sign via social media photo |
| Q5 — Tram station | Correlate May 7th post with Maps near the geolocated area |

*Answers intentionally omitted. Complete the room on [TryHackMe](https://tryhackme.com/room/cachemeoutside).*

---

## Conclusion

### What I learned

| Concept | Lesson |
|---------|--------|
| Git metadata | Commit patches leak emails even when profiles hide them |
| Active vs passive OSINT | Some data only surfaces through direct interaction (email) |
| Visual geolocation | Business signage in photos is a strong geolocation anchor |
| Cross-platform pivoting | One username (`jiml33t`) links Komoot → GitHub → Threads → IG |
| Correlation OSINT | Combine post text, maps, and infrastructure (tram/Auchan) for precision |

Always document your sources and stay within ethical OSINT boundaries — only use publicly available information.

---

## References

- [Sunjid Ahmed Siyem — Walkthrough](https://medium.com/@sunjid-ahmed/cache-me-outside-tryhackme-walkthrough-da97894dd200)
- [Sudoroot — Walkthrough](https://medium.com/@sudoroot523/tryhackme-cache-me-outside-ea9879a044e0)
