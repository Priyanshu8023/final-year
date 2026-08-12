# 08 — Read API

FastAPI, `fmf.api.app`, one uvicorn worker. Read-only over Postgres except for two portfolio
endpoints (§4). Never computes a feature, never loads a model (`02-architecture.md` §2.2).

Base path `/api/v1`. OpenAPI at `/openapi.json` — the dashboard generates its TypeScript client
from it, which is how the two stay in sync without a shared schema package.

---

## 1. Conventions

- **Dates** are `YYYY-MM-DD`, always NSE session dates, always IST-anchored. No timestamps in
  responses except `created_at`/`resolved_at` audit fields, which are UTC ISO-8601.
- **Money** serialises as a decimal string (`"1013422.55"`), not a JSON number. JSON numbers
  are IEEE doubles and a currency value must not round-trip through one.
- **Probabilities** are floats with 6 significant digits.
- **Provenance is never implicit.** Every response containing accuracy carries the `origin` tier
  it was computed from. There is no endpoint that returns a blended number
  (`10-backfill-and-accuracy.md` §2).
- **Pagination** is `limit`/`offset` with `limit ≤ 500`, default 100. Cursor pagination is
  unnecessary at 120k rows/year.
- **Errors** are RFC 7807 problem+json: `{type, title, status, detail}`.
- **No auth in this phase.** Single account, bound to localhost or a private network.
  `12-out-of-scope.md`. If exposed, put a reverse proxy with basic auth in front — do not add
  half an auth system to the application.

---

## 2. Prediction endpoints

### `GET /api/v1/predictions/latest`

Today's predictions — the dashboard's landing view.

Query: `model_version` (default: the active ensemble), `min_probability`, `label`,
`sort` (`prob_desc` default), `limit`, `offset`.

```json
{
  "trade_date": "2026-02-13",
  "target_date": "2026-02-16",
  "model_version": "ensemble-v1.0.0",
  "threshold_used": 0.41,
  "resolved": false,
  "exog_staleness": {"global": 1, "macro": 34, "gdelt": 0},
  "n_tickers": 96,
  "items": [
    {"ticker": "RELIANCE", "company_name": "Reliance Industries",
     "prob_up": 0.5731, "pred_label": 1, "close": "1487.300000",
     "volatility_regime": "MEDIUM"}
  ]
}
```

`resolved: false` and the absence of an outcome field are deliberate — today's predictions have
no result yet, and the API must not imply otherwise.

`exog_staleness` surfaces at the top level so the dashboard can show a banner when today's
inputs were degraded (`05-daily-batch-job.md` §4.7). Users should know that before reading the
list.

### `GET /api/v1/predictions/{date}`

Same shape for a historical date. Includes `realised_label`, `realised_return` and `correct`
per item once outcomes exist, plus `origin`.

### `GET /api/v1/tickers/{symbol}/history`

The ticker detail view. Query: `from`, `to` (default: last 180 sessions), `model_version`.

```json
{
  "ticker": "RELIANCE",
  "company_name": "Reliance Industries",
  "sector": "Energy",
  "bars": [
    {"trade_date": "2026-02-13", "open": "1479.000000", "high": "1492.100000",
     "low": "1474.500000", "close": "1487.300000", "volume": 4821003}
  ],
  "predictions": [
    {"trade_date": "2026-02-12", "target_date": "2026-02-13", "model_version": "ensemble-v1.0.0",
     "prob_up": 0.5512, "pred_label": 1, "realised_label": 1, "correct": true,
     "origin": "live"}
  ],
  "summary": {"origin": "live", "n": 41, "n_correct": 22, "accuracy": 0.5366,
              "ci95": [0.3849, 0.6883], "baseline_majority": 0.5122}
}
```

Bars and predictions in one response: the chart needs both, and two round trips for one view is
pointless at this size.

Note the confidence interval on 41 observations — ±15 points. That width is the honest answer
to "how is RELIANCE doing" and the UI must render it (`09-dashboard.md` §4).

---

## 3. Accuracy and comparison

### `GET /api/v1/accuracy`

Query: `model_version` (repeatable), `origin` (**required**, no default), `from`, `to`,
`granularity` (`daily` | `cumulative`).

`origin` has no default on purpose. A caller must state which provenance tier it wants. There
is no way to ask this endpoint for "accuracy" and receive a silent blend of live and backfilled
rows — that is the specific mistake the three-tier design exists to prevent
(`10-backfill-and-accuracy.md`).

```json
{
  "origin": "live",
  "granularity": "cumulative",
  "series": [{
    "model_version": "transformer-v1.0.0",
    "points": [{"trade_date": "2026-02-13", "n": 3936, "n_correct": 2001,
                "accuracy": 0.5084, "ci95": [0.4928, 0.5240],
                "baseline_majority": 0.5031}],
    "evidence": {
      "n_total": 3936,
      "n_required_for_significance": 23400,
      "progress": 0.168,
      "beats_baseline_significantly": false
    }
  }]
}
```

