# Print Track Daily Report — Project Reference

## What this is
A standalone daily production report tool for Evergreen Studio. It reads a
Firebase data export from the Print Track production tracker and renders an
interactive dashboard for reviewing production performance by date.

It is a **single self-contained HTML file** — no server, no build step, no
dependencies beyond Chart.js (loaded from CDN). Open `index.html` in any
browser to use it.

## Relationship to other projects
- **LM_Production_Tracker** — the Netlify/Firebase app that operators use
  daily to log production sessions, tally counts, maintenance events, and
  wait time. That app is the *source* of all data this report consumes.
- **Claude_Code_LM** — the Print Log app (EGStudio PrintLog) that tracks
  nesting files, gang job sheets, and order status. Separate system.

## Data structure (RAW object)
- `RAW.sessions` — keyed by machine name (`30`, `30+`, `H5`, `Colex`,
  `Wallets`, `H5`, etc.). Each session has:
  - `t` — date string (YYYY-MM-DD)
  - `ts` — ISO timestamp
  - `g` — good pieces
  - `b` — bad pieces
  - `s` — total seconds of active run time
  - `p` — piece type (e.g. `"Coir · 28x16 FC"`)
  - `o` — operator name
  - `m` — mode (`stopgo-fc`, `continuous-fc`, `tally`, `tally2`, `stamped`,
    etc.) — treat any mode starting with `"tally"` as tally-family, not just
    the exact string `"tally"` (found 2026-09-10: the floor's live data is
    entirely `tally2`, and an exact-match check silently missed it)
  - `st` / `et` — **not actually populated for tally-family sessions** (checked
    directly against the operator tracker's `tally.js`/`tally2.js` source
    2026-09-10: `tally.js` writes a `startTime` but never an `endTime`;
    `tally2.js` writes neither). Every tally-family session also hardcodes
    `s`/`totalSec` to a fixed `60` on each autosave — not real elapsed time
    either. The only trustworthy timing signal for tally-family sessions is
    each one's own `ts` (save timestamp): reconstruct elapsed time per piece
    type as (latest `ts` − earliest `ts`) across all its tally-family
    sessions that day, never from a single session's fields.
- `RAW.targets` — keyed by piece type. Each has `pph` (pieces per hour target)
  and optionally `ppt` (pieces per table, for coir).
- `RAW.maint` — maintenance and incident log. Each entry has `t`, `machine`,
  `type` (Cleaning / Machine Down / Operator Fix / Defective Material), `notes`,
  `ts` (raw timestamp, added 2026-09-14 to bucket Cleaning entries into
  Day/Night shift), and `detail` (added 2026-09-14 — for `type:"Cleaning"`
  entries, the first `" · "`-separated segment is the checklist tier: "Start
  of Shift"/"Mid Shift"/"End of Shift"/"40 Hour"/etc., set by `cleanSubmit()`
  in the operator tracker's `js/maintenance.js`). Used by the Weekly Machine
  Recap's cleaning-checklist indicators (`_cleaningStatusForShift`,
  `_cleaningFortyHourDone`).
- `RAW.wait` — wait/downtime log. Each entry has `t`, `machine`, `dur` (seconds),
  `notes`, `op`.
- `RAW.oeeParams` — OEE parameters per machine: `stdDownMin` (standard planned
  downtime in minutes), `idealCycleSec` (ideal seconds per piece), `targetOEE`,
  `targetAvail`, `targetPerf`, `targetQual`.
- `RAW.shiftMin` — planned shift length in minutes (570 = 6:30 AM to 4:30 PM
  minus 30-min break).

