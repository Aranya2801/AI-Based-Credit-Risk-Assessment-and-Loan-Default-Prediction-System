# Security Policy

## Supported versions

| Version | Supported |
|---------|-----------|
| 1.x     | ✅ Yes    |

## Reporting a vulnerability

**Do not open a public GitHub issue for security vulnerabilities.**

Email **security@your-org.example.com** with:

1. A description of the vulnerability and its potential impact
2. Steps to reproduce (proof-of-concept code if applicable)
3. The affected component(s)

You will receive an acknowledgement within 48 hours and a detailed response
within 7 business days.

We follow responsible disclosure: we ask that you give us 90 days to patch
the issue before any public disclosure.

## Security design notes

### Authentication
- All API endpoints (except `/health` and `/`) require a signed JWT issued
  by the `/api/v1/auth/login` endpoint.
- Tokens are signed with HS256 using a `SECRET_KEY` that **must** be set via
  environment variable or secrets manager — never committed to source control.
- Access tokens expire in 30 minutes; refresh tokens in 7 days.

### Authorisation (RBAC)
- Five roles: `admin`, `risk_analyst`, `loan_officer`, `auditor`, `viewer`
- Role permissions are enforced at the endpoint level via the
  `require_permission()` FastAPI dependency.
- See `backend/app/core/security.py` for the full role-permission matrix.

### Secrets management
- In development: `.env` file (never committed — see `.gitignore`)
- In production: AWS Secrets Manager (referenced in `infra/terraform/modules/ecs/main.tf`),
  injected into ECS task environment at runtime via `secrets:` in the task definition

### Transport security
- All inter-service communication in the Kubernetes/ECS stack is encrypted
  in transit (Redis TLS in the ElastiCache module, RDS encryption enabled)
- The ALB/Ingress enforces HTTPS; HTTP is redirected

### Audit trail
- Every credit decision (scoring result, fraud flag, manual override) is
  written to the `audit_logs` table, which is insert-only and retained for
  2,555 days (7 years) by default, meeting typical regulatory retention
  requirements for automated credit decisions

### Dependency security
- The CD pipeline builds Docker images with SBOM and SLSA Level 3 provenance
  attestations
- The `security-scan.yml` workflow runs weekly Trivy container scans and
  Bandit Python SAST, with results uploaded to GitHub Security tab as SARIF

### What this platform is NOT

This is an **open-source reference architecture**, not a regulated financial
product. If you are using or adapting this platform for production lending,
you are responsible for:
- Regulatory compliance (ECOA, FCRA, GDPR/CCPA as applicable)
- Model risk management and validation procedures
- Fair lending analysis with qualified legal counsel
- Data security and PII handling