`evidence` is computed, not stored. `n_required_for_significance` is the two-proportion sample
size for the model's measured edge over the majority baseline at α = 0.05, power 0.8 —
approximately 23,400 predictions (≈244 sessions) for a 0.64pp edge
(`10-backfill-and-accuracy.md` §4).

Returning `beats_baseline_significantly: false` alongside a 0.5084 accuracy is the single most
important piece of honesty in this API. The number alone invites over-reading; the flag makes
over-reading take deliberate effort.

### `GET /api/v1/models`

The model comparison table. All registered versions with `family`, `is_active`,
`train_start`/`train_end`, stored `test_metrics`, and live forward-tested accuracy for the same
`origin`.

```json
{
  "items": [{
    "model_version": "transformer-v1.0.0", "family": "transformer", "is_active": true,
    "test_metrics": {"accuracy": 0.5088, "roc_auc": 0.5123, "f1": 0.5514},
    "live": {"origin": "live", "n": 3936, "accuracy": 0.5084, "ci95": [0.4928, 0.5240]},
    "regime_breakdown": [
      {"regime": "HIGH", "n": 1024, "accuracy": 0.5215, "ci95": [0.4908, 0.5521]},
      {"regime": "MEDIUM", "n": 1088, "accuracy": 0.4991, "ci95": [0.4692, 0.5290]},
      {"regime": "LOW", "n": 1824, "accuracy": 0.5077, "ci95": [0.4846, 0.5307]}
    ]
  }]
}
```

`regime_breakdown` is the project's actual research question — *which model performs best under
which volatility regime* — answered on forward-tested rather than backtested data. It is the
main deliverable of running this system for a year.

`test_metrics` and `live` sit side by side so the gap between the evaluated and the deployed
number is legible. If they diverge widely, either the parity work failed or the market changed;
both are worth seeing immediately.

---

## 4. Portfolio

### `GET /api/v1/portfolio/{account_id}`

Current state: cash, holdings with live mark and unrealised P&L, realised P&L, cumulative
costs, `n_positions`, `deployment`.

### `GET /api/v1/portfolio/{account_id}/equity`

The equity curve from `portfolio_snapshots`, with `gross_equity = equity + costs_cum` alongside
`equity`, and the `^NSEI` buy-and-hold benchmark on the same axis (`07-paper-portfolio.md` §6).

### `GET /api/v1/portfolio/{account_id}/trades`

Paginated trade log with itemised `cost_breakdown`, `signal_return`, `realised_return`, and the
`prediction_id` that triggered each entry.

### `POST /api/v1/portfolio/{account_id}/reset` — mutation

Resets to `starting_cash`, deletes holdings, archives trades. Requires
`{"confirm": "<account_label>"}` in the body. Destructive and operator-facing.

### `POST /api/v1/portfolio` — mutation

Creates a paper account bound to a `model_version`, with a `cost_model` and `sizing_policy`.
The supported way to compare configurations is to create a second account, never to edit a
running one — editing would rewrite the meaning of an equity curve already recorded.

These two are the only writes in the API. Everything else the system does is written by the
daily job.

---

## 5. Operational

### `GET /healthz`

```json
{"status": "ok", "database": "ok",
 "last_successful_run": {"trade_date": "2026-02-13", "finished_at": "2026-02-13T13:02:11Z"},
 "active_model_versions": ["rf-v1.0.0","xgb-v1.0.0","lstm-v1.0.0","transformer-v1.0.0","ensemble-v1.0.0"],
 "exog_staleness_days": {"global": 1, "macro": 34, "gdelt": 0},
 "last_parity": {"passed": true, "max_deviation": 3.1e-13, "checked_at": "2026-02-13T13:00:44Z"}}
```

`status` degrades to `"stale"` if the last successful run predates the most recent NSE session.
This one endpoint answers every operational question the system raises.

### `GET /api/v1/admin/runs`

`job_runs`, paginated, filterable by `status`. The failure-mode history from
`05-daily-batch-job.md` §4, queryable — "how often does the fetch flake" is a filter, not a log
grep.

---

## 6. Caching

**None.** No Redis, no in-process cache, no `Cache-Control` beyond `no-store`.

The data changes once per day. The largest response — a full day of predictions for one model —
is 96 rows, ~50 KB of JSON, served by an index-only scan on `uq_prediction` from a table that
lives entirely in `shared_buffers`. Sub-millisecond in Postgres, single-digit milliseconds
end to end, at single-digit requests per second.

A cache in front of that adds an invalidation path for the one moment per day when the data
does change — which is precisely the moment a stale cache would be most misleading (showing
yesterday's predictions as today's). Negative value. Argued with numbers in
`11-trade-offs.md` §1.

The one exception, if it ever measures as slow: `accuracy_daily` is already a materialised
view, refreshed by the job. That is the cache, it lives in Postgres, and it is invalidated by
the only process that can change its inputs.
