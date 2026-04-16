# Threat Model: Payment Platform Audit Framework

**Version:** 1.0  
**Date:** 2026-04-16  
**Author:** Security Engineer  
**Methodology:** STRIDE  
**Status:** Initial draft — no production system exists yet. This model covers the *target audit domain*: payment gateways and platforms that this framework will assess. It serves as the reference threat surface for building audit checklists, automated checks, and reference architectures.

---

## 1. System Overview

### 1.1 Scope

This threat model covers the canonical **payment gateway platform** that this audit framework is designed to evaluate. The audit framework itself is a collection of documentation, checklists, and tooling — not a live payment system — but the threat model must reflect the real attack surface of payment platforms so that audit artifacts are threat-informed.

**In scope:**

- Payment gateway core engine (transaction processing, routing, cascading)
- Tokenization and card vault subsystem
- Key management infrastructure (HSM, KMS)
- Merchant-facing APIs and portal
- Authentication and authorization layer (JWT, OAuth 2.0)
- Webhook and callback delivery
- Logging, monitoring, and audit trail
- Incident response interfaces
- Multi-tenancy and white-label isolation layer

**Out of scope:**

- Card-scheme networks (Visa/Mastercard infrastructure)
- Issuing bank internal systems
- Physical HSM supply chain
- End-user browser/device security

---

### 1.2 Key Components and Trust Boundaries

```
┌───────────────────────────────────────────────────────────────────────┐
│  EXTERNAL TRUST ZONE                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────────────────┐  │
│  │  Merchant    │   │  Cardholder  │   │  Acquirer / Card Scheme  │  │
│  │  Application │   │  Browser     │   │  (Visa, MC, banks)       │  │
│  └──────┬───────┘   └──────┬───────┘   └───────────┬──────────────┘  │
└─────────┼──────────────────┼───────────────────────┼─────────────────┘
          │ HTTPS/API        │ HTTPS                 │ ISO 8583 / REST
          ▼                  ▼                       ▼
┌───────────────────────────────────────────────────────────────────────┐
│  DMZ / API GATEWAY ZONE                                               │
│  ┌──────────────────────────────────────────────────────────────────┐ │
│  │  API Gateway (rate-limiting, TLS termination, WAF)               │ │
│  └──────────────────┬─────────────────────────────────────────────┘  │
└─────────────────────┼──────────────────────────────────────────────── ┘
                      │ mTLS / internal JWT
                      ▼
┌───────────────────────────────────────────────────────────────────────┐
│  APPLICATION TRUST ZONE (PCI DSS CDE — Cardholder Data Environment)  │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────────────┐ │
│  │  Auth Service│  │ Routing      │  │  Tokenization / Card Vault  │ │
│  │  (JWT/OAuth) │  │ Engine       │  │  (HSM-backed)               │ │
│  └──────┬───────┘  └──────┬───────┘  └──────────────┬──────────────┘ │
│         │                 │                          │                │
│  ┌──────▼───────────────────────────────────────────▼──────────────┐ │
│  │  Core Transaction Engine                                         │ │
│  └──────────────────────────────┬───────────────────────────────────┘ │
│                                 │                                     │
│  ┌──────────────┐  ┌────────────▼───────┐  ┌───────────────────────┐ │
│  │  Merchant    │  │  Connector Layer   │  │  Reconciliation &     │ │
│  │  Portal      │  │  (acquirer adapters│  │  Dispute Module       │ │
│  └──────────────┘  └────────────────────┘  └───────────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
                      │ encrypted channel (TLS + signing)
                      ▼
┌───────────────────────────────────────────────────────────────────────┐
│  DATA TRUST ZONE                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐  │
│  │  Primary DB  │  │  Cache Layer │  │  Key Management (HSM/KMS)  │  │
│  │  (encrypted) │  │  (Redis)     │  │                            │  │
│  └──────────────┘  └──────────────┘  └────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
                      │
                      ▼
┌───────────────────────────────────────────────────────────────────────┐
│  OBSERVABILITY ZONE                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────────────────────┐  │
│  │  Audit Log   │  │  SIEM /      │  │  Incident Response         │  │
│  │  (append-    │  │  Monitoring  │  │  Interface                 │  │
│  │   only)      │  │              │  │                            │  │
│  └──────────────┘  └──────────────┘  └────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────┘
```

**Trust boundary crossings (attack surface concentrated here):**
1. External → DMZ/API Gateway: merchant API calls, cardholder payment submissions
2. DMZ → Application Zone: internal service calls (must use mTLS or signed tokens)
3. Application → Data Zone: database queries, cache reads/writes, KMS requests
4. Application → Observability Zone: log writes, metric emissions

