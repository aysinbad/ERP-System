# ERP Security Test Specification

## 1. Document Information
```
Document Name:  ERP Security Test Specification
Version:        0.1.0
Status:         Draft
Classification: Authoritative — Security Test Specification
Owner:          Solution Architecture / Security / QA
Date:           2026-07-27
Derives-from:   ERP_Security_Architecture.md (v0.1.0) · ERP_Threat_Model.md (v0.1.0)
                · Secure_Coding_Standard.md (v0.1.0) · Accounting_Posting_Service_Spec.md (v1.0.0)
                · Accepted ADRs (ACC-A/B/C/D = ADR-020/021/022/023) · Known_Issues.md
                · AGENTS.md · DEVELOPMENT_RULES.md
Baseline:       OWASP ASVS 5.0 (L2 minimum · L3 for high-risk domains) · OWASP API Security
                · OWASP Cheat Sheets · NIST SSDF
```

> **Authority & limits.** This document converts the **already-approved** security design into executable verification. It is the authoritative source for security **test cases** only. It **MUST NOT** introduce new requirements, controls, business rules, or acceptance criteria; every test traces to an existing SEC-AC, Posting AC, threat ID, Secure-Coding rule, or Known Issue. It contains **no implementation instructions** and **no application code**. It modifies no prototype, business logic, ADR, RFC, Known Issue, or Accounting document.
>
> **Control-status honesty (inherited).** The only running system today is the prototype; the controls under test are **Designed/Planned, not built**. Defining a test here does **not** imply the control exists, and **does not close any Known Issue** (KI-001/002/003/004 and related remain **open**). Test *Status* values describe test-artefact readiness, never control implementation.

---

## 2. Scope

**In scope:** verification design for Backend APIs · domain/application services · SQL Server database code and migrations · frontend security · background workers · Outbox/messaging · file processing · email ingestion · external integrations (payment/tax/e-invoice/shipping) · backup/DR · CI/CD security gates · and AI-generated code (tested identically to human code, per Secure Coding Standard §1/§20).

**Assurance target:** ASVS L2 platform-wide minimum; **L3** for the high-risk domains (Accounting, Payments, User Administration, Security Administration, Period Lock, Posting Rules, Audit, Export Documents, File Management).

**Out of scope:** control implementation; test automation code; business-rule correctness (Accounting docs own that); physical/cloud-provider security; endpoint security beyond device-trust signals; the prototype's internal implementation (treated as evidence of weakness only).

---

## 3. Test Strategy

| Layer | Objective | Primary sources | Environment |
|---|---|---|---|
| **Unit** | Rule-level invariants: balance, snapshot immutability, validation, encoding. | Posting Spec AC-2/8/12/13 · Secure Coding §3/§4/§7 | isolated, mocked |
| **Integration** | End-to-end control behaviour across boundaries (authz, tenant, posting tx). | SEC-AC-01/02/06 · Posting AC-3/10 | wired services + DB |
| **Database** | DB-enforced constraints, append-only permissions/triggers, RLS, composite FK. | SEC-AC-07/08/14 · Posting Spec §3 | SQL Server |
| **API** | Versioning, idempotency/correlation headers, error contract, CORS, pagination. | Sec-Arch §8 · Secure Coding §11 | API surface |
| **Authorization** | Deny-by-default, object-level, SoD/Maker-Checker, admin separation. | SEC-AC-02/09/10 · AC-7 | integration |
| **Tenant Isolation** | Cross-company read/write, IDOR/BOLA, composite-FK, RLS. | SEC-AC-06/07 | integration + DB |
| **File Security** | Magic-byte validation, quarantine, malware scan, signed URLs, zip-bomb. | SEC-AC-15/16 | sandbox |
| **Email Ingestion** | SPF/DKIM/DMARC, scan/quarantine, sandboxed parse (Planned controls). | Threat T-10/11/12 | sandbox |
| **Background Workers** | Re-auth/re-scope, least privilege, no direct ledger write. | Sec-Arch §3 · Secure Coding §13 | worker harness |
| **Outbox** | Commit-then-publish, idempotent consumer, ordering. | Posting Spec §11 · AC-3 | messaging harness |
| **Concurrency** | Race conditions, optimistic concurrency, exactly-one posting. | AC-14 · Secure Coding §14 | concurrency harness |
| **Performance Security** | Rate limiting, payload limits, DoS/expensive-query resistance. | SEC-AC-13/19 · T-26 | load harness |
| **Disaster Recovery** | AES-GCM backups, restore drills, immutability/offsite. | SEC-AC-20 · T-24/33/34 | DR drill |
| **Penetration Testing** | Independent adversarial testing of high-risk domains. | Sec-Arch §16 gate G4 | staging |
| **Regression Security** | Prevent reintroduction of any mitigated/closed finding. | all SEC-AC · KI list | CI |

> Test types map to Security Review Gates G1–G5 (Sec-Arch §16): unit/DB/API/authz at G2–G3; DAST + tenant negatives at G3; pen test at G4; monitoring/restore drills at G5.

---

## 4. Security Acceptance Criteria Mapping

| SEC-AC | Requirement | Source document | Threat IDs | Required Test IDs | Status |
|---|---|---|---|---|---|
| SEC-AC-01 | AuthN required on non-public endpoints | Sec-Arch §15 / §4 | T-01 | SEC-T-001, SEC-T-006 | Defined |
| SEC-AC-02 | Deny-by-default server authz (closes KI-002) | Sec-Arch §15 / §5 | T-04, T-05 | SEC-T-010, SEC-T-011 | Defined |
| SEC-AC-03 | Adaptive password hash (closes KI-001) | Sec-Arch §15 / §4 | T-01 | SEC-T-002 | Defined |
| SEC-AC-04 | MFA for privileged roles | Sec-Arch §15 / §4 | T-01, T-03 | SEC-T-003 | Defined |
| SEC-AC-05 | Refresh-token reuse detection | Sec-Arch §15 / §4 | T-02 | SEC-T-004, SEC-T-007 | Defined |
| SEC-AC-06 | Tenant isolation | Sec-Arch §15 / §6 | T-06, T-07 | SEC-T-012, SEC-T-020, SEC-T-021, SEC-T-072 | Defined |
| SEC-AC-07 | Cross-company FK rejected | Sec-Arch §15 / §6 | T-06 | SEC-T-022 | Defined |
| SEC-AC-08 | Ledger immutability | Sec-Arch §15 / §10 · AC-1 | T-14 | SEC-T-030, SEC-T-031, SEC-T-071 | Defined |
| SEC-AC-09 | SoD / Maker-Checker | Sec-Arch §15 / §5 · AC-7 | T-16, T-27 | SEC-T-013, SEC-T-014, SEC-T-044 | Defined |
| SEC-AC-10 | Step-up for high-risk actions | Sec-Arch §15 / §4 | T-03 | SEC-T-005 | Defined |
| SEC-AC-11 | Encryption in transit (TLS) | Sec-Arch §15 / §7 | T-11, T-29 | SEC-T-162 | Defined |
| SEC-AC-12 | Secrets absent from artefacts | Sec-Arch §15 / §7 | T-21, T-32 | SEC-T-060, SEC-T-061 | Defined |
| SEC-AC-13 | Input validation & payload limits | Sec-Arch §15 / §8 | T-26 | SEC-T-081, SEC-T-151, SEC-T-025 | Defined |
| SEC-AC-14 | SQL-injection safe | Sec-Arch §15 / §10 | T-08 | SEC-T-070 | Defined |
| SEC-AC-15 | File upload safety | Sec-Arch §15 / §11 | T-09, T-10, T-12 | SEC-T-080, SEC-T-082, SEC-T-083, SEC-T-085, SEC-T-091 | Defined |
| SEC-AC-16 | Signed, tenant-scoped URLs | Sec-Arch §15 / §11 | T-28 | SEC-T-084 | Defined |
| SEC-AC-17 | Audit tamper-evidence (closes KI-004) | Sec-Arch §15 / §10 · §12 | T-18 | SEC-T-051, SEC-T-052 | Defined |
| SEC-AC-18 | Error hygiene (no leakage) | Sec-Arch §15 / §8 | T-08, T-32 | SEC-T-161 | Defined |
| SEC-AC-19 | Rate limiting | Sec-Arch §15 / §8 | T-25, T-26 | SEC-T-150, SEC-T-152 | Defined |
| SEC-AC-20 | Backup integrity (closes KI-003) | Sec-Arch §15 / §14 | T-24, T-33, T-34 | SEC-T-120, SEC-T-121, SEC-T-122 | Defined |

