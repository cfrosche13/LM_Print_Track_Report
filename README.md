# Print Track Daily Report

A standalone production report for Evergreen Studio, built from Firebase data exported from the Print Track app.

## Quick start

1. Open `index.html` in any browser — no server needed.
2. Use the date picker to navigate to any production day.
3. Switch to the **Totals** tab for a monthly calendar overview.

## Refreshing the data

Data is embedded directly in `index.html`. To update it:

1. Export the relevant nodes from Firebase (`sessions`, `targets`, `maint`, `wait`, `tallyClicks`, `oeeParams`, `shiftMin`).
2. Replace the `var RAW = {...}` object near the top of the `<script>` block.
3. Update the `DAILY` pre-aggregated object to match.
4. Update the date picker `min`/`max` values and any hardcoded quick-jump buttons.

See `CLAUDE.md` for full project context and data structure details.

## Related projects

- **LM_Production_Tracker** — the Netlify/Firebase source app
- **Claude_Code_LM** — the Print Log / EGStudio PrintLog app
