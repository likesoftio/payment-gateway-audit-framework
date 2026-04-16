# Design System — Payment Gateway Audit Framework

_Owner: Engineer (PAY-3). Last updated: 2026-04-16._

This document covers the visual language for three surfaces:
1. **Documentation site** — MkDocs + Material theme
2. **CLI terminal output** — Rich library styling
3. **HTML audit reports** — Jinja2 report templates

---

## Color Palette

### Semantic colors (all surfaces)

| Token | Hex | Usage |
|-------|-----|-------|
| `severity-critical` | `#ef4444` (red-500) | Critical findings in CLI, reports, docs admonitions |
| `severity-high` | `#f97316` (orange-500) | High severity findings |
| `severity-medium` | `#eab308` (yellow-500) | Medium severity findings |
| `severity-low` | `#22c55e` (green-500) | Low severity / passed checks |
| `finding-pass` | `#22c55e` (green-500) | Check passed |
| `finding-fail` | `#ef4444` (red-500) | Check failed |
| `finding-warn` | `#eab308` (yellow-500) | Check warned |
| `finding-info` | `#6366f1` (indigo-500) | Informational finding |

### Documentation site (Material for MkDocs)

| Token | Value | Usage |
|-------|-------|-------|
| `primary` | `indigo` (#6366f1) | Navigation, links, headings |
| `accent` | `indigo` (#6366f1) | Interactive elements, inline code highlights |
| `background-light` | `#ffffff` | Page background (light mode) |
| `background-dark` | `#0d1117` | Page background (dark mode) |
| `surface-light` | `#f8f9fa` | Card / admonition background |
| `surface-dark` | `#161b22` | Card / admonition background (dark) |

### CLI output (Rich)

| Usage | Rich style |
|-------|-----------|
| Section headers | `bold white` |
| Critical finding | `bold red` |
| High finding | `bold orange1` |
| Medium finding | `bold yellow` |
| Low / passed | `bold green` |
| Informational | `bold cyan` |
| Muted / metadata | `dim` |
| Domain label | `bold blue` |

---

## Typography

### Documentation site

| Role | Font | Notes |
|------|------|-------|
| Body | System sans-serif (Roboto via Material) | Default Material theme |
| Code / CLI samples | `JetBrains Mono`, `Fira Code`, `monospace` | Use `pymdownx.highlight` for syntax color |
| Headings | Roboto (700 weight) | Auto-sized by Material h1–h4 |

### HTML reports

| Element | Style |
|---------|-------|
| Report title | `font-size: 2rem; font-weight: 700; color: #1e293b` |
| Section heading | `font-size: 1.25rem; font-weight: 600; color: #334155` |
| Finding title | `font-size: 1rem; font-weight: 600` |
| Body text | `font-size: 0.875rem; color: #475569; line-height: 1.6` |
| Code blocks | `font-family: 'JetBrains Mono', monospace; font-size: 0.8rem` |

---

## Finding Severity Badges

Standard badge format used in CLI, HTML reports, and checklists:

| Severity | Badge text | Color |
|----------|-----------|-------|
| CRITICAL | `[CRITICAL]` | Red |
| HIGH | `[HIGH]` | Orange |
| MEDIUM | `[MEDIUM]` | Yellow |
| LOW | `[LOW]` | Green |
| INFO | `[INFO]` | Indigo |
| PASS | `[PASS]` | Green |

HTML badge class names: `.badge-critical`, `.badge-high`, `.badge-medium`, `.badge-low`, `.badge-pass`.

---

## CLI Output Patterns

### Scan summary

```
Payment Gateway Audit Runner v0.1.0

Target: https://api.example-payment.com
Domains: security, architecture, tokenization
Checks: 24 total

 Running  [████████████████────────]  18/24  SEC-009: JWT algorithm pinning

 CRITICAL  SEC-001  JWT algorithm accepts 'alg:none'
 HIGH      SEC-007  TLS 1.0 still accepted on /api/v1
 PASS      SEC-012  HTTPS enforced on all endpoints
 MEDIUM    ARCH-003  No rate limiting on /api/v1/charges
 ...

Summary
  ✗ 2 critical    ✗ 3 high    ⚠ 5 medium    ✓ 14 passed

Report written to: reports/2026-04-16_143022/
```

### Check progress format

```
  Running  check-id  short description
  PASS     check-id  short description
  FAIL     check-id  short description [evidence one-liner]
  WARN     check-id  short description
```

---

## Report Template Structure

Every HTML report includes these sections in order:

1. **Header** — framework name, version, run date, target
2. **Executive Summary** — total findings by severity, overall risk level
3. **Domain Sections** — one section per audited domain
   - Domain summary (total checks, pass/fail)
   - Findings table (id, severity, title, standard reference)
   - Finding detail (description, evidence, remediation)
4. **Gap Analysis** — comparison to Corefy / PayAdmit baseline
5. **Security Score** — domain-by-domain score grid
6. **Remediation Backlog** — sorted by severity, with effort estimate
7. **Appendix** — raw check output, system information

---

## Spacing Scale (HTML reports only)

Base unit: `4px`

| Token | Value | Usage |
|-------|-------|-------|
| `xs` | `4px` | Icon padding, tight labels |
| `sm` | `8px` | Inline element gaps |
| `md` | `16px` | Card padding, section gaps |
| `lg` | `24px` | Section padding |
| `xl` | `32px` | Page section margins |
| `2xl` | `48px` | Major section breaks |

---

## Brand Guidelines

- **Name**: Payment Gateway Audit Framework (full); `pgaf` (CLI command / package name, pending final decision)
- **Tone**: Professional, precise, direct. Security tooling is read under pressure — no marketing language.
- **Icons**: Use security/shield iconography sparingly. Prefer text labels over icons in CLI output.
- **Logo**: None at M1. A simple text logo in the docs nav (`PGAF`) is sufficient.
- **Docs URL**: `https://likesoftio.github.io/payment-gateway-audit-framework`

---

_Generated by Engineer. References: [PAY-3](/PAY/issues/PAY-3), [TECH-STACK.md](TECH-STACK.md)._
