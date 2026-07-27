# ERP Threat Model

## 1. Document Information
```
Document Name:  ERP Threat Model
Version:        0.1.0
Status:         Draft
Owner:          Solution Architecture / Security
Classification: Authoritative — Threat Model
Governing baseline: docs/06_Architecture/Security/ERP_Security_Architecture.md (v0.1.0)
Related:        Accounting_Posting_Service_Spec.md · ADR-023 (ACC-D) · Known_Issues (KI-001/002/003/004, KI-008/009/024/025/030) · Prototype_Runtime_Verification_Report.md
Methodology:    STRIDE · qualitative risk (Likelihood × Impact) — no mathematical precision claimed
Review date:    2026-07-26 (next review per §13)
```

> **Control-status honesty (enforced throughout):** the only running system today is the **prototype**, which carries the real weaknesses. The Security Architecture and Posting Service Spec describe **designed** controls that are **not yet built**. Every control is tagged: **Existing** (running) · **Designed** (specified, not built) · **Planned** (intended, not yet specified) · **Missing**. Documenting a mitigation here **does not** close a Known Issue and **does not** claim the system is secure.

---

## 2. Scope and Assumptions

**In scope:** Browser/frontend · API · Application services · SQL Server · Object/file storage · Background workers · Message broker & Outbox · Email ingestion · External shipping/tax/payment/e-invoice services · Authentication/identity provider · Administrator & privileged operations · Backup & DR infrastructure.

**Assumptions:** target production is ASP.NET Core + SQL Server per the specs; controls in the Security Architecture are the intended target state; the prototype is reference-only and not deployed to production; multi-tenant (Company/Branch) operation.

**Out of scope (this version):** physical data-centre/cloud-provider security (inherited from provider); end-user device/endpoint security beyond device-trust signals; the prototype's internal implementation (treated as evidence of weaknesses, not a hardening target); business-rule correctness (covered by Accounting docs, not this model).

---

## 3. Protected Assets

| Asset | Owner | Classification | Integrity | Confidentiality | Availability |
|---|---|---|---|---|---|
| Journal entries & lines | Accounting | Internal | **Critical** (immutable, legal) | Medium | High |
| Payments & treasury data | Accounting/Treasury | Confidential | Critical | High | High |
| Audit events | Security/Accounting | Internal | **Critical** (tamper-evident) | Medium | High |
| User credentials & MFA secrets | IAM/Security | Restricted | Critical | **Critical** | High |
| Roles & permissions | Security Admin | Restricted | Critical | High | High |
| Customer & supplier data | CRM/Procurement | Confidential | High | High | Medium |
| Supplier banking details | Procurement/Treasury | Restricted | Critical | **Critical** | Medium |
| Pricing & profitability data | Commercial | Confidential | High | High | Medium |
| Inventory & production data | Operations | Internal | High | Medium | Medium |
| Export & shipping documents | Sales/Export | Confidential | High | High | High |
| Quality certificates | Quality | Confidential | High | Medium | Medium |
| API keys & integration secrets | DevSecOps | Restricted | Critical | **Critical** | High |
| Backups | SRE | Restricted | Critical | High | **Critical** |
| Posting rules | Accounting/Security | Internal | **Critical** | Medium | High |
| Accounting periods (lock state) | Accounting | Internal | Critical | Low | High |

---

## 4. Actors

**Legitimate:** normal employee · accountant · warehouse user · sales/CRM user · HR user · administrator · security administrator · Product Owner · external customer · forwarder/shipping partner · external service · background worker.
**Adversarial / abused:** malicious insider · compromised account · unauthenticated attacker · supply-chain attacker.

> Per Zero Trust, *every* actor (including background workers and internal services) is untrusted until authenticated, authorised, and tenant-scoped per request.

---

## 5. Entry Points

Login · password reset · APIs (`/api/v1/...`) · file upload · email ingestion · signed document URLs · admin screens · reporting & exports · integration callbacks/webhooks · background jobs · database migrations · CI/CD pipeline · backup restore.

---

## 6. Trust Boundaries

