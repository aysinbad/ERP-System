# ERP Security Architecture

## Document Information
```
Document Name:  ERP Security Architecture
Version:        0.1.0
Status:         Draft
Classification: Authoritative — Security Architecture (platform-wide)
Owner:          Solution Architecture / Security
Date:           2026-07-26
Baseline:       OWASP ASVS 5.0 Level 2 (minimum) + selected Level 3 controls
Related:        Accounting_Posting_Service_Spec.md · ADR-023 (ACC-D central lock) · KI-001/002/003/004 (security-relevant findings)
```

> **Scope & authority:** this document is the **authoritative security architecture** for the ERP platform. It defines *what must be true*; component specs (e.g. the Posting Service Spec) and the future `schema.sql` conform to it. **No implementation code**; no change to business logic, ADRs, RFCs, the Posting Service Specification, Accounting documentation, Test Cases, or Product Owner decisions.
>
> **Assurance target:** **OWASP ASVS 5.0 Level 2 is the minimum baseline for the whole platform.** **Level 3** controls are applied to the high-risk domains: **Accounting, Payments, User Administration, Security Administration, Period Lock, Posting Rules, Audit, Export Documents, File Management.**
>
> **No absolute-security claim.** These controls reduce risk to an acceptable, auditable level; they do not eliminate it. Security is continuous (see §13).

---

## 1. Security Principles

| Principle | Definition | ERP application |
|---|---|---|
| **Zero Trust** | No implicit trust by network location; every request is authenticated, authorised, and validated. | Internal service calls, background workers, and the browser are all treated as untrusted; every hop re-verifies identity + tenant + permission (§3, §5). |
| **Least Privilege** | Every identity (user, service, DB principal) holds the minimum rights needed. | The application DB principal has **no `UPDATE`/`DELETE` on posted ledger tables** and no DDL; users get scoped permissions, not roles-as-god (closes KI-002). |
| **Defence in Depth** | Multiple independent layers; no single control is the only barrier. | UI hides + API authorises + DB constrains + audit records — a bypass at one layer is caught by the next. |
| **Secure by Default** | The safe configuration is the default; opening up is explicit. | Deny-by-default authorisation; least-privilege defaults; MFA required for privileged roles; cookies `Secure`+`HttpOnly`+`SameSite`. |
| **Fail Secure** | On error/uncertainty, deny and preserve integrity rather than proceed. | A failed authz/lock/validation check **rejects** the operation; a failed audit or Outbox write **rolls back** the whole posting transaction (no false success — SP-06). |
| **Immutable Accounting Records** | Legal ledger entries are append-only; correction is by reversal, never edit/delete. | Enforced at DB (permissions + triggers) and service (ACC-A/ACC-C); no soft-delete flag permitted on ledger tables. |
| **Separation of Duties (SoD)** | No single actor can both initiate and approve a sensitive action. | Maker/Checker on payments, period unlock, posting-rule activation, and user/security administration (§5). |

---

## 2. Secure Defaults

The safe configuration is the default; relaxing it is always explicit, justified, and audited (operationalises the *Secure by Default* principle).

- **Every endpoint requires authentication** unless it is explicitly and deliberately marked anonymous.
- **Every new permission defaults to deny** — access is granted, never assumed.
- **Every uploaded file defaults to quarantine** and is released only after validation + malware scan.
- **Every background worker runs with least privilege** — scoped credentials, no ambient admin rights.
- **Every new API is rate limited by default**; limits are relaxed only with justification.
- **Every database object follows least privilege** — the application principal cannot `UPDATE`/`DELETE` posted ledger rows or run DDL.
- **Every outbound integration uses authenticated, encrypted transport** (TLS/mTLS); no plaintext or unauthenticated endpoints.
- **Every new feature must define its Security Acceptance Criteria before implementation** (gate G1, §16).

---

## 3. Trust Boundaries

Each arrow is a **trust boundary**: the downstream side treats upstream input as untrusted until re-validated. Data crossing a boundary is authenticated, authorised, size- and schema-validated, and tenant-scoped.

```
Browser → API → Application → Database → Storage → Email → External Services → Background Workers
```

