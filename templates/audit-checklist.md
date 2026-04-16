# Audit Checklist Template

Reusable template for all Payment Gateway Audit Framework domain checklists. Copy this file into the appropriate `methodology/<domain>.md` location and populate each item.

> **Usage:** Replace `[DOMAIN]` with the audit domain name (e.g., Authentication, Tokenization). Delete this preamble block before publishing the domain checklist.

---

## Domain: [DOMAIN]

**Version:** 1.0  
**Last reviewed:** YYYY-MM-DD  
**PCI DSS version:** v4.0  
**OWASP reference:** Payment Security Top 10 / ASVS v4.0  
**Status:** Draft | In Review | Approved

### Scope

Brief description of what this domain covers and why it matters for payment platform security and compliance.

### Prerequisites

- List any dependencies (access, credentials, systems) required before starting this checklist
- Minimum access level needed by the auditor
- Tools required (e.g., Burp Suite, curl, log viewer)

---

## Checklist Items

Each item follows the structure below. Minimum **15 items** per domain checklist.

---

### Item Structure (reference)

| Field | Description |
|-------|-------------|
| **ID** | Unique identifier: `<DOMAIN-PREFIX>-<NNN>` (e.g., `AUTH-001`, `TOK-001`) |
| **Title** | Short, action-oriented name |
| **Description** | What the control is and why it matters |
| **Risk Rating** | `Critical` / `High` / `Medium` / `Low` |
| **Evidence Required** | What the auditor must collect to evaluate this item |
| **Testing Procedure** | Step-by-step instructions for the auditor |
| **PCI DSS Mapping** | Requirement ID(s) from PCI DSS v4.0 (e.g., `Req 8.3.6`) |
| **OWASP Mapping** | OWASP reference(s) — Payment Security, ASVS, or Top 10 (e.g., `ASVS 2.1.1`, `PS-Top10 #3`) |
| **Pass Criteria** | Objective condition that constitutes a pass |
| **Fail Criteria** | Objective condition that constitutes a fail |
| **Notes** | Auditor observations, compensating controls, or follow-up actions |

---

## Example Items

The following three example items illustrate the expected level of detail. Remove these before publishing the domain checklist.

---

### [DOMAIN-PREFIX]-001 — Multi-Factor Authentication Enforcement

**Title:** Multi-Factor Authentication for Administrative Access  
**Risk Rating:** `Critical`

**Description:**  
All administrative and privileged accounts accessing the payment platform must require multi-factor authentication (MFA). Single-factor authentication for privileged access is a critical control gap under PCI DSS v4.0 and enables account takeover attacks.

**Evidence Required:**
- Screenshot or export of IAM/SSO configuration showing MFA enforcement policy
- List of all admin/privileged role definitions
- Sample login audit log entries showing MFA challenge events
- Any MFA exemption records and their approval trail

**Testing Procedure:**
1. Enumerate all accounts with administrative or elevated privileges via IAM/SSO admin console.
2. Verify that the MFA enforcement policy applies to 100% of those accounts — check for group exceptions or bypass rules.
3. Attempt to log in to a test admin account without providing an MFA factor; confirm the login is rejected.
4. Review audit logs for the past 30 days — confirm no admin logins without a corresponding MFA event.
5. Confirm that hardware tokens or app-based TOTP (not SMS-only) are accepted methods where required by policy.

**PCI DSS Mapping:** Req 8.4.2, Req 8.4.3

**OWASP Mapping:** ASVS 2.2.1, ASVS 2.2.2, PS-Top10 #2 (Broken Authentication)

**Pass Criteria:**  
MFA is enforced for all privileged accounts with no bypass rules, and audit logs confirm no unauthenticated admin sessions in the review period.

**Fail Criteria:**  
Any privileged account found that does not require MFA, or audit logs showing admin access without an MFA event.

**Notes:**  
_[Auditor notes, compensating controls, or remediation reference]_

---

### [DOMAIN-PREFIX]-002 — Session Token Expiry and Rotation

**Title:** API Session Tokens Expire and Rotate After Use  
**Risk Rating:** `High`

**Description:**  
Session tokens and refresh tokens issued by the payment platform must expire after a defined idle/absolute timeout and must be rotated on privilege escalation. Long-lived tokens dramatically increase the blast radius of a credential leak.

