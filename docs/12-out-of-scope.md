# 12 — Out of Scope

The boundary, on record. Each item states what is excluded, why, and what would have to change.

---

## 1. Intraday prediction

**Excluded.** The horizon is one trading day. The target is `close(t+1) > close(t)`
(notebook 11, `create_target`). Every feature is daily-bar derived, the exogenous series are
lagged by one day, and the sequence models consume 20 *daily* steps.

**Why:** an intraday horizon is not a configuration change. It requires a different target, a
different feature set (the 13 technical indicators are all daily-parameterised: RSI(14),
Volatility_50, SMA_20), intraday bar acquisition, and retraining from scratch. It is a
different project that reuses the infrastructure.

**Consequences accepted:** no WebSocket, no streaming, no message broker, no live price
display. `11-trade-offs.md` §1 rejects each on the load implied by one run per day; that
rejection is contingent on this exclusion.

**Would change it:** a decision to re-target the research question. Not a feature request.

---

## 2. Real-money execution

**Excluded.** No broker integration, no order routing, no API keys with trading permissions.
The portfolio is paper only, against real fetched prices (`07-paper-portfolio.md`).

**Why:** three independent reasons, any one sufficient.

1. **The evidence does not support it.** The best model's measured edge is 0.64 percentage
   points and round-trip costs are ~0.34% (`07-paper-portfolio.md` §2.2). Expected gross return
   per trade is roughly 0.015% against 0.338% of cost. Deploying capital against that is not a
   risk decision, it is an arithmetic error.
2. **Statistical.** One to two years of forward-testing are needed before the edge is
   distinguishable from chance (`10-backfill-and-accuracy.md` §4.2). Trading before then is
   trading on an unmeasured hypothesis.
3. **Regulatory.** Automated order placement on Indian exchanges requires broker-side algo
   approval and exchange registration. Out of scope for a final-year project by a wide margin.

**Explicitly not built:** order management, fills and partial fills, margin, position limits,
kill switches, pre-trade risk checks. These are load-bearing in a real system and their absence
must not be mistaken for an oversight.

**Also explicitly not built, per the constraints:** no matching engine, no order book, no
simulated price generator. Every price in `trades` is a real `open` or `close` from
`daily_bars`. The one place this bites — a suspended ticker on settlement day — is handled by
carrying the position at its last mark rather than fabricating a fill
(`07-paper-portfolio.md` §5).

---

## 3. Multi-user authentication

**Excluded.** Single operator. No login, no sessions, no roles, no per-user data isolation.

**Why:** there is one user. The deleted `server/` contained a working JWT + bcrypt
implementation with a `users` table (`01-repository-baseline.md` §1.1); it is preserved in
`667ea18` and is not being carried forward, because auth without a second user is a schema tax
and a login screen between the operator and the data.

**Note on `accounts`:** `04-data-model.md` §6.1 has multiple paper accounts. These are
*strategy* accounts — one per model version, so P&L is attributable — not user accounts. They
carry no ownership, no credentials, no isolation.

**Deployment consequence:** the API and dashboard bind to localhost or a private network. If
exposed, put a reverse proxy with HTTP basic auth in front. Do not add half an auth system to
the application — a partial implementation invites the assumption that it is complete.

**Would change it:** a genuine second user. Then FastAPI's `OAuth2PasswordBearer` + `passlib`,
a `users` table, and an owner column on `accounts`.

---

## 4. Notebooks 19–31: alpha and portfolio strategies

**Excluded.** Thirteen notebooks, none of them in the serving path:

| | |
|---|---|
| 19–20 | backtesting, strategy evaluation and optimisation |
| 21 | cross-sectional portfolio strategy |
| 22–23 | multi-horizon and regime-specific models, regime-aware dynamic trading |
| 24–25 | risk-managed trading, stress testing and validation |
| 26–31 | alpha signal enhancement, alternative and cross-sectional alpha, refined pipeline, ensemble alpha, robust ensemble recovery |

**Why:** this phase serves the four models from notebooks 13–14 with the ensemble from 16–17.
Adding a second modelling family multiplies the parity surface — notebook 22's artefacts under
`models/multi_horizon_regime/` are a **different** feature set at **five** horizons (t+1, t+3,
t+5) with **per-regime** models, each needing its own scaler, its own feature order and its own
parity fixture. That is a second serving system sharing a database.

**What is borrowed rather than served:** `models/multi_horizon_regime/` demonstrates the
artefact-bundling pattern this design adopts — `best_high_transformer_t1.pt` alongside
`best_high_transformer_t1_scaler.pkl` and `..._meta.json`. Notebook 22 persisted its scaler;
notebooks 13 and 14 did not (`03-feature-parity.md` §4.1). The pattern is imported; the models
are not.