| # | Boundary | Identities | Data crossing | Primary controls | Failure consequence |
|---|---|---|---|---|---|
| TB-1 | Browser → API | end user ↔ API | credentials, requests, files, headers | TLS1.3, authN, CSRF, validation, rate limit, authoritative headers | account/tenant compromise, injection |
| TB-2 | API → Application | authenticated principal | request DTOs, tenant context | deny-by-default authz, tenant resolved server-side, step-up | authz/tenant bypass (EoP, disclosure) |
| TB-3 | Application → Database | app DB principal | queries, ledger writes | least-privilege principal (no UPDATE/DELETE on ledger), parameterisation, RLS, triggers | ledger tampering, SQLi, cross-tenant |
| TB-4 | Application → Storage | app ↔ object store | files, exports | server-side encryption, signed URLs, non-enumerable paths | document leakage (IDOR) |
| TB-5 | Email → Ingestion Worker | inbound sender ↔ worker | attachments, message bodies | SPF/DKIM/DMARC, malware scan, quarantine, sandboxed parsing | malware entry, forged-sender action |
| TB-6 | External Service → API/Callback | third party ↔ API | webhooks, integration responses | signature verification, replay nonce, egress allow-list | forged callbacks, SSRF |
| TB-7 | Outbox → Broker | dispatcher ↔ broker | integration events | commit-then-dispatch, MessageUid, TLS | duplicate/replayed events |
| TB-8 | Broker → Worker | broker ↔ consumer | event payloads | idempotent consumers, re-authz, tenant re-scope | duplicate processing, privilege misuse |
| TB-9 | CI/CD → Runtime | pipeline ↔ environments | artefacts, secrets | signed packages, provenance, least-priv runners, approval gates | supply-chain compromise |
| TB-10 | Backup Store → Restore Environment | SRE ↔ restore env | backup images | AES-GCM, immutable/offsite, integrity verify, isolated creds | data theft, ransomware, DR corruption |

---

## 7. STRIDE Threat Register

**Control status:** E=Existing · D=Designed · P=Planned · M=Missing. **Risk = qualitative(Likelihood, Impact).**