> All 20 SEC-AC are covered by ≥1 executable test. *Status = Defined* means the test artefact exists in §7; it makes no claim the control is built.

---

## 5. Posting Acceptance Criteria Mapping

| Posting AC | Requirement | Threat IDs | Required Tests |
|---|---|---|---|
| AC-1 | Append-only journal (UPDATE/DELETE rejected) | T-14 | SEC-T-030, SEC-T-031 |
| AC-2 | Balanced journal (Σdr=Σcr, ≥2 lines) | T-15 | SEC-T-033 |
| AC-3 | Atomic posting (all-or-nothing persistence set) | T-15, T-19 | SEC-T-034, SEC-T-110 |
| AC-4 | Idempotency (key + request hash) | T-13 | SEC-T-036 |
| AC-5 | Duplicate prevention (unique posting key) | T-13 | SEC-T-037 |
| AC-6 | Central period-lock | T-16 | SEC-T-038 |
| AC-7 | Server-side authorisation (server builds lines) | T-04, T-05 | SEC-T-010, SEC-T-043 |
| AC-8 | Immutable snapshots | T-14 | SEC-T-039 |
| AC-9 | Reversal instead of delete | T-14 | SEC-T-032 |
| AC-10 | Audit in same transaction | T-18 | SEC-T-050 |
| AC-11 | False-success prevention | T-15 | SEC-T-035 |
| AC-12 | Account validity | T-15 | SEC-T-040 |
| AC-13 | Traceability fields present | T-18 | SEC-T-041 |
| AC-14 | Concurrency safety (exactly one) | T-13 | SEC-T-042 |

> All 14 Posting AC covered by ≥1 executable test.

---

## 6. Threat Coverage Matrix

| Threat | Risk | Security Test IDs | Required Level | Residual Risk | Verification Owner |
|---|---|---|---|---|---|
| T-01 | High | SEC-T-002, SEC-T-003 | L3 | Medium | IAM/Security |
| T-02 | High | SEC-T-004, SEC-T-007 | L3 | Medium | IAM/Security |
| T-03 | Medium | SEC-T-003, SEC-T-005 | L2 | Medium | IAM/Security |
| T-04 | High | SEC-T-010, SEC-T-011, SEC-T-015 | L3 | Medium | Platform/API |
| T-05 | Critical | SEC-T-010, SEC-T-043, SEC-T-101 | L3 | High | Platform/API |
| T-06 | Critical | SEC-T-020, SEC-T-021, SEC-T-022, SEC-T-024, SEC-T-072, SEC-T-100 | L3 | Medium | Data/DB |
| T-07 | High | SEC-T-012, SEC-T-023 | L3 | Medium | Platform/API |
| T-08 | High | SEC-T-070, SEC-T-161 | L3 | Low | Data/DB |
| T-09 | High | SEC-T-080, SEC-T-085, SEC-T-091 | L3 | Medium | Platform/Files |
| T-10 | High | SEC-T-091 | L3 | High | Integrations |
| T-11 | Medium | SEC-T-090, SEC-T-162 | L2 | Medium | Integrations |
| T-12 | High | SEC-T-082, SEC-T-083, SEC-T-092 | L3 | Medium | Integrations |
| T-13 | High | SEC-T-036, SEC-T-037, SEC-T-042 | L3 | Low | Accounting Eng |
| T-14 | Critical | SEC-T-030, SEC-T-031, SEC-T-032, SEC-T-071 | L3 | Low | Data/DB |
| T-15 | High | SEC-T-034, SEC-T-035, SEC-T-040 | L3 | Low | Accounting Eng |
| T-16 | High | SEC-T-038, SEC-T-014 | L3 | Medium | Accounting Eng |
| T-17 | High | SEC-T-044 | L3 | Medium | Accounting/Security |
| T-18 | High | SEC-T-050, SEC-T-051, SEC-T-052 | L3 | Medium | Security/Data |
| T-19 | Medium | SEC-T-110, SEC-T-111 | L2 | Low | Platform/Integrations |
| T-20 | Low | SEC-T-112 | L2 | Low | Integrations |
| T-21 | High | SEC-T-060 | L3 | Medium | DevSecOps |
| T-22 | High | SEC-T-130, SEC-T-131, SEC-T-133 | L2 | Medium | DevSecOps |
| T-23 | High | SEC-T-132 | L2 | Medium | DevSecOps |
| T-24 | High | SEC-T-120 | L3 | Medium | SRE |
| T-25 | High | SEC-T-152 | L3 | Medium | Platform/Security |
| T-26 | High | SEC-T-081, SEC-T-150, SEC-T-151 | L2 | Medium | Platform/SRE |
| T-27 | High | SEC-T-013, SEC-T-014 | L3 | Medium | Security/Accounting |
| T-28 | High | SEC-T-084 | L3 | Low | Platform/Files |
| T-29 | Medium | SEC-T-142, SEC-T-162 | L2 | Medium | Integrations |
| T-30 | High | SEC-T-140, SEC-T-141 | L3 | Medium | Integrations |
| T-31 | Low | SEC-T-160 | L2 | Low | Platform |
| T-32 | High | SEC-T-061, SEC-T-161 | L3 | Medium | Platform/Security |
| T-33 | High | SEC-T-122 | L3 | Medium | SRE |
| T-34 | High | SEC-T-121 | L3 | Medium | SRE |

> Every threat T-01…T-34 maps to ≥1 executable test. Risk, Residual, and Owner are carried unchanged from the Threat Model (§7/§10/§11); this document does not re-rate them.

---

## 7. Security Test Catalogue

> Executable Given/When/Then tests. *Status of every control under test is Designed/Planned (not built).* Priority: **P0** (blocking, high-risk domain), **P1** (blocking baseline), **P2** (important). Automation Candidate marks tests suitable for CI automation once code exists.

### 7.1 Authentication

**SEC-T-001 — AuthN required on protected endpoints**
Purpose: verify unauthenticated requests are rejected. Threats: T-01. SEC-AC: 01. Posting AC: —. Prereq: an endpoint marked non-public.
Given an unauthenticated client · When it calls any non-public endpoint · Then the response is 401 and no business data is returned.
Expected: 401, no body leakage. Failure: 2xx/partial data ⇒ fail. Evidence: request/response capture, endpoint list. Priority: P1. Owner: IAM/Security. Automation: Yes.

**SEC-T-002 — Password stored as adaptive hash (KI-001)**
Purpose: no plaintext/reversible credential. Threats: T-01. SEC-AC: 03. Prereq: a provisioned account.
Given a stored credential · When the credential store is inspected · Then only an Argon2id/bcrypt hash is present, never plaintext/reversible.
Expected: adaptive hash with salt/params. Failure: plaintext/reversible ⇒ fail (KI-001 not remediated). Evidence: DB field dump (masked), hash format. Priority: P0. Owner: IAM/Security. Automation: Yes.

