# Contributing Guide

Welcome to the Payment Gateway Audit Framework. This guide covers how to contribute methodology, tooling, and reference content.

## Who Contributes What

| Role | Primary contributions |
|------|----------------------|
| Engineer | Automated checks (`checks/`), audit runner CLI (`runner/`) |
| Security Engineer | Security domain checklists, threat model, OWASP/PCI DSS mappings |
| Technical Writer | Documentation, README, guides, API reference |
| Product Owner | Roadmap, backlog grooming, acceptance criteria |
| Code Reviewer | Code review for all PRs touching `checks/` and `runner/` |

## Development Setup

> The automation layer (M3) is not yet implemented. Setup instructions for the runner CLI will be added when it ships.

For methodology and documentation contributions, no special tooling is required:

1. Clone or pull the repository
2. Open `methodology/<domain>/checklist.md` for the domain you're working on
3. Make your changes
4. Follow the commit and PR conventions below

## Repository Layout

```
PaymentGatewayAuditFramework/
├── README.md
├── docs/
│   ├── ARCHITECTURE.md       # Framework architecture
│   ├── CONTRIBUTING.md       # This file
│   ├── TECH-STACK.md         # Technology decisions (pending PAY-5)
│   ├── THREAT-MODEL.md       # Threat model (pending PAY-9)
│   └── SECURITY-REVIEW.md   # Security review (pending PAY-10)
├── methodology/
│   ├── architecture/         # Architecture & Scalability checklist
│   ├── competitive/          # Competitive Functional Analysis checklist
│   ├── merchant/             # Merchant Operational Excellence checklist
│   ├── white-label/          # White Label Model Readiness checklist
│   └── security/             # Security & Compliance checklist
├── checks/                   # Machine-executable audit scripts (M3)
├── reference-architectures/  # Payment gateway patterns (M4)
└── templates/                # Deliverable templates
```

## Commit Conventions

Use [Conventional Commits](https://www.conventionalcommits.org/) format:

```
<type>: <short description>
```

Types: `feat`, `fix`, `docs`, `chore`, `refactor`, `test`, `perf`

Examples:
```
docs: add PCI DSS v4.0 requirement mapping to security checklist
feat: add automated JWT algorithm confusion check
fix: correct TPS capacity threshold in architecture checklist
```

Rules:
- Lowercase after colon
- No period at end
- Under 72 characters
- Reference Paperclip issue ID in commit body when applicable (e.g., `PAY-12`)

## Pull Request Workflow

### When to open a PR

**Requires a PR** (via branch + review):
- New audit checks (`checks/`)
- Changes to the audit runner CLI (`runner/`)
- New or significantly revised methodology checklists
- New reference architecture documentation

**Direct-to-main OK**:
- Typos and minor wording fixes
- Comment-only changes
- Minor doc fixes (must reference issue in commit body)

### Opening a PR

1. Branch from `main`: `pay-<N>/<short-description>` (e.g., `pay-15/add-tokenization-checks`)
2. Push your branch and open a PR with this template:

```markdown
## What changed
<Brief description>

## Why
<Motivation and context>

## How to test
<Steps to verify — for checks, include a test command>

## Related
Closes PAY-N
```

3. Set the originating Paperclip issue to `in_review`
4. @-mention @Code Reviewer and @Product Owner on the issue with the PR link

### Review requirements

- **Code Reviewer** must approve: correctness, security, code style, simplicity
- **Product Owner** must approve: intent match, scope, acceptance criteria
- CI must pass (when configured)
- No force pushes to main

### Merge

The PR author merges after all approvals: `gh pr merge <number> --merge`

## Checklist Quality Standards

When authoring or updating methodology checklists:

- Each item must be actionable (starts with a verb: "Verify", "Check", "Confirm")
- Each item must map to at least one PCI DSS v4.0 requirement or OWASP standard
- Each item must have a severity: `Critical`, `High`, `Medium`, or `Low`
- Items must be testable — include the test method (automated / manual / interview)

Example checklist item format:
```markdown
### CHK-SEC-001: Verify JWT algorithm pinning

**Severity**: Critical
**Standards**: PCI DSS v4.0 Req 6.3, OWASP ASVS 3.5.3
**Test method**: Automated
**Description**: Confirm the API rejects tokens signed with `alg: none` or an unexpected algorithm.
**Check**: `checks/security/jwt-algorithm-check.js`
```

## Reference Architecture Contributions (M4)

Each reference architecture document must include:

1. **System overview** — what this pattern is and when to use it
2. **Architecture diagram** — ASCII or linked image
3. **Threat surface map** — STRIDE-based, covering the payment flow
4. **Pre-filled audit checklist** — which methodology items apply and any pattern-specific items
5. **PCI DSS applicability notes** — which requirements are hardest to satisfy for this pattern

## Documentation Standards

- Write for a senior engineer who is not a payment domain expert.
- Define payment-specific terms on first use (e.g., "cascading — the process of automatically retrying a declined transaction through a different gateway").
- Keep a single source of truth per topic. Link, don't copy.
- File paths in docs should be relative to the repository root.

## Getting Help

- For questions about scope or priorities, check the [Paperclip issue tracker](/).
- For security-related questions, @-mention @Security Engineer on the relevant issue.
- For methodology questions, reference [PCI DSS v4.0](https://www.pcisecuritystandards.org/document_library/) or [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/).

---
_Last updated: 2026-04-16 by Technical Writer._
