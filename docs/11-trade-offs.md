# 11 — Trade-offs

Decisions argued with numbers, including the ones this design declines to make.

---

## 1. No Kafka, no message queue, no Redis

The constraint says to justify any of these against the actual load, and that the honest answer
is almost certainly that they are not warranted. Working through it: they are not warranted, and
the numbers are not close.

### 1.1 The load

| Quantity | Value |
|---|---|
| Inference runs per day | 1 |
| Feature rows per run | 96 |
| Prediction rows per run | 480 |
| Bytes of prediction data per run | ~50 KB |
| Events per second, if this were a stream | **0.0011** — one message per 900 seconds |
| Producers | 1 |
| Consumers | 1 |
| API requests per second | single digits, human-driven |
| Rows in the hottest query | 96 |

### 1.2 Kafka: rejected

Kafka solves decoupling of independently-scaled producers and consumers, replayable ordered
logs, and buffering when consumers cannot keep up.

Here there is one producer and one consumer, in the same process, in a linear sequence. Nothing
to decouple. Replay is already available — `daily_bars` and `exog_series` are the durable log,
and the panel is recomputed from the epoch every day anyway
(`03-feature-parity.md` §3.4), so replayability is a property of the *design*, not of a
transport. Buffering is meaningless when the consumer finishes in 90 seconds and the next
message is 24 hours away.

The cost: a broker plus a controller (~1 GB RAM committed to move 50 KB/day), a topic
retention policy, a consumer-offset failure mode that produces silent gaps, and a second
durability story alongside Postgres's. Then, because at-least-once delivery means duplicates,
the consumer needs idempotency keyed on `(trade_date, ticker, model_version)` — which is
`04-data-model.md` §4.2's unique constraint. **Kafka's main contribution would be recreating a
problem the schema already solves.**

Rejected.

### 1.3 Celery / RQ / any task queue: rejected

