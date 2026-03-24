# 🌱 Front Yard Garden Tracker

A personal garden management web app for a 12-bed south-facing front yard in zone 7a. Tracks crops, season logs, recurring care tasks, and harvest yields across a drip-irrigated raised bed system.

**Live app:** https://cdewittburrow.github.io/New-Garden-Tracker/

---

## How This Got Built

This started as a question about what an irrigation system might look like. It ended up as a full-stack web app with a real database, a crop task system, and harvest yield tracking. The whole thing was built conversationally — no prior coding experience required — using Claude.ai for planning and Claude Code in the VS Code terminal for implementation.

The workflow: talk through what you want in Claude.ai, get a spec, hand it to Claude Code, review the diffs, approve, push. GitHub Pages handles the rest.

---

## The Garden

12 raised beds, 2×4×1 ft each, in a 3-column grid (West, Center, East) running north to south. Beds are referenced by position label — W1 through W4, C1 through C4, E1 through E4.

**2026 season:**

| Bed | Crop | Notes |
|-----|------|-------|
| W1 | Everbearing Strawberries | Perennial. USDA Organic, Fast Growing Trees. |
| W2 | Tomato — Orange Hat (Determinate) | 2 plants. Transplant May 10–20. |
| W3 | Elephant & Music Garlic → Kyoto Red Carrots | Garlic planted Oct 2025. Carrots after harvest. |
| W4 | Italian Red Garlic → New Kuroda Carrots | Garlic planted Oct 2025. Carrots after harvest. |
| C1 | 18-Day French Radish → Good Mother Stallard Beans | Radish sown Mar 8. Pole beans need trellis. |
| C2 | Sugar Bon Snap Peas → Blue Lake Bush 274 | Peas sown Mar 8. Beans sow May 20. |
| C3 | Lincoln Garden Peas → Jade Bush Beans | Peas sown Mar 8. Beans sow May 20. |
| C4 | Lillian's Caseload Peas → Red Swan Bush Beans | Peas sown Mar 8. Beans sow May 20. |
| E1 | Martino's Roma Tomato (Determinate) | 3 plants. Transplant May 10–20. |
| E2 | Amish Paste Tomato (Indeterminate) | 2 plants. Transplant May 10–20. |
| E3 | Comstock Sauce & Slice Tomato (Indeterminate) | 2 plants. Transplant May 10–20. |
| E4 | Bonny Best Tomato (Indeterminate) | 2 plants. Transplant May 10–20. |

---

## The Irrigation System

Two-zone drip system fed from the front yard spigot (target 60+ PSI / 6+ GPM after fixing a PEX run).

- **Zone 1** — Tomatoes (E column): 2 GPH emitters
- **Zone 2** — Everything else (W + C columns): 1 GPH emitters
- **Timer:** Rachio smart controller
- **Trunk:** 3/4" PVC east–west along north edge (~30 ft)
- **Laterals:** 1/2" PVC south down each column (~22 ft each)
- **Install order:** Backflow preventer → Rachio timer → Pressure regulator (25–30 PSI) → Y-filter (150 mesh) → trunk → laterals → drip tubing

---

## The App

**Beds** — SVG flip-card. The front face is the garden map; tap any bed to flip to the inspector. Inspector shows the active planting, crop status, rotation card, irrigation details, season log, and harvest log. A "Watered" button (with duration input) logs a manual watering for that bed's zone and shows "last watered X days ago (Y min)" inline. If a bed has had multiple crops in a season, a pill strip lets you switch between planting records. Bed tiles show a task badge when something is due this week.

**Water** — Full irrigation system diagram with component callouts and zone color coding.

**Drip** — Cross-section showing how water gets from the lateral line into a raised bed (punched tee → poly tubing → emitters).

**Tasks** — All crop care tasks due this week across all beds, in chronological order. Filter by bed. Check off tasks and they persist to the database.

---

## Tech Stack

| Thing | What |
|-------|------|
| Frontend | Single `index.html` — embedded CSS and JS, no build step |
| Hosting | GitHub Pages, auto-deploys from `main` |
| Database | Supabase (Postgres) |
| Dev tools | VS Code + Live Server, Claude Code via terminal |
| Planning | Claude.ai |
| Irrigation timer | Rachio |

The Supabase anon key is intentionally baked into the frontend. It's the publishable key, designed to be public. No auth layer — this is a single-user personal tool with no sensitive data.

RLS is enabled on all tables. Policies follow a consistent naming convention and grant access to the `anon` role only (the publishable key). No DELETE policies exist anywhere — the app never deletes data.

