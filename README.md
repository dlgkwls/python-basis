# Wafer-Map Yield Visualizer

Paste or upload die-level CSV and instantly see the wafer: a color-coded bin map, overall
yield %, a fail-bin pareto, and an edge-vs-center yield split — in one zero-dependency HTML file.

![Wafer-Map Yield Visualizer — bin map, yield, pareto, and edge-vs-center split](assets/screenshot.png)

> **TODO (Hazel):** drop a screenshot at `assets/screenshot.png`. Load the **edge-ring** sample
> at seed 2026, then capture the full map + stats panel — it's the strongest single frame.

## Features

- **Reads raw die data** — paste a CSV or pick a `.csv` file; the parser validates every row,
  keeps the good ones, and tells you exactly which lines it skipped and why.
- **Answers the first three yield questions** — *what's the yield* (overall % to one decimal),
  *which bin is killing it* (a descending fail-bin pareto), and *is the failure spatial*
  (edge-vs-center yield split with a signed delta). Hover any die for its `(x, y) · bin · pass/fail`.
- **Reproducible synthetic wafers** — three built-in generators (edge ring, center cluster,
  random) driven by a seedable PRNG, so the same seed always draws the same wafer. Each one
  exports its CSV so you can re-paste or share it.

## What is a wafer map?

> **TODO (Hazel): rewrite this paragraph in your own words before publishing** — the PRD asks for
> it in your voice for interview authenticity. Draft below to react to:

A wafer is a disc of silicon holding hundreds or thousands of identical chips ("die"), laid out
on a grid. After fabrication, **wafer sort** tests each die in place and stamps it with a **bin**
number that says how it did: bin 1 means it passed, and higher bins record *how* it failed —
continuity (opens/shorts), leakage, parametric, or functional. **Yield** is simply the fraction
of die that passed (bin 1). A wafer map paints each die's bin as a colored square in its real grid
position, so the failure *pattern* jumps out: a ring of fails hugging the edge points at a
different root cause than a hot spot in the center. Reading that pattern is the daily work of a
yield engineer, and this tool renders it from raw die data.

## Data contract

Input is a comma-delimited, UTF-8 CSV with a **header row**:

```csv
die_x,die_y,bin
-3,0,1
-2,0,1
-1,0,3
0,0,1
1,0,1
```

- `die_x`, `die_y` — integer grid coordinates, origin at wafer center; negatives are valid.
- `bin` — integer. **1 = pass.** 2–5 are fail categories (2 = continuity · 3 = leakage ·
  4 = parametric · 5 = functional/other). Any bin ≥ 2 counts as a fail. Bins > 5 are accepted
  and drawn gray, labeled "other".
- Columns are matched **by header name** (case-insensitive, trimmed) in any order; extra columns
  are ignored.
- One row per tested die. Duplicate `(x, y)` → **last row wins**, with a warning showing the count.
- Bad rows (blank or non-integer fields) are skipped and logged as `line N: reason`; the rest
  still render. Missing a required column, zero valid rows, or more than 20,000 rows are blocked
  with a message.

**Formulas** — yield `= 100 · (bin-1 count) / (total die after dedupe)`; fail share per bin
`= 100 · bin_count / total_fails`; zone `r = sqrt(x²+y²) / R_data` where `R_data` is the largest
such distance in the data, edge if `r ≥ 0.85`; delta `= center_yield − edge_yield`.

## How the samples are generated

Every integer `(x, y)` with `sqrt(x² + y²) ≤ R` (R = 20, ~1,257 die) is a die. With
`r = sqrt(x² + y²) / R` (normalized 0–1), each generator sets a fail probability:

- **Edge ring** — `p_fail = 0.02 + 0.55 · max(0, (r − 0.75) / 0.25)²` (fails concentrate at the rim).
- **Center cluster** — `p_fail = 0.02 + 0.60 · exp(−(r / 0.25)²)` (a hot spot in the middle).
- **Random** — `p_fail = 0.05` everywhere.

On a fail, the bin is a weighted pick: 2 (40 %), 3 (30 %), 4 (20 %), 5 (10 %). A seedable
**mulberry32** PRNG (seed field defaults to `2026`) makes every wafer reproducible —
`Math.random()` is never used in the generators. Pre-exported CSVs live in [`samples/`](samples/).

## Run it

It's a single static file with no build step:

- **Locally:** just open `index.html` in a browser (works fully offline — the Google Fonts link
  degrades gracefully to system fonts if you're offline).
- **GitHub Pages:** push this repo, then in **Settings → Pages** serve from the `main` branch
  root. The live demo is `index.html` at the repo's Pages URL.

**Live demo:** _<add your GitHub Pages URL here>_

## Notes

Built with AI assistance; every formula and behavior hand-verified against the fixtures in the PRD.

## License

[MIT](LICENSE)