| Boundary | Untrusted input | Controls at the boundary |
|---|---|---|
| **Browser → API** | all headers, body, cookies, files | TLS 1.3; authN (token); CSRF defence; input validation + allow-lists; payload limits; rate limiting; **`Idempotency-Key`/`X-Correlation-Id` headers authoritative** (Posting Spec §4). |
| **API → Application** | request DTOs | deny-by-default authorisation; tenant resolution (Company/Branch) from the authenticated principal, **never** from client body; step-up for sensitive ops. |
| **Application → Database** | queries, parameters | parameterised queries only (no string SQL); least-privilege DB principal; row-level ownership filters; immutable-ledger constraints. |
| **Database → Storage** | file blobs, exports | server-side encryption at rest; signed, time-limited URLs; no direct public bucket access. |
| **Storage → Email** | generated documents, links | signed URLs (not attachments where avoidable); DLP checks; no secrets/PII beyond policy; SPF/DKIM/DMARC on sending domain. |
| **→ External Services** | third-party APIs (payment, tax, e-invoice) | mTLS/allow-listed endpoints; outbound secrets from a vault; response validation; no PII/secrets in URLs. |
| **→ Background Workers** | queued messages, Outbox events | workers are untrusted callers — they re-authorise and re-scope by tenant; Outbox consumers are **idempotent** (at-least-once delivery); workers never write the legal ledger directly. |

---

## 4. Authentication

- **Password policy (ASVS L2):** minimum length ≥ 12; screen against breached-password lists; no forced periodic rotation without cause; no composition rules that reduce entropy; **passwords stored with a strong adaptive hash (Argon2id/bcrypt)** — never plaintext or reversible (closes KI-001).
- **MFA:** required for all privileged roles (Accounting, Payments, User/Security Admin) and available to all users. Prefer **phishing-resistant MFA (WebAuthn/FIDO2)**; TOTP as fallback. SMS OTP is not accepted for privileged roles.
- **Session management:** short-lived access tokens; server-side session validity; idle + absolute timeouts; cookies `Secure` + `HttpOnly` + `SameSite=Strict`; session fixation prevented (rotate on privilege change); concurrent-session policy configurable.
- **Refresh tokens:** rotating refresh tokens with **reuse detection** — a replayed refresh token invalidates the whole token family and raises a security alert; refresh tokens bound to client/device where feasible.
- **Password reset:** time-limited, single-use, high-entropy tokens delivered out-of-band; no account enumeration (uniform responses); reset invalidates active sessions; reset of a privileged account requires MFA re-assertion.
- **Device trust:** optional device registration; new/unrecognised device triggers step-up MFA; device posture (where available) informs risk-based access. Device trust **augments**, never replaces, per-request authorisation (Zero Trust).

**L3 for high-risk domains:** step-up (re-authentication) before payments, period unlock, posting-rule activation, and security administration.

---

## 5. Authorization

- **RBAC:** roles group permissions; **permissions (not role name strings) gate every action** — server-side, deny-by-default (closes KI-002). UI visibility is UX only and is never a security control.
- **Policy-based authorization (PBAC/ABAC):** contextual policies layer on RBAC — tenant match (Company/Branch), resource ownership, period state, amount thresholds, time-of-day, and risk signals combine into an allow/deny decision.
- **Permission hierarchy:** coarse domain permission (`accounting.post`) → fine action/rule permission (rule-specific `RequiredPermission`) → administrative permission (`accounting.rules.admin`). Higher tiers never implicitly grant lower-tier bypass.
- **Separation of Duties:** conflicting permissions cannot be held by the same user simultaneously (e.g. create supplier + approve its payment); SoD conflicts are defined as policy and enforced at grant time and at run time.
- **Maker / Checker:** sensitive operations require a second authorised approver distinct from the initiator — **payments, period unlock/close, posting-rule activation/deprecation, user creation & permission grants, security-configuration changes**. The approval is audited with both identities.
- **Step-up authentication:** high-risk actions require a fresh authentication factor within a short window, independent of the active session.

