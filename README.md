# 🌱 Front Yard Garden Tracker

A personal garden management web app for a 12-bed south-facing front yard in zone 7a. Tracks crops, season logs, recurring care tasks, and harvest yields across a drip-irrigated raised bed system.

**Live app:** https://new-garden-tracker.vercel.app

---

## How This Got Built

This started as a question about what an irrigation system might look like. It ended up as a full-stack web app with a real database, a crop task system, and harvest yield tracking. The whole thing was built conversationally — no prior coding experience required — using Claude.ai for planning and Claude Code in the VS Code terminal for implementation.

The workflow: talk through what you want in Claude.ai, get a spec, hand it to Claude Code, review the diffs, approve, push. Vercel handles the rest.

---

## The Irrigation System

Two-zone drip system fed from the front yard spigot (target 60+ PSI / 6+ GPM after fixing a PEX run).

- **Zone 1** — 2 GPH emitters
- **Zone 2** — 1 GPH emitters
- **Timer:** Rachio smart controller
- **Trunk:** 3/4" PVC east–west along north edge (~30 ft)
- **Laterals:** 1/2" PVC south down each column (~22 ft each)
- **Install order:** Backflow preventer → Rachio timer → Pressure regulator (25–30 PSI) → Y-filter (150 mesh) → trunk → laterals → drip tubing

---

## The App

Three tabs: **Field**, **Planning**, and **Overview**.

**Field tab** — SVG garden map with 12 stadium-pill bed tiles filling the full screen. Each pill carries three status signals:

- **Progress arc ring** — sweeps clockwise from the top of the pill. Teal while growing; shifts to amber and pulses when ≥85% through the growing window or status is "ready to harvest". Arc endpoint uses `expected_harvest` date when set, falls back to `crops.days_to_maturity` otherwise. Perennials show a steady full teal ring with no countdown. No ring = draft/unplanted.
- **Draft outline** — unplanted beds show a dashed blue pill border and dimmed crop label.
- **Task tick marks** — pending tasks render as small amber squares arced along the inside of the pill's bottom curve (1–4 marks; count number for 5+).

Tap any bed to open a bottom sheet inspector.

The inspector is organized around two jobs: *what needs doing* (tasks due this week) and *let me record something* (log water, add note, log harvest). The header shows crop name, variety, bed chip, and status badge (tap to change) with a timeline strip below — a slim progress bar mirroring the arc ring, labeled with planted date + relative time on the left and expected harvest on the right. Perennials show "In ground since [date]" instead. A minimap shows all 12 beds. Tasks render as a checklist; tap to expand description, check to complete. The two-pill action row has a water pill (goes amber when 4+ days since last watering) and a note pill. A collapsible last-note strip shows the most recent note; tap to expand to last 3. Harvest logging is a quiet full-width button. "Close this planting" is a plain text link at the bottom.

When a bed has an active planting plus a queued draft, swipe left/right in the inspector to navigate between them. Dot indicators below the drag handle show current position.

Draft beds (not yet planted) show a simplified inspector with only a "Mark as planted today" CTA.

**Planning tab** — 3×4 grid of plan cards, one per bed, with a year selector. Active beds show a connected peek strip below the card: the queued draft crop name (tap to edit) or `+ plan next` if none yet. Clicking the main card edits the active planting; clicking the peek opens the draft or creates a new one. Draft beds show as dashed blue cards. The plan form has a searchable crop picker backed by the `crops` DB table (shows sun, water, spacing, frost-hardy metadata), a variety field with autocomplete from past planting history filtered to the selected crop, planting method selector, dates, seed source, spacing notes, and season notes.

To carry a season forward: open any plan card and use "Copy to [year+1] →" at the bottom of the form, or use "Use [year] as template for [year+1]" in the footer to bulk-copy all planned beds. Both jump the view to the target year and skip beds already planned.

A task calendar below the grid shows all tasks this season by bed, filterable by bed chip.

