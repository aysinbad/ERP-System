# Accounting — Posting Service Specification (Phase 1 Design)

## Document Information
```
Document Name:  Accounting Posting Service Specification (Phase 1)
Version:        1.0.0 (technical-review implementation-readiness pass)
Status:         Draft — Technical Design (Phase 1 contract)
Classification: Implementation Design Input
Owner:          Solution Architecture Team
Date:           2026-07-25
Baseline:       frozen repository (ADR-020 ACC-A · ADR-021 ACC-B · ADR-022 ACC-C · ADR-023 ACC-D · Known_Issues KI-002/004/008/009/016/020/024/025/027/030 · AINV-01→34 · Session_6_2/*)
Target stack:   ASP.NET Core Web API · C# · SQL Server
```

> **حدود التعديل:** هذا ملف **جديد** فقط. لا يعدّل أي ADR أو RFC أو `Accounting.md` أو Known Issues أو Test Cases أو حزمة قرارات Product Owner أو الـProttype. لا يُنتِج `schema.sql` تنفيذياً ولا كود تطبيق — عقود ومخططات منطقية فقط.
>
> **مبدأ حاكم:** الـAPI **لا يقبل أسطر قيد نهائية** من وحدات الأعمال. يقبل **هوية الحدث + مدخلاته**، والخادم يحلّ `PostingRule` ويبني الأسطر. هذا هو جذر إغلاق KI-002 (تجاوز الصلاحية) و AINV-02 (التتبّع).

---

## 1. Scope and Non-scope

### 1.1 Included (Phase 1 core)

Domain models: `JournalEntry` · `JournalLine` · `PostingRequest` · `PostingResult` · `PostingContext` · `PostingSnapshot` · `PostingRule` · `PostingSource` · `PostingAudit` · `AccountingPeriod` · `ReversalRequest` · `IdempotencyRecord`.

Services / builders: `PostingService` · `JournalBuilder`.

Interfaces (contracts only): `PeriodLockService` · `PermissionService` · `AuditService` · `SnapshotProvider` · `IdempotencyService`.

### 1.2 Excluded (out of Phase 1 core)

- Implementation of all **62** business posting rules (only the `IPostingRule` contract + resolver + one explicitly-authorised **manual-journal** rule are in scope).
- Final FX-engine decision → `PendingDecision` (**OQ-6.F**).
- **ACC-E** (account 2017 settlement) → `PendingDecision` (**OQ-6.A**).
- Reconciliation engine implementation (read-only; never mutates ledger).
- Month-close readiness enforcement rules → `PendingDecision` (**OQ-6.G**).
- Payroll policy (canonical path) → `PendingDecision` (**OQ-6.D**).
- Claims-provision policy → `PendingDecision` (**OQ-6.H**).

> Unresolved outcomes are marked **`PendingDecision`** and implemented as **extension points**, never hard-coded in Phase 1 core (see §10).

---

## 2. Architecture Invariants (non-negotiable)

| # | Invariant | Source |
|---|---|---|
| AI-1 | **PostingService is the only writer** to the general ledger. | ACC-A |
| AI-2 | No business module writes `JournalEntry`/`JournalLine` directly. | ACC-A · KI-002 |
| AI-3 | Posted journals are **append-only**. | ACC-A · ACC-C |
| AI-4 | Posted journals **cannot be updated or deleted**. | ACC-C · KI-008/024 |
| AI-5 | Correction uses **reversal** and, where required, reposting. | ACC-C · KI-020/027 |
| AI-6 | `JournalBuilder` **never reads mutable live values after snapshot creation**. | ACC-B · KI-025 |
| AI-7 | Every posting is **idempotent**. | ACC-A · KI-009 |
| AI-8 | Every posting is **balanced** (`Σ debit = Σ credit`). | AINV-01 |
| AI-9 | Every write is **authorised on the server**. | ACC-D · KI-002 |
| AI-10 | Every posting date is checked against the **central period lock**. | ACC-D · AINV-26 |
| AI-11 | **Audit and journal persistence are in the same DB transaction.** | ACC-C · KI-004 · SP-06 |
| AI-12 | Reconciliation **never mutates** the legal ledger. | ACC-A |

---

## 3. Logical Schema — implementation input for future schema.sql

> **ليست `schema.sql` تنفيذية.** تصميم منطقي فقط، مُدخَل لبناء المخطط لاحقاً. أنواع البيانات مقترحة لـSQL Server. `schema.sql` = **Required Phase 1 build artifact — not yet created** (الموقع المستقبلي: `src/Infrastructure/Persistence/schema.sql`).

**Conventions:** money = `DECIMAL(19,4)`; identifiers = `BIGINT IDENTITY` (surrogate) + `UNIQUEIDENTIFIER` (public); timestamps = `DATETIME2(3)` UTC; concurrency = `ROWVERSION`.

**Multi-tenancy ownership policy (replaces "CompanyId/BranchId on all tables"):**
- `CompanyId` is **mandatory on aggregate roots**.
- **Child tables inherit tenant ownership through foreign keys** to their root (no duplicated tenant column by default).
- A **duplicate tenant column is added to a child only when required** for row-level security (RLS), partitioning, or proven query performance.
- **Composite foreign keys (or equivalent constraints) must prevent cross-company child references** (a child cannot point at a parent in another company).
- `BranchId` is present **only for branch-scoped entities** — it is **not** added blindly to every table.
- **Shared accounting metadata** (e.g. rule definitions) may be **global or company-scoped**, but the choice is **explicit** per table (see matrix).

**Tenant ownership matrix:**

| Table | Role | `CompanyId` | `BranchId` | Tenant ownership |
|---|---|---|---|---|
| `JournalEntries` | aggregate root | Mandatory | Yes (branch-scoped entity) | own tenant columns |
| `JournalLines` | child of `JournalEntries` | inherited via `EntryId` (add only for RLS/partitioning) | No | via FK to root |
| `PostingEvents` | request aggregate | Mandatory | Yes (branch-scoped request) | own tenant columns |
| `IdempotencyRecords` | tenant-owned registry | Mandatory (part of PK) | No | own (`CompanyId` in PK) |
| `PostingSnapshots` | child of `JournalEntries` | inherited via `EntryId` | No | via FK to root |
| `AccountingPeriods` | company-scoped control | Mandatory | No (company-wide periods) | own `CompanyId` |
| `AuditEvents` | child (`EntryId`/`EventId`) + hash chain | Mandatory (hash sequenced per `CompanyId`, §9) | No | own `CompanyId`; links via FK |
| `PostingRules` | shared accounting metadata | **Explicit choice:** global by default; may be company-scoped via nullable `CompanyId` (NULL = global) | No | **explicitly global-or-company** |
| `OutboxMessages` | integration outbox | Mandatory (tenant-scoped dispatch/RLS) | No | own `CompanyId` |
| `JournalNumberSequences` | numbering allocator (§3.10) | Mandatory | Only if numbering is branch-specific (config) | own `CompanyId` (+ optional branch) |

