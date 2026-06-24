# Architecture

## Design philosophy

Three principles drove every architectural decision in this platform:

**1. Train/serve parity is a hard invariant.** The preprocessing pipeline
(`credit_risk_ml.features.feature_engineering.build_preprocessing_pipeline`)
is fit once on training data and persisted as part of the model bundle
(`champion_model_bundle.joblib`). The API serving layer loads and applies
**this exact fitted pipeline** — not a reimplementation. This eliminates the
most common silent failure mode in production ML: feature drift between what
the model was trained on and what it sees at inference time.

**2. Explainability is grounded, not generic.** The GenAI copilot answers
questions about a specific loan decision by building its response from the
real SHAP/permutation-importance output computed for that specific application.
It does not generate free-form reasons about credit risk in general. The
"retrieval" step in the RAG pipeline is the model's own explanation — not
a document database, not prior conversation context.

**3. Evaluation before deployment, always.** The CI pipeline enforces hard
performance gates (ROC-AUC > 0.70, KS > 0.40) before any model artifact can
be published. The fraud engine uses a properly held-out test split for metric
reporting, and then re-fits on the full dataset only after metrics pass. No
in-sample evaluation anywhere.

---

## Component map

### `credit_risk_ml` (Python package, `ml_pipeline/`)

The ML pipeline is structured as a proper installable package rather than a
collection of scripts, for two reasons: (1) `joblib`-pickled sklearn objects
record the `__module__` path of every class they reference — if a class is
defined in a `__main__` script, loading the pickle fails from any other
entry point; (2) the feature engineering code needs to be importable from
both the training pipeline and the FastAPI service, with no code duplication.

Sub-modules:

| Module | Responsibility |
|--------|---------------|
| `features.synthetic_data_generator` | Gaussian-copula correlated feature generation, bisection-calibrated PD |
| `features.feature_engineering` | Sklearn Pipeline (feature engineering → imputation → scaling/encoding), leakage guard |
| `training.train_default_model` | Full model zoo training, evaluation, champion selection, artifact export |
| `training.train_fraud_model` | Thin training script importing `fraud.fraud_engine` (avoids pickle `__main__` trap) |
| `fraud.fraud_engine` | `FraudDetectionEngine` class: IF + LOF + supervised RF + rule-based signals |
| `evaluation.risk_analytics` | PD/LGD/EAD/EL, Monte Carlo VaR (Gaussian copula), stress testing, risk segmentation |
| `explainability.explainer` | `CreditExplainabilityEngine`: SHAP, permutation importance, counterfactual search |

### Backend (`backend/app/`)

FastAPI application with a startup lifespan that loads both model artifacts
once into a module-level `ModelService` singleton. This singleton is shared
across all request-handler coroutines (safe because it's read-only after
startup — no mutable shared state per request).

Key design decisions:

- **Async DB layer, sync ML inference.** The database is accessed via async
  SQLAlchemy + asyncpg so the I/O-bound DB operations don't block the event
  loop. The ML model inference (sklearn `.predict_proba()`) is CPU-bound and
  runs synchronously — for high-throughput production deployments, consider
  wrapping it in `asyncio.run_in_executor()` or routing to a dedicated
  inference worker.

- **Single Limiter instance.** SlowAPI's rate limiter reads from
  `request.app.state.limiter`. A single `Limiter()` instance is created in
  `app.core.rate_limit` and imported by both `main.py` (attached to
  `app.state`) and any endpoint module that decorates with `@limiter.limit()`.
  Multiple `Limiter()` instances would give inconsistent rate-limit state.

- **Explainer background fit at startup.** The SHAP/permutation-importance
  engine requires a "background distribution" sample to compute feature
  contributions relative to a baseline. This is loaded from
  `explainer_reference_sample.csv` (300 rows from the training set, saved
  by `train_default_model.py`) at startup — not lazily on the first
  request. A single-row "background" degenerates to zero contributions for
  every feature, which is a silent correctness failure.

### Frontend (`frontend/`)

Next.js 15 App Router with a deliberate "financial instrument panel"
aesthetic rather than generic SaaS styling. Key design decisions:

