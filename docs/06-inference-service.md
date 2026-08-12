# 06 — Inference

`fmf.components.model_inference`. Called only by the daily job and the backfill pipeline. Never
by the API (`02-architecture.md` §2.2 — a second inference path would void the parity
guarantee).

---

## 1. Input

One `FeaturePanel` slice: 96 rows × 163 columns, in `fmf.constant.FEATURE_ORDER` order,
produced by `build_panel(..., as_of=trade_date)`. Plus, for the sequence models, the preceding
20 rows per ticker from the same panel — which the full-history recompute already has in memory,
so no second read.

---

## 2. Two input contracts

`01-repository-baseline.md` §3.1 established that the four models were trained on differently
prepared inputs. The dispatcher reads `model_versions.input_contract` rather than inferring from
the family name.

### 2.1 `tabular_raw` — RandomForest, XGBoost

```
X = panel.loc[panel.Date == trade_date, FEATURE_ORDER].to_numpy()   # (96, 163)
prob = model.predict_proba(X)[:, 1]
```

**No scaling.** Notebook 13 fit `RandomForestClassifier` and `XGBClassifier` on `X_train`
directly; the scaler in cell 24 was created afterward and used only for the sequence models.
Applying the scaler here would be a silent, total skew — `Close ≈ 3200` becoming `≈ 0.4` while
the tree's learned splits stay at 3200.

This is the most tempting mistake in the whole serving layer, because "load the scaler, apply
it, then predict" reads as obviously correct. Guard: `input_contract` is checked in an
assertion at the call site, and the tabular path has no reference to the scaler object at all —
it is not passed into the function.

### 2.2 `scaled_sequence` — LSTM, Transformer

```
Xs   = scaler.transform(panel[FEATURE_ORDER])                    # loaded, never fitted
seq  = per-ticker window of the 20 rows STRICTLY BEFORE trade_date   # (96, 20, 163) float32
prob = sigmoid(model(torch.from_numpy(seq)))
```

The window boundary reproduces notebook 14's `create_sequences_with_meta` exactly:

```python
for i in range(window, len(grp)):
    seq_X.append(values[i - window:i])   # rows i-20 .. i-1
    seq_y.append(labels[i])              # label at row i
```

`values[i-window:i]` is exclusive of row `i`. **The features of `trade_date` itself are not in
the sequence** — the model predicts row `i`'s target from the 20 rows before it. Getting this
off by one shifts every input by a day and produces a model that is confidently predicting
yesterday. It is worth an explicit unit test against a hand-built fixture, not just an
assertion on shape.

`float32`, matching `to_numpy(dtype=np.float32)` in the notebook. Feeding float64 would work
and would give marginally different results — parity is about matching, not about precision.

`model.eval()` and `torch.no_grad()`, batch 512 (the whole day is one batch of 96).
Deterministic: no dropout at eval, no `cudnn` non-determinism at this size. CPU is fine —
`05-daily-batch-job.md` §5 puts it under 2 seconds.

### 2.3 `ensemble`

From `ensemble_config.json`: `0.7 × Transformer_prob + 0.3 × XGB_prob`, threshold `0.41` from
`optimized_threshold.json`.

Stored as its own `model_versions` row with `family='ensemble'`, `input_contract='ensemble'`,
and a `components` key in its `runtime` jsonb naming the exact member versions
(`transformer-v1.0.0`, `xgb-v1.0.0`). The ensemble's identity therefore changes when a member
changes, which is what stops a member swap from silently redefining what "the ensemble" means
mid-accuracy-series.

Note the ensemble uses two of the four models. LSTM and RF are still scored and stored — they
are the comparison arm of the research question ("which model performs best under different
volatility regimes"), and the forward-tested comparison is a deliverable in its own right.

**The 0.41 threshold and the 0.7/0.3 weights must be re-derived after the retrain** — they were
tuned on a panel containing the leaked cluster features (`03-feature-parity.md` §7.2 step 6).

---

## 3. Artefact bundle

Layout, `meta.json` contents and the four load-time gates are specified in
`03-feature-parity.md` §4.2–4.3 and are not repeated here. The operative summary:

- One immutable directory per `model_version`. Never overwritten.
- All active bundles load and verify **before any inference runs** (`05-daily-batch-job.md`
  §4.5 — no partial days).
- Loaded once per process. The job is a fresh process each day, so there is no cache
  invalidation question.

---

## 4. Threshold handling

`pred_label = (prob_up >= threshold_used)`.

`threshold_used` is written onto **every prediction row** rather than read from config at query
time. `optimized_threshold.json` is a mutable file; if it is retuned in month eight, every
prior prediction's *label* must keep the meaning it had when it was made. Storing the threshold
per row makes that automatic, and makes "recompute accuracy under a different threshold" a
legitimate separate analysis rather than a retroactive edit.

Default per family from `model_versions.default_threshold`; RF/XGB/LSTM use 0.5, the ensemble
0.41.

---

## 5. Determinism

| Source of non-determinism | Handled by |
|---|---|
| `RandomForestClassifier.predict_proba` | Deterministic given a fitted estimator |
| `XGBClassifier.predict_proba` | Deterministic; `n_jobs` affects speed, not values |
| PyTorch dropout | `model.eval()` |
| PyTorch thread count affecting float reduction order | `torch.set_num_threads(1)` in the job — costs nothing at 96 rows and removes a genuine source of last-bit variation |
| pandas `groupby` ordering | `sort=False` with a pre-sorted frame, matching the notebooks |
| Dict/set iteration order in feature assembly | `FEATURE_ORDER` is an explicit tuple |

Determinism is checked, not assumed: re-running the job for a date already predicted must
produce byte-identical probabilities. This is verifiable because the natural key rejects the
second write — so the check runs in CI against a scratch database, comparing two independent
runs over the same fixture date.

---

## 6. What inference does not do

- No feature computation. It receives a `FeaturePanel` and never touches raw bars.
- No `.fit()` on anything (`03-feature-parity.md` §4.3).
- No fallback. If a bundle fails to load, the job fails; it does not substitute a different
  version or skip a model.
- No calibration. `calibrated_model.pkl` (769 B, notebook 16) is not in the serving path.
  Probability calibration is a legitimate improvement and is deferred to a later phase — but a
  calibrated probability is a *different* number than the one the recorded metrics describe,
  so introducing it means a new `model_version`, not a wrapper.