**L3:** Maker/Checker + step-up are mandatory (not optional) for the high-risk domains listed in scope.

---

## 6. Multi-tenancy Security

Aligned with the Posting Service Spec tenant-ownership policy.

- **Company isolation:** `CompanyId` is mandatory on aggregate roots; **every query is filtered by the authenticated principal's `CompanyId`** — resolved server-side, never from client input.
- **Branch isolation:** `BranchId`/`BranchScopeId` applies **only to branch-scoped entities**; branch-scoped users see only their branch(es).
- **Cross-company prevention:** child records inherit tenant ownership via foreign keys; **composite FKs (or equivalent constraints) prevent a child in one company from referencing a parent in another**; row-level security (RLS) enforces the tenant predicate at the database.
- **Tenant ownership rules:** shared metadata (e.g. posting rules) is explicitly global-or-company-scoped; a company-specific active rule takes precedence over the global one for that company; different companies' rules never conflict (Posting Spec §3.9).
- **Isolation testing:** cross-tenant access attempts are a standing negative-test and monitoring case (§12, §15).

---

## 7. Data Protection

- **Encryption in transit:** TLS 1.2 minimum, **TLS 1.3 preferred**, HSTS enabled; internal service-to-service traffic encrypted (mTLS where feasible); no plaintext protocols.
- **Encryption at rest:** database TDE + **column/field-level encryption for sensitive fields**; encrypted storage buckets; **encrypted, authenticated backups (AES-GCM)** — never XOR or obfuscation (closes KI-003).
- **Secrets management:** secrets in a dedicated **vault/secret store**, never in source control, config files, logs, or client code; short-lived dynamic credentials where supported.
- **Key rotation:** documented rotation schedule and on-compromise rotation; envelope encryption (data keys wrapped by a rotated master key); old keys retained only for decrypt/restore, then retired.
- **Data classification matrix:** every field/dataset is assigned one of four classifications; the classification drives encryption, masking, logging, export, and retention.

| Classification | Representative ERP data | Encryption | Masking | Logging | Export | Retention |
|---|---|---|---|---|---|---|
| **Public** | Product Catalogue (public listings) | in transit | none | allowed | allowed | standard |
| **Internal** | Journal Entries, Audit Events | in transit + at rest | none (internal) | reference/id only, no full payload | authorised, scoped | **permanent / statutory** (append-only, immutable) |
| **Confidential** | Customer Master, Export Documents | at rest + field-level where sensitive | mask in UI/logs | no sensitive fields in logs | signed-URL, tenant-scoped, Maker/Checker for bulk | per policy + legal hold |
| **Restricted** | Password Hashes, Supplier Banking, API Secrets | strong at rest + field-level; secrets in vault | always masked/omitted | **never logged** | **never exported**; secrets never leave the vault | rotate (secrets); minimal retention |

- **PII handling:** data minimisation; **no secrets or unrestricted PII in JSON payloads** (`InputPayload`, `FrozenInputs`, audit `Detail`, Outbox `Payload`) — allow-listed schemas only (Posting Spec §9); masking in UI/logs; retention and right-to-erasure handled **without** violating immutable-ledger law (erasure applies to non-ledger PII; ledger records are retained per statutory obligation).

---

## 8. API Security

- **Validation:** strict server-side validation of every field; reject-unknown-fields; canonicalise before validating.
- **Allow-lists:** allow-listed values, schemas, MIME types, and rule identifiers — never deny-lists as the primary control.
- **Rate limiting:** per-identity and per-IP limits on authentication and write endpoints; adaptive throttling on anomalies.
- **Replay protection:** `Idempotency-Key` + request-hash prevents duplicate side effects; nonce/timestamp checks on sensitive callbacks.
- **Idempotency:** authoritative `Idempotency-Key` header; identical key + identical request ⇒ stored result; same key + different request ⇒ `ACC_IDEMPOTENCY_CONFLICT` (Posting Spec §4/§7).
- **Correlation IDs:** authoritative `X-Correlation-Id` header; server generates when absent; propagated through app, DB context, Outbox, and logs.
- **API versioning:** URI major-version (`/api/v1/...`); breaking changes require a new version; deprecation window with `Deprecation`/`Sunset` headers (Posting Spec §4.6).
- **Error handling:** typed error codes to the client, full detail server-side only; **no stack traces, SQL, or internal identifiers leaked**; uniform errors to prevent enumeration.
- **Payload limits:** enforced request and per-field size limits; oversized ⇒ rejected (413/`ACC_INVALID_REQUEST`).

