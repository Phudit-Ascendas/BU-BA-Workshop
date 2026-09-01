# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Teaching material, not an application: a MATLAB workshop for *MG 410 Business Analytics for Strategic
Decisions*, built on one fictional Thai retailer (**Brighten Up Mart** — 3 distribution centres, 5
stores, 48 months of history). Everything is deliberately low-code — Live Tasks, the Regression Learner
app, the Optimize Live Task — so the audience reads results rather than writes algorithms. Keep it that
way: refactoring a Live Task into hand-written code destroys the point of the lesson.

`README.md` is the participant-facing description of the scenario and the expected numbers. It is the
source of truth for *pedagogy*; this file covers *mechanics*.

## Environment

MATLAB **R2026a** (verified installed). Required toolboxes: **Statistics and Machine Learning**
(`poissrnd`, Regression Learner, the exported model) and **Optimization** (`optimvar`, `optimproblem`,
`solve`/linprog, Optimize Live Task).

There is no build, no lint, and no test suite. "Running" means opening the artifacts in MATLAB:

```matlab
open BUBAWorkshop.prj                        % open FIRST — sets the project path
open LowCode_MATLAB_Business_Analytics.mlx   % run sections top to bottom
BrightenUpMartDashboardApp                   % dashboard alone
```

Scripts resolve `data/` **relative to the current folder**, so the project (or a `cd` to the repo root)
must be in place before running anything.

Use the MATLAB MCP tools to execute or check code — `evaluate_matlab_code`, `run_matlab_file`,
`check_matlab_code`. They drive the user's visible MATLAB desktop; figures appear there, not here.

## Everything is a binary MATLAB file

`.mlx`, `.mlapp` and `.mat` are marked `binary` in `.gitattributes` — git shows no diffs and merges are
not possible. Consequences:

- **Never** edit `.mlx`/`.mlapp` with text tools. Edit them in MATLAB (Live Editor / App Designer).
- To *read* one, it is a zip: extract it and strip the tags from `matlab/document.xml`. Do this in the
  scratchpad, never in the repo.

  ```bash
  unzip -q -d "$SCRATCH/x" LowCode_MATLAB_Business_Analytics.mlx
  sed -e 's|<w:p>|\n|g' -e 's|<[^>]*>||g' -e 's|<!\[CDATA\[||g' -e 's|\]\]>||g' "$SCRATCH/x/matlab/document.xml"
  ```

- Embedded **Live Tasks** (Import Data, Optimize) do not appear in that extraction — sections that look
  blank in the text are Live Tasks. Their generated code only exists once the task is expanded in MATLAB.
- `archive/` holds superseded `.mlapp` backups and an earlier plain-`.m` port of the dashboard
  (`archive/BrightenUpMartDashboard.m`). Useful for reading the app's logic as plain text, but it is
  **not** kept in sync with `BrightenUpMartDashboardApp.mlapp` — treat the `.mlapp` as authoritative.

## The one contract that ties the repo together

The live script and the dashboard are independent implementations of the *same* pipeline. Any change to
one is a change to a shared, undocumented contract:

1. **Feature schema.** `RbLin.predictFcn` accepts a one-row table with exactly
   `{MonthOfYear, LastMonth, SameMonthLastYear, MarketingSpend}` (that order, those names) and returns
   `Sales`. `data/BrightenUpMartSales_Features.csv` is the training form of the same schema; the
   Appendix of the solution script shows how it is derived from the raw CSV by `lag1`/`lag12`.
2. **Recursive forecast.** Each prediction feeds back as the next month's `LastMonth`.
   `SameMonthLastYear` is read from a growing `[history; forecast]` series in the app — that is what
   allows horizons past 12 months. The live script reads only `BUS.Sales(end-12+h)`, so it is correct
   only up to a 12-month horizon.