### 3.1 `JournalEntries`
- **Purpose:** legal ledger header; one per posted economic event.
- **Columns:** `EntryId BIGINT IDENTITY PK` · `EntryUid UNIQUEIDENTIFIER` · `JournalNumber NVARCHAR(30)` · `CompanyId INT` · `BranchId INT` · `PostingKey NVARCHAR(200)` · `SourceType NVARCHAR(40)` · `SourceId NVARCHAR(80)` · `RuleId NVARCHAR(40)` · `RuleVersion INT` · `EventDate DATE` · `PostedAtUtc DATETIME2(3)` · `PostedByUserId INT` · `BaseCurrency CHAR(3)` · `Description NVARCHAR(400)` · `EntryKind NVARCHAR(20)` (`Normal`|`Reversal`|`Manual`) · `ReversesEntryId BIGINT NULL` · `CorrelationId UNIQUEIDENTIFIER` · `RowVersion ROWVERSION`.
- **PK:** `EntryId`. **FKs:** `ReversesEntryId → JournalEntries.EntryId`.
- **Unique:** `UQ_JE_PostingKey (CompanyId, PostingKey)` · `UQ_JE_Uid (EntryUid)` · `UQ_JE_JournalNumber (CompanyId, JournalNumber)`.
- **Check:** `EventDate IS NOT NULL` · `SourceType/SourceId NOT NULL` · `BaseCurrency NOT NULL` · `JournalNumber NOT NULL` · `EntryKind IN ('Normal','Reversal','Manual')` · `(EntryKind='Reversal') = (ReversesEntryId IS NOT NULL)`.
- **Indexes:** `IX_JE_Source (SourceType, SourceId)` · `IX_JE_EventDate (CompanyId, EventDate)` · `IX_JE_Reverses (ReversesEntryId)` · `IX_JE_JournalNumber (CompanyId, JournalNumber)` (user-facing lookups).
- **`JournalNumber` (logical field):** **user-visible**, **immutable**, **sequential per legal entity (`CompanyId`)**, **unique**, **independent from `EntryId`** (surrogate), **never reused** (gap-tolerant, no recycling), and **survives migration** (carried, not regenerated). Allocation is server-side within the posting transaction from a per-entity sequence; the surrogate `EntryId` remains the internal key. Indexing: covering unique index `UQ_JE_JournalNumber (CompanyId, JournalNumber)` for immutability + fast user lookup.
- **Immutable enforcement:** `DENY UPDATE, DELETE` to app role; `INSTEAD OF UPDATE/DELETE` trigger raises error; append-only.
- **Retention:** permanent (legal record).
- **Concurrency:** insert-only; `RowVersion` for read models.

### 3.2 `JournalLines`
- **Purpose:** debit/credit lines of an entry.
- **Columns:** `LineId BIGINT IDENTITY PK` · `EntryId BIGINT` · `LineNo INT` · `AccountCode NVARCHAR(20)` · `Debit DECIMAL(19,4) NOT NULL DEFAULT 0` · `Credit DECIMAL(19,4) NOT NULL DEFAULT 0` · `Currency CHAR(3)` · `FxRate DECIMAL(19,8) NULL` · `AmountBase DECIMAL(19,4)` · `PartyType NVARCHAR(20) NULL` · `PartyId NVARCHAR(80) NULL` · `LineNote NVARCHAR(200) NULL`.
- **PK:** `LineId`. **FKs:** `EntryId → JournalEntries` · `AccountCode → ChartOfAccounts.AccountCode`.
- **Unique:** `UQ_JL_EntryLine (EntryId, LineNo)`.
- **Check:** `Debit >= 0` · `Credit >= 0` · `NOT (Debit > 0 AND Credit > 0)` · `NOT (Debit = 0 AND Credit = 0)`.
- **Indexes:** `IX_JL_Entry (EntryId)` · `IX_JL_Account (AccountCode)`.
- **Immutable enforcement:** same as 3.1. **“≥2 lines” + balance** enforced atomically in the posting transaction and re-validated by a DB check (see §3.8).
- **Retention:** permanent. **Concurrency:** insert-only.

### 3.3 `PostingEvents` (a.k.a. PostingRequests)
- **Purpose:** immutable record of each accepted posting request (the "what was asked").
- **Columns:** `EventId BIGINT IDENTITY PK` · `EventUid UNIQUEIDENTIFIER` · `CompanyId INT NOT NULL` · `BranchId INT NOT NULL` · `IdempotencyKey NVARCHAR(120)` · `CorrelationId UNIQUEIDENTIFIER` · `SourceType/SourceId` · `RuleId` · `ExpectedRuleVersion INT NULL` · `EventDate DATE` · `InputPayload NVARCHAR(MAX)` (JSON, business inputs only — never final lines) · `RequestedByUserId INT` · `ReceivedAtUtc DATETIME2(3)` · `ResultEntryId BIGINT NULL` · `Status NVARCHAR(20)` (`Accepted`|`Rejected`).
- **PK:** `EventId`. **FKs:** `ResultEntryId → JournalEntries`.
- **Unique:** `UQ_PE_Idem (CompanyId, IdempotencyKey)`.
- **Check:** `EventDate NOT NULL` · `Status IN ('Accepted','Rejected')`.
- **Immutable enforcement:** append-only. **Retention:** permanent. **Concurrency:** unique idempotency key serialises duplicates.

### 3.4 `IdempotencyRecords`
- **Purpose:** first-writer-wins registry mapping idempotency key → stored result.
- **Columns:** `IdempotencyKey NVARCHAR(120)` · `CompanyId INT NOT NULL` · `RequestHash CHAR(64)` (SHA-256 of canonical request) · `ResultEntryId BIGINT NULL` · `ResultCode NVARCHAR(40)` · `CreatedAtUtc DATETIME2(3)` · `ExpiresAtUtc DATETIME2(3) NULL`.
- **PRIMARY KEY (CompanyId, IdempotencyKey)**. **FK:** `ResultEntryId → JournalEntries`.
- **Check:** `RequestHash NOT NULL`.
- **Immutable enforcement:** result fields write-once (set on first commit). **Retention:** long-lived (≥ legal window). **Concurrency:** composite-PK `(CompanyId, IdempotencyKey)` insert conflict ⇒ replay path (return stored result if `RequestHash` matches, else `ACC_IDEMPOTENCY_CONFLICT`).

