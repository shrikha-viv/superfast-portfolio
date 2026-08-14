# Personal Portfolio & Personal OS — Knowledge Transfer Document

**Status:** Draft for review
**Purpose:** Consolidate every architectural and design decision made so far, before build begins.

---

## 1. Vision

Not a portfolio. A personal operating system with a public-facing website attached — part resume, part archive, part living dossier of one specific person.

Design intent: **classic power through restraint.** Institutional confidence (Swiss modernism, structural brutalism, museum/archive aesthetics) — not gamified, not gimmicky, not decorated. Personalization comes from specificity and consistency of voice and structure, not from mascots or skins.

Brand identity: scorpion (zodiac) + skateboard (hobby), used as **functional metaphor that drives decisions**, not literal illustration.

---

## 2. System Architecture

### 2.1 Stack

```
Django + Wagtail          — CMS / admin / content backend
PostgreSQL                — primary database
Redis                     — cache + job queue
Celery or Django-Q        — scheduled API syncs, background refresh jobs
Frontend: TBD             — Wagtail-rendered templates (Option A, simpler/faster)
                            OR Next.js/Astro headless via Wagtail REST API (Option B, more visual control)
```

**Open decision:** Option A vs B. Recommendation was Option A first (Wagtail-rendered) unless custom frontend polish is a hard requirement — revisit once visual direction (Section 5) is locked.

### 2.2 App structure

```
/apps/content       Wagtail page models (authored content)
/apps/integrations  GitHub, LeetCode, Spotify, Medium, workout provider clients
/apps/dashboard      Admin refresh/status/sync views
/apps/public         Templates or API serializers for the public site
```

### 2.3 Two kinds of content — the core rule

```
Authored content:  things you write and control  → Wagtail is the source of truth
Synced content:    things fetched from APIs        → cached snapshot in your DB, never live-called from the frontend
```

Public pages always read from your own DB. Nothing on the live site ever waits on a third-party API call.

---

## 3. Content Models

### Authored (Wagtail-managed)

```python
Experience
Project
ProjectReport
TechnicalNote
PersonalDoc
Achievement
WishlistItem
Grade
NowPage
```

### Synced (cache/snapshot tables, written by background jobs)

```python
GitHubStatsSnapshot
LeetCodeStatsSnapshot
WorkoutStatsSnapshot
SpotifyRecentlyPlayed
MediumArticleLink
```

---

## 4. Feature Mapping

| Section | Source | Source of Truth | Notes |
|---|---|---|---|
| Stats (LeetCode, GitHub, workouts) | APIs | Snapshot in DB | Scheduled sync + manual "Refresh" button in admin |
| Work experience | Manual, **not** LinkedIn-synced | Wagtail | Canonical, hand-maintained |
| Projects | Wagtail + GitHub link | Wagtail | High-level "report" written by you; low-level detail lives on GitHub, linked out |
| Technical blogs | Medium RSS | Cached snapshot | Titles/dates/links only — redirect out to Medium for full article, no full-content mirroring |
| Personal docs | Wagtail | Wagtail | Visibility states: private / public / unlisted / draft / archived |
| Music | Spotify API (`user-read-recently-played`) | Snapshot | Ambient/ticker treatment, not a standalone heavy page |
| Achievements / grades | Wagtail | Wagtail | Structured records, changelog/ledger styling — not badge icons |
| Wishlist | Wagtail | Wagtail | Checked/unchecked progress is one of the few places a literal progress indicator is justified |

**Confirmed feasible via public APIs/feeds:**
- Medium: official RSS feeds for profiles/publications, safe to fetch title/date/link.
- Spotify: official Web API, recently-played scope exists.
- GitHub: solid API for repos/commits/PRs/issues; be cautious — the literal contribution-graph visualization is not a clean public API the way repo/commit data is.
- LeetCode: no polished official public API — isolate behind a provider interface so a break doesn't affect the rest of the site.

---

## 5. Refresh Architecture

