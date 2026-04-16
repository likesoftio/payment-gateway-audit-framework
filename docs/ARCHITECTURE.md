# Audit Framework Architecture

## Overview

The Payment Gateway Audit Framework is a modular, document-driven audit system. It is not a runtime product — its "output" is a structured audit report, not running software. The architecture organizes methodology, automation, and reference content so that any team can execute a complete payment platform audit in under 4 hours per module.

## Components

| Component | Responsibility | Location |
|-----------|---------------|----------|
| Methodology checklists | Structured audit items per domain, mapped to PCI DSS / OWASP | `methodology/<domain>/` |
| Automated checks | Machine-executable scripts that query a target system | `checks/<domain>/` |
| Audit runner CLI | Orchestrates check execution and generates a findings report | `runner/` (M3) |
| Reference architectures | Documented payment gateway patterns with threat surface maps | `reference-architectures/` |
| Deliverable templates | Executive summary, gap analysis, and security scorecard templates | `templates/` |

## Audit Execution Flow

```
1. Scope definition
       │
       ▼
2. Load methodology checklists (per selected domains)
       │
       ▼
3. Run automated checks (runner CLI or manual)
       │
       ▼
4. Merge automated findings with manual checklist results
       │
       ▼
5. Apply reference architecture baseline (choose matching pattern)
       │
       ▼
6. Generate deliverables
   ├── Executive Summary
   ├── Gap Analysis (vs. Corefy / PayAdmit baseline)
   ├── Improvement Backlog (Critical / High / Medium)
   └── Security Score
```

## Audit Domains and Their Inputs

### 1. Architecture & Scalability
- **Inputs**: system architecture docs, deployment manifests (Kubernetes/Docker), DB schema, load-test results
- **Key checks**: single points of failure, balance-calculation locking, HA scheme completeness, acquirer-addition effort estimate
- **PCI DSS mapping**: Requirements 1 (network), 2 (secure config), 6 (secure systems)

### 2. Competitive Functional Analysis
- **Inputs**: feature matrix, routing rule configuration, tokenization API docs
- **Key checks**: BIN/country/amount/currency/time/metadata routing coverage, cascading on hard/soft declines, cross-acquirer token portability
- **Baseline**: Corefy and PayAdmit feature matrices

### 3. Merchant Operational Excellence
- **Inputs**: reconciliation exports, dispute/refund API docs, webhook delivery logs, merchant portal screenshots
- **Key checks**: bank statement matching automation, partial refund support, chargeback handling, webhook retry queue, decline reason analytics
- **PCI DSS mapping**: Requirements 7, 8, 10 (access control, audit logs)

### 4. White Label Model Readiness
- **Inputs**: multi-tenant data model, CSS/JS injection config, RBAC configuration, OpenAPI spec
- **Key checks**: database and cache isolation per tenant, branding override coverage, role permission granularity, API documentation completeness
- **PCI DSS mapping**: Requirements 7 (access control), 3 (cardholder data)

### 5. Security & Compliance
- **Inputs**: codebase (or black-box endpoints), auth configuration, logging config, data masking rules
- **Key checks**: OWASP ASVS compliance, JWT/OAuth vulnerabilities, OWASP Top 10, sensitive field masking in logs
- **PCI DSS mapping**: Requirements 3 (protect stored data), 4 (encrypt in transit), 6 (secure development), 8 (auth), 10 (logging)

## Reference Architecture Patterns (M4)

Five common payment gateway patterns will be documented with architecture diagrams, threat surface maps, and pre-filled audit checklists:

1. **Stripe-like** — hosted fields, direct-to-processor model
2. **Adyen-like** — platform model with sub-merchant onboarding
3. **In-house acquirer** — self-managed acquiring bank connections
4. **PSP aggregator** — multi-PSP routing with cascading
5. **Embedded finance** — payment capabilities embedded in a non-payment product

## API Boundaries

> Note: The API reference will be expanded in M3 when the audit runner CLI ships. At M1, there are no programmatic APIs — all interactions are through the CLI and file system.

### Planned CLI interface (M3)

```
audit-runner run \
  --target <target-system-url-or-manifest> \
  --domains architecture,security,merchant \
  --reference-arch psp-aggregator \
  --output-dir ./reports/
```

### Planned output format

Each run produces a findings directory:

```
reports/<run-id>/
├── findings.json          # Machine-readable findings
├── executive-summary.md   # Rendered executive summary
├── gap-analysis.md        # Gap analysis vs. baseline
├── backlog.csv            # Improvement backlog (exportable to Jira/Linear)
└── security-score.md      # Security scorecard
```