3. **Marketing-spend model.** `poissrnd(mode(BUS.MarketingSpend))` per month, with the focus month
   overwritten by `round(mode + 2*std)` — the promotional spike. `rng(7)` is fixed so a demo reproduces
   the same numbers; the app caches the draw over 24 months so changing the horizon does not reshuffle
   months already on screen.
4. **Forecast → network sizing.** The focus/shipping month's forecast *is* total demand. It is split by
   `storeShare = [0.265 0.217 0.181 0.169 0.168]`; supply is `ceil(totalDemand*1.03)` split by
   `dcShare = [0.47 0.29 0.24]`. The rounding remainder is pushed onto the last store / first DC so the
   totals stay exact.
5. **LP.** `ship` is 3×5, lower bound 0; minimise `sum(sum(cost.*ship))` subject to
   `sum(ship,2) <= supply` and `sum(ship,1)' == demand`. The teaching payoff is the comparison against
   the greedy first-come-first-served `alloc` benchmark.

Scenario constants — the `1e2*[20 6 9 25 15; 7 20 23 37 18; 37 25 26 7 28]` cost matrix, the two share
vectors, the 1.03 supply factor, `rng(7)` — are **duplicated** in the live script and in
`BrightenUpMartDashboardApp.startupFcn`. Change one and the other must change too, or the dashboard
silently contradicts the script it is meant to summarise. The newer solution draft moves these into
`data/TransportOpt.xlsx` (sheets `Shipping Cost`, `Supply Share`, `Demand Share`) — that is the direction
of travel, but the taught script and the app still hardcode them.

## Dashboard app structure

`BrightenUpMartDashboardApp.mlapp` is an App Designer UIFigure app, grid-layout throughout. The
data-flow spine is worth knowing before editing:

- `startupFcn` loads the CSV + model, sets the scenario constants, creates the heatmap **at runtime**
  (`heatmap` is not an App Designer component), then calls `refreshForecast`.
- `refreshForecast` → `runForecast` → `drawForecast` → `syncShippingMonths` → `applyShippingMonth(false)`
  → `solveTransportation`. Every control re-enters at `refreshForecast` or `applyShippingMonth`, so the
  shipping plan can never disagree with the forecast above it.
- `applyShippingMonth(resetCosts)` is the seam that makes this survivable in a live demo: `false`
  re-derives demand and supply but **preserves the user's cost edits**; `true` (the *Reload from
  forecast* button) also restores the scenario costs.
- Infeasible input routes to `setInfeasible` — red lamp and a message, never an exception, so a mistyped
  cell mid-demo is recoverable.
- `drawForecast` must `delete(findall(...,'Type','constantline'))` after `cla`: the focus-month `xline`
  is created with `HandleVisibility','off'`, so `cla` leaves it behind and markers accumulate on redraw.

## Known traps

- **Model filename casing.** The file on disk is `data/RbLin.mat`; the app loads `'Rblin.mat'` and the
  README says `Rblin.mat`. This only works because Windows is case-insensitive — it breaks on Linux or
  MATLAB Online. The variable inside is a struct named `RbLin` with a `predictFcn` field (the app reads
  it via `fieldnames`, so the variable name is not load-bearing there).
- **Hardcoded path** in the current solution draft:
  `readtable(".\Support\BU\BA Workshop\data\BrightenUpMartSales_Features.csv")`. It should be
  `"data/BrightenUpMartSales_Features.csv"`, as it is in draft1. Fix before teaching from that file.
- **File naming.** `LowCode_MATLAB_Business_Analytics.mlx` is the *taught* version — sections are
  deliberately left empty for participants to fill in during the workshop. The
  `Solution_..._draftN.mlx` files are the completed walkthroughs. Do not fill in the taught script's
  blanks; worked code belongs in a solution file.
- `resources/project/` is MATLAB Project metadata (opaque XML, LF line endings enforced by
  `.gitattributes`). MATLAB rewrites it on its own — commit the churn, do not hand-edit it.