```
Scheduled job   → runs every few hours, per integration
Manual job      → "Refresh" button in admin, per integration
Result          → written to the relevant *Snapshot table, with a last-synced timestamp
Frontend        → reads only from DB, never calls third-party APIs directly
```

Example flow:
```
Click "Refresh GitHub" (admin)
 → enqueue sync_github_stats
 → call GitHub API
 → normalize response
 → save GitHubStatsSnapshot
 → show last-synced time in admin
```

---

## 6. Admin Site (Wagtail)

### Structure
```
/admin
 - Work Experience
 - Projects
 - Project Reports
 - Personal Docs
 - Technical Blog Links
 - Achievements
 - Grades
 - Wishlist
 - Stats Sync Dashboard
 - API Credentials
 - Manual Refresh Buttons
```

### Permission tiers
```
Owner            — full access
Content admin    — edit projects/blogs/docs
Stats admin      — refresh API integrations only
Viewer           — read-only admin access
```

### Personal doc visibility states
```
private / public / unlisted / draft / archived
```
This lets private notes-to-self and polished public writing live in the same system without accidental leaks.

---

## 7. Visual Design System

### 7.1 Guiding principle
**Strength through restraint, executed with total confidence.** One typeface family, one accent color, a grid that's never broken, motion that's felt rather than seen. Power reads through precision and instant responsiveness, not through intensity or ornament.

### 7.2 Typography
- One serif or grotesk, used at extreme scale for headlines/hero content and hero stats (e.g., a commit count rendered huge, not as a small label).
- Avoid soft/rounded/"friendly" typefaces entirely (no Poppins/Nunito-type faces) — these read consumer/approachable, the opposite of the intended tone.
- Candidates: Söhne, Neue Haas Grotesk, tightly-set Inter, IBM Plex Mono for data/timestamps, a serif (Canela/Fraunces) for editorial weight in project reports/personal docs.
- Tight letter-spacing on headlines, generous line-height on body copy.