---

## 2. Data Flows

| Flow ID | Source | Destination | Data Carried | Classification |
|---------|--------|-------------|--------------|----------------|
| DF-1 | Merchant App | API Gateway | Auth token + payment request | Sensitive |
| DF-2 | Cardholder Browser | API Gateway | PAN, CVV, expiry (PCI-scoped) | Cardholder Data (CHD) |
| DF-3 | API Gateway | Auth Service | Credentials / JWT | Sensitive |
| DF-4 | Auth Service | Core Engine | Signed authorization claim | Internal |
| DF-5 | Core Engine | Routing Engine | Transaction params (no PAN) | Internal |
| DF-6 | Core Engine | Tokenization Vault | PAN + context | CHD |
| DF-7 | Tokenization Vault | KMS | Encryption key request | Highly Sensitive |
| DF-8 | Routing Engine | Connector Layer | Tokenized transaction | Internal |
| DF-9 | Connector Layer | Acquirer | ISO 8583 / REST (CHD or token) | CHD / Regulated |
| DF-10 | Core Engine | DB | Transaction record (masked CHD) | Sensitive |
| DF-11 | Core Engine | Cache | Session state, routing cache | Sensitive |
| DF-12 | Core Engine | Audit Log | Sanitized transaction log | Sensitive |
| DF-13 | Webhook Service | Merchant App | Event notification + HMAC sig | Sensitive |

---

## 3. Attack Surface Enumeration

| Surface ID | Surface | Protocol | Exposure |
|------------|---------|----------|----------|
| AS-1 | Merchant REST API | HTTPS | External |
| AS-2 | Cardholder payment form / 3DS redirect | HTTPS | External (high-value) |
| AS-3 | Admin / Merchant Portal | HTTPS + Auth | External (authenticated) |
| AS-4 | Webhook endpoints (merchant-side) | HTTPS outbound | Internal → External |
| AS-5 | Acquirer connector endpoints | HTTPS/ISO 8583 | External (partner) |
| AS-6 | Internal service-to-service APIs | mTLS / gRPC | Internal |
| AS-7 | KMS / HSM management interface | TLS + hardware auth | Internal (restricted) |
| AS-8 | Database and cache layer | TCP (encrypted) | Internal |
| AS-9 | Log aggregation pipeline | Syslog / Kafka | Internal |
| AS-10 | CI/CD pipeline and secrets manager | HTTPS | DevOps (privileged) |
| AS-11 | Incident response tooling | HTTPS | Internal + On-call |

---

## 4. STRIDE Threat Analysis

### 4.1 Authentication Flows

**Component:** Auth Service, JWT issuance, OAuth 2.0 flows, session management

| Threat ID | Category | Threat | Likelihood | Impact | Risk | Mitigation |
|-----------|----------|--------|------------|--------|------|------------|
| T-A1 | **Spoofing** | Forged or stolen JWT used to impersonate merchant/admin | High | Critical | **Critical** | Short JWT TTL (≤15 min), RS256/ES256 signing (no HS256 with shared secret), token binding, refresh token rotation |
| T-A2 | **Spoofing** | OAuth authorization code interception (redirect URI mismatch) | Medium | High | **High** | Strict redirect URI allowlist, PKCE mandatory for public clients |
| T-A3 | **Tampering** | JWT payload modification (none-algorithm attack, weak HS256 key) | Low | Critical | **High** | Reject "alg: none"; enforce asymmetric signing; validate `alg` header server-side |
| T-A4 | **Repudiation** | No audit trail of authentication events (logins, token grants, revocations) | Medium | High | **High** | Immutable auth event log; include IP, user agent, timestamp, outcome |
| T-A5 | **Information Disclosure** | JWT contains PAN or other CHD in claims | Medium | Critical | **Critical** | Audit all JWT claims; forbid CHD in tokens; use opaque references |
| T-A6 | **Denial of Service** | Brute-force or credential stuffing against auth endpoints | High | High | **Critical** | Rate limiting per IP and per account; CAPTCHA on repeated failures; account lockout with alerting |
| T-A7 | **Elevation of Privilege** | Privilege escalation via JWT role claim manipulation | Medium | Critical | **Critical** | Server-side RBAC; never trust client-supplied roles without re-validation |
| T-A8 | **Spoofing** | Weak or reused API keys for merchant authentication | High | High | **Critical** | Cryptographically random keys (≥256 bits); per-merchant key rotation; key scoping |

