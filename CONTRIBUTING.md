# Contributing to RiskCore

Thank you for your interest in contributing. This is a research-grade
production-style codebase — contributions are expected to meet the same
quality bar as the existing code.

## Quick decision tree

- **Bug fix** → Open an issue describing the bug (steps to reproduce + actual vs.
  expected output), then a PR referencing it.
- **New feature** → Open an issue first so we can discuss scope before you invest
  time writing it.
- **Documentation** → PRs welcome without a prior issue.

## Development setup

```bash
git clone https://github.com/your-org/credit-risk-platform.git
cd credit-risk-platform

# ML pipeline (editable install)
pip install -e "./ml_pipeline[full]"

# Backend
pip install -r backend/requirements.txt
pip install ruff black pytest pytest-asyncio

# Frontend
cd frontend && npm install
```

## Before you submit a pull request

### Python

```bash
# Lint
ruff check backend/app ml_pipeline/credit_risk_ml

# Format
black backend/app ml_pipeline/credit_risk_ml

# Type check
mypy backend/app --ignore-missing-imports

# Tests
pytest backend/tests -v
```

### TypeScript / Next.js

```bash
cd frontend
npm run typecheck
npm run lint
npm run build
```

### ML model changes

If you modify the feature engineering pipeline or model training code, you
**must** re-run the full training loop and include the updated
`training_results.json` output in your PR so reviewers can verify the
performance hasn't regressed below the CI gates (ROC-AUC > 0.70, KS > 0.40).

```bash
python -m credit_risk_ml.features.synthetic_data_generator \
  --n 25000 --out /tmp/pr_test_data.csv

python -m credit_risk_ml.training.train_default_model \
  --data /tmp/pr_test_data.csv --output-dir /tmp/pr_artifacts
```

## Code conventions

- **Python**: follow PEP 8, use type hints everywhere, docstrings on public classes/functions
- **TypeScript**: strict mode, no `any` except in explicitly justified cases
- **Commits**: conventional commits format: `feat:`, `fix:`, `docs:`, `refactor:`, `test:`
- **Tests**: new backend functionality must ship with a corresponding pytest test
- **No fabricated metrics**: never add benchmark numbers to docs that weren't produced by actually running the code

## Financial domain accuracy

This codebase models real credit risk concepts (PD, LGD, EAD, Basel IRB,
CCAR stress testing). If you spot a finance-domain inaccuracy — a formula
that's wrong, a concept that's mis-labelled, a regulatory reference that's
off — please open an issue and flag it. Getting this right matters.

## Code of Conduct

See [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md).