### 7.3 Color
- Near-black on near-white (off-black/off-white, not pure #000/#FFF).
- One accent color only, reserved for live data, links, and interactive states.
- No gradients, no glassmorphism, no neon — explicitly the "gimmick" territory being avoided.
- **Secondary "UV" palette** (see Section 8) as an optional/hidden mode: deep obsidian + a single amber-green accent, surfacing hidden-layer content.

### 7.4 Grid & Layout
- Strict, visible grid — consistent alignment across all sections.
- Generous negative space; a page that's majority whitespace around confident type feels considered, not sparse.
- Asymmetry preferred over centering where it reads as intentional rather than generic.

### 7.5 Motion
- Linear or slightly eased, fast (150–250ms). No bounce, no overshoot, no spring physics.
- Scroll reveals: subtle only (e.g., 8px rise + fade), never theatrical.
- Hover states: mechanical and precise — small exact scale (~1.02x) or hard color shift, not glow/soft transitions.
- Stat updates fade precisely into place rather than sliding with flourish.

### 7.6 Interaction / Performance
- Perceived instant response (sub-100ms) is treated as a core design element, not just an engineering target — the snapshot-DB architecture (Section 5) directly supports this.
- Optional subtle magnetic/cursor-follow effects on nav/cards — extremely subtle only.
- Data visualizations (activity pulse, timeline scrubber) styled like instrumentation: thin lines, precise gridlines, monospace numerals.

### 7.7 Wayfinding & Intuitiveness
- One consistent, always-present orientation element (e.g., persistent thin sidebar/index) — same role as room numbers in a museum.
- Predictable visual grammar (same type scale, spacing rhythm, interaction cues) across all sections even though content itself is personal/unconventional — the *form* teaches visitors how to read the *content*.
- Interactivity signaled through consistent visual cues (small arrow, slight shadow shift) applied identically everywhere — no tooltips or onboarding tours needed.

---

## 8. Brand Identity & Personalization Strategy

### 8.1 Core resolution
Scorpion + skateboard are **not** mascots or illustrated skins. They function as the *reasoning* behind design decisions — a visitor should feel the specificity even without consciously identifying the symbolism.

### 8.2 Where the symbolism does real work

| Metaphor | Real-world basis | Functional use |
|---|---|---|
| UV glow | Scorpions fluoresce under UV light | Optional/hidden "night mode" palette (obsidian + amber-green) that surfaces hidden-layer/private-leaning content differently from default mode |
| Molting | Scorpions shed exoskeletons to grow | Structural framing for Personal Docs — organized as versions-of-self over time ("molt log" as internal logic), not a flat blog list |
| Stillness → precision strike | Scorpion behavior: mostly still, then exact | Justifies the site's interaction pacing — quiet static pages, one precise fast motion on interaction (ties directly to Section 7.5) |
| Iteration/failure before success | Core structure of skateboarding — most tricks land on repeated attempts | Framing for Project Reports — show the compressed decision/failure sequence ("the line"), not just the polished outcome |
| Grip tape / truck geometry | Physical skate hardware textures/angles | Optional private structural reference — a faint grain texture or grid angle basis, never a visible illustrated skin |

### 8.3 Structural/voice-level personalization
- Site information architecture framed as an archive/dossier ("logs," specimen-style index numbers, monospace timestamps) rather than "Home / Projects / Blog."
- One consistent voice for all microcopy — dry, exact, slightly wry, never enthusiastic-startup-tone. Applies to captions, empty states, hover labels, 404 page.
- Prefer data-as-self-description over adjectives: e.g. a stat/caption like "current streak: 41 days" over "I'm a passionate developer."

### 8.4 Single visible brand mark
- One abstracted, geometric scorpion+skateboard logo/icon (e.g., tail curve doubling as a kicktail) — used as favicon, hero mark, loading state. This is the only place the symbolism becomes literally visual.

### 8.5 Bespoke display font (subtle scorpion details) — direction only, not yet built
- Applies to headlines/stat display only; body copy remains a clean, highly legible grotesk.
- 2–3 structural nods baked into the display face: a sharp/stinger-like terminal on select strokes, a tail-taper on descenders (g/y/j), possibly a redrawn ampersand referencing a tail curl.
- **Not started yet** — deliberately deferred until the rest of the visual system (Section 7) is locked. Building this requires either FontForge/fonttools scripting (available in the build environment; fonttools confirmed installed, fontforge module not currently available) for a geometric/precise result, or hand-refinement in a dedicated type tool for a more polished result. Revisit as a distinct, later phase.

---

## 9. What to Deliberately Hold Back On

- No badges, trophies, confetti, level-ups, mascots, or click sound effects anywhere.
- No literal scorpion illustrations in the UI beyond the single brand mark.
- No forced onboarding tours or tooltip-driven explanations.
- No arbitrary point/XP systems with no underlying meaning.
- No gradients, glassmorphism, or neon accents.
- No bounce/overshoot motion or theatrical scroll effects.
- Don't over-theme every section with a metaphor label — the symbolic system (Section 8.2) is deliberately limited to 2–3 places where it does real structural work; everything else stays clean and quiet.
- Exploration/progress indicators (if used) should stay understated (e.g. a thin footer indicator) — never a leaderboard-style HUD.

---

## 10. Open Decisions (need your input before/at build start)

1. **Frontend approach:** Wagtail-rendered templates (Option A) vs. headless Wagtail + Next.js/Astro (Option B).
2. **Workout data provider:** which service (Strava/Garmin/Fitbit/etc.) — integration approach is provider-specific.
3. **UV/night-mode palette:** confirm as a real toggle, or keep as an easter-egg-only hidden state.
4. **Bespoke display font:** confirm intent to proceed later, and whether geometric/scripted output is acceptable or hand-refined type design is required.
5. **"Hidden layer" easter egg:** confirm scope (e.g. a terminal-style hidden route) and whether it ties into the UV-mode concept or stands alone.
6. **Admin roles beyond yourself:** confirm which specific people (if any) get Content admin / Stats admin / Viewer access.

---

*This document reflects all decisions made in discussion prior to build. Once confirmed, next step is a static homepage mockup (type scale, grid, hero section) before Wagtail model implementation begins.*