### 4.2 Tokenization Subsystem

**Component:** Card Vault, token issuance, de-tokenization, HSM interface

| Threat ID | Category | Threat | Likelihood | Impact | Risk | Mitigation |
|-----------|----------|--------|------------|--------|------|------------|
| T-TK1 | **Spoofing** | Unauthorized de-tokenization request (internal service impersonation) | Medium | Critical | **Critical** | mTLS on all vault endpoints; service identity certificates; de-tokenization audit log |
| T-TK2 | **Tampering** | Token-to-PAN mapping table manipulation | Low | Critical | **High** | DB integrity controls; append-only token records; HSM-signed mappings where feasible |
| T-TK3 | **Information Disclosure** | Token leaks PAN information (format-preserving tokens that encode card data) | Medium | Critical | **Critical** | Use random/opaque tokens, not format-preserving encoding; token must not be derivable to PAN |
| T-TK4 | **Information Disclosure** | Vault backup exposes unencrypted CHD | Medium | Critical | **Critical** | Encrypt backups with separate KMS key; test restore procedures |
| T-TK5 | **Elevation of Privilege** | Overly broad token scope allows cross-merchant de-tokenization | Medium | Critical | **Critical** | Tenant-scoped tokens; strict tenant isolation enforced at vault layer |
| T-TK6 | **Denial of Service** | Vault service overwhelmed by high-volume de-tokenization requests | Medium | High | **High** | Rate limiting per service; circuit breakers; async token resolution where latency permits |
| T-TK7 | **Repudiation** | No record of which system de-tokenized a given PAN | Medium | High | **High** | Immutable de-tokenization log: requester, timestamp, token ID, purpose code |

### 4.3 Key Management

**Component:** KMS, HSM, encryption key lifecycle

| Threat ID | Category | Threat | Likelihood | Impact | Risk | Mitigation |
|-----------|----------|--------|------------|--------|------|------------|
| T-KM1 | **Information Disclosure** | Key exfiltration from software KMS (plaintext keys in memory/disk) | Medium | Critical | **Critical** | HSM for root keys; FIPS 140-2 Level 3 minimum; keys never leave HSM in plaintext |
| T-KM2 | **Tampering** | Key rotation failure leaves expired keys in active use | Medium | High | **High** | Automated key rotation with dual-control; rotation events logged and alerted |
| T-KM3 | **Elevation of Privilege** | Unrestricted KMS API allows any service to decrypt any key | Medium | Critical | **Critical** | Key usage policies: per-purpose encryption (DEK, KEK, signing key segregation); access scoped by service identity |
| T-KM4 | **Spoofing** | KMS API impersonation by compromised internal service | Low | Critical | **High** | mTLS + IAM policies on KMS endpoint; out-of-band key ceremony for root key generation |
| T-KM5 | **Denial of Service** | KMS unavailability causes total transaction processing failure | Medium | Critical | **Critical** | KMS HA design (multi-AZ); local key caching with short TTL for DEKs; graceful degradation plan |
| T-KM6 | **Repudiation** | No audit trail of key usage (who used which key to decrypt what) | Medium | High | **High** | KMS access logs with service identity; log forwarded to SIEM |

### 4.4 Data-in-Transit

**Component:** TLS configuration, certificate management, internal service mesh

| Threat ID | Category | Threat | Likelihood | Impact | Risk | Mitigation |
|-----------|----------|--------|------------|--------|------|------------|
| T-DT1 | **Information Disclosure** | Weak TLS configuration allows MITM or downgrade attack | Medium | Critical | **Critical** | TLS 1.2 minimum (TLS 1.3 preferred); disable SSLv3, TLS 1.0, RC4, NULL ciphers; HSTS enforced |
| T-DT2 | **Tampering** | Unsigned webhook payloads allow replay or injection | High | High | **Critical** | HMAC-SHA256 signature on all webhook events; include timestamp; verify on merchant side |
| T-DT3 | **Information Disclosure** | Internal service traffic unencrypted (east-west) | Medium | High | **High** | mTLS on all internal APIs; service mesh with automatic certificate rotation |
| T-DT4 | **Spoofing** | Expired or self-signed certificates accepted by internal services | Medium | High | **High** | Certificate pinning or SPIFFE/SPIRE for service identity; automated cert renewal (ACME/cert-manager) |
| T-DT5 | **Repudiation** | Acquirer communication lacks message signing | Medium | High | **High** | Sign acquirer API requests with platform key; verify acquirer responses; retain request/response pairs |
| T-DT6 | **Denial of Service** | TLS handshake flood exhausts connection pool | Medium | High | **High** | Rate limiting at load balancer; connection throttling; DDoS protection upstream |