**Overview tab** — Season summary: beds active, yield to date, most recent harvest. Activity feed of recent waterings and notes. Season notes section with bed filter chips.

A **daily briefing card** sits at the top of the Overview tab. Each morning at 6am MDT a Supabase Edge Function reads active plantings, open tasks, recent logs, harvests, and waterings from the database, fetches the 7-day NWS forecast and active weather alerts, and calls the Gemini API to synthesize a briefing. The briefing has two sections: **This week** — 2–5 weather facts each tied to a consequence (e.g. "Rain Tue–Wed — skip watering Mon") — and **Actions** — suggested tasks where every item includes a specific reason it's needed now (a weather window, crop timing milestone, or overdue action). Each action is accept/rejectable from the card; accepted suggestions are added to the task system for the relevant bed.

**Water** — Full irrigation system diagram with component callouts and zone color coding.

**Drip** — Cross-section showing how water gets from the lateral line into a raised bed (punched tee → poly tubing → emitters).

---

## Tech Stack

| Thing | What |
|-------|------|
| Frontend | Single `index.html` — embedded CSS and JS, no build step |
| Hosting | Vercel, auto-deploys from `main` |
| Database | Supabase (Postgres) |
| Daily briefing | Supabase Edge Function (`generate-briefing`) + Gemini 2.5 Flash API |
| Briefing schedule | pg_cron + pg_net — fires at 12:00 UTC (6am MDT) daily |
| Dev tools | VS Code + Live Server, Claude Code via terminal |
| Planning | Claude.ai |
| Irrigation timer | Rachio |

The Supabase anon key is intentionally baked into the frontend. It's the publishable key, designed to be public. No auth layer — this is a single-user personal tool with no sensitive data.

RLS is enabled on all tables. Policies follow a consistent naming convention and grant access to the `anon` role only (the publishable key). No DELETE policies exist anywhere — the app never deletes data.

| Table | SELECT | INSERT | UPDATE |
|-------|--------|--------|--------|
| `locations` | ✓ | ✓ | ✓ |
| `plantings` | ✓ | ✓ | ✓ (set `planted_date`, `ended_date`, `status`) |
| `logs` | ✓ | ✓ | ✓ |
| `harvests` | ✓ | ✓ | ✓ |
| `tasks` | ✓ | ✓ | ✓ |
| `waterings` | ✓ | ✓ | — (records are never patched) |

All policies use `USING (true)` / `WITH CHECK (true)` — open to the anon key, locked to everything else. If you enable RLS on a new table and the app breaks, add the same `anon_select_<table>` + `anon_insert_<table>` policies and an `anon_update_<table>` if the app sends PATCHes to it.

---

## Data Model

Everything hangs off **plantings** — a specific crop in a specific bed for a specific year and season. A bed is a place. A planting is an event.

```
locations
  id, label (e.g. "w1"), location_type (raised_bed|tree|shrub|vine|container|ground),
  zone, emitter_gph, grid_col, grid_row (null for non-bed types),
  is_in_ground, is_active, notes

plantings
  id, bed_id, location_id → locations.id,
  crop_name, variety, expected_harvest, year, season,
  planted_date, ended_date, status, planting_state,
  planned_date, planting_method, start_indoors_date,
  seed_source, spacing_notes, season_notes

crops
  id, name, water_needs, sun_needs, spacing_inches, frost_hardy, companion_plants

harvests
  id, planting_id, harvest_date, amount, unit (lbs/count), notes

logs
  id, planting_id, date, note

tasks
  id, planting_id, task_id, completed_date

briefings
  id, headline, payload (json), generated_at, suggestions_state (json)

waterings
  id, zone (1 or 2), watered_date, duration_minutes, source (manual | rachio)
```

`ended_date = null` means the planting is still active. `planting_state` is `draft` (planned, not yet planted), `active` (in ground), or `ended` (closed). When both exist for the same bed, active takes priority over draft in all display logic.