| ID | STRIDE | Component | Scenario (ERP-specific) | Asset | Likelihood | Impact | Risk | Existing controls (honest) | Required mitigation (status) | Verification | Owner | Residual |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| T-01 | S | Auth | Attacker reuses breached password to take over an accountant account (prototype stores plaintext — KI-001) | Credentials | M | Critical | **High** | Prototype plaintext pw (**M**) | Argon2id hashing, breached-pw check, MFA (**D**) | SEC-AC-03/04 | IAM/Security | Medium |
| T-02 | S | Session | Stolen/replayed refresh token grants persistent access | Credentials | M | High | **High** | none (**M**) | Rotating refresh + reuse detection (**D**) | SEC-AC-05 | IAM/Security | Medium |
| T-03 | S | Auth | MFA bypass via fallback/social-engineering on a privileged user | Credentials | L | High | **Medium** | none (**M**) | Phishing-resistant MFA, step-up (**D**) | SEC-AC-04/10 | IAM/Security | Medium |
| T-04 | E | AuthZ | Low-role user escalates by calling a privileged action the UI hid (client-side perms — KI-002) | Roles/permissions | M | Critical | **High** | Prototype client-side checks (**M**) | Server-side RBAC/PBAC deny-by-default (**D**) | SEC-AC-02 | Platform/API | Medium |
| T-05 | E | AuthZ | Server-side authorisation absent — viewer creates a supplier payment; accountant posts payroll (runtime SP-07/PAY-05/FX-01) | Payments/Payroll | H | Critical | **Critical** | Handlers unguarded (**M**, KI-002) | API authz middleware on every write (**D**) | SEC-AC-02 · PostAC-7 | Platform/API | High |
| T-06 | I | Multi-tenant | Company A user reads Company B journal/customer by changing an id | All tenant data | M | Critical | **Critical** | Single-tenant prototype; no isolation (**M**) | Server-side tenant filter + RLS + composite FK (**D**) | SEC-AC-06/07 | Data/DB | Medium |
| T-07 | I | API | IDOR/BOLA — direct object reference returns another tenant's/owner's record | Confidential data | M | High | **High** | none (**M**) | Object-level authz + tenant scoping (**D**) | SEC-AC-06 | Platform/API | Medium |
| T-08 | T | Database | SQL injection via an unparameterised query in production | Ledger/all | L | Critical | **High** | Prototype is client-side (N/A); prod risk (**M**) | Parameterised queries only (**D**) | SEC-AC-14 | Data/DB | Low |
| T-09 | T | Files | Malicious file uploaded as an export attachment (polyglot/oversized) | Storage/users | M | High | **High** | none (**M**) | Content-type inspection, malware scan, quarantine (**D**) | SEC-AC-15 | Platform/Files | Medium |
| T-10 | T | Email ingestion | Malware-laden attachment arrives via shipping email and is opened/parsed | Storage/hosts | H | High | **High** | none (**M**) | Ingestion scan, quarantine, sandboxed parse (**P**) | SEC-AC-15 · new test | Integrations | High |
| T-11 | S | Email ingestion | Forged shipping email spoofs a forwarder to trigger an action | Export docs | M | Medium | **Medium** | none (**M**) | SPF/DKIM/DMARC verify, sender allow-list, no auto-trust (**P**) | new test | Integrations | Medium |
| T-12 | T | Email ingestion | Unsafe attachment extraction executes macro/exploit during parsing | Hosts/data | M | High | **High** | none (**M**) | Sandboxed parsing, no macro exec, type/size limits (**P**) | new test | Integrations | Medium |
| T-13 | T | Posting | Same export order posted as two commercial invoices (prototype no dedup — KI-009) | Journal entries | M | High | **High** | Prototype: no duplicate guard (**M**) | Unique posting key + idempotency (**D**) | PostAC-4/5 | Accounting Eng | Low |
| T-14 | T | Ledger | Insider modifies/deletes a posted journal (prototype delete allowed — KI-008/024) | Journal entries | M | Critical | **Critical** | Prototype delete/edit possible (**M**) | Append-only + DB perms + triggers + reversal (**D**) | PostAC-1/9 · SEC-AC-08 | Data/DB | Low |
| T-15 | R | Posting | "False success" — deletion in a locked period reports success without effect (runtime SP-06) | Ledger/audit | M | High | **High** | Prototype false-success (**M**) | Atomic tx; no success unless commit (**D**) | PostAC-3/11 | Accounting Eng | Low |
| T-16 | E | Period lock | User unlocks a closed period without authorisation (runtime CLOSE-03/05) | Period state | M | High | **High** | Uncoordinated lock writers; unlock ungated (**M**) | Central PeriodLockService + authz + Maker/Checker (**D**) | PostAC-6 · SEC-AC-09 | Accounting Eng | Medium |
| T-17 | T | Posting rules | Admin/attacker activates a malicious PostingRule that mis-books to a controlled account | Posting rules | L | Critical | **High** | Registry designed, not built (**M**) | Admin-only registration + Maker/Checker + audit + version pinning (**D**) | SEC-AC-09 · PostSpec §5 | Accounting/Security | Medium |
| T-18 | R | Audit | Audit log altered/removed to hide fraud (prototype tamperable — KI-004) | Audit events | M | Critical | **High** | Prototype tamperable audit (**M**) | Append-only hash chain per Company / DB-ledger (**D**) | SEC-AC-17 | Security/Data | Medium |
| T-19 | T | Outbox/Broker | Replayed/duplicated integration event double-processes downstream | Integration data | M | Medium | **Medium** | none (**M**) | MessageUid dedup, idempotent consumers (**D**) | PostSpec §11 | Platform/Integrations | Low |
| T-20 | T | Broker | Event reordering causes out-of-order downstream state | Integration data | L | Medium | **Low** | none (**M**) | Per-aggregate ordering; consumers key on aggregate (**D**) | PostSpec §11 | Integrations | Low |
| T-21 | I | Secrets | Integration secret/API key leaks from config/logs/payload | API secrets | M | Critical | **High** | Prototype: no vault (**M**) | Vault; no secrets in payloads/logs (**D**) | SEC-AC-12 | DevSecOps | Medium |
| T-22 | T | Supply chain | Vulnerable/malicious dependency pulled into the build | Build/runtime | H | High | **High** | none (**M**) | SCA gate, SBOM, pinning, signed feeds (**P**) | SEC arch §9 · new test | DevSecOps | Medium |
| T-23 | T | CI/CD | Compromised pipeline injects code or exfiltrates secrets | All | L | Critical | **High** | none (**M**) | Signed packages, provenance, least-priv runners, approval gates (**P**) | SEC arch §9 | DevSecOps | Medium |
| T-24 | I | Backup | Backup stolen or tampered; prototype uses XOR obfuscation (KI-003) | Backups | M | Critical | **High** | Prototype XOR (**M**) | AES-GCM authenticated, immutable/offsite (**D**) | SEC-AC-20 | SRE | Medium |
| T-25 | I | Reporting/Export | Mass export exfiltrates customer/pricing/banking data | Confidential data | M | High | **High** | none (**M**) | Export authz, Maker/Checker for bulk, rate limit, alerting (**D/P**) | SEC-AC-19 · §7 · §12 | Platform/Security | Medium |
| T-26 | D | API | Volumetric/expensive-query DoS makes posting unavailable at close | Availability | M | High | **High** | none (**M**) | Rate limit, payload limits, autoscale/WAF (**D**) | SEC-AC-13/19 | Platform/SRE | Medium |
| T-27 | E | Business logic | Insider fraud: create supplier + approve its payment (no SoD) | Payments | M | Critical | **High** | No SoD in prototype (**M**) | SoD + Maker/Checker + audit + alerting (**D**) | SEC-AC-09 | Security/Accounting | Medium |
| T-28 | I | Storage | Enumerable/unsigned object path exposes another tenant's export document | Export docs | M | High | **High** | none (**M**) | Signed, short-lived, tenant-scoped URLs; non-enumerable paths (**D**) | SEC-AC-16 | Platform/Files | Low |
| T-29 | I | Integrations | SSRF via a user-influenced URL in an external call reaches internal metadata | Secrets/internal | L | High | **Medium** | none (**M**) | Egress allow-list, no user-controlled URLs, metadata block (**P**) | new test | Integrations | Medium |
| T-30 | S | Webhooks | Forged webhook (payment/e-invoice) triggers unauthorised state change | Payments/ledger | M | High | **High** | none (**M**) | Signature verification, replay nonce, source allow-list (**P**) | §3 · new test | Integrations | Medium |
| T-31 | T | Logging | Log injection forges/breaks audit or SIEM parsing | Logs/audit | L | Medium | **Low** | none (**M**) | Structured logs, encode user input (**D**) | §12 | Platform | Low |
| T-32 | I | Logging | Sensitive data (secrets/PII/banking) written to logs | Restricted data | M | High | **High** | Prototype leaks likely (**M**) | Classification-driven masking; no secrets/PII in logs (**D**) | SEC-AC-12 · §7 · §12 | Platform/Security | Medium |
| T-33 | D | Backup/Infra | Ransomware encrypts production and backups | Availability/all | L | Critical | **High** | none (**M**) | Immutable/WORM + offsite backups, least-priv, EDR (**P**) | SEC-AC-20 · §14 | SRE | Medium |
| T-34 | D | DR | Disaster-recovery restore is corrupted or untrusted | Availability/integrity | L | Critical | **High** | none (**M**) | Restore drills, integrity verification, immutable copies (**P**) | SEC-AC-20 · §14 | SRE | Medium |

