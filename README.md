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
| `BrightenUpMartAnalyticsApp.mlapp` | **Dashboard** — the interactive App Designer app the workshop ends on |
| `BUBAWorkshop.prj` | MATLAB project file — open it to put this folder on the path |
| `data/BrightenUpMartSales.csv` | 48 months of `Month, Sales, MarketingSpend` (Jan 2022 – Dec 2025) |
| `data/BrightenUpMartSales_Features.csv` | The same history reshaped as supervised-learning features (36 rows, Jan 2023 – Dec 2025) |
| `data/RbLin.mat` | Trained robust-linear model exported from Regression Learner (`RbLin.predictFcn`) |
| `data/TransportOpt.xlsx` | Distribution-network scenario — sheets `Shipping Cost`, `Supply Share`, `Demand Share` |


## Requirements

- MATLAB **R2026a**
- **Statistics and Machine Learning Toolbox** — `poissrnd`, Regression Learner, the exported model
- **Optimization Toolbox** — `optimvar`, `optimproblem`, `solve` (linprog), the Optimize Live Task

## Getting started

```matlab
cd <this folder>                            % the live script resolves data/ relative to the current folder
open LowCode_MATLAB_Business_Analytics.mlx  % run sections top to bottom
```

The live script's final part opens the dashboard, so running it through takes you from raw data to the
finished tool. To skip straight to the dashboard:

```matlab
BrightenUpMartAnalyticsApp
```

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
table starts in Jan 2023 and covers three years. The first two years (rows 1–24, 2023–2024) become
`BUS_train` and the third year (rows 25–36, 2025) becomes `BUS_test`, so the model is scored honestly on
months it has never seen.

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
spike. `rng(7)` is fixed inside `createForecast`, so a demo reproduces the same numbers every time. That
trailing median is 44 with a standard deviation of 38.36, so the focus month's spend comes out at 121.

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

**Setting up the network.** The scenario is not hardcoded: three **Import Data Live Tasks** read
`data/TransportOpt.xlsx`.

| Sheet | What it carries |
|---|---|
| `Shipping Cost` | 3 DCs × 5 stores of per-unit cost, in THB |
| `Supply Share` | each DC's share of capacity — Bangkok 0.47, Chiang Mai 0.29, Hat Yai 0.24 |
| `Demand Share` | each store's share of sales — North 0.217, Central 0.265, East 0.181, South 0.169, Isaan 0.168 |

Per-unit cost rises with distance, so every DC is cheapest into the store nearest it — Bangkok DC into
the Central Store (600), Chiang Mai DC into the North Store (700), Hat Yai DC into the South Store (700):

|  | North | Central | East | South | Isaan |
|---|---|---|---|---|---|
| **Bangkok DC** | 2000 | 600 | 900 | 2500 | 1500 |
| **Chiang Mai DC** | 700 | 2000 | 2300 | 3700 | 1800 |
| **Hat Yai DC** | 3700 | 2500 | 2600 | 700 | 2800 |

Total supply is set at 103% of total demand — a buffer so the plan is not knife-edge:

```matlab
totalDemand = round(forecastTbl.ForecastSales(focusmkt));   % 947
totalSupply = ceil(totalDemand*1.03);                       % 976
```

Splitting those totals by the shares and rounding rarely sums back to the total, so the remainder is
pushed onto the **last store** and onto the **first DC**. The store demands must sum to `totalDemand`
exactly, or the equality constraint in the next step cannot be satisfied. That gives store demand
`[205 251 171 160 160]` against DC supply `[459 283 234]`.

**Solving.** The **Optimize Live Task** states the decision in business terms — choose how many units to
ship on each route, so that no DC ships beyond capacity, every store's demand is fully met, and total
cost is as low as possible — and hands it to a linear-programming solver, which is guaranteed to return
the mathematically optimal plan rather than merely a good one:

|  | North | Central | East | South | Isaan |
|---|---|---|---|---|---|
| **Bangkok DC** | 0 | 251 | 171 | 0 | 37 |
| **Chiang Mai DC** | 205 | 0 | 0 | 0 | 78 |
| **Hat Yai DC** | 0 | 0 | 0 | 160 | 45 |

Every store is served from its cheapest centre, and only the Isaan Store — expensive from everywhere — is
split three ways, soaking up whatever capacity is left over.

**How much did optimization save?** `naiveAllocation` benchmarks the plan against a common real-world
habit — allocating stock first-come, first-served, store by store, ignoring cost entirely. For April 2026
that is **THB 881,900** optimized against **THB 1,854,700** naive: a saving of roughly **THB 972,800**,
about **52%**, with no change to demand, supply, or prices. The annotated bar chart shows the gap.

### Part 3 — AI agentics for data analytics and dashboard intelligence
A forecast and an optimal shipping plan are only useful if the right people see them in time to act. The
plan is also optimal only *for the numbers supplied* — demand came from a forecast, so the honest
follow-up question is how much the plan and the bill move if demand turns out five percent higher. That
question is the reason the dashboard exists.

