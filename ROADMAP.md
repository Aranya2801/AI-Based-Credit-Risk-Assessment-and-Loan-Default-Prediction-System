# Roadmap

## v1.1 — Deep Learning Models (Q3 2026)

- **TabNet** (PyTorch) — attention-based tabular network; built-in feature selection
  that produces decision-step-level explanations natively
- **Neural Additive Model (NAM)** — interpretable by design; one sub-network per
  feature, making global explanations exact rather than approximate
- **Autoencoder fraud detector** — reconstruction-error anomaly score as a fourth
  component in the fraud ensemble, replaces the LOF component on large datasets
  where LOF's O(n²) scaling becomes a bottleneck

## v1.2 — Time-Series and Vintage Analysis (Q4 2026)

- **Prophet / ARIMA forecasting** — PD trend forecasting by cohort vintage,
  enabling forward-looking expected-loss projections rather than point-in-time only
- **Cohort survival analysis** — Kaplan-Meier curves for time-to-default by
  loan purpose and origination quarter
- **Early warning system** — threshold-triggered re-scoring of existing
  accounts when macro indicators (unemployment, consumer sentiment) shift
  beyond a configurable sensitivity band

## v1.3 — Production Hardening (Q1 2027)

- **FAISS vector store** — replace the template copilot with a genuine
  document-retrieval RAG pipeline over regulatory guidance, underwriting
  policy documents, and historical decision rationale
- **Ollama model evaluation** — benchmark Llama 3.1 8B vs Phi-3-mini
  on the copilot Q&A task for the offline/on-premise deployment path
- **WebSocket real-time feed** — live scoring events streamed to the
  Dashboard without polling
- **Multi-tenant RBAC** — per-tenant permission overrides for SaaS
  deployment serving multiple lending institutions from one cluster
- **Model A/B testing** — canary routing between champion and challenger
  models, with automatic promotion when the challenger's KS statistic
  exceeds the champion on a held-out window

## v2.0 — Graph and Alternative Credit Scoring (2027)

- **Graph-based fraud detection** — GNN over the application-device-identity
  graph to detect fraud rings that are individually low-signal but collectively
  anomalous
- **Alternative credit scoring** — cash-flow-based scoring from bank transaction
  data (open banking / Plaid integration) for thin-file applicants with no bureau
  history
- **Regulatory capital calculation** — Basel III/IV Pillar 1 RWA computation
  for banks subject to IRB model requirements

## Research directions

- **Conformal prediction intervals** for PD estimates — output a calibrated
  confidence interval rather than a point estimate, providing a more honest
  representation of model uncertainty to credit officers
- **Causal inference for fairness** — move beyond disparate impact ratio
  checks to structural causal model analysis of whether protected attributes
  are mediating or confounding risk predictions
- **Federated learning** — train PD models across multiple lenders without
  sharing raw application data, relevant for industry-consortium credit models