---

## 8. Abuse Cases

**AC-1 — Viewer posts a supplier payment.** *Pre:* viewer role; server authz not enforced (KI-002). *Steps:* call `POST /api/v1/accounting/postings` (rule JS-16) directly. *Expected controls:* deny-by-default API authz (SEC-AC-02), rule-specific permission. *Detection:* authz-denial + anomaly alert (§12). *Response:* block, alert, review grants. *Residual:* High until server authz built.

**AC-2 — Accountant posts payroll without approval.** *Pre:* accountant with `hr:view` only (runtime PAY-05). *Steps:* invoke payroll posting. *Expected:* permission check + Maker/Checker (SEC-AC-09). *Detection:* SoD/approval alert. *Response:* reject, audit. *Residual:* Medium.

**AC-3 — Company A accesses Company B data.** *Pre:* valid A session. *Steps:* request B's entry id / by-source. *Expected:* server-side tenant filter + RLS + composite FK (SEC-AC-06/07). *Detection:* cross-tenant access alert. *Response:* block, investigate. *Residual:* Medium.

**AC-4 — Replay of the same posting request.** *Pre:* captured request. *Steps:* resend with same `Idempotency-Key`. *Expected:* idempotency returns stored result; different hash ⇒ `ACC_IDEMPOTENCY_CONFLICT` (PostAC-4/5). *Detection:* duplicate-key metric. *Response:* no double effect. *Residual:* Low.