The closing part covers two layers of business intelligence: the visual dashboard that summarizes
everything at a glance, and an early look at **AI agents** — a language model plus *tools* plus a loop,
which reads a plain-language business question, decides which analysis to run, and reports the answer,
with every number still computed by MATLAB. It is the idea behind the AI copilots now appearing inside
tools like Power BI and Tableau, and the **MATLAB Agentic AI toolkit** is what built the app this section
opens.

### Appendix
For anyone who wants the machinery rather than the buttons, the appendix holds:

- how the features table is built from the raw CSV with `lag1` / `lag12`;
- the `createForecast` local function — the recursive forecast loop and the Poisson spend model;
- the `naiveAllocation` local function — the greedy benchmark and the savings bar chart.

---

## The dashboard

`BrightenUpMartAnalyticsApp` puts both halves of the workshop side by side on one screen — no tabs — and
makes the assumptions editable, so the analysis can be re-run live in front of an audience. It mirrors
the solution script's pipeline, and reads the same `data/TransportOpt.xlsx` scenario rather than
hardcoding costs or shares.

```
┌────────────────────────────────────────────────────────────────────────────┐
│  Brighten Up Mart — Forecast and Distribution Planner                      │
├──────────────────────────────────┬─────────────────────────────────────────┤
│  Sales Forecast                  │  Transport Optimization                 │
│  Horizon [6▲▼]  Style [ ▼ ]      │  Shipping month [ ▼ ]  [Reload Excel]   │
│  Focus marketing month [ ▼ ]     │  Demand 947 units | supply 976 (+3%)    │
│  [Reset]                         │  ┌───────────────┬────────────────────┐ │
│                                  │  │ supply share %│ demand share %     │ │
│   blue  ─o─  actual sales        │  └───────────────┴────────────────────┘ │
│   red   --+  forecast            │  ┌────────────────────────────────────┐ │
│   grey  ┆    focus month         │  │ editable 3×5 shipping-cost table   │ │
│                                  │  ├────────────────────────────────────┤ │
│                                  │  │ heatmap of the optimal ship plan   │ │
│                                  │  └────────────────────────────────────┘ │
│  Horizon 6 months. Promotion …   │  Optimized | Naive first-come | Savings │
│                                  │  ● Optimal plan uses 7 of 15 routes.    │
└──────────────────────────────────┴─────────────────────────────────────────┘
```

The window is `[28 30 1480 762]`, sized to fit a 1536×864 logical screen.

### Left — Sales forecast

- **Forecast horizon** (1–12 months) re-runs the recursive forecast. The cap is 12 on purpose:
  `SameMonthLastYear` is read from `BUS.Sales(end-12+h)`, exactly as `createForecast` does, so the app's
  numbers stay identical to the taught script's.
- **Plot style** switches between two framings of the same numbers — *Continuous timeline* (all 48 months
  of history with the forecast appended) and *Year over year* (every calendar year overlaid on a shared
  1–12 month axis, oldest year palest, the forecast year in red).
- **Focus marketing month** applies the same promotional push as the live script: the chosen month's
  spend is replaced by `round(median + 2*std)` over the trailing 13 months. It defaults to **None**, and
  picking a month also **moves the shipping month across to match** — a promotion is the reason to ship
  extra stock. The status line beneath the axes reports the boosted spend against the normal median.
- **Reset** restores the opening state: horizon 6, continuous timeline, focus None, shipping month 1, and
  costs *and* shares reloaded from Excel.
- `rng(7)` is re-seeded on every forecast, so the marketing-spend draw is reproducible and changing the
  horizon never reshuffles the months you were already looking at.

### Right — Transport optimization

- **Shipping month** picks which forecast month to ship. Demand and DC capacity are re-derived from that
  month and the plan re-solves. The list tracks the forecast horizon, keeping your choice when it is
  still in range.
- **Both share tables are editable** — DC supply share and store demand share, shown as percentages. The
  shares are **rescaled to 100% before the split**, so an edit actually moves stock instead of being
  handed straight back by the rounding correction. Rescaling is a no-op at the Excel defaults, so the
  taught numbers are unchanged; when the shares no longer total 100% the lamp turns amber and the status
  line says so out loud rather than quietly changing what you typed.
- The **shipping-cost table** is editable cell by cell (3 DC rows × 5 store columns). Any edit re-solves
  immediately, and a negative or non-finite entry is rejected with the previous value restored.
- Whenever the forecast changes, **demand and supply re-derive automatically** so the shipping plan can
  never silently disagree with the forecast beside it. Your **cost and share edits survive** that
  refresh; **Reload Excel** is what restores the original scenario.
- The **heatmap** shows the optimal plan on the `sky` colormap — a busier route is a deeper blue.
- The **KPI strip** reports optimized cost, the naive first-come benchmark, and the savings with its
  percentage, beside a lamp for solver status.

Infeasible input — supply below demand, or shares driven negative — shows a red lamp and an explanatory
message rather than throwing an error, so a mistyped cell during a demo is always recoverable.

### Reference numbers

| State | Demand / supply | Optimized | Naive | Savings |
|---|---|---|---|---|
| As it opens — horizon 6, no promotion, Jan 2026 | 800 / 824 | THB 745,200 | THB 1,562,000 | THB 816,800 (52.3%) |
| Focus month April, shipping April — matches the script | 947 / 976 | THB 881,900 | THB 1,854,700 | THB 972,800 (52.5%) |