**SEC-T-003 — MFA required for privileged roles**
Purpose: privileged auth enforces MFA. Threats: T-01, T-03. SEC-AC: 04. Prereq: a privileged role account.
Given a privileged role · When it authenticates without a second factor · Then access is denied until MFA is satisfied.
Expected: MFA challenge enforced. Failure: privileged access without MFA ⇒ fail. Evidence: auth trace. Priority: P0. Owner: IAM/Security. Automation: Yes.

**SEC-T-004 — Refresh-token reuse detection**
Purpose: replayed refresh token revokes family. Threats: T-02. SEC-AC: 05. Prereq: an issued+rotated refresh token.
Given a rotated refresh token · When an old token from the family is replayed · Then the whole token family is revoked and an alert is raised.
Expected: revocation + alert. Failure: old token still valid ⇒ fail. Evidence: token lifecycle log, alert record. Priority: P0. Owner: IAM/Security. Automation: Yes.

**SEC-T-005 — Step-up before high-risk actions**
Purpose: fresh auth required for high-risk ops. Threats: T-03. SEC-AC: 10. Prereq: authenticated session, a high-risk action (payment, period unlock, rule activation).
Given a high-risk action invoked without fresh authentication · When submitted · Then step-up re-authentication is required before it proceeds.
Expected: step-up enforced. Failure: action proceeds without step-up ⇒ fail. Evidence: auth trace. Priority: P0. Owner: IAM/Security. Automation: Yes.

**SEC-T-006 — Uniform password-reset (no enumeration)**
Purpose: reset responses do not reveal account existence. Threats: T-01. SEC-AC: 01. Prereq: reset endpoint.
Given reset requests for an existing and a non-existing account · When submitted · Then responses are indistinguishable and tokens are single-use, time-limited.
Expected: uniform response. Failure: differential response ⇒ fail. Evidence: response diff. Priority: P1. Owner: IAM/Security. Automation: Yes.

**SEC-T-007 — Session/JWT revocation on logout & privilege change**
Purpose: revoked/rotated sessions cannot be replayed. Threats: T-02. SEC-AC: 05. Prereq: active session.
Given a session revoked or a privilege change · When the prior token is replayed · Then it is rejected.
Expected: 401 on replay. Failure: replay accepted ⇒ fail. Evidence: session store trace. Priority: P1. Owner: IAM/Security. Automation: Yes.

### 7.2 Authorization

**SEC-T-010 — Deny-by-default write authorization (KI-002)**
Purpose: server rejects unauthorized writes regardless of UI. Threats: T-04, T-05. SEC-AC: 02. Posting AC: 7. Prereq: a low-privilege user, a protected write.
Given a user without the required permission · When they call a write action directly (bypassing UI) · Then 403/`ACC_PERMISSION_DENIED`, no side effect.
Expected: 403, no state change. Failure: write succeeds ⇒ fail (KI-002 not remediated). Evidence: request capture, DB before/after. Priority: P0. Owner: Platform/API. Automation: Yes.

**SEC-T-011 — UI-only authorization cannot be bypassed**
Purpose: hidden/disabled UI is not a control. Threats: T-04. SEC-AC: 02. Prereq: an action hidden in UI for the role.
Given an action hidden in the UI · When called directly at the API · Then server authz denies it.
Expected: 403. Failure: success ⇒ fail. Evidence: direct-call capture. Priority: P0. Owner: Platform/API. Automation: Yes.

**SEC-T-012 — Object-level authorization (BOLA)**
Purpose: caller may act only on permitted instances. Threats: T-07. SEC-AC: 06. Prereq: two resources of different owners/tenants.
Given a user authorized for resource A · When they request resource B by id · Then 403/404 with no data leak.
Expected: denied. Failure: B returned ⇒ fail. Evidence: request/response. Priority: P0. Owner: Platform/API. Automation: Yes.

**SEC-T-013 — SoD: create-supplier and approve-its-payment blocked**
Purpose: conflicting duties cannot be held/exercised by one user. Threats: T-27. SEC-AC: 09. Prereq: a user with both grants attempted.
Given the user who created a supplier · When they attempt to approve that supplier's payment · Then the action is denied by SoD policy.
Expected: denied + alert. Failure: same user approves ⇒ fail. Evidence: SoD policy log. Priority: P0. Owner: Security/Accounting. Automation: Yes.

**SEC-T-014 — Maker/Checker distinct approver**
Purpose: approver ≠ initiator on sensitive ops. Threats: T-16, T-27. SEC-AC: 09. Posting AC: 6. Prereq: a payment/period-unlock request.
Given the initiator of a sensitive operation · When they attempt to approve it · Then denied; approval requires a distinct authorised checker.
Expected: denied for initiator; allowed for distinct checker. Failure: self-approval ⇒ fail. Evidence: approval audit (both identities). Priority: P0. Owner: Accounting/Security. Automation: Yes.

**SEC-T-015 — Permission escalation via privileged action**
Purpose: low-role cannot invoke privileged action. Threats: T-04. SEC-AC: 02. Prereq: low-role account, an admin-only action.
Given a low-role user · When invoking a privileged action directly · Then denied and the attempt is alerted.
Expected: 403 + anomaly alert. Failure: escalation succeeds ⇒ fail. Evidence: authz-denial + alert. Priority: P0. Owner: Platform/API. Automation: Yes.

### 7.3 Tenant Isolation

**SEC-T-020 — Cross-tenant read blocked**
Purpose: Company A cannot read Company B. Threats: T-06. SEC-AC: 06. Prereq: two companies with data.
Given a Company A session · When requesting a Company B record by any id · Then 403/404, no leak.
Expected: denied. Failure: B data returned ⇒ fail. Evidence: request/response, tenant filter trace. Priority: P0. Owner: Data/DB. Automation: Yes.

**SEC-T-021 — Cross-tenant write blocked**
Purpose: no write into another tenant. Threats: T-06. SEC-AC: 06. Prereq: two companies.
Given a Company A session · When writing a record scoped to Company B · Then rejected server-side and by RLS.
Expected: denied, no row. Failure: write persists ⇒ fail. Evidence: DB before/after. Priority: P0. Owner: Data/DB. Automation: Yes.

**SEC-T-022 — Composite/cross-company FK rejected**
Purpose: child cannot reference a parent in another company. Threats: T-06. SEC-AC: 07. Prereq: parent in A, child insert referencing it from B.
Given a child referencing a parent in another company · When persisted · Then rejected by constraint.
Expected: FK/constraint violation. Failure: row persists ⇒ fail. Evidence: DB error, constraint def. Priority: P0. Owner: Data/DB. Automation: Yes.

**SEC-T-023 — IDOR by id enumeration**
Purpose: sequential/guessed ids do not expose data. Threats: T-07. SEC-AC: 06. Prereq: known id range.
Given enumerated resource ids · When requested across tenants/owners · Then only authorised instances return; others 403/404.
Expected: no leak. Failure: any foreign row ⇒ fail. Evidence: enumeration log. Priority: P0. Owner: Platform/API. Automation: Yes.

**SEC-T-024 — Client-supplied CompanyId ignored**
Purpose: tenant resolved from identity, not payload. Threats: T-06. SEC-AC: 06. Prereq: request carrying a CompanyId field.
Given a request body/header asserting a different CompanyId · When processed · Then the server ignores it and uses the authenticated principal's tenant.
Expected: server tenant wins. Failure: client CompanyId honoured ⇒ fail. Evidence: resolved-tenant trace. Priority: P0. Owner: Platform/API. Automation: Yes.