### 3.5 `PostingSnapshots`
- **Purpose:** frozen inputs used to build the entry (ACC-B evidence).
- **Columns:** `SnapshotId BIGINT IDENTITY PK` · `EntryId BIGINT` · `RuleId` · `RuleVersion INT` · `SnapshotSchemaVersion INT NOT NULL` · `FrozenInputs NVARCHAR(MAX)` (JSON: rates, unit costs, pcts, bases…) · `SnapshotHash CHAR(64)` · `CreatedAtUtc DATETIME2(3)`.
- **PK:** `SnapshotId`. **FK:** `EntryId → JournalEntries` (1:1).
- **Unique:** `UQ_Snap_Entry (EntryId)`.
- **`SnapshotSchemaVersion` (note):** the **shape** of `FrozenInputs` may evolve **independently** from the posting rule. **`RuleVersion` ≠ `SnapshotSchemaVersion`**: `RuleVersion` versions the accounting logic (which accounts/amounts); `SnapshotSchemaVersion` versions the serialized snapshot structure (how frozen inputs are laid out). A reader deserializes `FrozenInputs` by `SnapshotSchemaVersion`, then applies rule semantics by `RuleVersion`. Both are `NOT NULL` and immutable.
- **Immutable enforcement:** append-only; hash-verified. **Retention:** permanent. **Concurrency:** insert-only.

### 3.6 `AccountingPeriods`
- **Purpose:** central period-lock authority (single source — ACC-D).
- **Columns:** `PeriodId INT IDENTITY PK` · `CompanyId INT` · `PeriodStart DATE` · `PeriodEnd DATE` · `State NVARCHAR(12)` (`Open`|`Locked`|`Closed`) · `LockedAtUtc DATETIME2(3) NULL` · `LockedByUserId INT NULL` · `RowVersion ROWVERSION`.
- **PK:** `PeriodId`. **Unique:** `UQ_Period (CompanyId, PeriodStart, PeriodEnd)`.
- **Check:** `PeriodEnd >= PeriodStart` · `State IN ('Open','Locked','Closed')`.
- **Indexes:** `IX_Period_Range (CompanyId, PeriodStart, PeriodEnd)`.
- **Immutable enforcement:** state transitions only via `PeriodLockService` (authorised + audited); no ad-hoc lock writers (closes CLOSE-05). **Retention:** permanent. **Concurrency:** `RowVersion` optimistic on state change.

### 3.7 `AuditEvents`
- **Purpose:** tamper-resistant audit of every write (KI-004).
- **Columns:** `AuditId BIGINT IDENTITY PK` · `CompanyId INT NOT NULL` · `EntryId BIGINT NULL` · `EventId BIGINT NULL` · `Action NVARCHAR(40)` · `ActorUserId INT` · `AtUtc DATETIME2(3)` · `CorrelationId UNIQUEIDENTIFIER` · `Detail NVARCHAR(MAX)` (no secrets) · `SeqNo BIGINT` (per-`CompanyId` monotonic) · `PrevHash CHAR(64) NULL` · `RowHash CHAR(64)`.
- **PK:** `AuditId`. **FKs:** `EntryId → JournalEntries` · `EventId → PostingEvents`.
- **Immutable enforcement:** append-only; **hash chain** (`RowHash = H(PrevHash‖canonical row)`) computed **server-side only**; `DENY UPDATE, DELETE`.
- **Retention:** permanent. **Concurrency:** insert-only; written **in the same transaction** as the journal (AI-11).

### 3.8 Cross-table minimum constraints (enforced in posting transaction + DB)
- `JournalEntry` has **≥ 2 lines**; total `Debit = Credit` (validated pre-commit; DB re-check via constraint/trigger).
- Each line **Debit XOR Credit**, both **≥ 0** (§3.2 checks).
- **Unique posting key** per company (§3.1).
- **One reversal per original** unless explicitly versioned (`UQ` filtered index `WHERE EntryKind='Reversal'` on `ReversesEntryId`).
- Original and reversal references **immutable** (append-only + triggers).
- `EventDate`, `SourceType`, `SourceId`, `BaseCurrency` **required**.
- `AccountCode` must reference an **active** account (`IChartOfAccountsService`).
- **No `UPDATE`/`DELETE`** on posted entries and lines (permissions + triggers).

### 3.9 `PostingRules` (rule registry catalogue — logical, not implemented)
- **Purpose:** metadata catalogue backing `IPostingRuleRegistry` (§5); discovery/version/activation of rules. **Does not** store journal data.
- **Columns:** `RuleId NVARCHAR(40)` · `Version INT` · `DisplayName NVARCHAR(120)` · `RequiredPermission NVARCHAR(60)` · `SnapshotSchemaVersion INT` · `State NVARCHAR(12)` (`Active`|`Deprecated`|`Retired`) · `EffectiveFromUtc DATETIME2(3)` · `DeprecatedAtUtc DATETIME2(3) NULL` · `CompatibleWith NVARCHAR(200) NULL` (prior versions) · `CreatedAtUtc DATETIME2(3)`.
- **PK:** `(RuleId, Version)`. **Active-version scope (logical uniqueness):** Only one active version is permitted within each effective scope. A global active rule may coexist with company-specific active overrides. Each company may have at most one active version for the same `RuleId`. Rules belonging to different companies do not conflict with each other. At runtime, an active company-specific rule takes precedence over the active global rule for that company. If no company-specific active rule exists, the global active rule is used. (Enforced later via a filtered uniqueness constraint per effective scope; no final executable SQL yet.)
- **Check:** `State IN ('Active','Deprecated','Retired')` · `Version >= 1`.
- **Indexes:** `IX_PR_State (State)` · `IX_PR_Rule (RuleId)`.
- **Immutable enforcement:** a published `(RuleId, Version)` row is immutable except `State`/`DeprecatedAtUtc` transitions (audited). New logic ⇒ new `Version`, never edit-in-place.
- **Retention:** permanent (needed to interpret historical `RuleVersion`). **Concurrency:** optimistic on `State`.

> **لا يُنفَّذ الجدول في هذه الجلسة** — تعريف منطقي فقط، مُدخَل لبناء `schema.sql` لاحقاً.

