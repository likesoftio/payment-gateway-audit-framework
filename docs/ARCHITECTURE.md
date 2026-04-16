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

## Deployment Model

The framework runs as a local CLI tool — no servers required for the core audit workflow.

- **Build**: `npm run build` (or `make build`) — compiles TypeScript checks to JavaScript (M3)
- **Test**: `npm test` — runs check unit tests against mock system responses
- **Distribute**: published as an npm package or Docker image for teams without Node.js

> Technology decisions are documented in `docs/TECH-STACK.md` (pending PAY-5).

## Key Decisions

| Decision | Context | Rationale |
|----------|---------|-----------|
| Document-first approach | Methodology created before automation | Allows manual audits before CLI is ready; catches methodology gaps early |
| Per-domain modularity | Each domain is an independent module | Teams can run a single domain (e.g., security only) without touching others |
| PCI DSS v4.0 as primary standard | Multiple standards available | PCI DSS v4.0 is the most current and commonly required for payment platforms |
| Reference architecture baselines | Gap analysis requires a comparison point | Corefy and PayAdmit are established market leaders, providing a realistic benchmark |

---
_Architecture subject to revision pending tech stack decisions (PAY-5) and system architecture design (PAY-6)._
_Last updated: 2026-04-16 by Technical Writer._