| Table | SELECT | INSERT | UPDATE |
|-------|--------|--------|--------|
| `plantings` | ✓ | ✓ | ✓ (set `planted_date`, `ended_date`) |
| `logs` | ✓ | ✓ | ✓ |
| `harvests` | ✓ | ✓ | ✓ |
| `tasks` | ✓ | ✓ | ✓ |
| `waterings` | ✓ | ✓ | — (records are never patched) |

All policies use `USING (true)` / `WITH CHECK (true)` — open to the anon key, locked to everything else. If you enable RLS on a new table and the app breaks, add the same `anon_select_<table>` + `anon_insert_<table>` policies and an `anon_update_<table>` if the app sends PATCHes to it.

---

## Data Model

Everything hangs off **plantings** — a specific crop in a specific bed for a specific year and season. A bed is a place. A planting is an event.

```
plantings
  id, bed_id, crop_name, variety, expected_harvest, year, season, planted_date, ended_date, notes

harvests
  id, planting_id, harvest_date, amount, unit (lbs/count), notes

logs
  id, planting_id, date, note

tasks
  id, planting_id, task_id, completed_date

waterings
  id, zone (1 or 2), watered_date, duration_minutes, source (manual | rachio)
```

`ended_date = null` means the planting is still active. This is what the app queries to find the current crop in a bed, drive task logic, and show the "Mark as Planted" button.

`variety` is the specific cultivar name (e.g. "Sugar Bon", "Elephant & Music"). `expected_harvest` is a free-text window (e.g. "~May 3, 2026", "Jul 20–Aug 10, 2026") — kept as text because harvest windows are ranges, not single dates. Both are optional and captured in the Add Crop form.

Waterings are zone-scoped, not planting-scoped — a watering applies to all beds on that zone. The `source` column is the Rachio seam: manual entries write `source='manual'`, and when Rachio integration ships it writes `source='rachio'` instead. The display layer never needs to change.

Succession crops (e.g. radish → beans in C1) get separate planting records. When you start a new crop from the bed inspector, the app sets `ended_date = today` on the outgoing planting and inserts a new row for the incoming one. Both records persist independently with their own logs and harvests.

---

## Development Workflow

```bash
cd ~/Projects/New-Garden-Tracker
claude          # opens Claude Code in the project directory
```

Claude Code reads the files, makes changes, shows diffs for approval, then:

```bash
git add index.html && git commit -m "..." && git push
# GitHub Pages picks it up in ~1 minute
```

For planning and architecture: Claude.ai. For implementation: Claude Code. Different tools, different jobs.

---

## Decision Log

> **Convention:** Use full date stamps (`Mar 23, 2026`) in the Date column, not just month and year.

| Date | Decision | Rationale | Alternatives |
|------|----------|-----------|--------------|
| Oct 2025 | Garlic in W3 and W4 | Overwinter, harvest early summer, succeed to carrots in same beds | Dedicated garlic section |
| Mar 2026 | Single HTML file, no build step | Maximum simplicity. Edit, push, done. | React, Vue, Next.js |
| Mar 2026 | Supabase over Google Apps Script + Sheets | Apps Script was flaky and had no error visibility. Supabase is a real database with a real API. | Firebase, PlanetScale, keep Sheets |
| Mar 2026 | Two-zone irrigation | Tomatoes need 2 GPH, everything else needs 1 GPH. Two zones = independent scheduling. | Single zone, four zones |
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

---

## Roadmap

### Priority 1 — Rachio Integration
The `waterings` table and display layer are already in place. Manual watering entries write `source='manual'`. When the Rachio timer and drip system are installed (~6 weeks), the integration is:
1. Authenticate with the Rachio API
2. Pull zone run history on a schedule or webhook
3. Write events to `waterings` with `source='rachio'`
4. Remove the manual "Watered" button from the inspector

The "last watered X days ago" display already reads from `waterings` regardless of source — no frontend changes needed.

### Priority 2 — Custom Tasks
The current task system is entirely crop-defined and hardcoded. There's no way to add a one-off task ("stake the E2 tomato today") or a recurring reminder that isn't already in the crop template. Custom tasks — both one-time and repeating — are the next meaningful addition to the task system.

### Task System
- Overdue task state — when a window closes unchecked, show it rather than silently dropping it
- Task stacking — "Remove suckers — 3 weeks behind"

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

### Infrastructure
- PWA support — installable on phone home screen, works offline

---

*Built conversationally with Claude.ai and Claude Code. Zone 7a, Utah.*
