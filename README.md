# Brighten Up Mart — Business Analytics Workshop

[![Open in MATLAB Online](https://www.mathworks.com/images/responsive/global/open-in-matlab-online.svg)](https://matlab.mathworks.com/open/github/v1?repo=Phudit-Ascendas/BU-BA-Workshop&file=https://github.com/Phudit-Ascendas/BU-BA-Workshop/blob/master/LowCode_MATLAB_Business_Analytics.mlx)

*MATLAB for Business Analytics: Forecasting, Optimization, and AI Agents*

A hands-on MATLAB workshop for **MG 410 Business Analytics for Strategic Decisions**, built around one
fictional Thai retail chain, *Brighten Up Mart*: three regional distribution centres, five stores, and
four years of monthly sales history.

You will not write algorithms from scratch. Every technique here is a single MATLAB function, Live Task,
or app — your job is to interpret what it says about the business, the same way you would read output
from Excel, Power BI, or SPSS. The workshop follows three connected decisions that any retail, logistics,
or supply-chain manager faces every month:

1. **Part 1 — Machine Learning for Forecasting.** How much will customers buy next quarter?
2. **Part 2 — Transportation Optimization.** Given that demand, what is the cheapest way to ship
   inventory to the stores?
3. **Part 3 — AI Agentics for Data Analytics and Dashboard Intelligence.** How do we turn all of this
   into a dashboard — and an assistant — that busy executives can actually use?

---

## Contents

| File | What it is |
|---|---|
| `LowCode_MATLAB_Business_Analytics.mlx` | **Main live script** — the taught workshop, with sections left blank for participants to fill in |
| `Solution_LowCode_MATLAB_Business_Analytics.mlx` | The completed walkthrough, for reference after the session |
| `BrightenUpMartDashboardApp.mlapp` | **Dashboard** — the interactive App Designer app the workshop ends on |
| `data/BrightenUpMartSales.csv` | 48 months of `Month, Sales, MarketingSpend` (Jan 2022 – Dec 2025) |
| `data/BrightenUpMartSales_Features.csv` | The same history reshaped as supervised-learning features |
| `data/RbLin.mat` | Trained robust-linear model exported from Regression Learner (`RbLin.predictFcn`) |
| `data/TransportOpt.xlsx` | Distribution-network scenario — sheets `Shipping Cost`, `Supply Share`, `Demand Share` |

## Requirements

- MATLAB **R2026a**
- **Statistics and Machine Learning Toolbox** — `poissrnd`, Regression Learner, the exported model
- **Optimization Toolbox** — `optimvar`, `optimproblem`, `solve` (linprog), the Optimize Live Task

## Getting started

```matlab
cd <this folder>                            % scripts resolve data/ relative to the current folder
open LowCode_MATLAB_Business_Analytics.mlx  % run sections top to bottom
```

The live script's final part opens the dashboard, so running it through takes you from raw data to the
finished tool. To skip straight to the dashboard:

```matlab
BrightenUpMartDashboardApp
```

Both resolve `data/` relative to the current folder, so `cd` into this folder rather than running the
files from elsewhere.

## The live script

### The business scenario
Brighten Up Mart is a Thai retail chain with three regional distribution centres and five stores. Every
month the management team has to answer three questions: *How much will we sell?*, *How do we distribute
stock at the lowest cost?*, and *How do we get answers fast, without waiting for an analyst?*

The script opens by loading four years of monthly sales history together with the marketing budget spent
each month — a common early-warning signal for demand — via an **Import Data Live Task**, with a data
summary alongside it.

### A first look at the data
Before modelling anything, plot the raw series. The script looks at the data two ways: a correlation
check between sales and marketing spend, to motivate marketing as a predictor, and unit sales over time,
which exposes the trend and the seasonal peaks (Songkran in April, the year-end sale).

### Part 1 — Machine learning for forecasting
Traditional rules ("this month equals the same month last year, plus a bit of growth") are simple but
blind to anything unusual, such as a bigger-than-usual marketing push. Machine learning instead learns
the relationship from many historical examples — the same idea behind product recommendations and
credit-risk scoring, applied to demand.

**Turning history into learning examples.** The problem is framed as *features in, sales out*, one row
per month with four predictors:

| Feature | Meaning |
|---|---|
| `MonthOfYear` | 1–12, captures seasonality |
| `LastMonth` | previous month's sales |
| `SameMonthLastYear` | the same month one year earlier |
| `MarketingSpend` | budget planned for that month |

**Preparing train and test data.** Because `SameMonthLastYear` needs a full year of run-up, the feature
table starts in Jan 2023 and covers three years. The first two years become `BUS_train` and the third
year becomes `BUS_test`, so the model is scored honestly on months it has never seen.

**Training.** The model is fitted in the **Regression Learner app** — point-and-click, no training code —
and exported to `data/RbLin.mat` as a robust linear fit. It can also be opened from the command line:

```matlab
regressionLearner
```

**Explaining the model.** A short section on model explainability, so the forecast is not a black box
before it is used to commit money.

**Validating on the held-out year.** The script builds the test-year input table, predicts, and plots
predicted against actual sales so participants can judge the fit for themselves.

**Forecasting.** With the approach validated, the model rolls forward **recursively** — each prediction
feeds back as the next month's `LastMonth`. This is taught twice on purpose: first *step by step*,
assembling one input row by hand and predicting a single month, then again as a loop once the pattern is
clear ("easy? what if we want 50 years of data?"). Two **interactive controls** drive it:

```matlab
targetm  = 6;   % how many months ahead to forecast
focusmkt = 4;   % which of those months gets the marketing push

[futureDates, plannedSpend, forecastSales] = createForecast(BUS, targetm, focusmkt, 'RbLin');
```

Planned marketing spend is drawn as `poissrnd(median(BUS.MarketingSpend(end-12:end)))` per month, and the
focus month is replaced by `round(median + 2*std)` over that same trailing window — the promotional
spike. `rng(7)` is fixed inside `createForecast`, so a demo reproduces the same numbers every time.

With the default settings the forecast reads:

| Month | Forecast sales | Planned spend |
|---|---|---|
| Jan 2026 | 800 | 42 |
| Feb 2026 | 710 | 38 |
| Mar 2026 | 716 | 42 |
| **Apr 2026** | **947** | **121**  ← focus month |
| May 2026 | 871 | 45 |
| Jun 2026 | 839 | 39 |

The plot picks up right where history ends and clearly shows the expected April spike from the planned
promotion — exactly the signal that feeds Part 2.

### Part 2 — Transportation optimization
The focus month's forecast becomes that month's total demand: about **947 units in April**. The question
is now purely operational — given three distribution centres with limited capacity and five stores that
must each receive their share, which routes meet every store's demand at the lowest total shipping cost?

**Setting up the network.** The scenario is no longer hardcoded: three **Import Data Live Tasks** read
`data/TransportOpt.xlsx`.

| Sheet | What it carries |
|---|---|
| `Shipping Cost` | 3 DCs × 5 stores of per-unit cost, in THB |
| `Supply Share` | each DC's share of capacity — Bangkok 0.47, Chiang Mai 0.29, Hat Yai 0.24 |
| `Demand Share` | each store's share of sales — 0.265, 0.217, 0.181, 0.169, 0.168 |

Cost rises with distance — Bangkok DC is nearest the Central Store, Chiang Mai DC the North Store, Hat
Yai DC the South Store. Total supply is set at 103% of total demand, and both vectors are rounded so
their totals stay exact:

```matlab
totalDemand = round(forecastTbl.ForecastSales(focusmkt));   % 947
totalSupply = ceil(totalDemand*1.03);                       % 976
```

which gives store demand `[251 205 171 160 160]` against DC supply `[459 283 234]`.

**Solving.** The **Optimize Live Task** states the decision in business terms — choose how many units to
ship on each route, so that no DC ships beyond capacity, every store's demand is fully met, and total
cost is as low as possible — and hands it to a linear-programming solver, which is guaranteed to return
the mathematically optimal plan rather than merely a good one:

|  | North | Central | East | South | Isaan |
|---|---|---|---|---|---|
| **Bangkok DC** | 0 | 205 | 171 | 0 | 83 |
| **Chiang Mai DC** | 251 | 0 | 0 | 0 | 32 |
| **Hat Yai DC** | 0 | 0 | 0 | 160 | 45 |

**How much did optimization save?** `naiveAllocation` benchmarks the plan against a common real-world
habit — allocating stock first-come, first-served, store by store, ignoring cost entirely. For April 2026
that is **THB 872,700** optimized against **THB 1,919,100** naive: a saving of roughly **THB 1,046,400**,
about **55%**, with no change to demand, supply, or prices. The annotated bar chart shows the gap.

### Part 3 — AI agentics for data analytics and dashboard intelligence
A forecast and an optimal shipping plan are only useful if the right people see them in time to act. The
closing part covers two layers of business intelligence: the visual dashboard that summarizes everything
at a glance, and an early look at **AI agents** — software that takes a plain-language business question
and routes it to the right analysis automatically, the idea behind the AI copilots now appearing inside
tools like Power BI and Tableau.

### Appendix
For anyone who wants the machinery rather than the buttons, the appendix holds:

- how the features table is built from the raw CSV with `lag1` / `lag12`;
- the `createForecast` local function — the recursive forecast loop and the Poisson spend model;
- the `naiveAllocation` local function — the greedy benchmark and the savings bar chart.

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

> **Note.** The app is an independent implementation of the same pipeline and still draws marketing spend
> from `mode(BUS.MarketingSpend)` over the whole series, where the live script now uses the median of the
> trailing 13 months. The two therefore report slightly different figures for the same month.

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
  before/after is a one-click toggle. At the default 6-month horizon that is 937 units in April boosted
  against 877 unboosted — THB 864,100 of shipping cost against THB 808,700.
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