---

### HTTP Security Headers

Applied at the hosting/edge layer so they are consistent across every API:
- **HSTS** (`Strict-Transport-Security`) — force HTTPS, long max-age, `includeSubDomains`.
- **Content-Security-Policy** — restrict sources; default-deny; no inline script where avoidable.
- **X-Content-Type-Options: nosniff** — prevent MIME sniffing.
- **X-Frame-Options: DENY** (and CSP `frame-ancestors`) — clickjacking defence.
- **Referrer-Policy** — `no-referrer`/`strict-origin-when-cross-origin`.
- **Permissions-Policy** — disable unused browser features.

> **Header configuration belongs to the hosting layer** (gateway/reverse proxy/edge). **Individual APIs do not override or weaken platform security headers**; a service may only request stricter, never looser.

---

## 9. Software Supply Chain Security

- **Signed packages** — only cryptographically signed packages are consumed; signatures verified in CI.
- **Trusted package feeds** — restore only from approved, private/proxying feeds; **packages are never restored from untrusted sources**.
- **Dependency pinning** — exact versions pinned with lock files; no floating ranges in production builds.
- **SBOM generation** — a Software Bill of Materials is produced for every build artefact and retained.
- **Dependency vulnerability scanning** — SCA on every build; **critical dependency vulnerabilities block release**.
- **Package provenance verification** — verify build provenance/attestations (e.g. SLSA-style) before promotion.
- **NuGet verification** — package signature + source validation; `nuget verify`/signed-only policy enforced.
- **npm verification** — integrity hashes (lockfile), signature/provenance checks, audit gate.
- **Build reproducibility** — deterministic, isolated builds from pinned inputs; artefacts traceable to source commit.
- **CI dependency approval gates** — new or upgraded dependencies pass an approval gate (licence + security review) before merge.

> **Unsigned or tampered packages are rejected.** No build proceeds on a failed signature, provenance, or critical-vulnerability check.

---

## 10. Database Security

- **Least privilege:** the application DB principal holds `INSERT`/`SELECT` on ledger tables, **no `UPDATE`/`DELETE` on posted entries/lines**, and **no DDL**; administrative/migration credentials are separate and vaulted.
- **Immutable journals:** posted `JournalEntries`/`JournalLines`/`PostingSnapshots`/`AuditEvents` are append-only, enforced by **permissions + `INSTEAD OF UPDATE/DELETE` triggers**; correction is reversal only.
- **Ledger protection:** consider database-ledger features (e.g. SQL Server ledger tables) or an application hash chain (per-`CompanyId`) for tamper evidence (closes KI-004).
- **Constraints:** balance, `Debit XOR Credit`, non-negative amounts, unique posting key, one-reversal-per-original, required tenant/source/date columns — enforced in-database, not only in code (Posting Spec §3).
- **SQL injection prevention:** parameterised queries / ORM binding **only**; no dynamic SQL string concatenation; input never used to build identifiers.
- **Row-level ownership:** RLS predicates enforce `CompanyId` (and branch scope) at the database, independent of application filters.

---

## 11. File Security

- **Upload validation:** validate size, extension, and **content type by inspection** (magic bytes), not just the declared header; reject mismatches.
- **Malware scanning:** every uploaded file scanned before it becomes accessible; scanning failure ⇒ quarantine, not accept.
- **Allowed MIME types:** allow-list per feature (e.g. PDF/PNG/JPEG for export attachments); everything else rejected.
- **Signed URLs:** downloads via short-lived, signed URLs scoped to tenant + user; no public/enumerable object paths; no long-lived direct links.
- **Document quarantine:** uploads land in a quarantine area; released to the tenant store only after successful validation + scan; export documents (invoices, packing lists) are generated server-side and integrity-stamped.