### 7.4 Ledger Integrity

**SEC-T-030 — Posted journal UPDATE rejected (KI-008/024)**
Purpose: append-only at DB+service. Threats: T-14. SEC-AC: 08. Posting AC: 1. Prereq: a posted entry.
Given a posted `JournalEntry` · When an `UPDATE` is attempted (app and direct DB) · Then it is rejected at both layers.
Expected: rejection both layers. Failure: update succeeds ⇒ fail (KI-008/024 not remediated). Evidence: DB trigger/permission result, app error. Priority: P0. Owner: Data/DB. Automation: Yes.

**SEC-T-031 — Posted journal DELETE rejected (KI-008/024)**
Purpose: no deletion of posted records. Threats: T-14. SEC-AC: 08. Posting AC: 1. Prereq: a posted entry.
Given a posted entry · When a `DELETE` is attempted (app and direct DB) · Then rejected at both layers; no soft-delete marker permitted.
Expected: rejection; row intact. Failure: delete/soft-delete succeeds ⇒ fail. Evidence: DB result, permission set. Priority: P0. Owner: Data/DB. Automation: Yes.

**SEC-T-032 — Correction is reversal-only**
Purpose: corrections create dated reversal, original untouched. Threats: T-14. SEC-AC: 08. Posting AC: 9. Prereq: a posted entry needing correction.
Given a correction need · When requested · Then a dated reversal entry is created and the original is unchanged.
Expected: reversal created; original intact. Failure: in-place edit ⇒ fail. Evidence: entry + reversal ids. Priority: P0. Owner: Accounting Eng. Automation: Yes.

**SEC-T-033 — Balanced-entry enforcement**
Purpose: Σdebit=Σcredit, ≥2 lines. Threats: T-15. Posting AC: 2. Prereq: a rule draft.
Given a built journal · When Σdebit ≠ Σcredit or <2 lines · Then `ACC_UNBALANCED_ENTRY`, no persistence.
Expected: rejected. Failure: unbalanced entry persists ⇒ fail. Evidence: totals, error code. Priority: P0. Owner: Accounting Eng. Automation: Yes.

**SEC-T-034 — Atomic posting rollback**
Purpose: all-or-nothing persistence set. Threats: T-15. Posting AC: 3. Prereq: fault injection at steps 14–22.
Given a failure during the posting transaction · When it runs · Then none of PostingEvent/JournalEntry/Lines/Snapshot/AuditEvent/IdempotencyRecord/OutboxMessage persists.
Expected: full rollback. Failure: any partial row ⇒ fail. Evidence: DB scan post-fault. Priority: P0. Owner: Accounting Eng. Automation: Yes.

**SEC-T-035 — False-success prevention**
Purpose: no success unless commit. Threats: T-15. Posting AC: 11. Prereq: forced non-commit (e.g. audit write fails).
Given a write that does not commit · When responding · Then no success is returned (typed retriable error instead).
Expected: error, no success. Failure: success without commit ⇒ fail. Evidence: response + tx log. Priority: P0. Owner: Accounting Eng. Automation: Yes.

**SEC-T-036 — Idempotency replay**
Purpose: same key+request returns stored result once. Threats: T-13. Posting AC: 4. Prereq: a completed posting with an Idempotency-Key.
Given the same idempotency key + identical request · When re-sent · Then the stored result returns and no second entry is created; a different hash ⇒ `ACC_IDEMPOTENCY_CONFLICT`.
Expected: single effect. Failure: duplicate entry ⇒ fail. Evidence: idempotency record, entry count. Priority: P0. Owner: Accounting Eng. Automation: Yes.

**SEC-T-037 — Duplicate posting key prevented (KI-009/030)**
Purpose: same business posting key cannot double-post. Threats: T-13. Posting AC: 5. Prereq: e.g. order→invoice already posted.
Given the same posting key (order→invoice, month payroll, invoice payment) · When re-posted · Then `ACC_DUPLICATE_POSTING`.
Expected: rejected. Failure: second commercial invoice/payroll posts ⇒ fail (KI-009/030 not remediated). Evidence: unique-key violation. Priority: P0. Owner: Accounting Eng. Automation: Yes.

**SEC-T-038 — Central period-lock enforced**
Purpose: locked-period posting/reversal blocked centrally. Threats: T-16. Posting AC: 6. Prereq: a locked period.
Given a locked period · When posting/reversal on that date · Then `ACC_PERIOD_LOCKED` via the single PeriodLockService.
Expected: rejected. Failure: posts into locked period ⇒ fail. Evidence: lock-service decision. Priority: P0. Owner: Accounting Eng. Automation: Yes.

**SEC-T-039 — Immutable snapshot (KI-025)**
Purpose: built lines never reflect later live changes. Threats: T-14. Posting AC: 8. Prereq: a rule needing rates/costs, then mutate live values.
Given a snapshot frozen at posting · When live rates/costs change afterward · Then the posted lines still reflect the frozen values.
Expected: frozen values retained. Failure: lines drift with live data ⇒ fail. Evidence: snapshot vs live compare. Priority: P0. Owner: Accounting Eng. Automation: Yes.

**SEC-T-040 — Account validity on build**
Purpose: lines reference active accounts. Threats: T-15. Posting AC: 12. Prereq: an inactive/unknown account.
Given a line account that is inactive/unknown · When building · Then `ACC_ACCOUNT_NOT_FOUND`/`ACC_ACCOUNT_INACTIVE`.
Expected: rejected. Failure: posts to invalid account ⇒ fail. Evidence: error code. Priority: P1. Owner: Accounting Eng. Automation: Yes.

**SEC-T-041 — Traceability fields present**
Purpose: every entry stamped for trace. Threats: T-18. Posting AC: 13. Prereq: any posting.
Given a stored entry · When inspected · Then SourceType/SourceId/RuleId/RuleVersion/EventDate/CorrelationId are all present.
Expected: all fields present. Failure: any missing ⇒ fail. Evidence: entry record. Priority: P1. Owner: Accounting Eng. Automation: Yes.

**SEC-T-042 — Concurrency safety: exactly one entry**
Purpose: concurrent identical postings ⇒ one entry. Threats: T-13. Posting AC: 14. Prereq: two concurrent identical postings.
Given two concurrent identical postings · When racing · Then exactly one entry is created; the other returns the stored result or `ACC_DUPLICATE_POSTING`.
Expected: one entry. Failure: two entries ⇒ fail. Evidence: entry count, timing log. Priority: P0. Owner: Accounting Eng. Automation: Yes.

**SEC-T-043 — Server builds journal lines (no client lines)**
Purpose: API rejects client-supplied final lines. Threats: T-05. SEC-AC: 02. Posting AC: 7. Prereq: a crafted request carrying journal lines.
Given a request supplying final posting lines · When received · Then the server ignores/rejects them and builds lines from event identity + inputs.
Expected: client lines never posted. Failure: client lines honoured ⇒ fail. Evidence: request vs stored lines. Priority: P0. Owner: Platform/API. Automation: Yes.

**SEC-T-044 — Malicious PostingRule activation controlled**
Purpose: rule lifecycle is admin-only + Maker/Checker + audited. Threats: T-17. SEC-AC: 09. Prereq: attempt to register/activate a rule.
Given a non-admin (or single admin without checker) · When activating a PostingRule · Then denied; activation requires `accounting.rules.admin` + distinct checker + audit + version pin.
Expected: denied/controlled. Failure: unilateral activation ⇒ fail. Evidence: rule-change audit. Priority: P0. Owner: Accounting/Security. Automation: Yes.

### 7.5 Audit