## SHIPPING_STATUS (not part of RAW — added 2026-09-16)
A separate top-level global, fetched from Firebase path `shippingStatus` —
the same live snapshot the TV production dashboard's Shipping view reads
(synced by `LM_Production_Tracker`'s `shipping_status_sync.py` from a
coworker's Postgres database). **Not part of `RAW`** and not date-scoped:
`shipping_status_sync.py` overwrites this one Firebase node every 5 minutes
with the current live state — there is no per-date history anywhere. It is
only ever meaningful for *today*; using it for a past date would silently
show today's numbers mislabeled as that date's. Shape: `{ collation, assembly,
sorting, readyToShip }` (today's piece-type breakdown per station),
`shiftBreakdown: { shift, current: {collation,assembly,sorting,readyToShip},
previous: {...} }`, `exceptions: { count, qty }`, `updatedAt`. The Daily
Recap print report (see below) gates its Shipping Recap section on
`date === today` for exactly this reason — don't remove that gate.

## Machines
- **30** — coir printer (larger pieces, slower cycle)
- **30+** — coir and signs printer (faster, higher volume)
- **H5** — signs, non-coir mats, flocked coir (fastest cycle)
- **Colex** — display pieces and specialty cuts
- **Wallets** — leather wallet stamping station
- **Drinkware M1 / M2** — new drinkware printing machines (added May 2026)
- **Unassigned / Unknown** — early sessions before machine assignment was enforced

OEE is calculated for `30`, `30+`, and `H5` only. Colex and Wallets have
`idealCycleSec: 0` and no OEE target.

## OEE calculation method
```
Availability = (SHIFT_SEC - stdDownSec - waitSec) / SHIFT_SEC
Performance  = (units * idealCycleSec) / runSec
Quality      = good / (good + bad)
OEE          = Availability × Performance × Quality
```
Units with fewer than 10 total pieces (good + bad) are excluded from OEE.
Tally-mode sessions ARE included in OEE (they have reliable timestamps).
Stamped sessions (Wallets) are excluded.

## Piece type naming convention
`Category · Size [Coat]`
Examples: `Coir · 28x16 FC`, `Signs · Yard Sign`, `Non-Coir Mats · PVC`
- FC = full color
- OC = one color
- No suffix = mixed or unspecified

## Known quirks
- Early sessions (March 6–9, 2026) have many `Unassigned` / `Unknown` machine
  entries from before the app enforced machine selection.
- Tally mode was introduced around April 15, 2026, replacing stop-go timing
  for some machines; by September 2026 the floor runs almost entirely on a
  newer `tally2` variant instead of plain `tally`. Neither variant has a
  reliable `s`/`st`/`et` on any single session (see RAW.sessions notes above).
  If matching tally-family modes anywhere, use `m.indexOf('tally') === 0`,
  never an exact `=== 'tally'` — an exact match silently missed `tally2` once
  already (2026-09-10) and broke a feature that depended on it.
- **The daily report (index.html `render()`) does NOT compute a per-piece-type
  PPH anymore** (removed 2026-09-14, after several rounds trying to make it
  reliable — see git history around commits `2ed6efb`..`aa70ed9` for the full
  saga). Piece-count-based rate targets can't be trusted for tally-tracked
  production for the reason above, so "Actual vs target — pieces per hour"
  was replaced with a "Trend vs Recent" section (today's count per piece type
  vs. that type's own trailing-5-day average) — pure piece counts, no timing
  math at all. Don't reintroduce a PPH table without discussing with the
  owner first; this was a deliberate strategic pivot, not an oversight.
- **OEE (`calcOEE()`) still has the tally-timing problem, not fixed** (found
  2026-09-10, deliberately left alone when PPH was abandoned rather than
  patched further): it prefers `st`/`et` and falls back to summing `s`/
  `totalSec` — for tally-family sessions that's silently `count × 60`, not
  real time. If OEE numbers for a tally-heavy day are ever questioned, the
  likely first response now is the same one PPH got: consider whether OEE is
  worth abandoning/de-emphasizing too, rather than another round of patching.

## Daily Recap print report (added 2026-09-16, enlarged same day)
A single-page printable summary for one day, triggered by the "Print Daily
Recap" button next to the datepicker on Today's Results (`printDailyRecap()`
→ `_buildDailyRecapHtml(date)`, print target `#daily-recap-print`, CSS under
`body.print-daily-recap`). Meant primarily for *past* days (unlike the live
on-screen report, which is most useful for today). Contents, top to bottom:
date/weekday header, Print Floor/Drinkware/Wallets piece totals, % to Plan
for Day shift then Night shift **stacked** (not side by side — each gets the
full page width), and a Shipping Recap. Numbers throughout are deliberately
large (owner request) and actual/expected are shown as a fraction — a big
"scoreboard" style stacked fraction (`.dr-fraction-num`/`-line`/`-den`) for
the Print Floor/Drinkware summary cards, a compact inline "actual / expected"
for the per-machine table rows.
- % to Plan reuses `buildPlanVsActual` directly (both the shift summary cards
  and, via `_drMachineRows`/`_plannedQtyFromPlan`, the per-machine table), so
  it inherits the same pacing behavior as the on-screen Plan vs Actual — a
  past day naturally shows its final ratio since elapsed time is 100%.
- Shipping Recap blends two different data sources with different lifecycles:
  Assembled/Sorted/Ready-to-Ship come from `SHIPPING_STATUS` (live only,
  today-only — see above); **# Ship Confirmed** comes from `SHIP_CONFIRM`
  (date-keyed, Firebase path `shipConfirm/{date}`, written once per day by
  `ship_confirm_sync.py` — has real history), so it's shown on *any* date
  that has a record, not just today. Collated is deliberately NOT shown
  (owner asked for it removed in favor of Ship Confirmed).
- Reads `_dailyRenderCache` (set at the end of `render()`) rather than
  recomputing the session aggregation — the date must already be loaded on
  Today's Results before printing.
- Sizing was tuned by rough estimate, not an actual print-preview render (no
  browser available in-session to verify) — if the owner reports it no longer
  fits one page (or has room to grow further), the font-size/padding/margin
  values in the `body.print-daily-recap` CSS block are all in one place to
  adjust together.

## Rules for Claude Code
- Do not change the data structure or variable names in the `RAW` object.
- Do not add external dependencies beyond Chart.js (already loaded from CDN).
- When adding new features, keep everything in the single `index.html` file
  unless explicitly asked to split it out.
- Preserve the existing color system (CSS variables: `--green-dark`,
  `--green-mid`, `--green-light`, `--red`, `--amber`, `--blue`, etc.).
- The Evergreen Studio logo is embedded as base64 in the topbar — do not
  remove or replace it.
- Always test date edge cases: dates with no data, dates with only tally
  sessions, dates where OEE cannot be calculated.
