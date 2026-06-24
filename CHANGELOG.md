# Changelog

All notable changes to this project will be documented in this file.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [1.0.0] — 2026-06-19

### Added

**ML Pipeline (`credit_risk_ml` package)**
- Synthetic credit dataset generator with Gaussian copula correlations,
  bisection-calibrated PD (target base rate 8.5%), and realistic fraud injection (1.5%)
- Feature engineering pipeline with train/serve parity and label-leakage guard
- Model zoo: Logistic Regression, Decision Tree, Random Forest, XGBoost (optional),
  LightGBM (optional), CatBoost (optional), VotingEnsemble (top-3 by AUC)
- Fraud detection engine: Isolation Forest + Local Outlier Factor + supervised RF +
  rule-based signals; properly evaluated on a held-out test split (ROC-AUC 0.978)
- Risk analytics: PD/LGD/EAD/EL, Monte Carlo VaR/CVaR (Gaussian copula, ρ=0.15),
  3-scenario macroeconomic stress testing (logit-space scenario shifts)
- Explainability engine: SHAP (primary), permutation importance (fallback),
  LIME (cross-check), counterfactual suggestion search
- MLflow experiment tracking integration (optional; graceful fallback when not installed)
- Optuna hyperparameter optimisation stubs (ready to wire in)

**Backend (FastAPI)**
- JWT authentication with RBAC (5 roles: admin, risk_analyst, loan_officer, auditor, viewer)
- Real-time scoring endpoint (`POST /api/v1/risk/score`)
- Synchronous batch scoring endpoint (up to 5,000 applications)
- Async Celery-backed batch endpoint with progress polling
- Fraud detection endpoint with blended fraud score
- Portfolio risk summary endpoint (PD/LGD/EAD/EL + VaR + stress test)
- Counterfactual suggestions endpoint
- GenAI Copilot endpoint (template / Ollama / OpenAI-compatible)
- Customer management CRUD
- Model monitoring + PSI drift detection endpoint
- Prometheus metrics at `/metrics` via `prometheus-fastapi-instrumentator`
- Rate limiting (120 req/min via SlowAPI)
- Async SQLAlchemy + asyncpg database layer
- Alembic migration (initial schema: 8 tables including insert-only audit log)
- Celery tasks: async batch scoring, scheduled drift check, retraining trigger

**Frontend (Next.js 15)**
- 10 pages: Landing, Dashboard, Credit Score Analysis, Explainability,
  Fraud Detection, Portfolio Risk, AI Copilot, Customer Profiles, Reports, Settings
- Signature `RiskGauge` SVG component (half-circle dial, 5-band risk colour arc)
- Risk-terminal design system: `#0B0E14` ink base, JetBrains Mono for all numeric data,
  signal green/amber/red semantic tokens
- Fully typed API client mirroring all Pydantic schemas
- Field-level type parity verification between Pydantic and TypeScript (all 7 shared types)

**Infrastructure**
- Multi-stage Dockerfiles for backend and frontend
- 9-service docker-compose stack (postgres, redis, mlflow, backend, celery worker+beat,
  frontend, prometheus, grafana)
- Kubernetes manifests: Deployment, Service, HPA, Ingress, Namespace, ConfigMap,
  Secret, PVC, StatefulSet
- Terraform modules: VPC (multi-AZ), RDS (multi-AZ production, encryption, secrets manager),
  ElastiCache Redis (TLS), ECS Fargate (ALB, task definitions, IAM roles), ECR
- GitHub Actions workflows: CI (lint + test + ML train + Docker build),
  CD (GHCR push + ECS deploy to staging → production), model retraining, security scan

---

## [Unreleased]

### Planned
- TabNet deep learning model (PyTorch)
- Prophet/ARIMA time-series PD forecasting
- FAISS vector store for document-based RAG copilot
- Full LIME integration (counterfactuals currently use greedy perturbation)
- Equalized-odds and calibration-by-subgroup fairness metrics
- Home Credit / LendingClub schema adapter modules
- WebSocket real-time scoring feed for the dashboard