### 3.10 `JournalNumberSequences` (atomic journal-number allocation — logical, not implemented)
- **Purpose:** backs `IJournalNumberAllocator` (§5); source of the sequential, never-reused `JournalNumber` per legal entity. **Not** `EntryId` (surrogate) — a distinct legal number.
- **Columns:** `CompanyId INT NOT NULL` · `NumberingSeries NVARCHAR(20) NOT NULL` · `FiscalYear INT NOT NULL DEFAULT 0` · `BranchScopeId INT NOT NULL DEFAULT 0` · `NextValue BIGINT NOT NULL` · `RowVersion ROWVERSION`.
- **Scope semantics:** `FiscalYear = 0` ⇒ the sequence **does not reset annually**; `BranchScopeId = 0` ⇒ **company-wide** numbering; `BranchScopeId > 0` ⇒ **branch-specific** numbering.
- **PK:** `PRIMARY KEY (CompanyId, NumberingSeries, FiscalYear, BranchScopeId)` — non-nullable scope columns (no NULL normalisation) — one counter per (company, series, year, branch-scope).
- **Check:** `NextValue >= 1` · `NumberingSeries NOT NULL`.
- **Allocation semantics:** the allocator reads-and-increments **inside the posting transaction** (steps 15–21) under `RowVersion` optimistic concurrency (or an atomic `UPDATE … OUTPUT`), so a committed entry owns a unique number and a rolled-back entry leaves **gaps allowed, no reuse**.
- **Immutable enforcement:** counter row is mutable (monotonic increment only); allocated `JournalNumber` values on `JournalEntries` are immutable (§3.1).
- **Retention:** permanent. **Concurrency:** duplicate numbers prevented by the unique `JournalNumber` index (§3.1) + atomic increment; a lost optimistic race retries the increment (not the whole posting).
- **Scope config:** whether numbering is **company-wide or branch-specific** is configuration-driven via `BranchScopeId` (`0` = company-wide; `>0` = branch-specific) in the key.

> **لا يُنفَّذ الجدول في هذه الجلسة** — تعريف منطقي فقط.

---

## 4. API Contract

**Global rules:** all endpoints are under the versioned base path **`/api/v1/accounting/...`**; JSON; `Authorization` bearer required. **HTTP header authority (external requests):** the **`Idempotency-Key` header is the authoritative idempotency source** and the **`X-Correlation-Id` header is the authoritative correlation source**; the **server generates a correlation ID when the header is absent**. `idempotencyKey`/`correlationId` are **not public body fields** — they are internal DTO fields populated **only by the controller** from headers. If a request body nonetheless carries a value that **conflicts** with the header, the request is **rejected as `ACC_INVALID_REQUEST`**. **The API never accepts final journal lines from ordinary modules** — it accepts business event identity, source identity, event date, permitted business inputs, and expected rule version; the server resolves the rule and builds lines. **Manual journal is a separate, explicitly authorised rule (`RuleId=MANUAL_JOURNAL`), not a bypass.**

### 4.1 `POST /api/v1/accounting/postings`
- **Purpose:** submit a business event for posting.
- **Auth:** `accounting.post` (+ rule-specific permission resolved server-side).
- **Request (body):** `{ sourceType, sourceId, ruleId, expectedRuleVersion?, eventDate, inputs{…} }`. **`idempotencyKey` comes from the `Idempotency-Key` header** and **`correlationId` from `X-Correlation-Id`** (controller-populated internal DTO fields, not public body fields); a conflicting body value ⇒ `ACC_INVALID_REQUEST`. For `MANUAL_JOURNAL`, `inputs` carries proposed lines **as a request** that still passes full validation + `accounting.post.manual` permission (not a raw write).
- **Response (201):** `PostingResult { entryId, entryUid, postingKey, ruleId, ruleVersion, lines[], balanced:true, status:'Posted', correlationId }`.
- **Validation:** shape → idempotency/correlation → authorise → event date → period lock → resolve rule (+version) → snapshot → build → accounts → debit/credit → balance (see §6).
- **Success:** `201 Created` (new) · `200 OK` (idempotent replay of identical request).
- **Errors:** 400/401/403/409/404/422 per §7.
- **Idempotency:** identical `idempotencyKey` + matching request hash ⇒ return stored result (`200`); same key + different hash ⇒ `409 ACC_IDEMPOTENCY_CONFLICT`.
- **Audit:** successful postings are recorded in `AuditEvents` **inside the ledger transaction** (§6 step 19). **Pre-transaction business rejections** are recorded **only in the operational/security rejection log** and **do not create** ledger `AuditEvents`, journal records, idempotency results, or `OutboxMessages` (consistent with §6 and §11).
- **Example (req):**

  Headers:
  ```
  Idempotency-Key: SPV-1042:v1
  X-Correlation-Id: <valid-guid>
  ```
  Body:
  ```json
  {
    "sourceType": "SupplierPayment",
    "sourceId": "SPV-1042",
    "ruleId": "JS-16",
    "eventDate": "2026-06-30",
    "inputs": {
      "purchaseId": "PINV-88",
      "amount": 50000,
      "treasury": "1021"
    }
  }
  ```
  → **201** with server-built lines (2010 Dr / 1021 Cr + FX lines per rule).

### 4.2 `POST /api/v1/accounting/postings/{entryId}/reverse`
- **Purpose:** create a dated reversal of an existing entry (ACC-C).
- **Auth:** `accounting.reverse`.
- **Request (body):** `ReversalRequest { reason, reversalDate }`. **`idempotencyKey` from the `Idempotency-Key` header** and **`correlationId` from `X-Correlation-Id`** (controller-populated; conflicting body value ⇒ `ACC_INVALID_REQUEST`).
- **Response (201):** `PostingResult` for the reversal entry (`EntryKind=Reversal`, `ReversesEntryId=entryId`).
- **Validation:** original exists → not already reversed (unless versioned) → reversal date not in locked period → authorise.
- **Success:** `201` · **Errors:** 404 `ACC_ORIGINAL_ENTRY_NOT_FOUND` · 409 `ACC_REVERSAL_ALREADY_EXISTS` · 422 `ACC_REVERSAL_NOT_ALLOWED` · 403 · 409 `ACC_PERIOD_LOCKED`.
- **Idempotency:** by `idempotencyKey`. **Audit:** reversal audited in same transaction. **No delete/update ever occurs.**

### 4.3 `GET /api/v1/accounting/journal-entries/{entryId}`
- **Purpose:** fetch a posted entry + lines (read model).
- **Auth:** `accounting.read`. **Response (200):** entry + lines + snapshot ref. **Errors:** 404 `ACC_ORIGINAL_ENTRY_NOT_FOUND`, 403. Read-only; no audit write.

### 4.4 `GET /api/v1/accounting/journal-entries/by-source?sourceType=&sourceId=`
- **Purpose:** fetch entries for a business source (incl. reversals).
- **Auth:** `accounting.read`. **Response (200):** array (may be empty). **Validation:** both params required (400 `ACC_INVALID_REQUEST`).

### 4.5 `GET /api/v1/accounting/postings/{idempotencyKey}`
- **Purpose:** look up the stored result for an idempotency key (replay/confirm).
- **Auth:** `accounting.read`. **Response (200):** stored `PostingResult` or mapped error result. **Errors:** 404 if unknown key. No mutation.