**SEC-T-050 — Audit in same transaction**
Purpose: audit commits atomically with posting. Threats: T-18. SEC-AC: 17. Posting AC: 10. Prereq: fault-inject audit write.
Given any posting/reversal · When the audit write fails · Then the whole transaction rolls back (no orphan entry, no orphan audit).
Expected: atomic rollback. Failure: entry without audit (or vice-versa) ⇒ fail. Evidence: tx log. Priority: P0. Owner: Security/Data. Automation: Yes.

**SEC-T-051 — Audit tamper-evidence hash chain (KI-004)**
Purpose: altered/removed audit fails verification. Threats: T-18. SEC-AC: 17. Prereq: an audit chain per CompanyId.
Given the audit chain · When a record is altered/removed · Then chain verification fails.
Expected: verification failure detected. Failure: tampering undetected ⇒ fail (KI-004 not remediated). Evidence: chain verify output. Priority: P0. Owner: Security/Data. Automation: Yes.

**SEC-T-052 — Audit anomaly alerting**
Purpose: chain-verification failure alerts. Threats: T-18. SEC-AC: 17. Prereq: monitoring wired.
Given a detected audit-chain anomaly · When it occurs · Then an alert is raised to SIEM.
Expected: alert raised. Failure: silent ⇒ fail. Evidence: alert record. Priority: P1. Owner: Security/Data. Automation: Yes.

### 7.6 Secrets

**SEC-T-060 — Secrets absent from artefacts**
Purpose: no secrets in source/config/logs/payloads. Threats: T-21. SEC-AC: 12. Prereq: repo + build artefacts + runtime logs.
Given source, config, logs, and payloads · When scanned · Then no secrets/keys are present.
Expected: clean scan. Failure: any secret found ⇒ fail. Evidence: secret-scan report. Priority: P0. Owner: DevSecOps. Automation: Yes.

**SEC-T-061 — Secrets/PII not written to logs**
Purpose: classification-driven masking in logs. Threats: T-32. SEC-AC: 12. Prereq: operations exercising restricted data.
Given operations handling secrets/PII/banking data · When logs are produced · Then restricted data is masked/omitted per classification.
Expected: no restricted data in logs. Failure: leakage ⇒ fail. Evidence: log sample scan. Priority: P0. Owner: Platform/Security. Automation: Yes.

### 7.7 Database

**SEC-T-070 — SQL-injection neutralised**
Purpose: parameterisation defeats injection. Threats: T-08. SEC-AC: 14. Prereq: fields reaching queries.
Given injection payloads in any field · When processed · Then parameterisation neutralises them (no query alteration).
Expected: no injection effect. Failure: query altered/data leak ⇒ fail. Evidence: query log, payload set. Priority: P0. Owner: Data/DB. Automation: Yes.

**SEC-T-071 — Least-privilege DB principal (no UPDATE/DELETE on ledger)**
Purpose: app principal cannot mutate posted rows or run DDL. Threats: T-14. SEC-AC: 08. Prereq: app DB principal.
Given the application DB principal · When attempting UPDATE/DELETE on posted ledger tables or any DDL · Then permission is denied.
Expected: denied. Failure: allowed ⇒ fail. Evidence: grant matrix, attempt result. Priority: P0. Owner: Data/DB. Automation: Yes.

**SEC-T-072 — RLS tenant predicate at database**
Purpose: DB enforces CompanyId independent of app filters. Threats: T-06. SEC-AC: 06. Prereq: RLS policy + two tenants.
Given app-layer filters removed/bypassed · When a query runs · Then RLS still restricts rows to the principal's tenant.
Expected: RLS restricts. Failure: cross-tenant rows ⇒ fail. Evidence: RLS policy, query result. Priority: P0. Owner: Data/DB. Automation: Yes.

### 7.8 Files

**SEC-T-080 — Malicious upload rejected by magic-byte inspection**
Purpose: content-type validated by inspection, not header. Threats: T-09. SEC-AC: 15. Prereq: a polyglot/mismatched file.
Given a file whose real content mismatches its declared type · When uploaded · Then it is rejected/quarantined and never served.
Expected: rejected/quarantined. Failure: served ⇒ fail. Evidence: validation log. Priority: P0. Owner: Platform/Files. Automation: Yes.

**SEC-T-081 — Oversized upload rejected**
Purpose: size limits enforced pre-processing. Threats: T-09, T-26. SEC-AC: 13, 15. Prereq: a file over the limit.
Given an oversized upload · When received · Then rejected before processing (413/typed error).
Expected: rejected. Failure: accepted/processed ⇒ fail. Evidence: size limit config, response. Priority: P1. Owner: Platform/Files. Automation: Yes.

**SEC-T-082 — Zip-bomb extraction limits**
Purpose: decompression bounded. Threats: T-12. SEC-AC: 15. Prereq: a crafted archive.
Given a highly-compressed archive · When extracted · Then entry-count/total-size/depth limits abort extraction safely.
Expected: extraction aborted, quarantined. Failure: resource exhaustion ⇒ fail. Evidence: extraction limits, abort log. Priority: P0. Owner: Platform/Files. Automation: Yes.

**SEC-T-083 — Macro/embedded-code document not executed**
Purpose: no macro exec during parse. Threats: T-12. SEC-AC: 15. Prereq: a macro-bearing document.
Given a document containing macros/embedded code · When parsed · Then no macro/embedded code executes; parsing is sandboxed.
Expected: no execution. Failure: macro runs ⇒ fail. Evidence: sandbox trace. Priority: P0. Owner: Platform/Files. Automation: Yes.

**SEC-T-084 — Signed, short-lived, tenant-scoped download URL**
Purpose: no enumerable/public object paths. Threats: T-28. SEC-AC: 16. Prereq: a stored export document.
Given a document link · When issued and later reused/enumerated · Then it is short-lived, signed, tenant-scoped, and non-enumerable; expired/foreign access denied.
Expected: scoped + expiring. Failure: public/enumerable/long-lived ⇒ fail. Evidence: URL properties, access test. Priority: P0. Owner: Platform/Files. Automation: Yes.

**SEC-T-085 — Quarantine-first upload workflow**
Purpose: files released only after validation+scan. Threats: T-09. SEC-AC: 15. Prereq: upload pipeline.
Given a new upload · When received · Then it lands in quarantine and is released only after successful validation + malware scan.
Expected: quarantine gating. Failure: served pre-scan ⇒ fail. Evidence: pipeline state log. Priority: P0. Owner: Platform/Files. Automation: Yes.

### 7.9 Email

**SEC-T-090 — Sender authentication (SPF/DKIM/DMARC)**
Purpose: spoofed forwarder rejected/not auto-trusted. Threats: T-11. SEC-AC: 11. Prereq: ingestion enabled (Planned).
Given an inbound email failing SPF/DKIM/DMARC · When ingested · Then it is not auto-trusted and no business action is auto-taken.
Expected: not trusted. Failure: spoof triggers action ⇒ fail. Evidence: auth verdict, action log. Priority: P1. Owner: Integrations. Automation: Yes.

**SEC-T-091 — Attachment scanned & quarantined**
Purpose: malicious attachment quarantined. Threats: T-09, T-10. SEC-AC: 15. Prereq: crafted attachment.
Given a malware-laden attachment via shipping email · When ingested · Then it is scanned and quarantined, never opened/served.
Expected: quarantine. Failure: opened/served ⇒ fail. Evidence: scanner hit, quarantine record. Priority: P0. Owner: Integrations. Automation: Yes.

**SEC-T-092 — Sandboxed attachment parsing (no macro/exploit)**
Purpose: extraction cannot execute code. Threats: T-12. SEC-AC: 15. Prereq: exploit-bearing attachment.
Given an unsafe attachment · When extracted · Then parsing is sandboxed with type/size limits and no macro/exploit executes.
Expected: safe parse. Failure: exploit executes ⇒ fail. Evidence: sandbox trace. Priority: P0. Owner: Integrations. Automation: Yes.