**Evidence Required:**
- Token expiry configuration (access token TTL, refresh token TTL)
- Code or configuration showing token invalidation on logout
- Sample token payload (JWT claims or session record) demonstrating `exp` and rotation logic
- Evidence of token revocation list or blocklist mechanism

**Testing Procedure:**
1. Obtain a valid session token via the API authentication endpoint.
2. Wait past the configured idle timeout (or advance system clock in a test environment) and confirm the token is rejected with `401 Unauthorized`.
3. Issue a logout request and confirm the token is immediately invalidated — re-using it must return `401`.
4. Elevate privileges (e.g., add a permission) using the same session; confirm a new token is issued or the old one is rotated.
5. Inspect JWT `exp` claim or session expiry metadata; confirm it matches policy.

**PCI DSS Mapping:** Req 8.2.8, Req 8.6.1

**OWASP Mapping:** ASVS 3.3.1, ASVS 3.3.3, PS-Top10 #2 (Broken Authentication)

**Pass Criteria:**  
Tokens expire at or before the configured TTL, are invalidated on logout, and are rotated on privilege change.

**Fail Criteria:**  
Tokens remain valid after logout, do not expire within policy limits, or are not rotated on privilege escalation.

**Notes:**  
_[Auditor notes, compensating controls, or remediation reference]_

---

### [DOMAIN-PREFIX]-003 — Sensitive Data in Logs

**Title:** Payment Card Data Absent from Application Logs  
**Risk Rating:** `Critical`

**Description:**  
Primary Account Numbers (PAN), CVV/CVCs, full magnetic stripe data, and PIN blocks must never appear in application logs, debug output, or error traces. Log exfiltration is one of the most common paths to PCI DSS scope expansion.

**Evidence Required:**
- Log aggregation configuration and log retention policy
- Sample log output from transaction processing, error handling, and debug paths
- Evidence of log masking or redaction rules
- Results of automated log scanning tool (grep/regex scan for card-number patterns)

**Testing Procedure:**
1. Execute a representative set of transactions (successful payment, declined payment, refund, 3DS challenge) against a test/staging environment.
2. Capture all logs generated: application, middleware, access, and error logs.
3. Run a pattern scan across captured logs for 16-digit PAN patterns (`\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b`), CVV patterns (3–4 digit sequences near "cvv"/"cvc"), and track data (`;` delimited strings with `=`).
4. Verify that any card-adjacent fields are masked (e.g., `4111****1111`) and that CVV is fully absent.
5. Repeat for error/exception logs triggered by invalid card input.

**PCI DSS Mapping:** Req 3.3.1, Req 3.3.2, Req 10.3.1

**OWASP Mapping:** ASVS 7.1.1, ASVS 7.1.2, PS-Top10 #6 (Sensitive Data Exposure)

**Pass Criteria:**  
No unmasked PAN, CVV, or track data found in any log path after pattern scan.

**Fail Criteria:**  
Any PAN, CVV, track data, or PIN found unmasked in any log.

**Notes:**  
_[Auditor notes, compensating controls, or remediation reference]_

---

## Scoring Summary

Complete this table after finishing all items.

| Risk Rating | Total Items | Pass | Fail | Not Applicable |
|-------------|-------------|------|------|----------------|
| Critical    |             |      |      |                |
| High        |             |      |      |                |
| Medium      |             |      |      |                |
| Low         |             |      |      |                |
| **Total**   |             |      |      |                |

**Overall Result:** Pass | Fail | Conditional Pass (describe compensating controls below)

**Compensating Controls:**  
_[Describe any accepted compensating controls for failed items]_

**Open Findings:**  
_[List item IDs requiring remediation with target dates]_

---

## References

- [PCI DSS v4.0 Requirements](https://www.pcisecuritystandards.org/document_library/)
- [OWASP ASVS v4.0](https://owasp.org/www-project-application-security-verification-standard/)
- [OWASP Payment Security Top 10](https://owasp.org/www-project-top-10-for-payment-security/)
- [PCI DSS cross-reference matrix](../methodology/pci-dss-mapping.md)
- [OWASP cross-reference](../methodology/owasp-mapping.md)