### 4.5 Logging and Monitoring

**Component:** Audit log pipeline, SIEM, log masking controls

| Threat ID | Category | Threat | Likelihood | Impact | Risk | Mitigation |
|-----------|----------|--------|------------|--------|------|------------|
| T-LM1 | **Information Disclosure** | PAN, CVV, or full card data logged in plaintext | High | Critical | **Critical** | Mandatory data masking before log write: PAN → first6+last4 only; CVV must never be logged; automated log scanning to detect leaks |
| T-LM2 | **Tampering** | Log records modified or deleted to cover tracks | Medium | High | **High** | Append-only log storage; write once/read many (WORM); cryptographic log chaining (hash chains or signed segments) |
| T-LM3 | **Repudiation** | Insufficient logging means breach cannot be reconstructed | Medium | High | **High** | Log all: auth events, transaction lifecycle, admin actions, config changes, key usage; retain for PCI DSS minimum 12 months |
| T-LM4 | **Information Disclosure** | Log aggregation pipeline transmits sensitive data unencrypted | Medium | High | **High** | Encrypt log streams (TLS); restrict access to log aggregator; apply masking before shipping |
| T-LM5 | **Denial of Service** | Log flood (log injection) overwhelms SIEM and masks real events | Medium | Medium | **Medium** | Rate limit log ingestion per source; alert on abnormal log volume; separate audit log from application log |
| T-LM6 | **Elevation of Privilege** | Excessive log data enables correlation attacks on cardholder behavior | Low | High | **Medium** | Minimize CHD in logs to what is strictly necessary; anonymize where possible; data retention minimization |

### 4.6 Incident Response Surfaces

**Component:** Incident response tooling, admin interfaces, break-glass access

| Threat ID | Category | Threat | Likelihood | Impact | Risk | Mitigation |
|-----------|----------|--------|------------|--------|------|------------|
| T-IR1 | **Elevation of Privilege** | Break-glass admin access used outside incident context | Medium | Critical | **Critical** | Just-in-time (JIT) privileged access; MFA required; automated session recording; all break-glass events alerted to security team |
| T-IR2 | **Spoofing** | Attacker uses incident response credentials to gain persistent access | Low | Critical | **High** | Temporary credentials with short TTL for break-glass; credential rotation after each use; dual approval for production access |
| T-IR3 | **Repudiation** | No record of actions taken during incident response | Medium | High | **High** | Immutable session recording (terminal/API session replay); mandatory incident ticket linking to all admin actions |
| T-IR4 | **Information Disclosure** | Incident communication channels expose CHD or security findings | Medium | High | **High** | Use encrypted channels (Signal, private Slack with DLP); restrict incident channel membership; sanitize reports before sharing |
| T-IR5 | **Tampering** | Incident playbooks can be modified to weaken response | Low | High | **Medium** | Version-control playbooks with signed commits; access-controlled to security team |
| T-IR6 | **Denial of Service** | Incident response tooling unavailable during an actual incident | Medium | High | **High** | IR tooling in separate, isolated infrastructure from production; offline runbooks maintained |

### 4.7 Multi-Tenancy and Merchant Isolation

**Component:** Tenant isolation layer, RBAC, white-label configuration

| Threat ID | Category | Threat | Likelihood | Impact | Risk | Mitigation |
|-----------|----------|--------|------------|--------|------|------------|
| T-MT1 | **Information Disclosure** | Cross-tenant data leakage via shared cache or DB query without tenant filter | High | Critical | **Critical** | Tenant ID enforced at ORM/query level; row-level security in DB; cache key namespacing by tenant |
| T-MT2 | **Elevation of Privilege** | Merchant sub-user accesses another merchant's data via RBAC misconfiguration | Medium | Critical | **Critical** | Centralized RBAC policy engine; tenant context always validated server-side; test with automated tenant-boundary tests |
| T-MT3 | **Spoofing** | White-label CSS/JS injection introduces malicious code into cardholder payment form | Medium | Critical | **Critical** | Strict Content Security Policy (CSP); sandbox white-label JS; validate CSS against allowlist; PCI DSS SAQ-A compliance for hosted form |

---

## 5. Risk Summary