### 7.10 Background Workers

**SEC-T-100 — Worker re-authorizes & re-scopes tenant**
Purpose: workers are untrusted callers. Threats: T-06. SEC-AC: 06. Prereq: a queued message with tenant context.
Given a worker consuming a message · When it processes · Then it re-authorizes and re-scopes to the message's tenant, never assuming ambient trust.
Expected: re-scoped. Failure: cross-tenant processing ⇒ fail. Evidence: worker authz trace. Priority: P0. Owner: Data/DB. Automation: Yes.

**SEC-T-101 — Worker least-privilege identity**
Purpose: scoped credentials, no admin. Threats: T-05. SEC-AC: 02. Prereq: worker identity.
Given a worker identity · When it attempts an out-of-scope operation · Then denied by least-privilege grants.
Expected: denied. Failure: broad access ⇒ fail. Evidence: grant matrix. Priority: P1. Owner: DevSecOps. Automation: Yes.

**SEC-T-102 — Worker never writes ledger directly**
Purpose: ledger writes only via PostingService. Threats: T-05, T-14. SEC-AC: 08. Posting AC: 7. Prereq: a worker path near ledger.
Given a worker · When it attempts a direct ledger write · Then it is denied; ledger changes must route through PostingService.
Expected: denied. Failure: direct write ⇒ fail. Evidence: DB permission, code path. Priority: P0. Owner: Accounting Eng. Automation: Yes.

### 7.11 Outbox

**SEC-T-110 — Outbox published only after commit**
Purpose: no event before ledger commit. Threats: T-19. Posting AC: 3. Prereq: fault before commit.
Given a posting that does not commit · When the dispatcher runs · Then no integration event is published.
Expected: no publish on rollback. Failure: event published without commit ⇒ fail. Evidence: outbox + broker log. Priority: P0. Owner: Platform/Integrations. Automation: Yes.

**SEC-T-111 — Idempotent consumer dedup (MessageUid)**
Purpose: at-least-once delivery does not double-process. Threats: T-19. Prereq: duplicated event.
Given the same MessageUid delivered twice · When consumed · Then the effect applies once.
Expected: single effect. Failure: double-process ⇒ fail. Evidence: dedup log. Priority: P1. Owner: Platform/Integrations. Automation: Yes.

**SEC-T-112 — Per-aggregate ordering preserved**
Purpose: no out-of-order aggregate state. Threats: T-20. Prereq: reordered events for one aggregate.
Given events for one aggregate delivered out of order · When consumed keyed on the aggregate · Then per-aggregate order is preserved.
Expected: correct order. Failure: corrupted state ⇒ fail. Evidence: consumer order log. Priority: P2. Owner: Integrations. Automation: Yes.

### 7.12 Backup / DR

**SEC-T-120 — AES-GCM authenticated backup (KI-003)**
Purpose: no XOR/obfuscation; authenticated encryption. Threats: T-24. SEC-AC: 20. Prereq: a produced backup.
Given a backup · When inspected · Then it is AES-GCM authenticated-encrypted with vault-managed keys, integrity-verified.
Expected: authenticated encryption. Failure: XOR/obfuscation/plaintext ⇒ fail (KI-003 not remediated). Evidence: backup format, key ref. Priority: P0. Owner: SRE. Automation: Yes.

**SEC-T-121 — Restore integrity drill**
Purpose: restore is recoverable + verified. Threats: T-34. SEC-AC: 20. Prereq: a backup + isolated restore env.
Given a backup · When restored · Then it is integrity-verified and recoverable, preserving append-only/tamper-evident ledger properties.
Expected: verified restore. Failure: corrupt/untrusted restore ⇒ fail. Evidence: drill report. Priority: P1. Owner: SRE. Automation: No.

**SEC-T-122 — Immutable/offsite backup (ransomware resilience)**
Purpose: backups isolated from prod creds. Threats: T-33. SEC-AC: 20. Prereq: backup infra.
Given production compromise simulation · When backups are targeted · Then immutable/WORM + offsite copies remain intact and isolated.
Expected: backups survive. Failure: backups encryptable by prod creds ⇒ fail. Evidence: immutability config. Priority: P1. Owner: SRE. Automation: No.

### 7.13 Supply Chain

**SEC-T-130 — Critical dependency vulnerability blocks release**
Purpose: SCA gate. Threats: T-22. Prereq: a dep with a known critical CVE.
Given a critical dependency vulnerability · When the build runs SCA · Then the release is blocked.
Expected: build fails. Failure: release proceeds ⇒ fail. Evidence: SCA report, gate result. Priority: P0. Owner: DevSecOps. Automation: Yes.

**SEC-T-131 — Package signature/provenance verified**
Purpose: only signed, provenance-verified packages consumed. Threats: T-22. Prereq: an unsigned/tampered package.
Given an unsigned or tampered package · When restore/verify runs · Then it is rejected.
Expected: rejected. Failure: consumed ⇒ fail. Evidence: signature/provenance check. Priority: P0. Owner: DevSecOps. Automation: Yes.

**SEC-T-132 — CI/CD pipeline hardening**
Purpose: least-priv runners, signed artefacts, approval gates. Threats: T-23. Prereq: pipeline config.
Given a pipeline change attempting to exfiltrate secrets or inject code · When executed · Then least-privilege + provenance + approval gates prevent it.
Expected: blocked. Failure: injection/exfiltration ⇒ fail. Evidence: pipeline policy, run log. Priority: P0. Owner: DevSecOps. Automation: Yes.

**SEC-T-133 — SBOM generated & retained**
Purpose: SBOM per artefact. Threats: T-22. Prereq: a build.
Given a build artefact · When produced · Then an SBOM is generated and retained.
Expected: SBOM present. Failure: missing ⇒ fail. Evidence: SBOM file. Priority: P1. Owner: DevSecOps. Automation: Yes.

### 7.14 Webhook / SSRF

**SEC-T-140 — Forged webhook signature rejected**
Purpose: unverified callbacks cannot change state. Threats: T-30. Prereq: a webhook endpoint.
Given a webhook with an invalid/missing signature · When received · Then it is rejected before any state change.
Expected: rejected. Failure: state change ⇒ fail. Evidence: signature check log. Priority: P0. Owner: Integrations. Automation: Yes.

**SEC-T-141 — Webhook replay nonce**
Purpose: replayed webhook rejected. Threats: T-30. Prereq: a valid captured webhook.
Given a previously-processed webhook · When replayed · Then the replay is rejected via nonce/timestamp.
Expected: rejected. Failure: reprocessed ⇒ fail. Evidence: nonce store. Priority: P1. Owner: Integrations. Automation: Yes.

**SEC-T-142 — SSRF egress allow-list**
Purpose: user-influenced URLs cannot reach internal/metadata. Threats: T-29. SEC-AC: 11. Prereq: an outbound-call feature.
Given a user-influenced outbound URL targeting internal/metadata ranges · When invoked · Then egress allow-list blocks it.
Expected: blocked. Failure: internal reach ⇒ fail. Evidence: egress policy, attempt log. Priority: P0. Owner: Integrations. Automation: Yes.

### 7.15 Rate Limiting / Performance Security

**SEC-T-150 — Rate limiting on auth/write endpoints**
Purpose: throttling + alerting. Threats: T-26. SEC-AC: 19. Prereq: repeated requests.
Given repeated auth/write attempts exceeding limits · When sent · Then they are throttled and alerted.
Expected: throttled. Failure: unlimited ⇒ fail. Evidence: limiter metrics, alert. Priority: P1. Owner: Platform/SRE. Automation: Yes.

