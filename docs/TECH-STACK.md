# Tech Stack — Payment Gateway Audit Framework

_Last updated: 2026-04-16. Owner: Engineer (PAY-12)._

---

## Chosen Stack

| Layer | Technology | Version | Notes |
|---|---|---|---|
| Language | Python | 3.11+ | Industry standard for security tooling |
| CLI framework | Typer | 0.12+ | Type-hint-based, built on Click |
| Audit runner | Custom check registry | — | Plugin pattern; checks are Python functions |
| Report output | JSON + HTML + Markdown | — | Machine-readable and human-readable findings |
| Documentation site | MkDocs + Material theme | 1.5+ | Markdown-native, GitHub Pages compatible |
| Testing | pytest + pytest-cov | 8+ | Standard Python test runner |
| Linting / formatting | ruff | 0.4+ | Replaces flake8, isort, black in one tool |
| Type checking | mypy | 1.9+ | Static analysis for check correctness |
| CI/CD | GitHub Actions | — | Native GitHub integration, free tier sufficient |
| Containerization | Docker | 24+ | Reproducible audit environments |
| Config / schema | YAML + JSON Schema | — | Human-readable check definitions and config |
| Packaging | pyproject.toml (PEP 517) | — | `pip install` and `pipx install` distribution |

---

## Rationale

### Python 3.11+

Python is the lingua franca of security tooling. The ecosystem includes `cryptography`, `pyOpenSSL`, `requests`, `paramiko`, `scapy`, `bandit`, and dozens of compliance-adjacent libraries. Writing audit checks as Python functions means external contributors from the security community can participate with zero ramp-up. Python 3.11+ is chosen for performance improvements (15–60% faster than 3.9) and improved error messages.

**Rejected alternatives:**

- **TypeScript/Node.js** — Strong CLI tooling, but the security library ecosystem is thin. Writing checks against TLS configuration, certificate chains, or key management APIs is significantly more verbose without native C bindings.
- **Go** — Excellent for shipping compiled security binaries (Trivy, Nuclei). Overkill here: our framework is methodology-first. A single-binary CLI is a nice-to-have, not a requirement. Go's verbosity slows methodology iteration speed.
- **Rust** — Bleeding edge for this use case. Strong security community interest but too few audit-domain libraries. Rejected per our "prefer boring technology" principle.

### Typer (CLI framework)

Typer provides a clean, type-annotated API over Click. Subcommands map directly to the audit runner's surface (`audit run`, `audit report`, `audit list-checks`). It generates help text and shell completion from type hints with zero boilerplate.

### Custom check registry (plugin pattern)

Each audit check is a Python function decorated with `@check(domain="tokenization", standard="PCI-DSS-v4.0", req="3.5")`. A central `CheckRegistry` discovers checks at import time (or via entry points for third-party plugins). This enables:

- Selective execution by domain, standard, or severity
- Easy contribution of new checks without touching the core runner
- Isolated unit testing of individual checks

### MkDocs + Material theme

Documentation is the primary deliverable. MkDocs converts Markdown files to a polished static site with zero configuration. The Material theme provides search, versioning tabs, mermaid diagram support, and a professional look used by major open-source projects (FastAPI, Pydantic, Terraform providers). Deploys to GitHub Pages via a single GitHub Actions workflow.

### GitHub Actions for CI/CD

GitHub Actions is free for public repositories and has a large marketplace of pre-built security actions (CodeQL, dependency review, secret scanning). The project will use matrix builds to test against Python 3.11 and 3.12, plus a separate workflow for publishing the docs site on merge to `main`.

---

## Trade-offs

| Decision | Considered | Rejected Because |
|---|---|---|
| Python over Go | Go produces single-binary CLIs | Methodology iteration speed matters more than distribution simplicity; `pipx` solves distribution |
| Typer over Click | Click is more mature | Typer's type-hint API is cleaner for maintainability; it is built on Click and compatible |
| MkDocs over Sphinx | Sphinx is more feature-rich | Sphinx has heavier configuration and RST friction; Markdown-native is essential for audit checklists |
| ruff over black + flake8 | Black + flake8 + isort is the legacy stack | ruff is 100x faster, maintained by Astral, and consolidates four tools into one |
| pytest over unittest | unittest is stdlib | pytest's fixture model and assertion introspection are essential for check unit tests |
| GitHub Actions over CircleCI/Jenkins | Jenkins has richer plugins | We have no existing infrastructure; GitHub Actions has zero setup cost and native GitHub integration |
| Docker for reproducibility | Virtual environments alone | Payment environment audits often require specific OS tooling (openssl, nmap). Docker ensures the audit environment is identical across machines |

---

## Key Dependencies

| Package | Purpose | License |
|---|---|---|
| `typer` | CLI framework for `audit` command | MIT |
| `rich` | Terminal output formatting (tables, progress bars) | MIT |
| `pydantic` | Check result schema validation | MIT |
| `cryptography` | TLS/certificate inspection checks | Apache-2.0 / BSD |
| `requests` + `httpx` | HTTP-level audit checks | Apache-2.0 / BSD |
| `PyYAML` | Check configuration and report templates | MIT |
| `jinja2` | HTML report rendering | BSD |
| `pytest` | Test runner | MIT |
| `pytest-cov` | Coverage reporting | MIT |
| `ruff` | Linting and formatting | MIT |
| `mypy` | Static type checking | MIT |
| `mkdocs-material` | Documentation site | MIT |

All dependencies are MIT or Apache-2.0 licensed — compatible with open-source distribution.

---

## Infrastructure Notes

- **No database required.** Findings are written to structured JSON files and optionally uploaded as CI artifacts. No persistent state is needed.
- **No paid services required.** The full CI/CD pipeline runs on GitHub Actions free tier. MkDocs deploys to GitHub Pages. No board approval needed for infrastructure spend.
- **Docker image** will be published to GitHub Container Registry (ghcr.io) — free for public repos.

---

## Open Questions

None at this time. All tooling decisions fall within free-tier or open-source constraints.

---

_Generated by Engineer. References: [PAY-12](/PAY/issues/PAY-12), [VISION.md](VISION.md)._