**AC-5 — Insider modifies/deletes a posted journal.** *Pre:* DB or app access. *Steps:* attempt UPDATE/DELETE. *Expected:* append-only perms + triggers; reversal-only (PostAC-1/9, SEC-AC-08). *Detection:* audit-chain verification (SEC-AC-17). *Response:* block, forensic capture. *Residual:* Low (design strong).

**AC-6 — Unauthorised period unlock.** *Pre:* user with app access (runtime CLOSE-03). *Steps:* call unlock. *Expected:* central lock service + authz + Maker/Checker (PostAC-6, SEC-AC-09). *Detection:* unlock alert. *Response:* reject, audit. *Residual:* Medium.

**AC-7 — Malicious attachment via shipping email.** *Pre:* ingestion enabled. *Steps:* send crafted attachment. *Expected:* scan + quarantine + sandbox parse (SEC-AC-15). *Detection:* scanner hit. *Response:* quarantine, alert. *Residual:* High until ingestion hardened.

**AC-8 — Attacker obtains an export-document URL.** *Pre:* leaked/guessed link. *Steps:* access object directly. *Expected:* short-lived signed, tenant-scoped, non-enumerable URLs (SEC-AC-16). *Detection:* access-pattern anomaly. *Response:* revoke, rotate. *Residual:* Low.

**AC-9 — Admin activates a malicious PostingRule.** *Pre:* admin or compromised admin. *Steps:* register + activate a bad rule. *Expected:* admin-only + Maker/Checker + audit + version pinning (SEC-AC-09, PostSpec §5). *Detection:* rule-change alert. *Response:* deactivate, reverse affected postings. *Residual:* Medium.

**AC-10 — Compromised dependency enters the build.** *Pre:* CI pulls a package. *Steps:* malicious/vulnerable dep introduced. *Expected:* SCA gate, SBOM, signed feeds, provenance (SEC arch §9). *Detection:* SCA/critical-vuln block. *Response:* fail build, quarantine artefact. *Residual:* Medium.

**AC-11 — Audit evidence altered.** *Pre:* DB access. *Steps:* edit/remove audit rows. *Expected:* append-only hash chain per Company / DB-ledger (SEC-AC-17). *Detection:* chain verification fails. *Response:* alert, forensic. *Residual:* Medium.

**AC-12 — Backup stolen or restored into an unsafe environment.** *Pre:* backup access. *Steps:* exfiltrate or restore to attacker env. *Expected:* AES-GCM, isolated creds, immutable/offsite (SEC-AC-20). *Detection:* backup-access anomaly. *Response:* rotate keys, investigate. *Residual:* Medium.

---

## 9. High-Risk Workflows (deeper analysis)

- **Posting Service** — the single ledger writer; threats T-05/13/14/15/17. Depends on server authz, idempotency, append-only, snapshot, atomic audit+Outbox. Highest concentration of Critical risk.
- **Supplier/customer payments** — T-05/27/30; requires authz + SoD/Maker-Checker + webhook signature verification.
- **Period lock/unlock** — T-16; central `PeriodLockService`, authorised + audited, Maker/Checker on unlock.
- **Payroll** — T-05/27 + duplication (KI-030); single canonical path + uniqueness key + approval.
- **Posting-rule lifecycle** — T-17; admin-only registration/activation/deprecation, Maker/Checker, version pinning, audit.
- **User & permission administration** — T-04/27; deny-by-default grants, SoD conflict prevention, step-up, full audit.
- **File & export-document access** — T-09/28; content inspection, malware scan, signed tenant-scoped URLs.
- **Email ingestion** — T-10/11/12; SPF/DKIM/DMARC, scan, quarantine, sandboxed parsing (all **Planned**).
- **Backup restore** — T-24/33/34; AES-GCM, immutability, isolated creds, restore drills, integrity verification.
- **CI/CD release** — T-22/23; signed packages, provenance, least-priv runners, approval gates, SBOM.

---