### 4.6 API Versioning Strategy
- **URI versioning** with a single major version segment: `/api/v1/accounting/...`.
- **Breaking changes require a new API version** (`/api/v2/...`) — e.g. removing/renaming a field, changing a type, changing validation semantics, or altering an error contract.
- **Non-breaking changes** (additive optional fields, new endpoints, new optional headers) stay within `v1`.
- **Deprecation:** an older version remains available during a published deprecation window; `Deprecation`/`Sunset` response headers announce end-of-life. Rule-level evolution is handled separately via `RuleVersion`/`expectedRuleVersion` (payload), independent of the API version.
- The internal domain contracts (§5) are not exposed directly; only the versioned HTTP surface is public.

---

## 5. Domain Contracts (C# interfaces — signatures only, no implementation)

> عقود فقط. لا كود تنفيذي كامل.

```csharp
public interface IPostingService {
    // Resolve rule, snapshot, build, validate, persist atomically. Idempotent.
    Task<PostingResult> PostAsync(PostingRequest request, PostingContext ctx, CancellationToken ct);
    Task<PostingResult> ReverseAsync(long entryId, ReversalRequest req, PostingContext ctx, CancellationToken ct);
    // Failure: returns typed error result / throws PostingException(errorCode); never partial writes.
}

public interface IPostingRule {
    string RuleId { get; }
    int Version { get; }
    string RequiredPermission { get; }               // server-side authz key
    // Build lines from FROZEN snapshot only (never live values). Pure/deterministic.
    JournalDraft Build(PostingSnapshot snapshot, PostingContext ctx);
    // Failure: throws PostingRuleException → ACC_INVALID_JOURNAL_LINE / ACC_UNBALANCED_ENTRY.
}

public interface IPostingRuleResolver {
    IPostingRule Resolve(string ruleId, int? expectedVersion);
    // Failure: ACC_RULE_NOT_FOUND / ACC_RULE_VERSION_MISMATCH.
}

public interface IPostingRuleRegistry {
    // Discovery + lifecycle of posting rules (backed by PostingRules catalogue, §3.9).
    IReadOnlyCollection<RuleDescriptor> Discover();                 // rule discovery
    RuleDescriptor GetVersion(string ruleId, int version);          // version lookup
    void Register(RuleDescriptor rule);                             // registration (new RuleId/Version)
    void Activate(string ruleId, int version, PostingContext ctx);  // activation (one Active per RuleId)
    void Deprecate(string ruleId, int version, PostingContext ctx); // deprecation (audited)
    void ValidateCompatibility(string ruleId, int fromVersion, int toVersion); // compatibility validation
    // Failure: ACC_RULE_NOT_FOUND / ACC_RULE_VERSION_MISMATCH. Never mutates published logic in place.
}

public interface IJournalBuilder {
    // Assembles JournalEntry+lines from a rule draft; enforces ≥2 lines, XOR, non-negative, balance.
    JournalEntry BuildEntry(JournalDraft draft, PostingContext ctx);
    // Failure: ACC_INVALID_JOURNAL_LINE / ACC_UNBALANCED_ENTRY. Never reads mutable live state.
}

public interface IPeriodLockService {
    PeriodState GetState(DateOnly eventDate, int companyId);
    void EnsurePostingAllowed(DateOnly eventDate, int companyId);   // throws ACC_PERIOD_LOCKED
    void Lock(int periodId, PostingContext ctx);                    // authorised + audited
    void Unlock(int periodId, PostingContext ctx);                  // authorised + audited
}

public interface IPermissionService {
    void Authorize(string permissionKey, PostingContext ctx);       // deny-by-default; throws ACC_PERMISSION_DENIED
    bool Has(string permissionKey, PostingContext ctx);
}

public interface IAuditService {
    // Enlisted in the SAME unit of work as the journal write (AI-11).
    void Record(AuditEvent evt, IUnitOfWork uow);                   // failure ⇒ ACC_AUDIT_WRITE_FAILED ⇒ rollback all
}

public interface ISnapshotProvider {
    // Freezes all rule inputs at posting time; returns immutable snapshot + hash.
    PostingSnapshot Capture(PostingRequest req, IPostingRule rule, PostingContext ctx); // failure ⇒ ACC_SNAPSHOT_FAILED
}

public interface IIdempotencyService {
    IdempotencyLookup Check(string key, string requestHash, int companyId);  // Hit/Miss/Conflict
    void Store(string key, string requestHash, PostingResult result, IUnitOfWork uow); // write-once, same tx
}

public interface IChartOfAccountsService {
    bool IsActive(string accountCode, int companyId);               // ACC_ACCOUNT_NOT_FOUND / ACC_ACCOUNT_INACTIVE
    AccountRef Get(string accountCode, int companyId);
}

public interface IJournalNumberAllocator {
    // Allocate a legal JournalNumber atomically INSIDE the posting transaction (steps 15–21).
    // Never reuses numbers; gaps allowed; duplicates prevented under concurrency.
    string Allocate(int companyId, string numberingSeries, int fiscalYear, int branchScopeId, IUnitOfWork uow);
    // Scope (company-wide vs branch-specific) is configuration-driven. Never uses EntryId as the number.
    // Failure: ACC_PERSISTENCE_FAILED (rolls back the posting transaction).
}

public interface IUnitOfWork : IDisposable {
    void Begin(IsolationLevel level);                               // see §9 recommendation
    Task CommitAsync(CancellationToken ct);                         // failure ⇒ ACC_PERSISTENCE_FAILED
    void Rollback();                                                // total rollback (§6)
}
```

---

## 6. Posting Transaction Flow (exact order + rollback)

