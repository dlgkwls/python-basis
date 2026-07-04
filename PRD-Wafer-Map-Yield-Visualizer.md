# PRD — Wafer-Map Yield Visualizer

| | |
|---|---|
| **Project** | `wafer-map-viz` — vibecoded app #1 (the "decently easy" one) |
| **Owner** | Hazel Lee |
| **Build window** | Kickoff Fri **Jun 12** (~1 h: spec + fixtures) → build Sat **Jun 13** & Sun **Jun 14**, 2026 (~8 h) · ships **Jun 14**, launches with the portfolio Jun 15 |
| **Repo** | `wafer-map-viz` (public) · live demo via GitHub Pages |
| **Status** | Spec frozen — build against this document |

---

## 1. One-liner

Paste or upload a die-level CSV and instantly see the wafer: a color-coded bin map, overall yield %, a fail-bin pareto, and an edge-vs-center yield split — in one zero-dependency HTML file.

**Elevator pitch (for the README and portfolio card):** Wafer maps are the iconic visual of semiconductor yield engineering — every test, product, and yield engineer reads them daily. This tool renders one from raw die data and answers the first three questions an engineer asks: *what's the yield, which bin is killing it, and is the failure spatial?*

## 2. Background, goals, non-goals

**Why it exists.** Hazel is targeting yield / product / test / FA roles. This app (a) makes her portfolio instantly legible to those recruiters, (b) teaches bin/yield/wafer-sort vocabulary two months before the test-literacy block, and (c) is the rehearsal for vibecoded app #2.

**Goals:** a recruiter clicks the live demo and *gets it* in 10 seconds; Hazel can explain every formula and every line of behavior; total build ≤ 9.5 hrs.

**Non-goals (explicitly out of scope for v1):** real STDF/wafer-sort file formats, multi-wafer lots, zoom/pan, reticle/shot overlays, defect-image anything, server or database, login, dark mode.

## 3. Users & use cases

1. **Recruiter / hiring manager (primary):** opens the live demo, clicks "Load sample → edge-ring," sees the map + stats. Must require zero instructions.
2. **Hazel (secondary):** pastes her own generated CSVs, screenshots results for the portfolio and interviews.
3. **Curious engineer (tertiary):** opens the code; should find readable, commented vanilla JS.

## 4. Constraints