| Risk Level | Count | Threat IDs |
|------------|-------|------------|
| **Critical** | 13 | T-A1, T-A5, T-A6, T-A7, T-A8, T-TK1, T-TK3, T-TK5, T-KM1, T-KM3, T-DT1, T-DT2, T-LM1, T-IR1, T-MT1, T-MT2, T-MT3 |
| **High** | 17 | T-A2, T-A3, T-A4, T-TK2, T-TK4, T-TK6, T-TK7, T-KM2, T-KM4, T-KM5, T-KM6, T-DT3, T-DT4, T-DT5, T-DT6, T-LM2, T-LM3, T-LM4, T-IR2, T-IR3, T-IR4, T-IR6 |
| **Medium** | 5 | T-LM5, T-LM6, T-IR5, T-MT2 (overlap), see full table |
| **Low** | 1 | T-TK2 (initial rating Low–High, see table) |

> **Critical findings (16 distinct threats) must be addressed before any system goes into production scope for PCI DSS assessment.**

---

## 6. Top Priority Mitigations (Critical Only)

| Priority | Threat ID | Recommended Control | PCI DSS Req. | OWASP |
|----------|-----------|--------------------|--------------|-|
| P1 | T-LM1 | Automated PAN masking in all log pipelines; block CVV logging at framework level | Req. 3.3, 10.3 | A9 |
| P2 | T-A1, T-A8 | Short-lived JWT (RS256/ES256), per-merchant API key scoping, key rotation | Req. 8.2, 8.3 | A07 |
| P3 | T-TK3, T-TK5 | Opaque random tokens; strict tenant isolation at vault; no cross-tenant de-tokenization | Req. 3.4, 3.5 | — |
| P4 | T-KM1, T-KM3 | HSM for root keys; per-purpose key policies; no plaintext key export | Req. 3.5, 3.6 | — |
| P5 | T-MT1 | Tenant ID enforced at every DB query; row-level security; automated tenant-leak tests | Req. 7.2 | A01 |
| P6 | T-IR1 | JIT privileged access with MFA; session recording; break-glass alerting | Req. 7.3, 10.2 | A04 |
| P7 | T-A5, T-A7 | JWT claims audit; server-side RBAC; forbid CHD in tokens | Req. 3.3, 7.1 | A07, A01 |
| P8 | T-DT1, T-DT2 | TLS 1.2+ only; HSTS; HMAC-signed webhooks with timestamp | Req. 4.2, 6.4 | A02 |
| P9 | T-MT3 | CSP for white-label forms; sandboxed JS; hosted-form PCI SAQ-A path | Req. 6.4 | A03 |

---

## 7. Audit Checklist Mapping

The following audit checklist domains directly address the threats above. These map to the audit methodology (Goal M2):

| Audit Domain | Threat IDs Covered | Priority |
|--------------|-------------------|----------|
| Authentication & API Security | T-A1 to T-A8 | Critical |
| Tokenization & Card Vault | T-TK1 to T-TK7 | Critical |
| Key Management | T-KM1 to T-KM6 | Critical |
| Data-in-Transit / TLS | T-DT1 to T-DT6 | High |
| Logging & Monitoring | T-LM1 to T-LM6 | Critical |
| Incident Response | T-IR1 to T-IR6 | High |
| Multi-Tenancy & White Label | T-MT1 to T-MT3 | Critical |

---

## 8. Follow-up Issues Required

The following issues should be created to drive remediation and deeper audit checklist development:

1. **[Critical] Build Authentication Security Audit Checklist** — JWT/OAuth hardening checks per T-A1 through T-A8
2. **[Critical] Build Tokenization & Key Management Audit Checklist** — CHD protection checks per T-TK1/TK5, T-KM1/KM3
3. **[Critical] Build Logging & Data Masking Audit Checklist** — PAN masking, audit trail, WORM storage per T-LM1/LM2
4. **[High] Build Data-in-Transit Security Audit Checklist** — TLS config, mTLS, webhook signing per T-DT1 through T-DT6
5. **[High] Build Multi-Tenancy Isolation Audit Checklist** — Tenant leak prevention, RBAC per T-MT1/MT3
6. **[High] Build Incident Response Audit Checklist** — JIT access, session recording, playbooks per T-IR1 through T-IR6

---

## 9. Revision History

| Version | Date | Author | Change |
|---------|------|--------|--------|
| 1.0 | 2026-04-16 | Security Engineer | Initial threat model — covers all six audit domains, 35+ threats rated by STRIDE |

---

_This document feeds M2 — Core Methodology per `docs/VISION.md`. Update when architecture decisions are finalized in `docs/ARCHITECTURE.md`._