| Step | Action | Failure → error | Tx state |
|---|---|---|---|
| 1 | Receive request | — | pre-tx |
| 2 | Validate request shape | `ACC_INVALID_REQUEST` (400) | pre-tx |
| 3 | Validate correlation + idempotency keys | `ACC_INVALID_REQUEST` (400) | pre-tx |
| 4 | **Authorise action** (deny-by-default) | `ACC_PERMISSION_DENIED` (403) | pre-tx |
| 5 | Validate event date | `ACC_INVALID_EVENT_DATE` (400) | pre-tx |
| 6 | **Check period lock** | `ACC_PERIOD_LOCKED` (409) | pre-tx |
| 7 | Resolve posting rule + version | `ACC_RULE_NOT_FOUND` (404) / `ACC_RULE_VERSION_MISMATCH` (409) | pre-tx |
| 8 | Check existing idempotency record | Hit ⇒ return stored (200); Conflict ⇒ `ACC_IDEMPOTENCY_CONFLICT` (409) | pre-tx |
| 9 | **Create immutable snapshot** | `ACC_SNAPSHOT_FAILED` (500) | pre-tx |
| 10 | Build journal lines (from snapshot) | `ACC_INVALID_JOURNAL_LINE` (422) | pre-tx |
| 11 | Validate accounts (active) | `ACC_ACCOUNT_NOT_FOUND` (404) / `ACC_ACCOUNT_INACTIVE` (422) | pre-tx |
| 12 | Validate debit/credit rules (XOR, ≥0) | `ACC_INVALID_JOURNAL_LINE` (422) | pre-tx |
| 13 | Validate total balance + ≥2 lines | `ACC_UNBALANCED_ENTRY` (422) | pre-tx |
| 14 | **BEGIN SQL transaction** | `ACC_PERSISTENCE_FAILED` (500) | tx open |
| 15 | Persist `PostingEvent` | ↓ rollback | tx |
| 16 | Persist `JournalEntry` | ↓ rollback | tx |
| 17 | Persist `JournalLines` | ↓ rollback | tx |
| 18 | Persist `PostingSnapshot` | ↓ rollback | tx |
| 19 | Persist `AuditEvent` | `ACC_AUDIT_WRITE_FAILED` → rollback | tx |
| 20 | Persist idempotency result (write-once) | `ACC_IDEMPOTENCY_CONFLICT`/`ACC_PERSISTENCE_FAILED` → rollback | tx |
| 21 | Persist `OutboxMessage` (`PostingCompleted`/`PostingReversed`) | `ACC_PERSISTENCE_FAILED` → rollback | tx |
| 22 | **Commit** | `ACC_PERSISTENCE_FAILED` → rollback | commit |
| 23 | Return stored result | — | post |

**Atomic persistence set (steps 15–21, one SQL transaction):** `PostingEvent` · `JournalEntry` · `JournalLines` · `PostingSnapshot` · `AuditEvent` · `IdempotencyRecord` · `OutboxMessage`.

**Rollback rule (any failure at steps 14–22):** the entire unit of work is rolled back — **no journal, no lines, no snapshot, no audit, no idempotency, and no Outbox message** persist partially; **failure to write the `OutboxMessage` rolls back the whole transaction.** **No success is returned unless commit succeeds** (kills false-success SP-06). **No integration event is published inside the posting transaction** — the dispatcher publishes only *after* commit, from the committed Outbox (§11). Steps 1–13 fail before any write (safe). Reversal flow (§4.2) uses the same 14–23 envelope.

**Rejected-request recording (pre-transaction):** rejections at **steps 1–13** (shape, keys, authorisation, event date, period lock, rule resolution, snapshot/build/account/balance validation) occur **before** `BEGIN` and are recorded **only in an operational/security rejection log** (`PostingRejected`, §11). Such rejections **do not create** a `JournalEntry`, `JournalLines`, `PostingSnapshot`, idempotency result, or `OutboxMessage`, and are **not accounting integration events**. Only **successful posting and reversal** `AuditEvent`s live **inside** the ledger transaction (step 19). **Infrastructure failures** (`ACC_*_FAILED`) are operational errors only.

---

## 7. Error Catalogue

| Code | HTTP | Retryable | Client message | Internal logging |
|---|---|---|---|---|
| `ACC_INVALID_REQUEST` | 400 | No | "Request is invalid." | field-level validation detail |
| `ACC_INVALID_EVENT_DATE` | 400 | No | "Event date is invalid." | date + rule |
| `ACC_PERIOD_LOCKED` | 409 | No | "Accounting period is locked." | period id, event date, actor |
| `ACC_PERMISSION_DENIED` | 403 | No | "Not authorised." | permission key, actor (no data leak) |
| `ACC_DUPLICATE_POSTING` | 409 | No | "This posting already exists." | posting key, existing entryId |
| `ACC_IDEMPOTENCY_CONFLICT` | 409 | No | "Idempotency key reused with different request." | key, both hashes |
| `ACC_CONCURRENCY_CONFLICT` | 409 | No | "State changed since it was loaded — reload current state before retry." | entity, expected vs actual RowVersion, actor |
| `ACC_RULE_NOT_FOUND` | 404 | No | "Posting rule not found." | ruleId |
| `ACC_RULE_VERSION_MISMATCH` | 409 | No | "Rule version mismatch." | expected/actual version |
| `ACC_ACCOUNT_NOT_FOUND` | 404 | No | "Account not found." | accountCode |
| `ACC_ACCOUNT_INACTIVE` | 422 | No | "Account is inactive." | accountCode |
| `ACC_INVALID_JOURNAL_LINE` | 422 | No | "Invalid journal line." | line detail, rule |
| `ACC_UNBALANCED_ENTRY` | 422 | No | "Entry is not balanced." | debit/credit totals |
| `ACC_REVERSAL_NOT_ALLOWED` | 422 | No | "Reversal not allowed." | entryId, reason |
| `ACC_REVERSAL_ALREADY_EXISTS` | 409 | No | "Entry already reversed." | original + reversal id |
| `ACC_ORIGINAL_ENTRY_NOT_FOUND` | 404 | No | "Original entry not found." | entryId |
| `ACC_SNAPSHOT_FAILED` | 500 | Yes | "Temporary error, retry." | rule, inputs hash |
| `ACC_AUDIT_WRITE_FAILED` | 500 | Yes | "Temporary error, retry." | correlationId (triggers full rollback) |
| `ACC_PERSISTENCE_FAILED` | 500 | Yes | "Temporary error, retry." | tx step, correlationId |

---

## 8. P0 Acceptance Criteria

> Format: **Given / When / Then · ADR · AINV · KI/runtime · test type.**

**AC-1 Append-only journal.** Given a posted entry · When any `UPDATE`/`DELETE` is attempted · Then it is rejected at DB and service. ADR: ACC-A/ACC-C · AINV-02 · KI-008/024 · *database, integration*.

**AC-2 Balanced journal.** Given a rule draft · When built · Then `Σdebit=Σcredit` and ≥2 lines, else `ACC_UNBALANCED_ENTRY`. ADR: ACC-A · AINV-01 · runtime `_jrnBad` · *unit*.

**AC-3 Atomic posting.** Given a failure at steps **14–22** · When posting runs · Then **nothing from the complete atomic persistence set persists** — no `PostingEvent`, `JournalEntry`, `JournalLines`, `PostingSnapshot`, `AuditEvent`, `IdempotencyRecord`, or `OutboxMessage`. ADR: ACC-A · AINV-01 · SP-06 · *integration, database*.

**AC-4 Idempotency.** Given the same idempotency key + identical request · When re-sent · Then the stored result is returned once, no second entry. ADR: ACC-A · AINV-04 · KI-009 · *integration, concurrency*.

**AC-5 Duplicate prevention.** Given the same posting key (e.g. order→invoice, month payroll, invoice payment) · When re-posted · Then `ACC_DUPLICATE_POSTING`. ADR: ACC-A · AINV-04 · KI-009/030 · SP-03 · *integration, database*.