**The design accommodates their later addition** without rework: `model_versions.family` and
`feature_set_version` already namespace them, `predictions` already keys on `model_version`, and
`accounts` already binds a portfolio to a model. Adding a multi-horizon family means a new
`FEATURE_ORDER`, a new parity fixture, and a `target_date` offset greater than one session —
which `predictions.target_date` already stores explicitly rather than deriving
(`04-data-model.md` §4.2).

**Also excluded from 18:** selective prediction and confidence thresholds beyond the simple
`min_probability` gate in the paper portfolio's sizing policy.

---

## 5. Other exclusions

| Excluded | Why | Note |
|---|---|---|
| **Model retraining automation** | Retraining is a deliberate, versioned event producing a new immutable artefact directory. Automating it invites silent model drift under a stable name. | Run by hand; `03-feature-parity.md` §7. |
| **Probability calibration in serving** | `calibrated_model.pkl` (notebook 16) is not in the path. A calibrated probability is a different number than the recorded metrics describe. | Legitimate later work — as a new `model_version`, not a wrapper. `06-inference-service.md` §6. |
| **News sentiment features** | Alpha Vantage's `NEWS_SENTIMENT` columns did not survive into the 163-feature set upstream of notebook 11, and the free tier's 25 requests/day cannot cover 96 tickers. | Fetched on rotation for future work; never blocks the job. `05-daily-batch-job.md` §4.1. |
| **MongoDB** | `project_analysis.md` and `AGENTS.md` plan a MongoDB Atlas push; `pymongo` and `dnspython` are in `requirements.txt`. This design uses Postgres exclusively. | Two datastores for one workload, one of which is unused. Drop the dependencies. |
| **MLflow / Evidently** | Both in `requirements.txt`, neither used in any notebook. Experiment tracking for a fixed set of trained artefacts is answered by `model_versions` and `meta.json`. | Drop from serving requirements. `01-repository-baseline.md` §5. |
| **Streamlit / Flask / TensorFlow** | All in `requirements.txt`; none used. TensorFlow in particular is a ~600 MB dependency for a project that trains in PyTorch. | Drop. |
| **Feature store** | 163 features, one consumer, one producer, recomputed from the epoch daily. | `feature_rows` exists for forensics only. `04-data-model.md` §3.4. |
| **A/B testing framework** | Running two model versions concurrently is already supported by `is_active` and per-version accuracy series. | `03-feature-parity.md` §8.3. |
| **Alerting / metrics stack** | One job per day. `job_runs` plus one cron-driven email query. | `02-architecture.md` §7. |
| **Horizontal scaling, replicas, sharding** | Well over 10x away from need. | `11-trade-offs.md` §3. |
| **Sector, market-cap or index-weight features** | Not in the trained feature set. Adding one is a `FEATURE_ORDER` change and a major version bump. | |
| **Short selling** | Requires intraday square-off or SLB modelling. | Portfolio is long-only; stated as a limitation. `07-paper-portfolio.md` §4. |
| **Tax and capital-gains accounting** | Not needed for paper trading; it is why weighted-average cost is acceptable over FIFO. | `07-paper-portfolio.md` §3. |

---

## 6. Known limitations carried forward

Not excluded work — defects in the current state that this design surfaces rather than hides.
Each is recorded so it appears in the write-up rather than being discovered by a reader.

1. **The regime clustering leaked across the train/test boundary.** Notebook 12 fitted KMeans
   and GMM on all 63,541 rows. Five of 163 features saw test-period data before evaluation.
   The reported 0.5088 is therefore mildly optimistic. Fixed by the retrain
   (`03-feature-parity.md` §7); the current figures belong in an appendix with this explanation.
2. **The rule-based regime thresholds** (`LOW = 0.0146`, `HIGH = 0.0192`) were chosen by
   inspecting the whole panel. Reproducible, so not a serving blocker, but a soft leak. Re-derive
   from the training slice.
3. **The 0.41 threshold and 0.7/0.3 ensemble weights** were tuned on a panel containing the
   leaked features and must be re-derived (`06-inference-service.md` §2.3).
4. **Four Nifty 100 constituents are absent** — TMPV, TATACAP, ENRIN, HYUNDAI — for insufficient
   history. The universe is 96, not 100, and every accuracy figure is over 96
   (`04-data-model.md` §2.1).
5. **The library versions used to train are unrecorded.** The pickles in `models/` cannot be
   verified against the environment that produced them. Fixed as a side effect of the retrain
   (`03-feature-parity.md` §7.2 step 7).
6. **The portfolio trades a different quantity than the model predicts** — open-to-close, not
   close-to-close. Recorded per trade rather than papered over
   (`07-paper-portfolio.md` §1).
7. **`Market_Data/processed/*.parquet` is gitignored**, so the training panel is not in the
   repository. The parity fixture must be committed separately
   (`01-repository-baseline.md` §6).
