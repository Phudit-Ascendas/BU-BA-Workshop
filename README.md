# Brighten Up Mart — Business Analytics Workshop

[![Open in MATLAB Online](https://www.mathworks.com/images/responsive/global/open-in-matlab-online.svg)](https://matlab.mathworks.com/open/github/v1?repo=Phudit-Ascendas/BU-BA-Workshop&project=BUBAWorkshop.prj&file=LowCode_MATLAB_Business_Analytics.mlx)

A hands-on MATLAB workshop for **MG 410 Business Analytics for Strategic Decisions**, built around one
fictional Thai retail chain, *Brighten Up Mart*: three regional distribution centres, five stores, and
four years of monthly sales history.

The workshop answers two connected business questions with low-code MATLAB tools, then hands both to an
interactive dashboard:

1. **How much will we sell?** — machine-learning demand forecasting.
2. **What is the cheapest way to ship it?** — least-cost transportation optimization.

No algorithms are written from scratch. Every technique is one MATLAB function, Live Task, or app.

---

## Contents

| File | What it is |
|---|---|
| `LowCode_MATLAB_Business_Analytics.mlx` | **Main live script** — the taught workshop, run top to bottom |
| `BrightenUpMartDashboardApp.mlapp` | **Dashboard** — the interactive App Designer app the workshop ends on |
| `BUBAWorkshop.prj` | MATLAB Project; open this first so paths resolve |
| `data/BrightenUpMartSales.csv` | 48 months of `Month, Sales, MarketingSpend` (Jan 2022 – Dec 2025) |
| `data/BrightenUpMartSales_Features.csv` | The same history reshaped as supervised-learning features |
| `data/Rblin.mat` | Trained robust-linear model exported from Regression Learner (`RbLin.predictFcn`) |

## Requirements

- MATLAB **R2026a**
- **Statistics and Machine Learning Toolbox** — `poissrnd`, Regression Learner, the exported model
- **Optimization Toolbox** — `optimvar`, `optimproblem`, `solve` (linprog), the Optimize Live Task

## Getting started

```matlab
open BUBAWorkshop.prj                      % sets the project path
open LowCode_MATLAB_Business_Analytics.mlx  % run sections top to bottom
```

The live script's final line opens the dashboard, so running it through takes you from raw data to the
finished tool. To skip straight to the dashboard:

```matlab
BrightenUpMartDashboardApp
```

Both resolve `data/` relative to their own location, so open the `.prj` (or `cd` into this folder) rather
than running the files from elsewhere.

## The live script

### The data
`BrightenUpMartSales.csv` is read as a **timetable** via an Import Data Live Task, then plotted two ways:
`Sales` against `Month` to expose the trend and the seasonal peaks (Songkran in April, the year-end
sale), and a scatter of `Sales` against `MarketingSpend` to motivate marketing as a predictor.

### Part 1 — Machine learning for forecasting
The problem is framed as *features in, sales out*. Each row carries four predictors:

| Feature | Meaning |
|---|---|
| `MonthOfYear` | 1–12, captures seasonality |
| `LastMonth` | previous month's sales |
| `SameMonthLastYear` | the same month one year earlier |
| `MarketingSpend` | budget planned for that month |

The model is trained in the **Regression Learner app** (point-and-click, no training code) and exported
to `data/Rblin.mat` as a robust linear fit. Two **interactive controls** drive the forecast — edit them
in place and re-run:

```matlab
targetm  = 6;   % how many months ahead to forecast
focusmkt = 4;   % which of those months gets the marketing push
```

Planned marketing spend is drawn as `poissrnd(mode(BUS.MarketingSpend))` per month, and the focus month
is replaced by `round(mode + 2*std)` — the promotional spike. The forecast then runs **recursively**:
each prediction feeds back as the next month's `LastMonth`.

With the default settings the forecast reads:

| Month | Forecast sales | Planned spend |
|---|---|---|
| Jan 2026 | 794 | 37 |
| Feb 2026 | 703 | 35 |
| Mar 2026 | 730 | 36 |
| **Apr 2026** | **952** | **111**  ← focus month |
| May 2026 | 868 | 42 |
| Jun 2026 | 851 | 35 |

### Part 2 — Transportation optimization
The focus month's forecast becomes that month's total demand. It is split across the five stores by their
usual share of sales, and DC capacity is set at 103% of that total, so **the distribution plan is driven
by the forecast rather than a fixed guess**.

Shipping cost rises with distance — Bangkok DC is nearest the Central Store, Chiang Mai DC the North
Store, Hat Yai DC the South Store. The decision is stated in business terms with the **Optimize Live
Task** and handed to a linear-programming solver:

- **Minimise** total shipping cost
- **Subject to** no DC shipping beyond capacity, and every store's demand met exactly

For April 2026 that is 952 units, an optimal cost of **THB 876,200**, against **THB 1,925,300** for a
naive first-come-first-served allocation that ignores cost — a saving of about **THB 1,049,100**. The
annotated bar chart shows the gap, with no change to demand, supply, or prices.

### Part 3 — From analysis to a tool
The closing section makes the business-intelligence point: a forecast and an optimal plan only matter if
decision-makers see them in time to act. It opens the dashboard.

---

## The dashboard

`BrightenUpMartDashboardApp` puts both halves of the workshop on one screen and makes the assumptions
editable, so the analysis can be re-run live in front of an audience.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  Brighten Up Mart — Business Analytics Dashboard       [New random draw] │
├───────────────────────────────┬──────────────────────────────────────────┤
│ 1.  SALES FORECAST            │ 2.  TRANSPORTATION OPTIMIZATION          │
│ Horizon [6▲▼]  View [ ▼ ]     │ Shipping month [ ▼ ]  [Reload]           │
│ Focus month [ ▼ ] [Reset]     │  ┌────────────────────────────────────┐  │
│                               │  │ editable tableau: costs + Supply   │  │
│   blue  ─o─  actual sales     │  │ column + Demand row                │  │
│   red   ─◆─  forecast         │  ├────────────────────────────────────┤  │
│   grey  ┆    focus month      │  │ heatmap of the optimal ship plan   │  │
│                               │  └────────────────────────────────────┘  │
│                               │  ● Optimal   Naive   Savings             │
└───────────────────────────────┴──────────────────────────────────────────┘
```

### Component 1 — Sales forecast

- **Forecast horizon** (1–24 months) re-runs the recursive forecast.
- **View** switches between two framings of the same numbers:
  - *Full history + forecast* — all 48 months, forecast appended (blue line/circles for actuals, red
    line/diamonds for the forecast).
  - *Year over year + forecast* — one Jan–Dec axis comparing the current year, the previous year, and
    the forecast. A horizon past 12 months spans more than one calendar year, so each forecast year gets
    its own line (2026 solid, 2027 dark dashed) instead of overwriting the axis.
- **Focus marketing month** applies the same promotional push as the live script — the chosen month's
  spend is replaced by `round(mode + 2*std)`. It defaults to April, matching the script's `focusmkt = 4`.
  A dashed marker labels the month on the chart, and the label beside the dropdown reports the spend and
  **the resulting lift in units**, so the effect is visible rather than assumed.
- **Reset** clears the push (focus month back to `(none)`) and redraws the unboosted forecast, so the
  before/after is a one-click toggle. With the default data that is 952 units in April boosted against
  940 unboosted — THB 876,200 of shipping cost against THB 866,100.
- **New random draw** redraws the Poisson marketing-spend series.

Marketing spend is cached across the full 24-month span, so changing the horizon does **not** reshuffle
the months you were already looking at. `SameMonthLastYear` is read from a growing `[history; forecast]`
series, which is what lets horizons run past 12 months.

### Component 2 — Transportation optimization

- **Shipping month** picks which forecast month to ship. Demand and DC capacity are re-derived from that
  month and the plan re-solves. The list tracks the forecast horizon, keeping your choice when it is
  still in range and otherwise falling back to April.
- The **tableau** is fully editable — 3 DC rows × 5 store columns of unit costs, plus a shaded `Supply`
  column and `Demand` row. Any edit re-solves immediately.
- Whenever the forecast changes, **demand and supply re-derive automatically** so the shipping plan can
  never silently disagree with the forecast above it. Your **cost edits are preserved** through that
  refresh; **Reload from forecast** is what restores the original scenario costs.
- The **heatmap** shows the optimal plan on an inverted `hot` colormap; unused routes read light, heavy
  flows dark.
- The **KPI strip** reports optimized cost, the naive benchmark, savings and percentage, with a lamp for
  solver status.

Infeasible input — total supply below total demand — shows a red lamp and an explanatory message rather
than throwing an error, so a mistyped cell during a demo is recoverable.