**AC-6 Central period-lock.** Given a locked period · When posting/reversal on that date · Then `ACC_PERIOD_LOCKED` (via the single `PeriodLockService`). ADR: ACC-D · AINV-26 · CLOSE-05 · *integration, security*.

**AC-7 Server-side authorisation.** Given a client without permission · When it calls any write · Then `ACC_PERMISSION_DENIED`, regardless of UI. ADR: ACC-D · KI-002 · SP-07/PAY-05/FX-01 · *security, integration*.

**AC-8 Immutable snapshots.** Given a rule needing rates/costs · When posted · Then the snapshot freezes them and the built lines never reflect later live changes. ADR: ACC-B · AINV-29 · KI-022/023/025 · *unit, integration*.

**AC-9 Reversal instead of delete.** Given a correction need · When requested · Then a dated reversal entry is created; original untouched; no delete. ADR: ACC-C · AINV-02 · KI-008/020/024/027 · *integration, database*.

**AC-10 Audit in same transaction.** Given any posting/reversal · When committed · Then the audit row commits atomically with it; audit failure rolls back all. ADR: ACC-C · KI-004 · SP-06 · *integration, database*.

**AC-11 False-success prevention.** Given a write that does not commit · When responding · Then no success is returned. ADR: ACC-C/ACC-D · SP-06 · *integration*.

**AC-12 Account validity.** Given a line account · When building · Then it must reference an active account, else `ACC_ACCOUNT_NOT_FOUND`/`ACC_ACCOUNT_INACTIVE`. ADR: ACC-A · Glossary §10 · *unit, database*.

**AC-13 Traceability.** Given any entry · When stored · Then `SourceType/SourceId/RuleId/RuleVersion/EventDate/CorrelationId` are all present. ADR: ACC-A · AINV-02 · *unit, database*.

**AC-14 Concurrency safety.** Given two concurrent identical postings · When racing · Then exactly one entry is created (unique posting key + idempotency), the other returns the stored result or `ACC_DUPLICATE_POSTING`. ADR: ACC-A · AINV-04 · *concurrency, database*.

---

## 9. Security Requirements

- **Deny-by-default authorisation**; every write authorised on the server (§5 `IPermissionService`).
- **No trust in client-supplied role or account lines** — API takes business inputs, server builds lines.
- **Least-privilege DB user**: app role has `INSERT`/`SELECT` on ledger tables, **no `UPDATE`/`DELETE`**; no DDL.
- **No direct table access for business modules** — only `IPostingService`.
- **Audit tamper resistance**: append-only + server-side hash chain; DB `DENY UPDATE, DELETE`.
- **Secrets outside source control**; config/secret store; no secrets in logs.
- **Structured logs** with correlation IDs and **no financial secrets / PII amounts in plaintext logs** beyond what policy allows.
- **Rate limiting** on write endpoints; **replay protection** via idempotency + request hash.
- **Correlation IDs** end-to-end.
- **Secure exception handling**: typed error codes to client, full detail server-side only.
- **Concurrency decision:** default **optimistic** (`ROWVERSION`) for period state + read models; rely on **unique posting key + idempotency PK** for insert races. Pessimistic locks only if a proven hotspot.
- **Optimistic concurrency handling (explicit):**
  - **Client-driven retry:** on a concurrency failure the server returns a distinct response; the **client decides** whether to re-submit (re-reading current state first). Idempotency key makes a safe re-submit non-duplicating.
  - **No automatic silent retry:** the server does **not** transparently retry a conflicting write; silent retries could mask a changed baseline. (Transient infra faults `ACC_*_FAILED` are separately retryable per §7.)
  - **Stale rowversion handling:** a state-changing call (e.g. period lock/unlock) carries the `RowVersion` it read; if it no longer matches, the write is rejected — the caller must re-read and re-decide.
  - **Optimistic concurrency failure response:** reused idempotency keys return `409 ACC_IDEMPOTENCY_CONFLICT`; **stale `RowVersion` state changes (e.g. period lock/unlock) return `409 ACC_CONCURRENCY_CONFLICT`**, `retryable = No` — the client must **reload current state before retry**. Idempotency behaviour is unchanged.
- **Transaction isolation recommendation:** `READ COMMITTED SNAPSHOT` (RCSI) for the posting transaction; uniqueness/idempotency constraints (not isolation) prevent duplicates.
- **Tenant-scoped idempotency lookup:** `GET /api/v1/accounting/postings/{idempotencyKey}` (and the idempotency check at §6 step 8) resolve **only within the caller's `CompanyId`** — an idempotency key from another tenant is never returned or matched.
- **Payload size limits:** enforced maximum request/body and per-field JSON size (`inputs`, `Detail`, `Payload`); oversized ⇒ `ACC_INVALID_REQUEST` (413 at the gateway).
- **Allow-listed schemas:** `InputPayload` (PostingEvents), `FrozenInputs` (PostingSnapshots), audit `Detail` (AuditEvents), and Outbox `Payload` are validated against **registered allow-listed JSON schemas** (keyed by rule/`SnapshotSchemaVersion`/event type); unknown/extra fields rejected.
- **Secrets & PII prohibition:** **no secrets and no unrestricted PII** in any JSON payload (`InputPayload`/`FrozenInputs`/`Detail`/`Payload`); only fields required for the accounting fact, per the allow-listed schema.
- **Atomic audit-hash sequencing:** the audit hash chain is sequenced **per `CompanyId`** (`SeqNo` + `PrevHash`→`RowHash`) computed server-side within the ledger transaction, **or** an approved **database-ledger** alternative (e.g. SQL Server ledger tables). No cross-tenant chain interleaving.
- **Administrative-only rule lifecycle:** `IPostingRuleRegistry` **registration, activation, and deprecation** require an **administrative permission** (`accounting.rules.admin`), deny-by-default, fully audited — never available to ordinary posting callers.
- **No absolute-security claim** — these reduce, not eliminate, risk; subject to security review.

---

## 10. Pending Decisions (extension points — Phase 1 core must not hard-code outcomes)

| OQ | Affects | Blocking scope (Phase 1) | Extension point | Default Phase 1 behaviour |
|---|---|---|---|---|
| **OQ-6.A** | ACC-E / account 2017 settlement | Does not block Phase 1 core (blocks ACC-E only) | `IPostingRule` for a future settlement rule | no settlement rule registered; 2017 accrual posts normally |
| **OQ-6.D** | payroll rule | Does not block Phase 1 core | payroll `IPostingRule` + posting-key strategy | payroll rule **not registered**; duplicate-prevention key reserved |
| **OQ-6.F** | final FX rule | Does not block Phase 1 core | FX `IPostingRule` behind a feature flag | no FX engine bound; `PendingDecision` flag off |
| **OQ-6.G** | close-readiness enforcement | Does not block Phase 1 core | `IPeriodLockService` readiness hook | lock enforced; readiness checks advisory only |
| **OQ-6.H** | claims-provision rule | Does not block Phase 1 core | provision `IPostingRule` + basis provider | provision rule not registered |