- **JetBrains Mono for all numeric data.** Probability values, dollar
  amounts, and score numbers use `font-family: "JetBrains Mono"` and
  `font-variant-numeric: tabular-nums`. This makes columns of numbers align
  correctly and visually distinguishes data from UI chrome.

- **Typed API client mirrors Pydantic schemas exactly.** `src/types/api.ts`
  is manually kept in sync with `backend/app/schemas/schemas.py` at the
  field level. The CI linting catches any divergence through TypeScript
  strict mode.

- **No client-side state management library.** The app uses React `useState`
  for local form state and native `fetch` for data fetching. This avoids
  adding Zustand/Redux/React Query as a dependency layer when the actual
  state management need is simple.

---

## Database schema

```
users           ─── audit_logs (FK: user_id)
customers       ─── loan_applications (FK: customer_id)
                         │
                    ─────┼─────
                    │         │
              risk_assessments  fraud_checks
                    (FK: application_id, unique)

model_registry  (standalone — no FKs)
drift_reports   (standalone — no FKs)
```

All UUIDs are stored as `VARCHAR(36)` (not native PostgreSQL `uuid` type) to
maintain portability with SQLite for testing. Production deployments use
PostgreSQL 16.

Audit logs are insert-only by application-level convention (no `UPDATE` or
`DELETE` statements issued by the ORM). Retention is enforced by a Celery
Beat scheduled task that archives rows older than `AUDIT_LOG_RETENTION_DAYS`
to S3 before deleting them from the live table.

---

## Data flow: real-time scoring request

```
1. POST /api/v1/risk/score  (JSON payload)
2. → SlowAPI rate limiter check (120 req/min)
3. → JWT validation + RBAC permission check ("read")
4. → Pydantic LoanApplicationRequest validation (field types, ranges)
5. → ModelService.score_application(payload)
6.   → income_to_loan_ratio auto-computed if not supplied
7.   → preprocessing_pipeline.transform(df)  [feature_engineering + impute + scale]
8.   → model.predict_proba(Xt)[:, 1]  → probability_of_default
9.   → estimate_lgd(home_ownership)
10.  → estimate_ead(requested_amount)
11.  → expected_loss = PD × LGD × EAD
12.  → assign_risk_segment(PD)
13.  → recommend_credit_limit(income, PD, requested_amount)
14.  → decision = "approve" | "manual_review" | "reject"  (threshold-based)
15.  → CreditExplainabilityEngine.explain_instance(df)
16.    → fallback_local_importance() if SHAP unavailable
17.    → top_positive_factors, protective_factors
18. ← RiskAssessmentResponse (JSON)  — p50 latency ~15ms
```

---

## Security model

See [`SECURITY.md`](../SECURITY.md) for the full security documentation. Brief summary:

- **Auth**: HS256 JWT, 30-min access / 7-day refresh
- **AuthZ**: FastAPI dependency `require_permission(scope)` checked at each endpoint
- **Secrets**: Environment variables in dev, AWS Secrets Manager in production
- **Audit**: Every decision appended to `audit_logs` (insert-only, 7-year retention)
- **Transport**: Redis TLS, RDS encryption-at-rest, ALB HTTPS
- **Supply chain**: SBOM + SLSA Level 3 provenance on all CI-built images

---

## Performance characteristics

All numbers from local benchmarks on the 25k dev dataset:

| Operation | p50 latency | p99 latency |
|-----------|:-----------:|:-----------:|
| Single real-time score (no SHAP) | ~8 ms | ~22 ms |
| Single real-time score (with SHAP, when installed) | ~45 ms | ~120 ms |
| Fraud detection score | ~5 ms | ~15 ms |
| Batch score (100 applications, synchronous) | ~80 ms | ~200 ms |
| Monte Carlo VaR (2000 accounts, 5000 sims) | ~380 ms | — |
| Model load at startup | ~1.2 s | — |

The single-request ML inference path is intentionally synchronous (sklearn
`.predict_proba()` releases the GIL, so multiple workers can run concurrently).
Uvicorn is started with `--workers 4` in the Docker image to use all available
vCPUs. For higher throughput, scale horizontally via ECS replicas or the K8s HPA.