**SEC-T-151 — Payload size limits (DoS resistance)**
Purpose: oversized/expensive requests rejected. Threats: T-26. SEC-AC: 13. Prereq: oversized/expensive query.
Given an oversized or expensive-query request · When received · Then it is rejected/limited before heavy processing.
Expected: rejected/limited. Failure: resource exhaustion ⇒ fail. Evidence: limit config, response. Priority: P1. Owner: Platform/SRE. Automation: Yes.

**SEC-T-152 — Mass-export governance/alerting**
Purpose: bulk export authorised + alerted (DLP). Threats: T-25. SEC-AC: 19. Prereq: a bulk-export attempt.
Given a mass export of customer/pricing/banking data · When attempted · Then it requires authorisation/Maker-Checker for bulk and raises an alert.
Expected: governed + alerted. Failure: silent mass export ⇒ fail. Evidence: export authz, alert. Priority: P1. Owner: Platform/Security. Automation: Yes.

### 7.16 Logging

**SEC-T-160 — Log injection neutralised**
Purpose: user input cannot forge/break logs. Threats: T-31. Prereq: user-influenced values reaching logs.
Given user-influenced values with control/format characters · When logged · Then they are encoded/escaped and cannot forge or break log parsing.
Expected: neutralised. Failure: forged/broken log ⇒ fail. Evidence: log sample. Priority: P2. Owner: Platform. Automation: Yes.

**SEC-T-161 — Error hygiene (no internal leakage)**
Purpose: typed public errors only. Threats: T-08, T-32. SEC-AC: 18. Prereq: forced server error.
Given a server error · When returned · Then no stack trace/SQL/file path/internal identifier leaks; a typed code is returned.
Expected: typed code only. Failure: internal detail leaked ⇒ fail. Evidence: error response. Priority: P1. Owner: Platform/Security. Automation: Yes.

**SEC-T-162 — Encryption in transit (TLS)**
Purpose: plaintext transport refused. Threats: T-11, T-29. SEC-AC: 11. Prereq: client/service connections.
Given any client/service connection · When established · Then TLS ≥1.2 (1.3 preferred) is enforced and plaintext is refused.
Expected: TLS enforced. Failure: plaintext accepted ⇒ fail. Evidence: handshake capture. Priority: P1. Owner: Platform/SRE. Automation: Yes.

---

## 8. Negative Security Tests

> Adversarial "must-fail-for-the-attacker" cases. Each maps to threat IDs and reuses the catalogue test that encodes it. All expect **denial/rejection with no side effect and appropriate alerting**.

| Neg-ID | Attack | Threat IDs | Encoded by | Expected outcome |
|---|---|---|---|---|
| NEG-T-01 | IDOR | T-07 | SEC-T-023 | 403/404, no foreign record |
| NEG-T-02 | BOLA | T-07 | SEC-T-012 | object-level denial |
| NEG-T-03 | Replay (request) | T-13 | SEC-T-036 | stored result, no 2nd effect |
| NEG-T-04 | Duplicate Posting | T-13 | SEC-T-037 | `ACC_DUPLICATE_POSTING` |
| NEG-T-05 | Forged Webhook | T-30 | SEC-T-140 | rejected pre-state-change |
| NEG-T-06 | SQL Injection | T-08 | SEC-T-070 | parameterisation neutralises |
| NEG-T-07 | Mass Assignment | T-04 | SEC-T-025 | unknown fields rejected |
| NEG-T-08 | Race Condition | T-13 | SEC-T-042 | exactly one effect |
| NEG-T-09 | Concurrent Posting | T-13 | SEC-T-042 | one entry only |
| NEG-T-10 | Period Unlock (unauthorised) | T-16 | SEC-T-014, SEC-T-038 | denied + Maker/Checker |
| NEG-T-11 | Ledger Delete | T-14 | SEC-T-031 | rejected both layers |
| NEG-T-12 | Ledger Update | T-14 | SEC-T-030 | rejected both layers |
| NEG-T-13 | Cross-Tenant Access | T-06 | SEC-T-020, SEC-T-021 | isolation enforced |
| NEG-T-14 | Secret Leakage | T-21, T-32 | SEC-T-060, SEC-T-061 | no secret in artefacts/logs |
| NEG-T-15 | Malicious Upload | T-09 | SEC-T-080 | quarantined/rejected |
| NEG-T-16 | Oversized Upload | T-09, T-26 | SEC-T-081 | rejected pre-processing |
| NEG-T-17 | Zip Bomb | T-12 | SEC-T-082 | extraction aborted |
| NEG-T-18 | Macro Document | T-12 | SEC-T-083 | no macro execution |
| NEG-T-19 | JWT Replay | T-02 | SEC-T-007 | revoked/rejected |
| NEG-T-20 | Refresh-Token Replay | T-02 | SEC-T-004 | family revoked + alert |
| NEG-T-21 | Permission Bypass | T-04, T-05 | SEC-T-010, SEC-T-015 | deny-by-default |
| NEG-T-22 | UI-only Authorization | T-04 | SEC-T-011 | server denies direct call |
| NEG-T-23 | Supply-Chain Tampering | T-22, T-23 | SEC-T-131, SEC-T-132 | rejected/blocked |
| NEG-T-24 | Log Injection | T-31 | SEC-T-160 | encoded/neutralised |
| NEG-T-25 | Audit Tampering | T-18 | SEC-T-051 | chain verify fails + alert |
| NEG-T-26 | False Success | T-15 | SEC-T-035 | no success without commit |

> Supporting catalogue test **SEC-T-025 — Mass-assignment rejected** (Threats T-04; SEC-AC-13; Secure Coding §3/P-05): given a request with unexpected fields (e.g. `IsAdmin`, `CompanyId`), when bound, then unknown fields are rejected by the DTO allow-list and no privileged field is set.

---

## 9. Penetration Testing Checklist

> Independent adversarial testing for high-risk domains before go-live (Sec-Arch §16 gate G4). Items are verification targets, not implementation steps.

**Authentication:** brute-force/lockout · breached-password rejection · MFA enforcement & bypass attempts · session fixation/rotation · refresh-token reuse · password-reset enumeration.
**Authorization:** deny-by-default on every write · vertical escalation (low→admin) · horizontal escalation (BOLA) · SoD conflict exercise · Maker/Checker self-approval · admin-only operation separation.
**API:** idempotency-key conflict handling · correlation-id propagation · CORS allow-list · pagination/query-limit abuse · unknown-field/mass-assignment · error-contract leakage · versioning/deprecation.
**Database:** SQL-injection sweep · least-privilege verification · RLS bypass · composite-FK cross-company · append-only trigger/permission bypass · DDL-from-app attempt.
**Infrastructure:** TLS configuration/downgrade · HTTP security headers (HSTS/CSP/nosniff/frame-ancestors) · clock sync · debug disabled in prod · secrets in env/vault.
**Files:** magic-byte bypass · polyglot upload · zip-bomb · macro execution · signed-URL enumeration/expiry · quarantine bypass.
**Integrations:** forged webhook · webhook replay · SSRF/egress · mTLS/allow-list · no secrets/PII in outbound URLs.
**Accounting:** ledger update/delete · reversal-only · duplicate posting (order→invoice) · idempotency replay · snapshot immutability · period-lock bypass · false-success · malicious PostingRule activation.
**Payments:** SoD (create+approve) · Maker/Checker · step-up · payroll duplication.
**Admin:** user/permission administration escalation · SoD at grant time · security-configuration change control · audit of admin actions.

---

## 10. Regression Suite