> **Rule:** Phase 1 core (PostingService, JournalBuilder, persistence, idempotency, lock, auth, audit) is **decision-agnostic**. Each unresolved outcome is injected later as a registered `IPostingRule`/hook. No `PendingDecision` outcome is compiled into the core path.

---

## 11. Integration Events and Outbox

> **The journal, journal lines, audit record, idempotency record, posting snapshot, and Outbox message are committed atomically in one database transaction. Only after that transaction commits may the dispatcher publish integration events.** This does **not** change the §6 posting transaction order and does **not** move the Outbox into a separate transaction.

- **Outbox table `OutboxMessages`:** `OutboxId BIGINT IDENTITY PK` · `CompanyId INT NOT NULL` · `MessageUid UNIQUEIDENTIFIER` · `EventType NVARCHAR(60)` · `EntryId BIGINT NULL` · `CorrelationId UNIQUEIDENTIFIER` · `Payload NVARCHAR(MAX)` (JSON) · `OccurredAtUtc DATETIME2(3)` · `Status NVARCHAR(12)` (`Pending`|`Dispatched`|`Failed`) · `Attempts INT DEFAULT 0` · `NextAttemptUtc DATETIME2(3) NULL` · `DispatchedAtUtc DATETIME2(3) NULL`. **Unique:** `UQ_Outbox_Uid (MessageUid)`. **Index:** `IX_Outbox_Pending (CompanyId, Status, NextAttemptUtc)`. Written **in the same DB transaction** as the journal (the Outbox row is persisted at **step 21** of §6 and committed atomically with the ledger).
- **`IntegrationEvent` entity (payload contract):** `{ messageUid, eventType, entryId?, journalNumber?, sourceType, sourceId, correlationId, occurredAtUtc, data{…} }`. Carries **no** mutable state; a stable, serialized fact.
- **Events:**
  - **`PostingCompleted`** — published from the **committed Outbox** after a successful posting commit (carries `entryId`, `journalNumber`, source identity).
  - **`PostingReversed`** — published from the **committed Outbox** after a reversal commit (carries reversal `entryId` + `reversesEntryId`).
  - **`PostingRejected`** — a **business rejection**. **Recorded as Audit / Operational information**, **not** treated as a committed accounting integration event (no ledger transaction committed, so it is not an Outbox event).
  - **Infrastructure failures** (transient `ACC_*_FAILED`) — **never published as business integration events**; handled operationally (logs/alerts) only.
- **Dispatcher responsibility:** a separate background dispatcher reads `Pending` Outbox rows and publishes to the message broker; it never writes the ledger. Publication and status update (`Dispatched`) are isolated from the posting path.
- **Delivery guarantees:** **at-least-once**. The single atomic transaction guarantees the event is *recorded* in the Outbox together with the ledger; delivery after commit is at-least-once, so consumers must be idempotent.
- **Retry policy:** exponential backoff on `NextAttemptUtc`; bounded attempts, then `Failed` + alert; poison messages quarantined, never silently dropped.
- **Idempotent event publication:** each message carries a stable `MessageUid`; consumers de-duplicate on it. Re-dispatch after a crash cannot create a second logical event.
- **Ordering guarantees:** per-aggregate ordering (by `EntryId`/source) is preserved via `OccurredAtUtc` + dispatch order; **no global total order** is promised. Consumers needing order key on the aggregate.

## 12. Currency Precision Policy

> لا تُضمَّن أمثلة عملات مُصمَّتة داخل منطق الأعمال — القواعد مدفوعة ببيانات العملة، لا ثوابت مكتوبة.

- **Storage precision:** monetary amounts stored as `DECIMAL(19,4)` (base + line amounts); FX rates as `DECIMAL(19,8)`.
- **Calculation precision:** intermediate calculations carried at **higher precision** than storage (e.g. decimal 28-digit) and **rounded once** at the point a value becomes a stored journal amount.
- **Rounding policy:** **banker's rounding (round-half-to-even)** applied at the final storage step; the rounding mode is a single configured policy, not per-call.
- **Currency-specific decimal handling:** the number of decimals is a **property of the currency** (ISO 4217 minor units) resolved at runtime from currency reference data — **not hard-coded** in rules.
- **Zero-decimal currencies:** currencies with **0 minor units** are rounded/stored to whole units; no fractional minor unit is presented or posted.
- **Three-decimal currencies:** currencies with **3 minor units** round/present to 3 decimals; storage `DECIMAL(19,4)` accommodates them, and rounding uses the currency's declared scale.
- **Balance rule interaction:** the balanced-entry check (AINV-01) is evaluated on **stored (rounded) base amounts** so that `Σdebit = Σcredit` holds exactly after rounding.

## 13. Soft Delete Policy

> **Governance statement (non-negotiable):**
> - **Soft delete is prohibited for all legal accounting journals.**
> - **No `IsDeleted` flag (or equivalent hide/void-in-place marker) is permitted** on `JournalEntries`, `JournalLines`, `PostingSnapshots`, `AuditEvents`, or Outbox ledger records.
> - **Correction must always use reversal** (dated reversal entry per ACC-C / §4.2), never hiding or flagging a row as deleted.

This restates AI-3/AI-4/AI-5 at the data-governance level and forecloses a soft-delete workaround.

## 14. Validation

- **Created file:** `docs/03_Business_Logic/Accounting/Accounting_Posting_Service_Spec.md`
- **Document version:** 1.0.0
- **Section count:** 14
- **Transaction-flow step count:** 23 (Outbox persisted at step 21, inside the transaction)
- **Table count:** 17 (invariants · 8 ledger/support schema tables incl. `JournalNumberSequences` · `PostingRules` · `OutboxMessages` · cross-table constraints · tenant-ownership matrix · transaction flow · error catalogue · pending-decisions)
- **Interface count:** 13 (added `IJournalNumberAllocator`)
- **Endpoint count:** 5 (all under `/api/v1/accounting/...`)
- **Error-code count:** 19 (added `ACC_CONCURRENCY_CONFLICT` for stale RowVersion)
- **Acceptance-criteria count:** 14 (AC-1 → AC-14, unchanged)
- **PendingDecision count:** 5 (OQ-6.A · OQ-6.D · OQ-6.F · OQ-6.G · OQ-6.H, unchanged)
- **No existing governance or accounting document modified** — this file only.
- **No executable schema or application code created** — logical schema + interface signatures only.

**Final verdict: `READY FOR SOLUTION ARCHITECT SIGN-OFF`**
