# TryHackMe - Searchlight (IMINT)

> *These notes were written with AI assistance for English clarity. Attack paths and commands reflect my own work on the room.*

**Difficulty:** Easy  
**Category:** OSINT, IMINT / GEOINT  
**Room:** [Searchlight - IMINT](https://tryhackme.com/room/searchlightosint)  
**Type:** Image and video geolocation (no deployable machine)

## Introduction

> *Time to enter the warren of visual intelligence...*

This room teaches **IMINT** (Imagery Intelligence) and **GEOINT** (Geospatial Intelligence) — locating places and answering context questions using only what you can see in photos and videos.

**Flag format:** every answer is submitted as `sl{your answer}` (lowercase is fine).

**IMINT checklist** (Benjamin Strick / Geoint principles):

| Element | What to look for |
|---------|------------------|
| Context | Source, title, description, who shared it |
| Foreground | Signs, text, landmarks, vehicles |
| Background | Architecture, terrain, skyline |
| Map markings | Street layout, intersections, satellite/street view |
| Trial and error | Refine searches when the first guess fails |

**Core skills covered:**
1. Visual extraction before touching tools
2. Google dorking for context questions
3. Reverse image search (Google, Yandex, TinEye)
4. FFmpeg frame extraction from video

---

## Task 1 — Welcome to Searchlight IMINT

Read the room introduction. All answers use the format `sl{...}`.

| Question | Notes |
|----------|-------|
| Did you understand the flag format? | Confirm readiness with the room's example answer |

---

## Task 2 — Your first challenge

**Method:** pure visual analysis — no reverse search required.

Download the attached image. A curved street banner and shop signs are clearly readable. Ask:

- Are there **obvious text clues** (street names, store names)?
- What region does architecture/language suggest?

Search the visible street name on Google to confirm location and cross-reference with Street View.

| Question | Approach |
|----------|----------|
| Street name | Read signage in the image |

---

## Task 3 — Just Google it

**Method:** Google dorking + Wikipedia for context questions.

Image shows a **London Underground** entrance. Partial station name visible on the sign (`..LY CIRCUS ST..`). Google the fragment → identify the station → use Wikipedia for history and infrastructure details.

| Question | Approach |
|----------|----------|
| City | Recognize London Underground branding |
| Tube station | Partial sign text + Google |
| Year opened | Wikipedia — Bakerloo line opening date |
| Number of platforms | Wikipedia infobox |

Practice: [Google Dorking room](https://tryhackme.com/room/googledorking) on TryHackMe.

---

## Task 4 — Keep at it

**Method:** scan foreground signs → Google unique text.

Canteen photo with **"YVR CONNECTS"** signage → Vancouver International Airport. Airports are often in a **different municipality** than the city they serve.

| Question | Approach |
|----------|----------|
| Building name | Sign text in background |
| Country | YVR = Canada |
| City | Airport location (not Vancouver city proper) — check official airport address |

---

## Task 5 — Coffee and a light lunch

**Method:** cross-reference two photos + Google Maps manual verification.

Clue: coffee shop in **Scotland**. Photo 1 shows **The Edinburgh Woollen Mill** across the street — but EWM is a chain; do **not** assume Edinburgh.

Workflow:
1. EWM store finder → filter Scotland locations
2. Match intersection layout, corner building, traffic signs on Maps/Street View
3. Identify coffee shop opposite the woollen mill
4. Business details from Facebook / TripAdvisor / website

| Question | Approach |
|----------|----------|
| City | Maps matching of both photos |
| Street | Street View / Maps label |
| Phone / email | Business social media or listing |
| Owners' surname | TripAdvisor reviews, local news |

---

## Task 6 — Reverse your thinking

**Method:** reverse image search.

Read [Bellingcat reverse image guide](https://www.bellingcat.com/resources/how-tos/2019/09/26/guide-to-using-reverse-image-search-tools/) and OSINT Curious write-ups first.

**Yandex** often outperforms Google for this room's restaurant interior. Distinctive features: hanging lamps, circular cutouts, wall of framed photos → **Katz's Delicatessen** (NYC).

Second question: Google `bon appétit katz's deli 24 hours` → Bon Appétit video.

| Question | Approach |
|----------|----------|
| Restaurant | Yandex reverse image search |
| Bon Appétit editor | YouTube / article about 24-hour shift |

**Browser extension:** RevEye (Chrome/Firefox) for one-click multi-engine reverse search.

---

## Task 7 — Locate this sculpture

**Method:** visual clues + Google dorking + reverse image search.

Foreground: chrome **motorcycle-deer** sculpture. Background: modern waterfront apartments.

Search terms: `motorcycle deer statue`, `chrome reindeer sculpture oslo` → **Tjuvholmen Sculpture Park**, Oslo.

Photographer: reverse search or Visit Oslo / tourism pages with image credits.

| Question | Approach |
|----------|----------|
| Statue name | Google + tourism sites |
| Photographer | Image credits on official tourism page |

---

## Task 8 — …and justice for all

**Method:** reverse image search + news articles + Google Maps Street View.

Blindfolded **Lady Justice / Themis** statue reflected in courthouse windows. Yandex reverse search → news stock photos referencing **Albert V. Bryan U.S. Courthouse**.

| Question | Approach |
|----------|----------|
| Character depicted | Statue symbolism (Themis = Lady Justice) |
| Location | Courthouse name → city, state |
| Building opposite | Google Maps Street View — face courthouse, identify hotel across street |

Recommended viewing: Amy Herman TED talk *"A lesson on looking"* for visual intelligence mindset.

---

## Task 9 — The view from my hotel room

**Method:** FFmpeg frame extraction + reverse image search + landmark correlation.

Download the video. Extract stills with FFmpeg ([Nixintel guide](https://nixintel.info/osint-tools/using-ffmpeg-to-grab-stills-and-audio-for-osint/)):

```bash
ffmpeg -i task9_video.mp4 -r 1 img%06d.png -hide_banner
```

`-r 1` = one frame per second (avoids thousands of duplicates).

Key landmarks visible from the balcony:
- **Clarke Quay** signage
- **Riverside Point** mall
- Singapore Flyer (ferris wheel)
- Marina Bay Sands (three-tower building)
- Colourful warehouse-style shophouses

Correlate on Google Maps → Singapore river area → identify hotel from reverse search of balcony view (review photos, Alamy stock, etc.).

| Question | Approach |
|----------|----------|
| Hotel name | Landmark triangulation + reverse image of balcony view |

---

## Answers Reference

| Task | Questions | Method summary |
|------|-----------|----------------|
| 1 | Flag format | Read room intro |
| 2 | Street name | Visual — read banner |
| 3 | Tube station context | Google + Wikipedia |
| 4 | Airport context | Sign text + geography |
| 5 | Scottish coffee shop | Maps cross-reference |
| 6 | NYC deli | Reverse image search |
| 7 | Oslo sculpture | Google + tourism credits |
| 8 | Courthouse statue | Reverse search + Street View |
| 9 | Singapore hotel | FFmpeg + landmark ID |

*Specific `sl{...}` values intentionally omitted. Complete the room on [TryHackMe](https://tryhackme.com/room/searchlightosint).*

---

## Conclusion

### What I learned

| Skill | Takeaway |
|-------|----------|
| Visual first | Read every sign before opening tools |
| Google dorking | Fragment text (`..LY CIRCUS..`) is enough to start |
| Reverse image search | Yandex often beats Google for interiors and art |
| Chain stores trap | Edinburgh Woollen Mill ≠ Edinburgh city |
| Airport geography | YVR is in Richmond, not Vancouver |
| Video IMINT | FFmpeg `-r 1` extracts searchable key frames |
| Context questions | Location first, then Wikipedia/Maps for metadata |

Searchlight builds IMINT fundamentals progressively — from obvious signage to multi-landmark video geolocation.
