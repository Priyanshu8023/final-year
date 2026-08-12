# 09 — Dashboard

Next.js App Router, server components, five pages. Reads `08-read-api.md` and renders it.

---

## 1. Start from `667ea18`, not `create-next-app`

`client/` is deleted in the working tree but present in `HEAD`
(`01-repository-baseline.md` §1.2) with a page structure that is already close to what is
needed: `app/dashboard/`, `app/portfolio/`, `app/stocks/[symbol]/`, chart components
(`CandlestickChart`, `SparklineChart`), a shadcn-style `components/ui/` set, and Tailwind
configured.

Restore it, then:

| Keep | `components/ui/*`, `CandlestickChart`, `SparklineChart`, `StockTable`, layout, Tailwind config |
| Rework | `app/dashboard` → predictions view; `app/portfolio` → paper account; `app/stocks/[symbol]` → ticker detail with prediction overlay |
| Delete | `app/auth/*`, `app/profile`, `app/orders`, `store/auth-store.ts`, `store/watchlist-store.ts`, `hooks/useWebSocket.ts`, `components/dashboard/{WatchlistPanel,NewsSection,MarketIndices,GainersLosers}.tsx`, `components/stocks/TradePanel.tsx`, `services/{auth,watchlist}-api.ts` |

Deletions follow from scope: no auth, no watchlists, no manual trading, no live price socket at
a one-day horizon (`12-out-of-scope.md`).

**No zustand.** The remaining state is a couple of URL query params (selected model version,
date range). Server components plus `searchParams` cover it. A client store for data that
changes once a day is machinery without a job.

**No client-side polling.** `useWebSocket` and any `setInterval` refetch go. Data updates at
18:30 IST; the page is server-rendered per request with `revalidate: 300`.

---

## 2. Pages

| Route | Purpose |
|---|---|
| `/` | Today's predictions |
| `/stocks/[symbol]` | Ticker detail: price history + past predictions |
| `/accuracy` | Live accuracy vs baseline, model comparison, regime breakdown |
| `/portfolio` | Paper account state and equity curve |
| `/status` | Job health, parity, data freshness |

---

## 3. `/` — Today's predictions

- Header: `trade_date`, `target_date`, active model selector, `threshold_used`.
- **Freshness banner**, rendered when `exog_staleness.global > 1` or the last run failed:
  *"Today's global market inputs are 3 sessions old."* Not a footnote — a banner. A user
  reading a prediction is entitled to know its inputs were degraded before they read it.
- Table: ticker, company, close, `prob_up`, UP/DOWN, volatility regime. Sortable, filterable by
  label and probability.
- Probability rendered as a bar centred on the threshold, not a bare percentage. The models sit
  between 0.47 and 0.55; a column of "51.2%" values reads as meaningful discrimination when it
  is nearly none. A centred bar makes the compression visible at a glance.
- Summary strip: count UP / DOWN, mean `prob_up`, count above the portfolio's
  `min_probability` gate.

**Explicitly absent:** any "recommended action", "buy signal", or confidence badge. The system
outputs a direction and a probability. Anything stronger is a claim the evidence does not
support.

---

## 4. `/accuracy` — the page that must not mislead

This page carries the project's headline claim, and the claim is thin (`00-overview.md` §4).
Four rules govern it.

**1. Never a bare accuracy number.** Every accuracy is rendered with its 95% Wilson interval.
At 96 predictions/day the single-day standard error is 5.1 percentage points — a daily accuracy
figure is noise drawn as a line. The daily series is plotted as a faint scatter behind the
cumulative line, with the cumulative interval as a shaded band that visibly narrows over time.

**2. The baseline is on the same axis, always.** A majority-class reference line, not a
separate widget. The chart's subject is the *gap*, and a gap needs both lines.

**3. Provenance is a segmented control, never a merge.** Three tabs — `live`,
`backfill_oos`, `backfill_insample` — with a persistent one-line explanation of the selected
tier. `backfill_insample` is additionally shown with a hatched fill and the label *"in-sample —
these dates were in the model's training data and are not evidence of predictive skill"*
(`10-backfill-and-accuracy.md` §2). There is no "all" option, because the API does not offer
one.

**4. Evidence sufficiency is stated numerically.** A progress element:
*"3,936 of ~23,400 predictions needed to distinguish this model's edge from chance (17%).
Approximately 203 more trading sessions."* This converts an ambiguous chart into a specific,
honest status.

Below: the model comparison table from `GET /api/v1/models` — stored test metrics beside live
forward-tested accuracy — and the regime breakdown, which is the research question's actual
answer.

---

## 5. `/stocks/[symbol]`

- Candlestick chart of `daily_bars` (reuse `CandlestickChart`), with prediction markers on each
  session: up/down triangle, filled if correct, hollow if wrong, greyed if unresolved. One
  glance conveys the hit pattern without a second chart.
- A `prob_up` sparkline beneath, sharing the x-axis.
- Table of past predictions across model versions for this ticker.
- Per-ticker accuracy with its confidence interval — which, at ~40 observations, will be
  ±15 points. Render the interval prominently. A per-ticker accuracy is the number most likely
  to be over-read on the whole site, and the interval is the correction.

---

## 6. `/portfolio`

- Equity curve with three series: net equity, gross-of-cost equity, and `^NSEI` buy-and-hold.
  The gap between the first two is the cost drag, and per `07-paper-portfolio.md` §2.2 it is
  expected to dominate the strategy's return. That chart is the most informative object in the
  dashboard and should be the largest thing on the page.
- Holdings table: ticker, qty, avg cost, current mark, unrealised P&L.
- Trade log with itemised costs, `signal_return` beside `realised_return`
  (`07-paper-portfolio.md` §1) — the close-to-close/open-to-close gap made visible per trade.
- Stat strip: total return, max drawdown, hit rate, cost drag, deployment.

---

## 7. `/status`

`job_runs` history, last parity result with `max_deviation`, exogenous staleness per group,
active model versions, last successful run. Effectively `/healthz` rendered as a page. It
exists so that "is the number on the front page trustworthy today" is answerable without shell
access.

---

## 8. Non-goals

No dark/light theming work beyond what the restored client already has. No mobile-first
redesign — this is an operator and examiner tool. No animation. No login. No real-time updates.
The dashboard's job is to render evidence accurately, and every hour spent on polish is an hour
not spent on the parity work in `03-feature-parity.md` §7, which is what determines whether the
evidence means anything.