## 10. Mitigation Mapping (every Critical & High threat)

| Threat | Risk | Sec-Arch § | SEC-AC | Posting AC | Known Issue | Planned test | Owner | Residual acceptance required |
|---|---|---|---|---|---|---|---|---|
| T-05 | Critical | §5, §8 | SEC-AC-02 | PostAC-7 | KI-002 (open) | authorization suite | Platform/API | **Yes** (until built) |
| T-06 | Critical | §6, §10 | SEC-AC-06/07 | — | — | tenant-isolation suite | Data/DB | **Yes** |
| T-14 | Critical | §1, §10 | SEC-AC-08/17 | PostAC-1/9 | KI-008/024 (open) | posting-integrity suite | Data/DB | Yes |
| T-01 | High | §4 | SEC-AC-03/04 | — | KI-001 (open) | authentication suite | IAM/Security | Yes |
| T-02 | High | §4 | SEC-AC-05 | — | — | authentication suite | IAM/Security | Yes |
| T-04 | High | §5 | SEC-AC-02 | PostAC-7 | KI-002 (open) | authorization suite | Platform/API | Yes |
| T-07 | High | §5, §6 | SEC-AC-06 | — | — | authorization/tenant | Platform/API | Yes |
| T-08 | High | §10 | SEC-AC-14 | — | — | injection tests | Data/DB | Yes |
| T-09 | High | §11 | SEC-AC-15 | — | — | file-security suite | Platform/Files | Yes |
| T-10 | High | §3, §11 | SEC-AC-15 | — | — | email-ingestion suite | Integrations | Yes |
| T-12 | High | §11 | SEC-AC-15 | — | — | email-ingestion suite | Integrations | Yes |
| T-13 | High | §8 | — | PostAC-4/5 | KI-009 (open) | idempotency/dedup | Accounting Eng | Yes |
| T-15 | High | §1 | SEC-AC-08 | PostAC-3/11 | — (SP-06 evidence) | posting-integrity | Accounting Eng | Yes |
| T-16 | High | §5 | SEC-AC-09 | PostAC-6 | — (CLOSE-03/05) | lock-enforcement | Accounting Eng | Yes |
| T-17 | High | §5 | SEC-AC-09 | PostSpec §5 | — | rule-lifecycle tests | Accounting/Security | Yes |
| T-18 | High | §10, §12 | SEC-AC-17 | — | KI-004 (open) | audit-tamper tests | Security/Data | Yes |
| T-21 | High | §7 | SEC-AC-12 | — | — | secrets-scan | DevSecOps | Yes |
| T-22 | High | §9 | — | — | — | supply-chain tests | DevSecOps | Yes |
| T-23 | High | §9 | — | — | — | supply-chain tests | DevSecOps | Yes |
| T-24 | High | §14 | SEC-AC-20 | — | KI-003 (open) | backup-restore tests | SRE | Yes |
| T-25 | High | §7, §12 | SEC-AC-19 | — | — | export/DLP tests | Platform/Security | Yes |
| T-26 | High | §8 | SEC-AC-13/19 | — | — | rate-limit/DoS tests | Platform/SRE | Yes |
| T-27 | High | §5 | SEC-AC-09 | — | — | SoD tests | Security/Accounting | Yes |
| T-28 | High | §11 | SEC-AC-16 | — | — | file-security suite | Platform/Files | Yes |
| T-30 | High | §3 | — | — | — | webhook tests | Integrations | Yes |
| T-32 | High | §7, §12 | SEC-AC-12 | — | — | log-hygiene tests | Platform/Security | Yes |
| T-33 | High | §14 | SEC-AC-20 | — | — | DR/ransomware drill | SRE | Yes |
| T-34 | High | §14 | SEC-AC-20 | — | — | restore-integrity drill | SRE | Yes |

> Every Critical/High threat above has an **owner** and a **verification method**; each requires **explicit residual-risk acceptance** until its mitigation is *Existing* (built + verified). **No High/Critical threat is left without mitigation, owner, and verification.**

---

## 11. Residual-Risk Register (unresolved)