- **One file:** `index.html`. Inline CSS + JS. No frameworks, no CDNs, no build step, works offline and on GitHub Pages. Target < 120 KB.
- **Vanilla JS + SVG** for the wafer and pareto (SVG keeps die inspectable in devtools and crisp on retina). Canvas acceptable only if SVG performance fails at 2,000 die (it shouldn't).
- **Responsive:** usable at 380 px wide (stats stack under the map). No horizontal scroll.
- **No storage APIs needed.** Session-only state.
- **Browser support:** current Chrome/Safari/Firefox. No IE consideration.

**Design tokens (Hazel's lab-notebook system — use exactly):**

| Token | Value | Use |
|---|---|---|
| paper | `#f3ede2` | page background |
| card | `#fbf7ef` | panels |
| ink / ink-soft / ink-faint | `#1c1813` / `#574d42` / `#8a7e6f` | text |
| line | `#d9cdba` | borders, gridlines |
| accent (blueprint) | `#41637f` | headers, buttons, links |
| pass (sage) | `#7e9472` | bin 1 die |
| fail bins 2–5 | `#cf3a26` / `#c0740b` / `#b8860b` / `#b8502e` | signal-red, amber, gold, clay |
| Fonts | Fraunces (display) · Hanken Grotesk (body) · JetBrains Mono (numbers, labels) | Google Fonts `<link>` is allowed; must degrade gracefully offline |
| Color scheme | **light only** (`color-scheme: only light`) | |

## 5. Data contract

**Input CSV** — header row required, comma-delimited, UTF-8:

```csv
die_x,die_y,bin
-3,0,1
-2,0,1
-1,0,3
0,0,1
1,0,1
```

- `die_x`, `die_y`: integers, die-grid coordinates, origin at wafer center. Negative values valid.
- `bin`: integer. **1 = pass.** 2–5 = fail categories (nominal labels: 2 = continuity (open/short) · 3 = leakage · 4 = parametric · 5 = functional/other). Any bin ≥ 2 counts as fail. Bins > 5 are accepted and colored gray with label "other".
- Every row = one tested die. Duplicate `(x, y)` → **last row wins**, and a warning is shown with the count of overwritten die.
- Extra columns: ignored silently. Column order: any (match by header name, case-insensitive, trimmed).

**Acceptance fixture A (paste-able, known answers):**

```csv
die_x,die_y,bin
0,0,1
1,0,1
0,1,2
1,1,1
2,0,1
```

Expected: total die **5**, pass **4**, **yield 80.0 %**, pareto shows bin 2 = 1 (100 % of fails), bin legend counts {1: 4, 2: 1}.

## 6. Synthetic sample generators (built into the app)

A "Load sample" control with three options. Generator parameters:

- **Wafer geometry:** include every integer (x, y) with `sqrt(x² + y²) ≤ R`, **R = 20** → ~1,257 die. Let `r = sqrt(x² + y²) / R` (normalized 0–1).
- **Edge ring:** `p_fail = 0.02 + 0.55 · max(0, (r − 0.75) / 0.25)²` — failures concentrate in the outer ring. Expected overall yield roughly 88–93 %.
- **Center cluster:** `p_fail = 0.02 + 0.60 · exp(−(r / 0.25)²)` — a hot spot at the center.
- **Random:** `p_fail = 0.05` everywhere.
- On fail, assign bin by weighted pick: 2 (40 %), 3 (30 %), 4 (20 %), 5 (10 %).
- **Seedable PRNG required** (e.g., mulberry32) with a visible seed field defaulting to `2026`, so results are reproducible — this is what makes the verification plan possible. `Math.random()` is forbidden in generators.
- Each generator must also **export its CSV** (download link) so the same data can be re-pasted or shared.

## 7. Functional requirements

| ID | Requirement | Acceptance criterion |
|---|---|---|
| FR-1 | CSV input via paste-textarea **and** file picker (`.csv`) | Fixture A via either path renders identically |
| FR-2 | Parser validates per §5; bad rows collected, good rows kept | A file with 1 bad row renders the rest + shows "1 row skipped (line 4: non-integer bin)" |
| FR-3 | Wafer render: one SVG square per die, centered circle outline, bin-colored per tokens | Fixture A shows 5 squares, one signal-red |
| FR-4 | Stats panel: total die, pass die, **yield % to 1 decimal**, per-bin counts with color legend | Fixture A → 80.0 % |
| FR-5 | **Pareto** of fail bins: horizontal bars, descending count, each labeled `bin · count · % of fails` | Edge-ring sample → bin 2 is the top bar at ~40 % of fails |
| FR-6 | **Edge vs center split:** edge = die with `r ≥ 0.85` (outer 15 % of radius ≈ outer 28 % of area — say so in the UI tooltip); show yield for each zone side-by-side with the delta | Edge-ring sample → edge yield at least 15 points below center; random sample → zones within ~3 points |
| FR-7 | **Hover tooltip** on any die: `(x, y) · bin n · pass/fail` | Hovering the red die in Fixture A shows `(0, 1) · bin 2 · fail` |
| FR-8 | Sample menu per §6 incl. seed field + CSV export | Seed 2026 twice → identical maps |
| FR-9 | Empty state: short explainer + "Load sample" button + the CSV schema | First load shows it; no console errors |
| FR-10 | Reset/clear control returns to empty state | One click |
| FR-11 | **Inline info icons:** a click-to-open popover next to seed, yield, bin legend, pareto, and edge-vs-center, each with a short concept explanation | Clicking each icon opens its popover with the correct text; click-outside / Escape / re-click closes it; default view is unchanged; no clash with the FR-7 die hover tooltip |

## 8. Math appendix (exact definitions)

- **Yield** `= 100 · (count of bin 1) / (total rows after dedupe)` — display 1 decimal.
- **Fail share per bin** `= 100 · bin_count / total_fails`.
- **Zone assignment:** `r = sqrt(x² + y²) / R_data` where `R_data = max r over all die` (data-driven, so partial maps still work). Edge if `r ≥ 0.85`.
- **Delta** `= center_yield − edge_yield`, signed, 1 decimal, labeled "center − edge".

## 9. UI spec

```
┌──────────────────────────────────────────────────────────┐
│ WAFER-MAP YIELD VISUALIZER          [Load sample ▾] seed │  header strip, blueprint accent
├───────────────┬──────────────────────────────────────────┤
│  INPUT        │            WAFER MAP (SVG)               │
│  paste box    │         circle + die grid                │
│  [file]       │                                          │
│  [Render]     ├──────────────────────────────────────────┤
│  [Clear]      │  YIELD 87.4 %   die 1257   pass 1099     │  mono numerals
│  parse log    │  [bin legend]  [pareto]  [edge vs ctr]   │
└───────────────┴──────────────────────────────────────────┘
```

- Stats use JetBrains Mono; the yield figure is the visual hero (large Fraunces numeral).
- Copy is plain and active-voice: "Paste die-level CSV", "Render map", "1 row skipped".
- Mobile (< 700 px): input collapses into a details/accordion above the map; stats stack.
- States: **empty** (explainer + schema + sample button) · **loaded** · **error** (parse log panel, clay border).
- Muted info icons sit next to the seed, yield, bin-legend, pareto, and edge-vs-center labels; their explanations are click-to-reveal (hidden by default), so the 10-second legibility of the default view is preserved.

## 10. Error handling

| Case | Behavior |
|---|---|
| Missing required header | Block render; message names the missing column(s) |
| Non-integer / blank field | Skip row; log `line N: reason`; continue |
| Duplicate (x, y) | Last wins; warn with count |
| 0 valid rows | Error state, schema reminder |
| > 20,000 rows | Refuse with message (keeps SVG responsive) |
| Empty paste + Render | Gentle prompt, not an error |

## 11. Milestones (mapped to the roadmap)

- **M1 — Fri Jun 12 (~1 h: commit this PRD, lock the spec, generate fixtures) + Sat Jun 13 (~4 h):** FR-1→FR-4 + FR-8 generators + FR-9/10. *Demo exists.*
- **M2 — Sat Jun 13, ~2.5 h:** FR-5, FR-6, FR-7.
- **M3 — Sun Jun 14, ~2 h:** error-handling table, mobile pass, README, deploy to Pages, portfolio card. **Ship.**

## 12. Verification plan (Hazel runs by hand — no coding)

1. Paste **Fixture A** → confirm 80.0 %, counts {1:4, 2:1}, red die at (0,1), tooltip text exact.
2. Seed `2026` → load each sample twice → maps pixel-identical.
3. Edge-ring sample → confirm edge yield < center yield by ≥ 15 points; pareto ordering descending.
4. Break it on purpose: bad row, missing header, empty file, 25k-row file → behaviors match §10.
5. Phone test at 380 px: no horizontal scroll, all stats readable.
6. Click each of the 5 info icons: popover text correct, only one open at a time, closes on outside-click / Escape, works at 380 px, and does not interfere with hovering a die.

## 13. README requirements

Screenshot at top → 3 feature bullets → "What is a wafer map?" paragraph **in Hazel's own words** (bins, wafer sort, yield) → data contract → how the samples are generated (formulas from §6) → *"Built with AI assistance; every formula and behavior hand-verified against the fixtures in the PRD"* → live-demo link → MIT license.

- One bullet: the app includes a small inline glossary — click the info icons beside the key labels for a plain-English explanation of each concept.

## 14. Definition of done

All FR acceptance criteria pass · §12 checklist complete · deployed on Pages · portfolio card live with screenshot + demo + code links · README per §13 · repo contains `index.html`, `README.md`, `samples/` (3 exported CSVs), and this PRD.

- FR-11 info icons present and verified.

## 15. Kickoff prompt — copy-paste to start the build

> Upload this PRD file with the message below.

```
You are my coding partner for a small zero-dependency web app. I've attached the full PRD
(PRD-Wafer-Map-Yield-Visualizer.md). Read it completely before writing any code. Then:

1. Confirm the data contract back to me in 3 lines, and list any ambiguities as questions
   (max 5). If there are none, say "no blockers."
2. Give me a build plan mapped to milestones M1–M3 (1–2 sentences each).
3. Then produce MILESTONE M1 ONLY: one complete, self-contained index.html implementing
   FR-1, FR-2, FR-3, FR-4, FR-8, FR-9 and FR-10, styled exactly with the PRD §4 design
   tokens, with the three seedable sample generators from §6.

Standing rules for this whole project:
- ONE file: index.html. Vanilla JS + SVG. No frameworks, no CDNs (Google Fonts link only),
  no build step. It must work offline and on GitHub Pages.
- Follow the PRD's formulas and fixtures exactly. Where the PRD states a known answer
  (Fixture A = 80.0 % yield), your code must reproduce it.
- After the code, give me a "VERIFY THIS" checklist of 5 hand-checks I can run in the
  browser for this milestone.
- If you deviate from the PRD for any reason, add a "DEVIATIONS" note explaining why.
  Never silently improvise.
- I'm a near-beginner: when something non-obvious appears in the code, explain it in one
  line, not an essay.
```

**Resume prompt (later sessions):** *"Continuing wafer-map-viz. PRD attached, current index.html attached. Last completed: M[n] / FR-[list]. Verify-this results: [paste]. Next: M[n+1] — produce the full updated single file, then the new VERIFY THIS checklist. Same standing rules."*

## 16. v1.5 stretch (only if time remains after §14)

PNG export of the map · bin-filter toggles (click legend to isolate a bin) · notch/orientation marker · yield-by-radius sparkline.