---

## 12. Logging and Monitoring

- **Security audit log:** authentication events, authorisation denials, privilege changes, MFA events, step-up, Maker/Checker approvals, secret access, admin actions — **tamper-resistant, append-only**, retained per policy.
- **Operational audit log:** business/operational events and **pre-transaction rejections** (e.g. `PostingRejected`) — distinct from the legal ledger `AuditEvents`, which live inside the posting transaction (Posting Spec §6/§11).
- **Correlation IDs:** every log line carries the `X-Correlation-Id` for end-to-end tracing across API, workers, and Outbox.
- **Alerting:** real-time alerts on brute-force, refresh-token reuse, cross-tenant access attempts, SoD violations, mass export, permission escalation, and audit-chain anomalies; forwarded to SIEM.
- **No secrets in logs:** structured logs; secrets/PII masked or omitted by classification (§7).

---

## 13. Security Incident Response

A defined lifecycle governs security incidents; it draws on the **Security Audit** and **Operational Audit** logs (§12) for detection and evidence.

- **Detection** — alerts from monitoring/SIEM (§12): brute-force, refresh-token reuse, cross-tenant access, SoD violation, audit-chain anomaly, mass export.
- **Triage** — classify severity/impact, assign an incident owner, open an incident record.
- **Containment** — isolate affected accounts/sessions/services; revoke tokens/keys; block source; prevent spread while preserving evidence.
- **Eradication** — remove the root cause (patch, rotate secrets, close the gap).
- **Recovery** — restore service from trusted state; verify integrity (ledger + audit chain) before reopening.
- **Post-incident review** — blameless retrospective; corrective actions tracked to closure; update controls/threat models.
- **Evidence preservation** — capture and retain relevant logs/artefacts under chain-of-custody; do **not** alter the immutable audit trail.
- **Forensic readiness** — logs are time-synchronised, correlation-ID-linked, tamper-evident, and retained long enough to investigate.
- **Communication responsibilities** — defined internal escalation and, where legally required, regulator/customer notification within mandated timelines; a single accountable owner coordinates messaging.

---

## 14. Backup and Disaster Recovery

- **Encrypted backups:** AES-GCM authenticated encryption; keys managed in the vault; **integrity-verified** (no XOR/obfuscation — KI-003).
- **Immutability & isolation:** immutable/WORM backup copies; offsite/second-region copy; backups isolated from production credentials (ransomware resilience).
- **RPO/RTO:** defined per tier (ledger and audit are the highest tier); documented and tested.
- **Restore testing:** periodic restore drills validate recoverability and integrity; results recorded.
- **Ledger continuity:** DR preserves the append-only, tamper-evident properties of the ledger and audit chain.

---

## 15. Security Acceptance Criteria

> Format: **ID · Given / When / Then · ASVS reference · test type.** These are platform-level; component specs add their own (e.g. Posting Spec AC-1…AC-14).