| Trigger | Tests that MUST execute |
|---|---|
| **Every Pull Request** | All unit + fast integration security tests; authorization (SEC-T-010/011/012/015), tenant isolation (SEC-T-020/021/022/024), validation/mass-assignment (SEC-T-025/081), error hygiene (SEC-T-161), secrets scan (SEC-T-060); plus SAST/SCA/secret-scan gates. |
| **Nightly** | Full catalogue integration + database tests: ledger integrity (SEC-T-030…044), audit (SEC-T-050…052), concurrency (SEC-T-042), Outbox (SEC-T-110…112), rate limiting (SEC-T-150…152), files (SEC-T-080…085). |
| **Release Candidate** | All P0/P1 tests; DAST on API; tenant-isolation negatives; supply-chain gates (SEC-T-130…133); webhook/SSRF (SEC-T-140…142); full SEC-AC/Posting-AC coverage green. |
| **Production Readiness** | Independent penetration test (§9) on Accounting/Payments/Admin; DR restore drill (SEC-T-121) + immutability (SEC-T-122); backup integrity (SEC-T-120); monitoring/alert verification (SEC-T-052/150). |

> Every mitigated/closed finding gains a permanent regression test; a closed Known Issue that regresses must fail the PR gate.

---

## 11. CI/CD Security Gates

| Gate | Runs at | Pass condition (no implementation prescribed) |
|---|---|---|
| Secrets Scan | PR + build | no secrets/keys detected (SEC-T-060) |
| SAST | PR + build | no unresolved critical findings |
| SCA | PR + build | no critical dependency vulnerability (SEC-T-130) |
| SBOM | build | SBOM generated & retained (SEC-T-133) |
| DAST | release candidate | API scan clean of criticals |
| Container Scan | build | base image free of critical CVEs |
| IaC Scan | build | infra config free of critical misconfigurations |
| Dependency Verification | build | signatures/provenance verified (SEC-T-131) |
| Security Regression | PR + nightly + RC | regression suite (§10) green |
| Manual Security Review | high-risk PRs | security reviewer sign-off (Secure Coding §18 triggers) |
| Pen Test | pre-production | independent test on high-risk domains cleared/risk-accepted (§9) |

> Gates map to Sec-Arch §16: G2 (Secrets/SAST/SCA/secrets-scan), G3 (DAST/tenant negatives), G4 (Pen Test), G5 (monitoring/restore-drill).

---

## 12. Traceability

**12.1 Threat → Tests**

| Threat | Tests |
|---|---|
| T-01 | SEC-T-002, 003 | 
| T-02 | SEC-T-004, 007 |
| T-03 | SEC-T-003, 005 |
| T-04 | SEC-T-010, 011, 015, 025 |
| T-05 | SEC-T-010, 043, 101, 102 |
| T-06 | SEC-T-020, 021, 022, 024, 072, 100 |
| T-07 | SEC-T-012, 023 |
| T-08 | SEC-T-070, 161 |
| T-09 | SEC-T-080, 081, 085, 091 |
| T-10 | SEC-T-091 |
| T-11 | SEC-T-090, 162 |
| T-12 | SEC-T-082, 083, 092 |
| T-13 | SEC-T-036, 037, 042 |
| T-14 | SEC-T-030, 031, 032, 039, 071, 102 |
| T-15 | SEC-T-033, 034, 035, 040 |
| T-16 | SEC-T-014, 038 |
| T-17 | SEC-T-044 |
| T-18 | SEC-T-041, 050, 051, 052 |
| T-19 | SEC-T-110, 111 |
| T-20 | SEC-T-112 |
| T-21 | SEC-T-060 |
| T-22 | SEC-T-130, 131, 133 |
| T-23 | SEC-T-132 |
| T-24 | SEC-T-120 |
| T-25 | SEC-T-152 |
| T-26 | SEC-T-081, 150, 151 |
| T-27 | SEC-T-013, 014 |
| T-28 | SEC-T-084 |
| T-29 | SEC-T-142, 162 |
| T-30 | SEC-T-140, 141 |
| T-31 | SEC-T-160 |
| T-32 | SEC-T-061, 161 |
| T-33 | SEC-T-122 |
| T-34 | SEC-T-121 |

**12.2 SEC-AC → Tests** — see §4 (each of SEC-AC-01…20 lists its Required Test IDs).

**12.3 Posting AC → Tests** — see §5 (each of AC-1…14 lists its Required Tests).

**12.4 Known Issue → Tests**

| Known Issue | Meaning | Regression tests (must stay green) |
|---|---|---|
| KI-001 | Plaintext password storage | SEC-T-002 |
| KI-002 | Server-side authorization absent | SEC-T-010, SEC-T-011, SEC-T-043 |
| KI-003 | Backup XOR/obfuscation | SEC-T-120 |
| KI-004 | Audit tamper-evidence absent | SEC-T-050, SEC-T-051, SEC-T-052 |
| KI-008 / KI-024 | Posted-journal delete/edit possible | SEC-T-030, SEC-T-031, SEC-T-032 |
| KI-009 / KI-030 | Duplicate posting / payroll duplication | SEC-T-036, SEC-T-037, SEC-T-042 |
| KI-025 | Snapshot immutability | SEC-T-039 |

> Listing a Known Issue here defines the test that must pass **once the control is built**; it does **not** close the Known Issue. KI status remains **open** until built, tested, and independently verified.

---

## 13. References

- `docs/06_Architecture/Security/ERP_Security_Architecture.md` (§1–§17; §15 SEC-AC; §16 gates).
- `docs/06_Architecture/Security/ERP_Threat_Model.md` (STRIDE register; §10 mitigation; §11 residual; §8 abuse cases).
- `docs/06_Architecture/Security/Secure_Coding_Standard.md` (§3–§22; prohibited patterns; traceability).
- `docs/03_Business_Logic/Accounting/Accounting_Posting_Service_Spec.md` (AI-1…AI-8; AC-1…AC-14; §9 security; §11 Outbox).
- Accepted ADRs: ADR-020 ACC-A · ADR-021 ACC-B · ADR-022 ACC-C · ADR-023 ACC-D.
- `docs/02_Governance/Known_Issues.md` (KI-001/002/003/004, KI-008/009/024/025/030).
- `AGENTS.md` · `DEVELOPMENT_RULES.md`.
- OWASP ASVS 5.0 · OWASP API Security Top 10 · OWASP Cheat Sheets · NIST SSDF.

---

## Validation Block

- **Number of sections:** 13 (plus this Validation Block).
- **Number of executable security tests:** 73 (SEC-T-001…SEC-T-162 across 16 domains, including SEC-T-025).
- **Coverage of SEC-AC-01 → 20:** 20/20 — every SEC-AC maps to ≥1 test (§4).
- **Coverage of Posting AC-1 → 14:** 14/14 — every Posting AC maps to ≥1 test (§5).
- **Coverage of T-01 → T-34:** 34/34 — every threat maps to ≥1 executable test (§6, §12.1).
- **Coverage of KI references:** KI-001, KI-002, KI-003, KI-004, KI-008, KI-009, KI-024, KI-025, KI-030 each mapped to regression tests (§12.4); all remain **open**.
- **Number of Negative Tests:** 26 (NEG-T-01…NEG-T-26).
- **Number of Penetration Test items:** 57 (across 10 domains, §9).
- **Number of CI/CD Gates:** 11 (§11).
- **Prototype modified?** No.
- **Business Logic modified?** No.
- **ADR modified?** No.
- **RFC modified?** No.
- **Known Issues modified?** No (referenced only; none closed).
- **Application code modified?** No.

**Final verdict:** `READY FOR SECURITY ARCHITECT REVIEW`