`status` is a user-settable growth stage: `planned`, `sown`, `germinating`, `growing`, `ready`, or `harvested`. Stored in the DB and updated from the inspector status badge.

`variety` is the specific cultivar name. The planning form autocompletes from past planting history filtered to the selected crop — no separate varieties table needed. `expected_harvest` is free text because harvest windows are ranges.

**Succession** is modeled as multiple planting records for the same bed — no foreign-key linking needed. The first draft planting for a bed (sorted by `planned_date`) is implicitly the next crop. When you "close" a planting, it gets `ended_date = today`. The draft becomes the next thing to plant.

Waterings are zone-scoped, not planting-scoped. The `source` column is the Rachio integration seam.

`crops` is a reference table of known crop types with agronomic metadata. The planning form uses it as a searchable picker and auto-fills spacing notes when a crop is selected.

---

## Development Workflow

```bash
cd ~/Projects/New-Garden-Tracker
claude          # opens Claude Code in the project directory
```

Claude Code reads the files, makes changes, shows diffs for approval, then:

```bash
git add index.html && git commit -m "..." && git push
# Vercel picks it up in ~30 seconds
```

For planning and architecture: Claude.ai. For implementation: Claude Code. Different tools, different jobs.

---

## Decision Log

> **Convention:** Use full date stamps (`Mar 23, 2026`) in the Date column, not just month and year.