- **SEC-AC-01 AuthN required.** Given an unauthenticated request · When it hits any non-public endpoint · Then it is rejected 401. *(ASVS V2 · integration, security)*
- **SEC-AC-02 Deny-by-default authz.** Given a user without the required permission · When they call a write action · Then 403, regardless of UI. *(ASVS V4 · security)* — closes KI-002.
- **SEC-AC-03 Password storage.** Given a stored credential · When inspected · Then it is an adaptive hash, never plaintext/reversible. *(ASVS V2 · database)* — closes KI-001.
- **SEC-AC-04 MFA for privileged.** Given a privileged role · When authenticating · Then MFA is required. *(ASVS V2 · security)*
- **SEC-AC-05 Refresh-token reuse.** Given a rotated refresh token · When an old one is replayed · Then the token family is revoked and an alert raised. *(ASVS V3 · security)*
- **SEC-AC-06 Tenant isolation.** Given a user of Company A · When requesting a Company B record (any id) · Then 404/403 and no data leak. *(ASVS V4 · security, database)*
- **SEC-AC-07 Cross-company FK.** Given a child referencing a parent in another company · When persisted · Then rejected by constraint. *(ASVS V4 · database)*
- **SEC-AC-08 Ledger immutability.** Given a posted entry · When `UPDATE`/`DELETE` is attempted (app or DB) · Then rejected. *(ASVS V1 · database)*
- **SEC-AC-09 SoD / Maker-Checker.** Given the initiator of a payment · When they attempt to approve it · Then denied; approval requires a distinct authorised checker. *(ASVS V4 · integration)*
- **SEC-AC-10 Step-up.** Given a high-risk action · When invoked without fresh auth · Then step-up is required. *(ASVS V2 · security)*
- **SEC-AC-11 Encryption in transit.** Given any client/service connection · When established · Then TLS ≥1.2 (1.3 preferred), plaintext refused. *(ASVS V9 · integration)*
- **SEC-AC-12 Secrets absent from artefacts.** Given source, config, logs, or payloads · When scanned · Then no secrets/keys present. *(ASVS V6 · security)*
- **SEC-AC-13 Input validation & payload limits.** Given an oversized or malformed request · When received · Then rejected before processing. *(ASVS V5 · security)*
- **SEC-AC-14 SQLi safe.** Given injection payloads in any field · When processed · Then parameterisation neutralises them. *(ASVS V5 · security, database)*
- **SEC-AC-15 File upload safety.** Given a disallowed or malicious file · When uploaded · Then quarantined/rejected, never served. *(ASVS V12 · security)*
- **SEC-AC-16 Signed URLs.** Given a document link · When issued · Then it is short-lived, signed, tenant-scoped; object paths are non-enumerable. *(ASVS V12 · security)*
- **SEC-AC-17 Audit tamper-evidence.** Given the audit chain · When a record is altered/removed · Then verification fails and alerts. *(ASVS V7 · database, security)* — closes KI-004.
- **SEC-AC-18 Error hygiene.** Given a server error · When returned · Then no stack trace/SQL/internal detail leaks; typed code only. *(ASVS V7 · security)*
- **SEC-AC-19 Rate limiting.** Given repeated auth/write attempts · When exceeding limits · Then throttled and alerted. *(ASVS V11 · security)*
- **SEC-AC-20 Backup integrity.** Given a backup · When restored · Then it is AES-GCM encrypted, integrity-verified, and recoverable. *(ASVS V1 · integration)* — closes KI-003.

---

## 16. Security Review Gates

Security is enforced at defined SDLC gates; no high-risk change ships without passing its gate.

| Gate | When | Requirement to pass |
|---|---|---|
| **G1 — Design/Threat Model** | before build of a feature | threat model (STRIDE) for new trust boundaries; security-architecture conformance review. |
| **G2 — Secure Development** | during build | SAST + dependency/SCA scans clean of criticals; secrets-scan clean; peer review incl. security checklist. |
| **G3 — Test** | before release candidate | Security Acceptance Criteria (§15) pass; DAST on the API; tenant-isolation negative tests pass. |
| **G4 — Pre-Production** | before go-live of high-risk domains | independent **penetration test** on Accounting/Payments/Admin; findings remediated or risk-accepted by the owner. |
| **G5 — Operations** | continuous | monitoring/alerting live; incident-response runbook; periodic access reviews; restore-drill evidence; key-rotation adherence. |

L3 domains (scope list) require G1, G4, and G5 without exception.

---

## 17. References

- **OWASP ASVS 5.0** — Application Security Verification Standard (Levels 2 & 3).
- **OWASP** — Top 10, ASVS verification chapters (V1 architecture, V2 authentication, V3 session, V4 access control, V5 validation, V6 cryptography/secrets, V7 logging & error handling, V9 communications, V11 business logic, V12 files).
- **NIST SP 800-63B** — authentication & lifecycle (password/MFA guidance).
- **NIST SP 800-57** — key management.
- **Repository:** Accounting Posting Service Spec (§3 schema, §4 API, §6 flow, §9 security, §11 Outbox) · ADR-023 ACC-D (central period lock) · Known Issues KI-001 (password storage), KI-002 (server-side authorisation), KI-003 (backup encryption), KI-004 (audit tamper-evidence).
