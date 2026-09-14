# ⚡ Clean Bus Tool — KE Priority Feeder Reassignment Dashboard

**A single-file, browser-based engineering dashboard that proposes how to reassign 11 kV feeders onto each grid station's "clean bus", so priority customers, quality customers and the most fault-prone feeders end up on the same PTR.**

Upload the Mastersheet → get a grid-by-grid swap proposal in which no proposed swap may push a PTR past 93 % of its ampacity, plus a downloadable Excel proposal.

![Static](https://img.shields.io/badge/build-none%20needed-brightgreen)
![Single file](https://img.shields.io/badge/app-one%20index.html-blue)
![Runtime](https://img.shields.io/badge/runs-100%25%20in%20the%20browser-9cf)
![Excel](https://img.shields.io/badge/io-.xlsx%20via%20SheetJS-217346)
![Domain](https://img.shields.io/badge/domain-11%20kV%20Network%20Planning-E4002B)

---

## Overview

`index.html` is the entire application — HTML, CSS, data and logic in one file with **no build step, no backend, no installation and no data leaving the machine**. It reads a two-sheet Excel Mastersheet, detects the relevant columns by fuzzy header matching, scores every feeder on the grid, then searches for the set of swaps that concentrates the best feeders on a chosen **clean bus** while respecting a 93 % PTR loading ceiling on every swap it proposes. Results are shown as KPIs, a sortable grid table with expandable per-PTR detail, and are exportable to a four-sheet `.xlsx`.

It is built for K-Electric Network Planning & Engineering work on the 11 kV distribution network — 511 priority feeders across ~30 grids are handled in a single run.

---

## The problem

On a grid station, faults on one PTR bus affect every feeder connected to it. If priority (Industrial / Strategic) feeders, Platinum/Gold/Silver customers, and feeders with a history of cable and busbar faults are spread randomly across PTRs, a fault on any one bus takes out strategically important load.

The tool answers a narrow, practical question:

> *For each grid, which PTR should be the "clean bus", and exactly which feeders should be swapped in and out of it — without overloading any PTR — to maximise the value of the customers sitting on the clean bus while minimising fault exposure?*

The maintenance plan is modelled as an **80 % reduction in the fault contribution of priority feeders** (their 12-month trip / cable / busbar counts are de-rated to 20 % in the calculations and reporting).

---

## Features

| | |
|---|---|
| 📥 **Drag-and-drop input** | Reads a 2-sheet `.xlsx` / `.xls` entirely in the browser; nothing is uploaded anywhere. |
| 🧠 **Fuzzy column detection** | Finds columns by header name aliases and substring matching, then shows a ✓/✕ chip for every column it needs — no fixed template is enforced. |
| 🏷️ **Grid name canonisation** | Spelling/case/whitespace variants of the same grid are merged automatically (the most frequent spelling wins). |
| 🔌 **Ampacity fallback cascade** | Uses the provided PTR loading from Sheet 2; if a PTR is missing it falls back to the grid's most common ampacity, then estimates it from the power transformer rating (MVA → A at 11 kV). Estimated values are flagged with `~`. |
| 🎯 **Score-driven clean-bus selection** | Every PTR bus in the grid is evaluated as a candidate clean bus; the best-scoring outcome wins, with fewer swaps as the tie-breaker. |
| ⚖️ **Two-phase optimisation** | (1) *Quality swaps* to raise clean-bus score, (2) a *water-filling load-balance* pass that shifts load off the most heavily loaded PTRs onto lighter ones. Both phases respect the `≤ 93 %` ampacity ceiling when placing a swap. |
| 📊 **Live KPIs** | Grids shown, total swaps, priority feeders moved onto clean buses, Platinum moved, and the number of PTRs still above 93 %. All recomputed against the active filters. |
| 🔎 **Filters** | Multi-select grid dropdown plus Feeder ID and Feeder Name lookups (with autocomplete). |
| 🔬 **Drill-down detail** | Expand any grid for the before/after PTR fault-contribution stacked bars, a per-PTR customer & loading table with utilisation bars, and the full numbered swap list. |
| 📤 **Excel export** | `KE_Priority_Clean_Bus_Proposal.xlsx` with Grid_Summary, PTR_Customers_Loading, Swaps and New_Assignment sheets — filtered views are exported as shown. |
| 🖨️ **KE-branded UI** | K-Electric gradient styling, Plus Jakarta Sans / JetBrains Mono, responsive down to tablet width. |

---

## How it works

### 1. Parse and normalise the workbook

The two sheets are located by looking for headers containing `Grid` + `Grid and PTR` + `Peak Load` (feeders) and `Grid and PTR`/`Grid Station` + capacity (PTR loading). Every row is then normalised:

* Grid keys are trimmed, upper-cased and whitespace-collapsed, then mapped back to a canonical label.
* Each feeder is bound to a bus identified as `GRID#TRAFO` (bus label `GRID:TRAFO`). If the Trafo column is absent, the trailing number of the *Grid and PTR* text is used.
* **Priority feeders** are matched against a list of **511 feeder IDs embedded in the file**. Flags such as Platinum/Gold/Silver are read as 0/1 (accepting blanks, `0`, `No`, `False`).
* Trip, HT-cable-fault, busbar-fault counts of priority feeders are scaled by `0.20` to represent the −80 % maintenance effect.

### 2. Score every feeder

Each feeder gets a score, normalised per grid so the factors are comparable between grids:

| Factor | Weight | Notes |
|---|---|---|
| Priority feeder (embedded ID list) | **+60** | Industrial / Strategic |
| Platinum customer | **+100** | Highest weight — quality of supply |
| Gold customer | **+45** | |
| Silver customer | **+18** | |
| "Good" loss category (`IND`, `INDUSTRIAL`, `LL`) | **+15** | |
| 12-month trips | **−25** × (feeder ÷ grid max) | |
| HT cable faults | **−30** × (feeder ÷ grid max) | |
| S/S busbar faults | **−25** × (feeder ÷ grid max) | |
| Main cable length (m) | **−15** × (feeder ÷ grid max) | |
| Loss category `ML`, `VHL`, `HL` | **hard exclusion** | Never allowed onto the clean bus |

### 3. Pick the clean bus and make quality swaps

For every candidate bus in a grid, the optimiser repeatedly finds the single best swap: the lowest-scoring feeder currently on the candidate clean bus is exchanged with the highest-scoring eligible feeder elsewhere in the grid. A swap is only accepted if

* it improves the score by more than `MIN_GAIN` (0.5 points),
* the receiving bus stays within **93 % of its ampacity** (an already-overloaded bus is never made worse), and
* the donor bus does not become overloaded either.

The candidate that yields the highest total clean-bus score wins (tie-break: fewer swaps). Grids with a single bus are reported as "1 bus" and left unchanged. Up to `MAX_SWAPS` (500) quality swaps per grid.

### 4. Load balancing (water-filling)

A second phase redistributes load between PTRs. It moves a heavier feeder out of the more heavily loaded bus `X` in exchange for a lighter one out of bus `Y`, accepting the transfer only when it reduces the sum of squared shortfalls to the 93 % ceiling — so PTRs sitting above the limit are relieved and lightly loaded PTRs are filled, pushing the grid's utilisation toward a common level. Again, clean-bus quality is protected:

* a bad-loss-category feeder may never be moved onto the clean bus, and
* a swap that would lower the clean bus's total score is rejected.

The practical consequence: **load-balance swaps are sparse and only fire when they genuinely improve packing.** If relieving an overloaded PTR would mean putting a worse feeder on the clean bus, the optimiser leaves the overload in place rather than trading grid quality for it — which is exactly what the `PTRs > 93 %` KPI is there to surface for manual follow-up.

### 5. Report

Everything is rendered client-side: KPI cards, the sortable grid table (grid, clean bus, Q+L swaps, priority pre→post, P·G·S counts, clean-bus fault share pre→post, max loading %) and the expandable per-grid detail. Exports use the active filters.

---

## Input format

A single workbook. Sheet names are irrelevant — sheets are detected by their headers.

### Sheet 1 — Feeders (required)

| Column | Used for |
|---|---|
| `Grid` | Grid grouping *(required)* |
| `Grid and PTR` | Bus assignment *(required)* |
| `Peak Load (Amp)` | Feeder loading *(required)* |
| `Feeder ID` | Priority matching (against the 511 embedded IDs) |
| `Feeder name` | Display, filtering, export |
| `Trafo. No.` | PTR number (fallback: trailing digits of *Grid and PTR*) |
| `Platinum` / `Gold` / `Silver` | Customer-tier weights |
| `Total Trips (Faults)` | Fault history penalty |
| `HT Cable Faults` | Fault history penalty |
| `S/S Busbar Fault` | Fault history penalty |
| `MC Length (m)` | Cable-length penalty |
| `Power Trafo Rating` | Ampacity estimation fallback |
| `Loss Category` | Good/bad category scoring (optional but recommended) |

### Sheet 2 — PTR loading / ampacity

| Column | Used for |
|---|---|
| `Grid Station` (or `Grid`) + `Trafo. No.` — or `Grid and PTR` | Bus key |
| `Provided Loading`, `PTR Ampacity`, `Capacity (A)`, `Rated Loading`, … | The PTR ampacity taken as-is; the 93 % cap is derived from it |

Column matching is case-, space- and punctuation-insensitive (`Peak Load (Amp)`, `peak_load_amp` and `PEAK LOAD` all resolve to the same column), and falls back to substring matching. A detection strip under the drop zone shows which columns were recognised and warns when PTR ampacity had to be estimated.

---

## Output

### On screen

* **Priority banner** — priority feeders in scope and the grids they live in.
* **KPI row** — grids shown, swaps (quality + load), priority feeders added to clean buses, Platinum added, PTRs > 93 % after.
* **Grid-wise proposal table** — sortable on every data column; red/amber/green pills for post-swap loading.
* **Expandable detail** — PTR fault contribution before/after, per-PTR campaign table (Platinum/Gold/Silver, fault % pre→post, load, ampacity, 93 % cap, utilisation bar) and the numbered swap list (`quality` vs `load`, `★` flag for priority feeders).

### Exported workbook — `KE_Priority_Clean_Bus_Proposal.xlsx`

| Sheet | Contents |
|---|---|
| `Grid_Summary` | One row per grid: clean bus, quality/load swap counts, priority + Platinum/Gold/Silver pre/post, clean-bus fault share pre/post, max load %, PTRs > 93 %, 12-month grid faults |
| `PTR_Customers_Loading` | One row per PTR: tiers, fault % pre/post, load, ampacity, 93 % cap, loading %, `OK` / `OVER 93 %` status |
| `Swaps` | Every proposed swap: type, out/in feeder with IDs and priority flags, amperages, context (source bus or exchange pair) |
| `New_Assignment` | Feeder-level before/after bus assignment with `Changed` and `On clean` markers |

---

## Getting started

**Use it**

1. Download or clone this repository.
2. Open `index.html` in any modern browser (double-click works — no server required).
3. Drop the Mastersheet onto the drop zone, wait for the column chips to appear, then click **Run proposal**.
4. Review the grid table and export the proposal.

**Serve it locally (optional)**

```bash
git clone https://github.com/WaizaSiddiqui/Clean-Bus-Tool.git
cd Clean-Bus-Tool

python3 -m http.server 8000      # or: npx serve .
# open http://localhost:8000/index.html
```

**Publish it as a GitHub Page**

`Settings → Pages → Build and deployment → Source: Deploy from a branch → Branch: main, Folder: / (root) → Save`.
The dashboard is then live at `https://waizasiddiqui.github.io/Clean-Bus-Tool/`.

**Dependencies** (loaded from CDN, nothing to install): SheetJS `xlsx.full.min.js` 0.18.5 from cdnjs, and the Plus Jakarta Sans / JetBrains Mono web fonts from Google Fonts. Only SheetJS is functionally required — fonts degrade gracefully.

---

## Configuration

All tunables live in one block near the top of the `<script>` in `index.html`:

```js
const PRIORITY = new Set([ ... ]);          // 511 priority feeder IDs

const W = { priority:60, plat:100, gold:45, silver:18,
            goodLoss:15, trips:25, cable:30, busbar:25, mcLen:15 };

const FAULT_FACTOR = 0.20;    // 20% of faults remain → −80% after maintenance
const MIN_GAIN     = 0.5;     // minimum score gain to accept a quality swap
const MAX_SWAPS    = 500;     // quality-swap cap per grid
const LOAD_LIMIT   = 0.93;    // PTR loading ceiling (93% of ampacity)

const BAD_LOSS  = new Set(['ML','VHL','HL']);   // never placed on the clean bus
const GOOD_LOSS = new Set(['IND','INDUSTRIAL','LL']);
```

Changing `LOAD_LIMIT` updates both the optimiser constraint and every "93 %" figure in the UI and export. To use a different priority list or a different set of loss categories, edit the two sets — nothing else needs to change.

---

## Assumptions & limitations

* **Ampacity is taken as provided.** When Sheet 2 has no entry for a PTR, ampacity is inferred (grid's most common value, else `MVA × 10⁶ / (√3 × 11 kV)`), and those PTRs are flagged `~` in the detail table and left blank in the export. Engineering review is required for estimated values.
* **The 93 % limit constrains proposals, it does not repair the input.** A swap is never accepted if it would push a PTR past 93 % (or increase the load on a PTR already above it), and the load-balance pass relieves existing overloads where it can — but a PTR that arrives above 93 % may still finish above 93 %. Those PTRs are counted in the `PTRs > 93 %` KPI and flagged `OVER 93 %` in the export, and need separate engineering action.
* **The optimiser is a greedy heuristic**, not a global optimum — it evaluates each bus as a candidate clean bus and iteratively applies the best single swap. It is deliberately fast and explainable; results are proposals, not switching orders.
* **Single-bus grids** cannot be improved by swapping and are reported as "1 bus" with no proposal.
* **Fault reduction is a planning assumption** (−80 % on priority feeders via `FAULT_FACTOR`), applied so that the proposal reflects the post-maintenance state; it is not a measured outcome.
* **Feeder prioritisation is ID-based.** Feeders missing a `Feeder ID`, or IDs not in the embedded list, are treated as non-priority.
* The workbook is parsed in memory only; the tool performs no validation of data quality beyond column detection, so dirty source data will produce dirty proposals.

> **Decision support only.** Proposed reassignments must be checked against protection and coordination settings, earthing, physical cable routing, switching constraints and field verification before implementation, and approved by the relevant Network Planning & Engineering authority.

---

## Repository structure

```
Clean-Bus-Tool/
├── index.html   # the entire application: UI, styles, input parsing, optimiser, reporting, export
└── README.md    # this document
```

---

## Roadmap ideas

- [ ] Load the priority feeder list from a file / editable on-screen list instead of a hard-coded array
- [ ] Split the monolith into `app.js` / `styles.css` (or a small module structure) with the optimiser unit-tested against sample workbooks
- [ ] Configurable weights and load limit exposed in the UI
- [ ] Consider protection/coordination feasibility signals and manual swap overrides
- [ ] Persist a run (inputs + proposal) for audit and side-by-side comparison of scenarios
- [ ] Bundle SheetJS locally so the dashboard works fully offline

---

## License

No license file is included yet. Until one is added, the code is **all rights reserved** to the author — add a `LICENSE` file (e.g. MIT for open source, or an internal/proprietary notice) to make the terms explicit.

---

*Built for K-Electric · Network Planning & Engineering · 11 kV feeder reassignment planning.*