## Check Registry — Extensibility Model

Each audit check is a Python function decorated with `@check(...)`:

```python
from audit_runner import check, Finding, Severity

@check(
    id="SEC-001",
    domain="security",
    standard="PCI-DSS-v4.0",
    req="6.3.2",
    severity=Severity.CRITICAL,
    test_method="automated",
)
def jwt_algorithm_pinning(target: AuditTarget) -> list[Finding]:
    """Verify the API rejects tokens signed with alg:none."""
    ...
```

The `CheckRegistry` discovers checks via:
1. **Static import** — built-in checks in `checks/<domain>/`
2. **Entry points** — third-party plugins installed via pip declare `audit_runner.checks` entry point

This means external teams can ship domain-specific check packages without forking the core runner.

## Deployment Model

The framework runs as a local CLI tool — no servers, no persistent state.

```
pip install payment-audit-runner     # or pipx install for isolated install
audit-runner --help
```

- **Test**: `pytest checks/ runner/ -q` — runs check unit tests against mock system responses
- **Distribute**: `pyproject.toml`-based package, published to PyPI; Docker image available via ghcr.io for teams that want a pre-configured environment
- **Docs**: `mkdocs build` or `mkdocs serve` for local preview; auto-deployed to GitHub Pages on push to `main`

> Technology decisions are documented in [`docs/TECH-STACK.md`](TECH-STACK.md).

## Directory Layout

```
PaymentGatewayAuditFramework/
├── README.md
├── ROADMAP.md
├── mkdocs.yml                      # Docs site config
├── pyproject.toml                  # Package definition (M3)
├── docs/
│   ├── ARCHITECTURE.md             # This file
│   ├── TECH-STACK.md               # Technology decisions (PAY-12)
│   ├── CONTRIBUTING.md
│   ├── THREAT-MODEL.md
│   └── SECURITY-REVIEW.md          # (M2 — Security Engineer)
├── methodology/
│   ├── index.md
│   ├── architecture/checklist.md   # M2
│   ├── competitive/checklist.md    # M2
│   ├── merchant/checklist.md       # M2
│   ├── white-label/checklist.md    # M2
│   └── security/checklist.md       # M2
├── checks/                         # M3 — machine-executable check modules
│   ├── __init__.py
│   ├── architecture/
│   ├── competitive/
│   ├── merchant/
│   ├── white_label/
│   └── security/
├── runner/                         # M3 — CLI and orchestration
│   ├── __init__.py
│   ├── cli.py                      # Typer app — `audit run`, `audit report`
│   ├── registry.py                 # CheckRegistry, @check decorator
│   ├── target.py                   # AuditTarget — wraps the system under audit
│   ├── runner.py                   # Executes checks, aggregates findings
│   └── report.py                   # Renders JSON, Markdown, HTML outputs
├── templates/                      # Jinja2 report templates
│   ├── executive-summary.md.j2
│   ├── gap-analysis.md.j2
│   └── security-score.md.j2
├── reference-architectures/        # M4
│   ├── stripe-like/
│   ├── adyen-like/
│   ├── in-house-acquirer/
│   ├── psp-aggregator/
│   └── embedded-finance/
└── .github/workflows/
    ├── ci.yml                      # Lint, type check, tests
    └── docs-deploy.yml             # MkDocs → GitHub Pages
```

## Key Decisions

| Decision | Context | Rationale |
|----------|---------|-----------|
| Document-first approach | Methodology created before automation | Allows manual audits before CLI is ready; catches methodology gaps early |
| Python as core language | Security tooling ecosystem | Industry standard; check contributors need zero ramp-up |
| Per-domain modularity | Each domain is an independent module | Teams can run a single domain (e.g., security only) without touching others |
| Plugin extensibility via entry points | Third-party check packages | Teams can ship private checks without forking the framework |
| PCI DSS v4.0 as primary standard | Multiple standards available | PCI DSS v4.0 is the most current and commonly required for payment platforms |
| Reference architecture baselines | Gap analysis requires a comparison point | Corefy and PayAdmit are established market leaders, providing a realistic benchmark |
| No persistent state / no database | Audit is a point-in-time assessment | Keeps the tool simple; findings are files, not records |

---
_Last updated: 2026-04-16 by Engineer ([PAY-10](/PAY/issues/PAY-10))._