| # | Risk | Why it remains | Temporary control | Decision owner | Due phase | Acceptance authority | Review/expiry |
|---|---|---|---|---|---|---|---|
| R-01 | Server-side authz not built (T-05/04) | Production not implemented; prototype bypassable | restrict prototype to non-production; no prod exposure | Platform/API lead | Phase 1 (Posting/Perms) | CISO/Security Architect | Phase 1 exit |
| R-02 | Tenant isolation unbuilt (T-06/07) | No production RLS yet | single-tenant deployments only until built | Data/DB lead | Phase 1–2 | CISO | Phase 2 exit |
| R-03 | Audit tamper-evidence unbuilt (T-18) | Hash chain/DB-ledger not implemented (KI-004 open) | restricted DB access; manual review | Security | Phase 1 | CISO | Phase 1 exit |
| R-04 | Email ingestion hardening Planned (T-10/11/12) | Not yet specified | disable ingestion until scan+sandbox ready | Integrations lead | Phase (Ingestion) | Security Architect | when feature designed |
| R-05 | Supply-chain controls Planned (T-22/23) | CI hardening not implemented | manual dependency review; locked feeds | DevSecOps | Phase 1 (CI) | Security Architect | Phase 1 exit |
| R-06 | Backup encryption unbuilt (T-24/33/34) | Prototype XOR (KI-003 open); AES-GCM not built | restrict backup access; offline copy | SRE | Phase (DR) | CISO | before prod |
| R-07 | Webhook/SSRF controls Planned (T-29/30) | Integration signing/egress control not built | disable external callbacks until signed | Integrations | Phase (Integrations) | Security Architect | when integration added |
| R-08 | Secrets management unbuilt (T-21/32) | Vault not integrated | keep secrets out of repo; manual handling | DevSecOps | Phase 1 | CISO | Phase 1 exit |
| R-09 | MFA/session hardening unbuilt (T-01/02/03) | IAM not implemented | limit access; strong admin passwords | IAM lead | Phase 1 (IAM) | CISO | Phase 1 exit |
| R-10 | Mass-export/DLP controls partial (T-25) | Bulk export governance not built | manual approval for bulk export | Security | Phase 2 | CISO | Phase 2 exit |

> **No risk is silently accepted** — each has an owner, acceptance authority, and a review/expiry.

---

## 12. Security Test Derivation

To be created later in `docs/05_Testing/Security/Security_Test_Specification.md` (not created here):

- **Authentication** — password hashing, breached-pw, MFA, session/refresh reuse (SEC-AC-01/03/04/05).
- **Authorization** — deny-by-default, RBAC/PBAC, server-side bypass, SoD/Maker-Checker (SEC-AC-02/09; PostAC-7).
- **Tenant isolation** — cross-company read/write, IDOR/BOLA, composite-FK, RLS (SEC-AC-06/07).
- **Posting integrity** — append-only, reversal-only, balance, false-success (PostAC-1/2/3/9/11; SEC-AC-08).
- **Idempotency** — replay, duplicate key, concurrency (PostAC-4/5/14; Sec-Arch §8 — replay protection).
- **Lock enforcement** — central lock, unauthorised unlock (PostAC-6; SEC-AC-09).
- **File security** — content inspection, malware scan, quarantine, signed URLs (SEC-AC-15/16).
- **Email ingestion** — spoof (SPF/DKIM/DMARC), malware, sandboxed extraction (T-10/11/12).
- **API rate limiting / DoS** — throttling, payload limits (SEC-AC-13/19).
- **Audit tamper detection** — hash-chain verification, alerting (SEC-AC-17).
- **Dependency / supply-chain** — SCA gate, SBOM, signature/provenance (SEC arch §9).
- **Backup restore integrity** — AES-GCM, restore drill, integrity verify (SEC-AC-20).

---

## 13. Review and Maintenance

This threat model MUST be reviewed:
- **before implementing a new trust boundary**;
- **after a material architecture change**;
- **after a security incident**;
- **before every production release** (gate G4, Sec-Arch §16);
- **at least annually**;
- **when a new external integration is added**.

Each review updates the register, re-rates risk, and re-checks that no High/Critical threat lacks an owner, verification method, and residual-risk acceptance.

---

> **Standing caveats:** control status is **Existing / Designed / Planned / Missing** — most production controls are **Designed/Planned, not built**. Known Issues **KI-001/002/003/004 (and related)** remain **open**; nothing here closes them. **No claim is made that the system is secure.**