Task queues buy concurrency across workers, retries, and scheduling. The job is 96 rows and
under two minutes; concurrency is not the bottleneck (60%+ of the runtime is a serialised
external fetch that a queue would not parallelise without hitting Yahoo's rate limits). Retries
are the 20:30 cron attempt, made safe by the natural key (`05-daily-batch-job.md` §1).
Scheduling is one crontab line.

The cost: a broker, a worker pool to keep alive, serialisation of task arguments, and a second
place where "did it run" is recorded that can disagree with `job_runs`.

Rejected.

### 1.4 Redis: rejected

Redis would serve as a cache in front of the read API or as a message bus. As a bus, see §1.2.

As a cache: the hottest query returns 96 rows from a table whose entire annual volume is 120k
rows — a few tens of MB, permanently in `shared_buffers`. The query is an index-only scan on
`uq_prediction`, sub-millisecond. Adding Redis in front replaces a sub-millisecond Postgres read
with a sub-millisecond Redis read plus an invalidation path.

That invalidation path is the actual objection. The data changes once per day, at 18:30. That
is the single moment when a stale cache is maximally wrong — serving yesterday's predictions
labelled as today's, on the page whose entire purpose is to state what today's prediction is.
A cache that is correct 99.9% of the time and wrong at the one moment anybody checks is worse
than no cache.

Note that `server/package.json` already declares `redis` as a dependency. That is another point
in favour of dropping `server/` (§2) rather than inheriting an unused dependency and, later,
someone's reasonable-seeming decision to use it.

Rejected. `08-read-api.md` §6 states the no-cache posture, and `accuracy_daily` — the one
aggregation heavy enough to matter — is a materialised view refreshed by the job that changes
its inputs. That is the cache, and it lives where invalidation is not a distributed problem.

### 1.5 What would change this

| Trigger | Then reconsider |
|---|---|
| Intraday prediction (multiple runs/day, minutes apart) | A queue for fetch fan-out. Still not Kafka. |
| Multiple independent consumers of the prediction stream (alerting, external subscribers) | A pub/sub layer, or Postgres `LISTEN/NOTIFY` first. |
| Read load past ~500 req/s | A cache — but a CDN in front of a static daily JSON would come first. |
| Universe past ~5,000 tickers with a per-ticker fetch fan-out | Parallel workers; a queue becomes reasonable. |

None of these is on the roadmap for this phase.

---

## 2. Express dropped, not kept as a gateway

The full argument is in `02-architecture.md` §3. Summarised, with the honest cost:

**For dropping:** zero functional overlap between what `server/` implements (JWT auth,
watchlists, a live quote proxy, a WebSocket price feed) and what this system needs (predictions,
history, model comparison, accuracy, portfolio). Every endpoint here is a query over tables the
Python job writes. Keeping Express means every response shape is defined twice — Pydantic and
TypeScript — and the two drift. It also means a second connection pool, a second deploy unit,
and a second runtime in an otherwise single-language deployment, to save nothing at single-digit
req/s.

**Against dropping:** `server/` is working code with helmet, CORS, rate limiting and a validated
route layer already in place. Rewriting equivalent middleware in FastAPI is real work, even if
it is small. And `yahoo-finance2` is a better-maintained client than `yfinance`.

**Why the first wins:** the `yahoo-finance2` point actually inverts. The training panel was
built with `yfinance`; switching the fetch client at serving time changes the raw inputs to a
parity-critical pipeline, and raw-input skew propagates through all 163 features
(`03-feature-parity.md` §5.4). Keeping `yfinance` is required, so the best argument for
retaining `server/` is not merely weak — it points the other way.

**Committed. The reversal condition, on record:** if intraday or sub-minute streaming enters
scope, restore `server/` from `667ea18` as a dedicated market-data gateway and leave the
prediction API in FastAPI. Do not merge them.

---

## 3. Single machine, and what changes at 10x

"10x" here means ~1,000 tickers (the full NSE mid-cap universe) and a longer history.

### 3.1 What does not change

| | Now (96 tickers) | 10x (1,000 tickers) | Verdict |
|---|---|---|---|
| Feature panel rows | 71,000 | 740,000 | Pandas handles it. ~1 GB float64 — fits in RAM. |
| Panel recompute time | 5–20 s | 1–3 min | Still under the daily window. |
| Prediction rows/day | 480 | 5,000 | Trivial insert. |
| Prediction rows/year | 120,000 | 1.25 M | Postgres is uninterested. |
| DL inference tensor | 1.3 MB | 13 MB | CPU, seconds. |
| API response size | 50 KB | 520 KB | Add server-side pagination defaults. |
| Database size after 5 years | < 1 GB | ~12 GB | One volume. |

The epoch-recompute decision in `03-feature-parity.md` §3.4 survives 10x comfortably. That is
worth noting explicitly, because it is the decision most likely to be second-guessed on
performance grounds, and the performance grounds do not appear until far past 10x.

### 3.2 What breaks first: the fetch

At 96 tickers the fetch is 20–60 s and is already 60%+ of the runtime. At 1,000 tickers, naive
chunked fetching is 4–10 minutes and starts colliding with Yahoo's undocumented rate limits.
Symptoms will be intermittent empty frames — which `05-daily-batch-job.md` §4.1 correctly turns
into `failed_fetch`, so the failure is loud rather than silent, but the job stops finishing.

**First change at 10x:** parallelise the fetch with a bounded worker pool (8–16 concurrent,
jittered backoff, per-host rate limiting) and persist a per-ticker fetch cursor so a partial
fetch resumes rather than restarting. This is the point at which a task queue becomes a
reasonable thing to want (§1.5), and it is the *only* component that reaches that point at 10x.

### 3.3 Second: memory during panel construction

At 1,000 tickers the panel with intermediate columns approaches 2–3 GB during construction —
notebook 11's `create_lag_features` already emits a `PerformanceWarning` about DataFrame
fragmentation at 96 tickers, and that gets worse linearly.

**Change:** build per-ticker and concatenate, rather than one wide frame with 102 sequential
`frame.insert` calls. This is a refactor of `add_lags` inside the shared module, so it applies
identically to training and serving and is covered by the existing parity assertion — a
correctness-preserving change validated by machinery that already exists. Fix the fragmentation
warning at the same time.

### 3.4 Third: nothing

Postgres, the API, the models and the dashboard are all more than an order of magnitude from
their limits at 10x. Splitting the database, adding read replicas, containerising the models
behind a serving framework, or introducing a scheduler are all changes that would be made for
reasons other than measured need.

### 3.5 The scaling axis that actually matters

Not tickers — **time**. The value of this system is `n` accumulating in the accuracy record
(`10-backfill-and-accuracy.md` §4.2): one year to a result, two to a robust one. Ten times the
tickers gets to significance ten times faster *only if* the per-ticker signal is homogeneous,
which the regime breakdown in `dl_metrics_summary.json` suggests it is not — LSTM accuracy
ranges from 0.4934 (LOW) to 0.5311 (HIGH) across regimes.

So the honest scaling statement is: **expanding the universe is a cheaper path to statistical
power than waiting, but only if it does not change the signal's composition — and there is
evidence it would.** The uptime and correctness of the daily job over a long period is the
binding constraint, not throughput. That is why `05-daily-batch-job.md` spends its length on
failure modes rather than on performance.

---

## 4. Other decisions, briefly

| Decision | Alternative | Why |
|---|---|---|
| Host cron | Airflow, Prefect, Dagster | One job, one schedule, no dependency graph. `05-daily-batch-job.md` §2. |
| Full panel recompute daily | Incremental / rolling window | Rolling window would need ~570 bars for float64 parity (77% of full history) to save 23% of a 90-second job, while reintroducing OBV and IIR skew. `03-feature-parity.md` §3. |
| `numeric` for prices and money | `double precision` | Prices arrive as decimal strings and feed a parity-critical pipeline; money must not round-trip through IEEE754. `04-data-model.md` §3.1. |
| `double precision[]` for `feature_rows` | 163 columns, or jsonb | 1.3 KB/row vs ~4 KB for jsonb; no migration on feature-set change. `04-data-model.md` §3.4. |
| Materialised `accuracy_daily` | Compute per request | 250 rows/model/year; refreshed by the only writer of its inputs. |
| Weighted-average cost | FIFO lots | No tax computation in scope; ~1-day holds. `07-paper-portfolio.md` §3. |
| No Sharpe ratio | Report it | Standard error on 250 observations of a near-zero-edge strategy is too wide to inform. `07-paper-portfolio.md` §6. |
| Parity check in the job, not only CI | CI only | The artefact mount and the database are mutable between deploys. `03-feature-parity.md` §6.2. |
| Retrain before serving | Serve current artefacts | The scaler and regime clusterer were never persisted; 5 of 163 features are unreproducible. Not a preference. `03-feature-parity.md` §7. |
