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
  - `st` / `et` — tally-family session start/end timestamps (tally/tally2
    mode only) — this span, not `s`, is the reliable elapsed-time source for
    these modes; `s` is not trustworthy for tally-family sessions
- `RAW.targets` — keyed by piece type. Each has `pph` (pieces per hour target)
  and optionally `ppt` (pieces per table, for coir).
- `RAW.maint` — maintenance and incident log. Each entry has `t`, `machine`,
  `type` (Cleaning / Machine Down / Operator Fix / Defective Material), `notes`.
- `RAW.wait` — wait/downtime log. Each entry has `t`, `machine`, `dur` (seconds),
  `notes`, `op`.
- `RAW.oeeParams` — OEE parameters per machine: `stdDownMin` (standard planned
  downtime in minutes), `idealCycleSec` (ideal seconds per piece), `targetOEE`,
  `targetAvail`, `targetPerf`, `targetQual`.
- `RAW.shiftMin` — planned shift length in minutes (570 = 6:30 AM to 4:30 PM
  minus 30-min break).

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
  newer `tally2` variant instead of plain `tally`. Tally-family sessions lack
  a reliable `s` (active-run-seconds) field, but their `st`/`et` start/end
  span is a reasonably reliable proxy for real elapsed production time and
  is what actual-PPH is computed from for these sessions (`render()`'s main
  session-aggregation loop in index.html). Match tally-family modes with
  `m.indexOf('tally') === 0`, never an exact `=== 'tally'`, or a new tally
  variant will silently break PPH again the way `tally2` did.
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