| Date | Decision | Rationale | Alternatives |
|------|----------|-----------|--------------|
| Mar 2026 | Single HTML file, no build step | Maximum simplicity. Edit, push, done. | React, Vue, Next.js |
| Mar 2026 | Supabase over Google Apps Script + Sheets | Apps Script was flaky and had no error visibility. Supabase is a real database with a real API. | Firebase, PlanetScale, keep Sheets |
| Mar 2026 | Two-zone irrigation | Different crop types need different emitter rates. Two zones = independent scheduling. | Single zone, four zones |
| Mar 2026 | Bed IDs as position labels (W1–E4) | Self-documenting. W3 tells you where the bed is. 3 does not. | Numeric IDs |
| Mar 2026 | Static crop-defined tasks, not user-managed | Experience building another app ("Oh Grow Up") showed that user-managed task systems get complicated fast. Pre-baked schedules per crop require zero management. | Full task manager |
| Mar 2026 | Calendar date windows for tasks, not week numbers | "Jun 3–9" is more useful at a glance than "week 13". | Display week offsets directly |
| Mar 2026 | No overdue task state in v1 | Adds complexity without proven need for a single-user personal tool. | Overdue badges and stacking |
| Mar 2026 | Plantings table as central data anchor | A bed is a place, not a crop. Attaching logs and harvests to a planting (crop + bed + year + season) makes the data meaningful across time and enables year-over-year comparison. | Keep using bed_id everywhere |
| Mar 2026 | Harvest logging per pick event, not as a running total | Log each pick separately and let the app sum them. More flexible, more honest, and you don't have to do math before logging. | Single total field per planting |
| Mar 2026 | Claude Code + Supabase MCP | Claude Code can talk directly to Supabase to create tables and manage schema, which removes an entire category of manual setup steps. | Manual schema setup in Supabase UI |
| Mar 2026 | `ended_date = null` to mark active planting | Simpler to query than a separate boolean. A planting that hasn't ended is active by definition — no extra field needed. | Separate `is_active` boolean |
| Mar 2026 | Succession crops as separate planting records | Each crop gets its own log and harvest history even if it's the same bed. Updating in place would destroy the record of what came before. | Update existing planting row |
| Mar 2026 | No auth layer, publishable Supabase key in frontend | Single-user personal tool with no sensitive data. The anon key is designed to be public. Adding auth would add friction with zero security benefit here. RLS is enabled on all tables; access is granted to the anon role only. | Auth.js, Supabase Auth, env vars |
| Mar 2026 | SVG for all diagrams | No external chart library. Keeps the single-file constraint. Full control over layout and theming. SVG elements respond to CSS custom properties so dark/light mode works cleanly. | D3.js, Canvas, image files |
| Mar 2026 | Dynamic crop labels on the map SVG — read from database, not hardcoded | Hardcoded labels drifted from reality the moment the first planting changed. Labels now update from `plantingCache` on load and after any planting save. Shows `fallow` when no active planting exists. | Manually update SVG text on each crop change |
| Mar 2026 | Waterings table is zone-scoped, not planting-scoped | A watering event applies to a zone, not a crop. One button on any bed in zone 2 logs a watering for all zone 2 beds. | Attach waterings to planting_id |
| Mar 2026 | `source` column as Rachio integration seam | The manual watering button writes `source='manual'`. When Rachio ships, it writes `source='rachio'` to the same table. The display layer (last watered X days ago) reads rows regardless of source — it never needs to change. | Separate manual/rachio tables, migration later |
| Mar 2026 | Bed detail panel reads crop/variety/harvest from DB, falls back to BEDS static data | The hardcoded `BEDS` constant was the source of truth for the inspector panel, so new plantings saved to Supabase never appeared correctly. DB is now authoritative; BEDS serves only as fallback for fields not yet in the DB. | Keep BEDS as sole source of truth |
| Mar 2026 | `variety` and `expected_harvest` as explicit columns on `plantings`, not derived from crop name | Variety is meaningful data — "Tomato" and "Orange Hat" are different things. Storing only `crop_name` meant variety was lost the moment a new crop was saved. `expected_harvest` kept as free text because harvest windows are ranges, not single dates. | Embed variety in crop_name, or add a separate varieties table |
| Mar 2026 | `crop_name` stores the crop type, `variety` stores the cultivar | Consistent with how gardeners talk: "I'm growing tomatoes — specifically Orange Hat." Keeps `crop_name` groupable and `variety` searchable independently. | Store the full name ("Orange Hat Tomato") in crop_name |
| Mar 27, 2026 | Replace flip-card inspector with a fixed bottom sheet | The 3D flip card required a card wrapper with a background, making the map feel framed and visually heavy. A bottom sheet slides up over the full-screen map, leaving the garden visible behind the backdrop — more native-feeling on mobile. | Side drawer, modal dialog |
| Mar 27, 2026 | Strip all SVG chrome from the garden map — pills only, no house band, zone labels, compass, or grid | Each extra element shrank the usable bed area. Removing decorative chrome let the beds scale to 76×140 px with equal 53 px spacing, filling the full iPhone viewport. | Keep chrome, reduce pill size |
| Mar 27, 2026 | Dark mode uses deep forest green (`#0d2016`) instead of near-black | Pure dark backgrounds read as generic and cold on the garden map. A saturated dark green grounds the UI in the context of the garden without washing out bed colors. | Standard #111 dark background |
| Mar 27, 2026 | `status` stored as an explicit DB column, not derived from `planted_date` | Deriving status from `planted_date` only distinguished planned vs growing — no way to express sown, germinating, ready, or harvested. Storing it explicitly in the DB makes the full 6-stage lifecycle writable from the inspector and persistent across sessions. | Derive more states from dates; keep status frontend-only |
| Mar 27, 2026 | Glowing bed-ring as status indicator instead of an overlay dot | Each pill already has a ring element with a glow filter attached — it sits at opacity 0 by default. Lighting it up with a status color (jade, lime, teal, amber, brown) is cheaper than adding new SVG elements and produces a more organic, bloom-like effect. "Ready" beds pulse amber on a CSS animation. | Small dot inside pill, colored gradient overlay, border color change |
| Mar 27, 2026 | Crop name as primary heading in the inspector, bed ID demoted to a chip | When you tap a bed, you know which bed you tapped — the map is right behind the sheet. What you want confirmed is what's growing there. Crop name at 1.65rem Playfair is the answer; "W3" as a small monospace chip is the address. | Keep bed ID as large heading, add crop as subtitle |
| Mar 27, 2026 | Inspector dates in natural format with relative time | ISO dates (2026-03-08) require mental math. "Mar 8, 2026" is readable at a glance. "3 weeks ago" under the sown date answers the most common follow-up question without opening a calendar. | ISO dates only; relative time in a tooltip |
| Mar 27, 2026 | Harvest form hidden behind a toggle by default | The harvest entry form (date, amount, unit, notes) was always expanded in the Yield accordion, including for crops that haven't been planted yet. Collapsing it behind "+ Log harvest" reduces noise and auto-collapses again after each successful entry. | Always visible; collapse only on mobile |
| Mar 27, 2026 | Section labels in sentence case, not ALL-CAPS | ALL-CAPS with 2.5px letter-spacing reads as shouting. Sentence case at 0.5px spacing reads as a structured header — same visual weight, less aggressive. | Title Case; keep ALL-CAPS |
| Mar 30, 2026 | Field inspector redesigned around two jobs: tasks + quick actions | With Overview absorbing ambient status (yield history, harvest countdowns, notes history), the field inspector no longer needs to carry that weight. Stripped to: tasks checklist, water/note pill row, collapsible note strip, quiet harvest button. All writes go through modal overlays. | Keep full-detail inspector |
| Mar 30, 2026 | Succession = any draft planting for the same bed, no FK linking | An explicit `succession_planting_id` FK required pre-linking before the current crop was even done. Dropping it means any draft planting for a bed is implicitly the successor — naturally ordered by `planned_date`. Simpler to create, simpler to query. | FK-based succession_planting_id |
| Mar 30, 2026 | Active planting takes priority over draft when both exist for same bed | `getPlanningPlanting` and `getActivePlanting` check active before draft. Without this, a newly created draft would hide the in-ground crop in both field and plan views. | First-created or most-recent wins |
| Mar 30, 2026 | Cross-year carryover scoped to current year only | Active plantings from prior years (garlic planted fall 2025) appear in the 2026 planning view but NOT in 2027+. Future years are a clean slate — showing active 2026 plantings in 2027 pre-pollutes the planning canvas. | Show carryover in all future years |
| Mar 30, 2026 | Copy-to-next-year uses `planYear + 1`, never a hardcoded year | Any button or footer label showing a target year computes it from the current `planYear` variable. Prevents the app from reading "Copy to 2027" after 2027 has passed. | Hardcode target year, update annually |
| Mar 30, 2026 | Variety autocomplete from planting history, no separate varieties table | `varieties` for a crop are whatever you've actually grown — queried as `DISTINCT variety WHERE crop_name = X` from `plantings`. Grows richer over time automatically. A managed vocabulary table would require seeding and maintenance for marginal gain in a single-user app. | Separate varieties table with FK to crops |
| Mar 30, 2026 | Plan card peek strip replaces inline succession text | The old `→ Beans` amber text inside the active card mentally linked the succession crop to the current planting. A connected peek strip below the card reads as "something queued here" without implying relationship to the active crop. Active beds always show a peek — draft queued or `+ plan next` if empty. | Inline text; separate succession section |
| Mar 30, 2026 | Inspector swipe for multi-planting navigation | When a bed has active + draft plantings, swipe left/right in the inspector to move between them. Dot indicator shows position. Horizontal swipe uses a 1.5× axis ratio to avoid conflicting with the existing swipe-down-to-close gesture. No tap-to-pick intermediary — common case (active planting) is always index 0. | Tap-to-pick planting selector; separate sheet |
| Mar 30, 2026 | `crops` table integrated into plan form as searchable picker | Crop name was a free-text field with no connection to the crops reference table. The picker surfaces agronomic metadata (sun, water, spacing, frost hardiness) inline and auto-fills spacing notes on selection. Typed names that don't match are saved as "new" crops with no metadata. | Keep free-text crop name field |
| Mar 30, 2026 | Bed ring replaced with a clockwise progress arc using `stroke-dasharray` | A binary on/off ring only answers "is something growing?" The arc answers "how far through the season is it?" — a continuous signal computed from planted_date and expected_harvest or days_to_maturity. Starts at the top of the pill (12 o'clock) because SVG rect stroke with rx=width/2 naturally begins there. | Static ring with color states; separate progress bar overlay |
| Mar 30, 2026 | Arc endpoint uses `expected_harvest` date first, falls back to DTM | The generic `crops.days_to_maturity` range is an approximation. When the user sets a specific `expected_harvest` date, that is more accurate and should drive the arc. Parsing `expected_harvest` free text (strip ~, split ranges, append year if missing) is sufficient for the formats in use. | Always use DTM; add a separate `expected_harvest_date` date column |
| Mar 30, 2026 | Perennials get a steady full teal ring, not a harvest countdown | Perennials don't have a harvest window to count down to — their `days_to_maturity` is null and they just keep going. The `is_perennial` flag on the `crops` table gates the arc logic so perennials never trigger the amber state. | Check for null DTM only (works but loses semantic clarity as more perennials are added) |
| Mar 30, 2026 | Inspector timeline strip mirrors the arc in text form | The arc ring answers "how far along?" visually but doesn't give the actual dates. A slim 4px progress bar below the header status row, labeled with planted date and expected harvest, surfaces both dates without adding a new section or disrupting the inspector layout. | Add a separate dates section; show dates in the header directly |
| Mar 30, 2026 | Task badges moved inside the pill as arced tick marks | An external circle badge overlapping the pill edge felt like a wart — it broke the pill's clean boundary. Small amber squares arced along the inside of the pill's bottom curve (y = arc_center_y + sqrt(innerR²-dx²)) sit within the pill shape and follow its curvature. 1–4 marks are readable at a glance; 5+ falls back to a count number. | Keep external badge; move to pill interior as flat row |
| Mar 30, 2026 | Split v2.2 into schema-first (v2.2a) and frontend-refactor (v2.2b) | Schema migration is purely additive and zero-risk; frontend refactor touches 9 functions and 4 hardcoded arrays and deserves its own ship cycle | Ship together; keep BEDS constant forever |
| Mar 30, 2026 | `locations.label` matches existing `bed_id` strings — no cache key changes | `plantingCache` is keyed by `bed_id` string. Using `label` as the canonical identifier means the cache key, Supabase queries, and SVG element IDs (`pill-w1` etc.) all stay unchanged through v2.2 | Use UUID as primary key throughout; rename everything |
| Mar 30, 2026 | Non-bed locations get DB records in v2.2a, map/planning UI deferred to v3 | Map and planning redesigns require real design work. The peach tree and blackberry need to be plantable and loggable now, not mappable yet. | Wait for v3 before inserting any non-bed locations |
| Apr 2, 2026 | Daily briefing via Supabase Edge Function, not a Claude CCR remote trigger | CCR remote trigger has no outbound network access — the sandbox blocks all external HTTP (Supabase, weather APIs). A Supabase Edge Function runs on Supabase's own infrastructure, reads the DB directly without HTTP auth, and has full outbound network access. pg_cron + pg_net schedule it daily at 12:00 UTC. | CCR remote trigger (abandoned), local Mac cron (machine must be on), GitHub Actions |
| Apr 2, 2026 | Gemini 2.5 Flash for AI synthesis in the edge function | Free tier with billing attached. As of April 2026, `gemini-2.0-flash` and `gemini-2.0-flash-lite` have `limit: 0` on the free tier for new projects — only `gemini-2.5-flash` carries actual free quota (20 RPD). Sufficient for 1 call/day with headroom. | Anthropic Claude Haiku (~$2/season), Gemini 2.0 Flash (quota blocked) |
| Apr 2, 2026 | Google Cloud project must have "Generative Language API" enabled AND billing attached to unlock free tier | New Google Cloud projects created from AI Studio have the API enabled but free tier quotas default to 0. Attaching a billing account unlocks the quotas without charging for usage within the free limits. Not obvious from Google's documentation. | — |
| Apr 2, 2026 | Service worker (`sw.js`) with network-first strategy for iOS PWA caching | iOS standalone PWA mode ignores `http-equiv` no-cache meta tags and aggressively caches the app shell. A service worker is the correct mechanism — always fetches fresh HTML when online, falls back to cache offline. `http-equiv` tags alone are insufficient for iOS home screen apps. | Strip no-cache meta tags; accept stale PWA behavior |
| Apr 12, 2026 | Briefing card stripped to weather facts + actions only — no narrative, no insights | The narrative and insights sections produced verbose prose that restated garden state the gardener already knows. The briefing's job is: here is the weather for the next N days, and here is what you need to do because of it. Every action now requires a specific reason (weather window or crop timing milestone) to appear. | Keep narrative with a shorter word limit |

---

## Roadmap

### v2.2 — Dynamic Locations

**v2.2a — Schema** ✓ Done
`locations` table created and seeded with all 12 raised beds. `location_id` FK added to `plantings` and backfilled from `bed_id`. RLS policies in place. Zero user-visible change — the schema now exists for v2.2b to wire into.

**v2.2b — Frontend refactor** (next)
Replace the hardcoded `BEDS` constant with a `fetchLocations()` call that loads location data from the DB at startup. Derive `ALL_BEDS` arrays and the planning grid `ROWS` layout from `grid_col/grid_row` on the fetched records. Strip `slotA` fallback planting data from `BEDS` — the database is now authoritative.

**v2.2c — Non-bed locations in Overview**
Add a section to the Overview tab listing non-raised-bed locations (peach tree, blackberry) with their most recent planting or log. Non-bed locations are already plantable and loggable after v2.2a — this just surfaces them in the UI.

---

### Priority 1 — Rachio Integration
The `waterings` table and display layer are already in place. Manual watering entries write `source='manual'`. When the Rachio timer and drip system are installed (~6 weeks), the integration is:
1. Authenticate with the Rachio API
2. Pull zone run history on a schedule or webhook
3. Write events to `waterings` with `source='rachio'`
4. Remove the manual "Log watering" button from the inspector

The "last watered X days ago" display already reads from `waterings` regardless of source — no frontend changes needed.

### ~~Priority 2 — Custom Tasks~~ ✓ Done
Custom task creation is implemented. The inspector task list includes an "Add task" entry that opens a modal for free-text title, optional description, and due date. Custom tasks are stored in the `tasks` table with `is_custom = true`.

### Priority 3 — Strip BEDS fallback data
The hardcoded `BEDS` constant still carries `slotA` planting fallbacks. Strip it to infrastructure facts only (zone, emitters, GPH). The database is now fully authoritative for crop data; the fallback just drifts stale.

### Task System
- Overdue task state — when a window closes unchecked, show it rather than silently dropping it
- Task stacking — "Remove suckers — 3 weeks behind"

### Variety vocabulary
Currently variety suggestions come from planting history. Once there's meaningful history, consider whether a managed `varieties` table (keyed to `crops`) would be worth the maintenance cost for autocomplete quality. Not urgent for a single-user app.

### Logging & Yield
- Photo logging — attach a photo to a log entry
- Printable season summary — end of year export

### Irrigation
- Rachio schedule display — show the configured zone schedule on the plumbing tab
- Per-zone water usage over time — total minutes run by week/month

### Planning & History
- Year-over-year comparison view — "W3 in 2025 vs W3 in 2026"
- Multi-year crop rotation suggestions — "tomatoes were in E1 last year, consider moving"
- Seed inventory tracking — what's on hand, what needs reordering
- End of season bed prep checklist — triggered by first frost

---

*Built conversationally with Claude.ai and Claude Code. Zone 7a, Utah.*
