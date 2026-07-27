# Prototype Runtime Verification Report

**Target:** `reference/prototype/prototype_v2.html`
**Type:** Runtime investigation only — the original prototype, project, and documentation were **not** modified. All execution was performed against an in-memory runtime copy loaded from a byte-identical file; no fixes were applied.
**Method:** The prototype was loaded and executed in a headless DOM runtime (jsdom). Every material screen and action was reached through the application's own functions and DOM (login → `render()`/`renderShell()` → view function → handler → central `buildJournal()` derivation), then the resulting `DB` records and derived journal entries were read back. State was reset to the seed baseline between destructive scenarios.

---

## 1. Executive summary

| Metric | Value |
|---|---|
| Screens opened & exercised (Phase 2) | **120** (113 statically reachable + 7 statically-unlinked invoked directly) |
| PASS | **112** reachable screens (+ 7 unlinked rendered OK) |
| PASS WITH DATA LIMITATION | FX Engine B posting (`fxReval`) — reachable but produces no exposure on any data |
| BROKEN | **1** — `pricing` (`ReferenceError: esc is not defined`) |
| PERMISSION FAILURE (runtime-confirmed bypass) | **5 write actions**: supplier-payment create/edit/delete, payroll post, Engine A revaluation post, recurring create/edit, period unlock |
| NOT VERIFIABLE | **1** — Engine B (`fxReval`) posted journal lines (1265/3020) could not be produced end-to-end because its exposure calc yields net 0 |
| Runtime console errors | **1 render error** (`esc` on `pricing`) + 1 benign jsdom CSS-parser warning at startup |
| **Final acceptance verdict** | **PASS WITH VERIFICATION PATCH REQUIRED** |

**Headline runtime outcomes**

1. **Every material accounting workflow executed and was observed** — supplier payments (all 7 scenarios), both payroll paths, both recurring behaviours, both FX engines (Engine A end-to-end; Engine B up to the "no difference" guard), all five period-lock paths, fixed-asset acquisition + depreciation, and both opening-balance journals.
2. **Two static-audit claims are REFUTED by runtime:**
   - *"A new fixed-asset acquisition posts no journal."* → **False.** `buildJournalCore` derives `Dr 1300 / Cr bank` for every `!a.opening` asset. A newly created asset **does** book its acquisition. (§4 FA-01)
   - *The FX double-count is a live double-posting risk.* → **Overstated.** Engine B (`fxReval`) is effectively inert: its exposure basis is recomputed at the same current rate it revalues against, so it returns net 0 and refuses to post even after a 9% rate move. The double-count is a code-level latency, not a reproducible runtime double-posting. (§4 FX-02, FX-03)
3. **Five write handlers are not permission-gated and are runtime-bypassable:** `editSuppPay`, `postPayroll`, `postRevaluation` (Engine A), `editRecurring`, and `lockPeriodNow`/`unlockPeriodNow`. Three others are correctly gated at the handler (`postFxRevaluation`/`closeFxRevaluation`, `doClosing`, `editAsset`). (§6)
4. **Confirmed defects:** supplier-payment duplicate + overpayment acceptance; locked-period delete producing a **false "deleted" success** while retaining the record; salary double-charge (5410 payroll + 5400 RC-002) with no cross-guard; five uncoordinated `DB.lockDate` writers producing inconsistent state; one hard render failure (`pricing`).

---

## 2. Environment and baseline

| Item | Value |
|---|---|
| Runtime | Node.js v22 + jsdom (headless DOM; scripts executed; `localStorage`/`sessionStorage` present) |
| Prototype file SHA-256 | `8996dd68a9f4eea241d4289f094f9f5088baf148b79efc6fa2f8ef5759d7c6b2` |
| Prototype size | 24,149 lines / 2,038,617 bytes |
| Startup console | 1 message only: jsdom "Could not parse CSS stylesheet" (styling-engine limitation, **not** an application error) |
| Persistence | `localStorage` key `vortex_accounting_v3`; `DB` falls back to `SEED` when key absent |
| Reset method | Restore `DB` from a captured seed-baseline snapshot (equivalent to clearing `localStorage` and reloading). Performed between every destructive scenario. |
| Default user | `admin` / role `owner` (all-access) |
| Seed login users | `admin`→owner, `mona`→sales, `karim`→sales, `hoda`→accountant |
| Roles defined | owner, admin, accountant, sales, warehouse, viewer (admin/warehouse/viewer have **no** seed login user; exercised by creating a user of that role) |
| Report date / lock date | `2026-06-03` / empty (unlocked) |

**Seed realities that differ from the static audit's assumptions (important):**

- `DB.employees` is **empty** and `DB.payroll` is empty — payroll posting is a no-op on pure seed data; a test employee was created to exercise PAY-02/04/05.
- `DB.autoRecurring` is **undefined** (auto-generation is OFF by default).
- The salary template **RC-002 (80,000) lives in `DB.recurring` (Engine B)**, not `DB.recurringExp` (Engine A, which is empty).
- `DB.revaluations`, `DB.fxRevaluations`, `DB.closings`, `DB.manualJournals` all start empty.
- Foreign-currency exposure exists in seed: customers in EUR/GBP/AED, a USD bank (TR-003), and FC purchase invoices (PINV-0001 EUR, PINV-0002 GBP).

A clean baseline snapshot was captured before any destructive scenario and restored between independent scenarios.

---

## 3. Screen smoke-test matrix

All 120 candidate screens were opened as `owner` (full access) and rendered via `render()`. Result column: **PASS** = rendered with substantive content, no runtime error; **BROKEN** = threw at render; **UNLINKED-OK** = statically unlinked but renders when reached directly; **DATA-DEP** = renders but content depends on records beyond seed.

| Screen ID | Role | Opens | Blank | Error | Primary controls present | Runtime verdict | Evidence |
|---|---|---|---|---|---|---|---|
| pricing | owner | **No (throws)** | — | **Yes** | — | **BROKEN** | `ReferenceError: esc is not defined` at render; `esc` is a top-level `const` duplicated across 7 `<script>` blocks (21983, 22611, 22806, 23167, 23569, 23808…) → resolves `undefined` |
| dashboard, execDash, actionCenter, today, quickEntry, customDash, reminders, hrHome, alerts | owner | Yes | No | No | Yes | PASS | Home group |
| crmDash, leads, pipeline, crmActivities, crmTasks | owner | Yes | No | No | Yes | PASS (leads/pipeline DATA-DEP) | CRM group |
| salesDocs, invoices, tracking, shipmentCosts, payments, advances, claims, dunning, commission, lcs, customers, custStatement, serviceInvoices, creditNotes | owner | Yes | No | No | Yes | PASS (tracking/custStatement DATA-DEP) | Sales/Export |
| purchases, receipts, purchaseOrders, returns, expenses, recurringExp, supplierPayments, suppliers, suppStatement, payables | owner | Yes | No | No | Yes | PASS | Purchasing |
| inventory, warehouses, workOrders, scrap, stocktake, lots, traceability, slowMoving, invRecon | owner | Yes | No | No | Yes | PASS (lots/stocktake DATA-DEP) | Warehouse |
| journal, manualJV, accounts, assets, custody, costCenters, audit, trialBalance, ledger, closing, monthClose, varianceSettle, easPanel, taxCenter, fxReval | owner | Yes | No | No | Yes | PASS | Accounting |
| banks, banksTx, cheques, treasuryStmt, reconcile | owner | Yes | No | No | Yes | PASS | Treasury |
| reportsHub, pl, balanceSheet, cashFlowStmt, financialPosition, periodCompare, aging, supplierAging, cashflow, cashForecast, cashForecast90, vat, vatReturn, withholding, currencyPos, revaluation, budget, expenseReport, expenseTrend, kpiDash, breakEven, monthEnd, wasteReport, prodReport, shipmentProfit, productProfit, customerProfit, profit, supplierCompare, invTurnover, salesCountry, lcReport | owner | Yes | No | No | Yes | PASS | Reports |
| employees, payroll, attendance, leaves, insurance | owner | Yes | No¹ | No | Yes | PASS (empty employee list is data, not defect) | HR |
| import, invoiceSettings, products, users, backup, settings, dataProtect, taskHome | owner | Yes | No | No | Yes | PASS | System |
| smartSearch, fiscalYears, stockMoves, ccBudget, docs, cashboxes, cashTx | owner | Yes | No | No | Yes | **UNLINKED-OK** (render fine when `CURRENT` set directly; no NAV entry) | Phase 4 |

¹ HR screens show a friendly "no active employees" empty state because seed has zero employees — a data condition, not a render failure.

**Totals:** 120 opened → 112 reachable PASS + 7 unlinked render OK + **1 BROKEN (`pricing`)**. This is **>99%** of statically reachable screens opened and exercised.

---

## 4. Priority workflow results

Journal lines below are the entries derived by the central `buildJournalCore()` (the app has no per-save inline journal; journals are computed from `DB`). All amounts in EGP unless a currency is shown.

### Supplier payments

**SP-01 — normal linked payment** — *PASS (high)*
- Setup: owner; PINV-0001 (EUR, balance 8,580 @ invoice rate 55.2). Action: create payment linked to PINV-0001, 1,000 EUR @ pay-rate 56.
- DB after: `SPV-002 {purchaseId:'PINV-0001', cur:'EUR', fxRate:56, amount:1000}` appended.
- Journal: `Dr 2010 55,200 / Cr 1021 56,000 / Dr 5500 800` (balanced). FX-loss line **present** because pay-rate ≠ invoice rate. Toast "تم الحفظ".
- Verdict: Dr 2010 / Cr treasury confirmed; FX line on rate difference confirmed.

**SP-02 — unlinked foreign-currency payment** — *PASS (high)*
- Action: unlinked payment, 500 USD @ 51 (no `purchaseId`/`expId`).
- Journal: `Dr 2010 25,500 / Cr 1021 25,500` — **no 4030/5500 line**. Confirmed: with no linked invoice, `invRate = payRate` ⇒ `fxDiff = 0`.

**SP-03 — duplicate payment** — *DEFECT confirmed (high)*
- Action: two payments against PINV-0001. Result: **both accepted**, `DB.supplierPayments` now holds 2 records for the same invoice. No dedup.

**SP-04 — overpayment** — *DEFECT confirmed (high)*
- Setup: PINV-0001 balance 8,580 EUR. Action: enter 999,999. Result: **accepted** (`amount:999999`), save not blocked. No ceiling / outstanding-balance check.

**SP-05 — locked-period create/edit** — *CONTROL works (high)*
- Setup: `DB.lockDate='2026-06-30'`. Create (date 2026-06-02) → blocked, toast "الفترة مقفولة", record not added. Edit seed SPV-001 (date 2026-05-20, locked) → blocked, amount unchanged (5000). `isLocked()` guard on save is effective.

**SP-06 — locked-period delete** — *DEFECT confirmed: FALSE SUCCESS (high)*
- Setup: lockDate covers SPV-001. Action: `delSuppPay(0)` → confirm.
- Result: **two toasts** — error "لا يمكن الحذف: الفترة مقفولة محاسبياً" **then** success "تم الحذف" — and the record is **not** deleted (count 1→1). The ternary skips the `splice` but `save();render();toast('تم الحذف','ok')` still runs. Misleading success message + no deletion.

**SP-07 — viewer permission bypass** — *DEFECT confirmed (high)*
- Setup: viewer (supplierPayments = view). UI: create/edit/delete buttons **visible**. Handler: `editSuppPay(-1)` opens the editor and **save succeeds** → payment created (count 1→2). No `can()` guard at UI or handler.

### Payroll and recurring salary

**PAY-01 — render + Post visibility** — *PASS with finding (high)*
- Owner and accountant both render payroll; **Post button visible to both**, though accountant has `hr:view` only (`hr:edit=false`). Button is not gated.

**PAY-02 — dedicated payroll posting** — *PASS (high)*
- Setup: 1 test employee (salary 50,000, ins 11%/18.75%, tax 2,000), month 2026-06.
- DB after: `DB.manualJournals` gains `{ref:'PAYROLL-2026-06'}` (balanced):
  `Dr 5410 50,000 / Dr 5420 9,375 / Cr 2130 14,875 / Cr 2120 2,000 / Cr 2140 42,500`. All required accounts present.
- Duplicate same-month post → **correctly blocked** (guard `some(j=>j.ref===ref)`; toast "تم الترحيل لهذا الشهر من قبل"; journal count stays 1). *(Verified precisely after eliminating a harness artifact where a stale confirm dialog was re-clicked.)*

**PAY-03 — recurring salary posting** — *PASS (high)*
- Action: `runRecurring('RC-002')` for 2026-06.
- DB after: `DB.expenses` gains `EXP-006 {amount:80000, cat:'رواتب', allocType:'overhead', recurringId:'RC-002'}`.
- Journal: `Dr 5400 80,000 / Cr 2015 80,000`. Posts to **5400** (overhead → admin-expenses account). Confirmed.

**PAY-04 — cross-path duplicate payroll** — *DEFECT confirmed (high)*
- Action: post dedicated payroll (E-001) **and** generate RC-002 for the same month.
- Account totals: `5410 = 50,000` (payroll gross) **and** `5400` includes the `80,000` RC-002 salary charge. Both persist; **no cross-guard**. Personnel cost is double-counted across two accounts (5410 + 5400). Duplicated impact ≈ 80,000.

**PAY-05 — accountant view-only bypass** — *DEFECT confirmed (high)*
- Setup: accountant (`hr:edit=false`). Post button **visible**; `postPayroll()` **executes and posts** the `PAYROLL-2026-06` journal. Handler performs no `can()` check → **ALLOWED (no gate)**.

### Recurring engines

**REC-01 — reachable engine** — *confirmed (high)*
- Menu item id `recurringExp` (label "المصروفات الدورية") renders `viewRecurring()` over **`DB.recurring`** (Engine B, "القيود المتكررة"). Active editors are `editRecurring`/`delRecurring` at 11510/11530 (Engine B). Save destination: `DB.recurring` and generated rows → `DB.expenses`. Label vs content mismatch confirmed at runtime (screen shows recurring-journal data under a "recurring-expenses" label).

**REC-02 — shadowed editor** — *confirmed: overwritten (high)*
- Engine A editors (`window.editRecurring`/`delRecurring` at 8104/8132 for `DB.recurringExp`) are redefined at 11510/11530. At runtime the later definitions win; the Engine A editors are unreachable dead code. There is no UI path to `DB.recurringExp`.

**REC-03 — auto generation** — *confirmed off by default (high)*
- `DB.autoRecurring` is undefined at boot, so `autoGenerateRecurring()` does not populate anything on a clean load. The reachable UI reads/writes `DB.recurring`/`DB.expenses`; Engine A's `DB.recurringExp` remains empty and has no inspection UI. `runGenerateRecurring` (8100) has no UI caller. When generation is driven through the reachable engine (`runRecurring`/`runAllRecurring`), records land in `DB.expenses` with a per-`recurringId`+month dedup guard (no duplicates within the reachable engine).

**REC-04 — locked-period generation** — *PARTIALLY CONFIRMED (med)*
- `genRecurring` writes the expense date as `monthStr-day` and does **not** call `isLocked()` before writing; it dedups only on `recurringId`+month. Generating into a month whose date falls in a locked period is **not blocked** by the recurring path itself (the lock guard lives on interactive editors, not the generator). No error/toast is raised by the generator for a locked target.

### FX revaluation engines

**FX-01 — Engine A (`viewRevaluation`/`postRevaluation`)** — *PASS (high)*
- Setup: closing rates raised above book (EUR 57, USD 52, GBP 66); auto-reverse on.
- Action: `postRevaluation()` → confirm. DB after: `DB.revaluations` gains `REV-001 {net:27,621, reverse:true, reverseDate:'2026-07-01', items:4}`.
- Journal (period-end): `Dr 1100 (customers) … / Dr 1020 (bank) … / Cr 4030 27,621`. Auto-reversal (2026-07-01): the mirror entry. Both present and balanced. Permission: post button gated on `journal:edit`. Confirmed 4030/5500 + auto-reversal.

**FX-02 — Engine B (`viewFxReval`/`postFxRevaluation`/`closeFxRevaluation`)** — *PASS WITH DATA LIMITATION / NOT VERIFIABLE for posted lines (high)*
- Screen renders; handler `postFxRevaluation` **correctly checks `can('accounts','edit')`** and `isLocked()`.
- **Runtime finding:** `fxExposure()` returns **empty even after a 9% rate move** (EUR 55.2→60, USD 50.5→53). Its "booked EGP" is recomputed at the same current rate used to revalue, so `diff = revalued − booked ≈ 0` for every party. Net is 0 → the handler correctly returns "لا يوجد فرق تقييم للترحيل" and posts nothing.
- Consequence: the actual posted journal lines (`Dr 1265 / Cr 4030` on post; `Dr 3020 / Cr 1265` on close, per `buildJournalCore` 3374–3400) **could not be produced end-to-end** through the engine's own exposure calc → those lines are **NOT VERIFIABLE at runtime** (present in code, unreachable via data). Engine B is, in practice, inert.

**FX-03 — double recognition** — *PARTIALLY CONFIRMED / REFINED (high)*
- Both engines have unconditional `4030/5500` emitters and **no cross-guard** (code-confirmed). At runtime, only Engine A posts (net → 4030); Engine B produces no exposure (see FX-02), so simultaneous double-posting **could not be reproduced end-to-end**. The double-count is therefore a latent code risk, **not** an observed runtime double-posting — a refinement of the static claim.

**FX-04 — role visibility** — *confirmed (high)*
- Owner: reaches both; can post both (`journal:edit` and `accounts:edit`). Accountant: reaches both (Engine A under reportsHub via `reports:view`; `fxReval` nav via `accounts:view`); can post Engine A (`journal:edit`) but Engine B post is **blocked at handler** (`accounts:edit=false`).

### Closing and period lock

**CLOSE-01 — `doClosing()`** — *PASS (high)*
- Permission gate `can('accounts','edit')` present. Preview: 2026-01-01 → 2026-06-03, net profit 544,014.33.
- DB after: `DB.manualJournals` gains closing entry `Dr 3099 544,014.33 / Cr 3020 544,014.33`; `DB.closings` records `{from,to,netProfit,by:'admin'}`; `DB.lockDate = 2026-06-03`; audit entry created (`create / إقفال فترة / JV-M0001`).
- Repeat-close guard: second `doClosing()` shows no confirm dialog and does not add a second closing (`alreadyClosed`). Confirmed.

**CLOSE-02 — `doMonthEndClose()`** — *confirmed lightweight (high)*
- Only sets `DB.lockDate` (to the `#me_lockdate` value, e.g. 2026-06-30). Creates **no** closing journal and **no** `DB.closings` record. **No permission check and no audit** in the handler.

**CLOSE-03 — wizard lock/unlock (`lockPeriodNow`/`unlockPeriodNow`)** — *confirmed ungated-but-audited (high)*
- `lockPeriodNow()` sets `DB.lockDate = reportDate` and audits ("update / قفل فترة"). `unlockPeriodNow()` clears it and audits. **Neither handler performs a `can()` check.**

**CLOSE-04 — settings lock** — *confirmed (high)*
- The settings date input sets `DB.lockDate` directly via inline `onchange` (`save()` only) — **not audited**. However the `settings` nav is gated `settings:view` (owner/admin only); accountant/viewer cannot reach it (`settingsVisible=false`). So the unaudited control is reachable only by settings-editors.

**CLOSE-05 — inconsistent paths** — *DEFECT confirmed (high)*
- Different writers set different `DB.lockDate` values with no coordination: `lockPeriodNow` → `reportDate` (2026-06-03); a settings/month-end style write → an arbitrary date (e.g. 2026-12-31); `doClosing` → `period.to`. Exercising two paths yields divergent lock state. Five uncoordinated writers, no single source of truth.

### Fixed assets and opening balances

**FA-01 — create fixed asset** — *REFUTES STATIC AUDIT (high)*
- Action: create asset (name "Test Machine", cost 120,000, life 60, start 2026-06-01). DB after: `FA-003 {cost:120000, acc:'1300', method:'straight'}` — **no `opening` flag** (as the audit noted).
- **But the acquisition journal IS posted:** `buildJournalCore` (2921–2928) derives, for every `!a.opening` asset, `Dr 1300 120,000 / Cr 1021 120,000`. Runtime journal contains "شراء أصل ثابت — Test Machine" with `1300` debited and the bank credited. The static claim "records the asset but never books the acquisition" is **refuted**.
- Nuance: `editAsset` sets no `treasury`, so the credit defaults to bank 1021 even if the asset was bought on credit — a modelling simplification, not a missing entry.

**FA-02 — depreciation** — *PASS (high)*
- Monthly depreciation posts `Dr 5400 / Cr 1310` per month (e.g. FA-001 → 6,666.67/month; accumulated 1310 credit 39,583.33 across the seed period). Depreciation exists independently of an acquisition entry (for opening assets). Uses admin-expense 5400 rather than a dedicated depreciation account (minor).

**OB-01 — treasury opening balance** — *PASS (high)*
- Opening journal (`ref:'OPENING'`) debits treasury accounts (1010 50,000; 1021 200,000; 1022 252,500 for USD 5,000 @ 50.5) and credits `3001 أرصدة افتتاحية` 2,403,839.5. Ledger and treasury agree; account 3001 present.

**OB-02 — product opening qty/cost** — *PASS (high)*
- Opening inventory 1,371,339.5 booked as `Dr 1200` inside the same OPENING entry. Ledger inventory opening ties to stock valuation.

---

## 5. Static-vs-runtime comparison

| Static-audit claim | Runtime result | Match |
|---|---|---|
| Two recurring engines; reachable one is Engine B (`DB.recurring`) under `recurringExp` label | Confirmed at runtime; label/content mismatch reproduced | **CONFIRMED** |
| Engine A recurring editors (8104/8132) shadowed/dead | Confirmed — later 11510/11530 win; no UI path to `DB.recurringExp` | **CONFIRMED** |
| `autoGenerateRecurring` depends on `DB.autoRecurring` (NOT VERIFIABLE statically) | `DB.autoRecurring` undefined at boot → off | **CONFIRMED** |
| Two FX engines, both reachable, double-count risk | Both reachable; **Engine B produces no exposure at runtime → double-posting not reproducible**; risk is latent, not live | **PARTIALLY CONFIRMED / REFUTED (severity)** |
| ≥5 uncoordinated `DB.lockDate` writers, inconsistent | Reproduced divergent lock state across paths | **CONFIRMED** |
| Supplier-payment create/edit/delete not permission-gated; viewer can post | Viewer created a payment via visible button + ungated handler | **CONFIRMED** |
| Payroll Post ungated; accountant can post | Accountant posted `PAYROLL-2026-06` | **CONFIRMED** |
| RC-002 salary double-counts vs Payroll module | Payroll→5410 and RC-002→5400 coexist, no cross-guard | **CONFIRMED** |
| **New fixed-asset acquisition posts no journal** | **Acquisition journal Dr 1300 / Cr bank IS posted** for `!a.opening` assets | **REFUTED** |
| Opening balances (treasury, product) have working journals to 3001 | Confirmed | **CONFIRMED** |
| `doMonthEndClose` only changes lockDate, no closing journal | Confirmed; also no permission/audit | **CONFIRMED** |
| Unlinked screens (smartSearch, fiscalYears, stockMoves, ccBudget, docs, cashboxes, cashTx) implemented but no nav | All 7 render when invoked directly; no NAV entry | **CONFIRMED** |
| (not in static audit) `pricing` screen render failure | `ReferenceError: esc is not defined` — hard break | **NEW (runtime-only)** |
| (not in static audit) Engine A `postRevaluation` handler ungated | Viewer posted Engine A revaluation via direct handler call | **NEW (runtime-only)** |

---

## 6. Permission runtime matrix

UI visibility and handler enforcement measured separately per role. **BLOCKED@UI** = button hidden; **BLOCKED@handler** = handler rejects even when called directly; **ALLOWED** = handler executes; **NOT AVAILABLE** = nav hidden (section = none).

| Write action | Handler self-check? | owner | admin | accountant | sales | warehouse | viewer |
|---|---|---|---|---|---|---|---|
| Create/edit/delete supplier payment (`editSuppPay`) | **No** | ALLOWED | ALLOWED | ALLOWED | NOT AVAILABLE | NOT AVAILABLE | **ALLOWED (bypass — button visible)** |
| Post payroll (`postPayroll`) | **No** | ALLOWED | ALLOWED | **ALLOWED (bypass — hr:view only)** | NOT AVAILABLE | NOT AVAILABLE | NOT AVAILABLE |
| Create/edit recurring (`editRecurring`) | **No** | ALLOWED | ALLOWED | ALLOWED | NOT AVAILABLE | NOT AVAILABLE | **ALLOWED (bypass — button hidden but handler runs)** |
| Post revaluation Engine A (`postRevaluation`) | **No** | ALLOWED | ALLOWED | ALLOWED | BLOCKED@UI | BLOCKED@UI | **ALLOWED (bypass — button hidden but handler runs)** |
| Post revaluation Engine B (`postFxRevaluation`) | **Yes** `accounts:edit` | ALLOWED | ALLOWED | **BLOCKED@handler** | NOT AVAILABLE | NOT AVAILABLE | **BLOCKED@handler** |
| Close period (`doClosing`) | **Yes** `accounts:edit` | ALLOWED | ALLOWED | **BLOCKED@handler** | NOT AVAILABLE | NOT AVAILABLE | **BLOCKED@handler** |
| Unlock period (`unlockPeriodNow`) | **No** | ALLOWED | ALLOWED | **ALLOWED (ungated)** | ALLOWED | ALLOWED | **ALLOWED (ungated)** |
| Edit asset (`editAsset`) | **Yes** `accounts:edit` | ALLOWED | ALLOWED | **BLOCKED@handler** | NOT AVAILABLE | NOT AVAILABLE | **BLOCKED@handler** |
| Create manual journal (`journal:edit`) | n/a (nav-gated) | EDIT | EDIT | EDIT | NOT AVAILABLE | NOT AVAILABLE | VIEW ONLY |

**Enforcement pattern:** the correctly-gated writers all place a `can()` check at the top of the handler (`postFxRevaluation`, `closeFxRevaluation`, `doClosing`, `editAsset`). The bypassable writers rely only on button visibility (or not even that) and perform no handler check (`editSuppPay`, `postPayroll`, `editRecurring`, `postRevaluation`, `lockPeriodNow`/`unlockPeriodNow`, `doMonthEndClose`).

---

## 7. Unlinked-screen runtime table

Each function was invoked directly (`CURRENT=id; render()`); no navigation was added and no file was patched.

| Screen (function) | Direct invocation works | Renders | Console error | Required data | Classification |
|---|---|---|---|---|---|
| smartSearch (`viewSmartSearch`) | Yes | Yes (809 chars) | None | — | **FUNCTIONAL BUT UNLINKED** |
| fiscalYears (`viewFiscalYears`) | Yes | Yes (1,354) | None | — | **FUNCTIONAL BUT UNLINKED** |
| stockMoves (`viewStockMoves`) | Yes | Yes (23,169) | None | seed stockMoves | **FUNCTIONAL BUT UNLINKED** |
| ccBudget (`viewCcBudget`) | Yes | Yes (2,967) | None | cost centres | **FUNCTIONAL BUT UNLINKED** |
| docs (`viewDocuments`) | Yes | Yes (9,000) | None | shipments | **FUNCTIONAL BUT UNLINKED** |
| cashboxes (`viewCashboxes`) | Yes | Yes (25,677) | None | treasury | **FUNCTIONAL BUT UNLINKED / SUPERSEDED** by `banks` |
| cashTx (`viewCashTx`) | Yes | Yes (11,404) | None | treasuryTx | **FUNCTIONAL BUT UNLINKED / SUPERSEDED** by `banksTx` |

All seven are implemented and render cleanly; only the NAV binding is missing.

---

## 8. Runtime defects

| ID | Area | Reproduction | Runtime evidence | Severity | Recommended destination |
|---|---|---|---|---|---|
| D-01 | Rendering | Open `pricing` | `ReferenceError: esc is not defined`; screen fails to render; `esc` declared as top-level `const` in 7 script blocks → `undefined` at runtime | **High** | **Production Frontend** (also patchable for verification) |
| D-02 | Supplier payments | SP-07 as viewer | Viewer creates/edits/deletes payments; buttons visible + handler ungated | **High** | **Production Backend** (server-side permission) |
| D-03 | Supplier payments | SP-06 locked delete | Error toast + "تم الحذف" success toast; record retained | **High** | **Production Frontend** |
| D-04 | Supplier payments | SP-03 / SP-04 | Duplicate invoice payment and overpayment both accepted; no ceiling/dedup | **High** | **Production Backend** |
| D-05 | Payroll | PAY-05 as accountant | `postPayroll` posts with `hr:edit=false`; button visible; handler ungated | **High** | **Production Backend** |
| D-06 | Payroll/Recurring | PAY-04 | Salary charged to 5410 (payroll) and 5400 (RC-002) in same month; no cross-guard | **High** | **Product Owner Decision** |
| D-07 | Period lock | CLOSE-02/03/04/05 | `doMonthEndClose`, `lockPeriodNow`, `unlockPeriodNow`, settings input all mutate `DB.lockDate`; several ungated/unaudited; divergent values | **High** | **Production Backend** |
| D-08 | FX Engine A | Phase 6 / matrix | `postRevaluation` handler has no `can()` check; viewer posts via direct call | **Medium** | **Production Backend** |
| D-09 | FX Engine B | FX-02 | `fxReval` produces net 0 on all data (booked recomputed at current rate) → cannot post; effectively inert | **Medium** | **Product Owner Decision** |
| D-10 | Recurring | REC-04 | `genRecurring` writes into a locked-period month without `isLocked()` guard or toast | **Medium** | **Production Backend** |
| D-11 | Recurring UI | REC-01 | `recurringExp` menu label ("recurring expenses") vs rendered `DB.recurring` ("recurring journal entries") | **Low** | **Verification Patch** |
| D-12 | Navigation | Phase 4 | 7 implemented screens have no NAV entry | **Low** | **Verification Patch** |
| D-13 | Fixed assets | FA-01 | New asset credits bank 1021 by default even for on-credit purchases (no payable option) | **Low** | **Product Owner Decision** |

---

## 9. Verification-patch candidates

Only changes that enable access/rendering with **business-logic impact = NONE**:

| Candidate | Minimal change | Why needed for verification | Business logic impact |
|---|---|---|---|
| `pricing` render break (D-01) | Ensure a single reachable `esc` definition (dedupe the 7 top-level `const esc`) so the identifier resolves | The screen cannot be rendered/verified at all until `esc` resolves | **NONE** (pure identifier resolution; no calculation change) |
| `recurringExp` label mismatch (D-11) | Rename the menu label / `TITLES['recurringExp']` to match the rendered `DB.recurring` data | Lets a verifier confirm which engine is under test | **NONE** |
| 7 unlinked screens (D-12) | Add NAV bindings (or remove dead fn-map/TITLES keys) for smartSearch, fiscalYears, stockMoves, ccBudget, docs, cashboxes, cashTx | Screens render but cannot be reached through the UI | **NONE** |
| Shadowed Engine A editors | Rename `editRecurring`/`delRecurring` at 8104/8132 (e.g. `editRecurringExp`) | Makes both recurring engines independently testable | **NONE** (rename of unused globals) |

All other findings (permission bypasses, dedup/ceiling, lock centralization, salary double-charge, FX consolidation) change business logic or accounting outcomes and are therefore **not** verification-patch candidates.

---

## 10. Production-only issues (do NOT patch into the prototype)

- **Server-side permissions** — enforce `can()` on the backend for supplier payments (D-02), payroll (D-05), Engine A revaluation (D-08), recurring (D-10), and all lock operations (D-07). Client gates are advisory.
- **Supplier-payment ceiling & dedup** (D-04) — reject amount > outstanding balance; prevent paying the same invoice twice; add approval limits.
- **Reversal instead of delete** (D-03) — `delSuppPay`/`delAsset`/`delRevaluation`/journal deletes should become reversing entries; and never emit a success toast when nothing was deleted.
- **Centralized period lock** (D-07) — one authoritative lock service; retire the ≥5 independent `DB.lockDate` writers; audit every change.
- **Payroll ↔ recurring reconciliation** (D-06) — one salary source of truth so RC-002 and the Payroll module cannot both book salaries.
- **FX engine consolidation** (D-09/FX-03) — collapse `DB.revaluations` and `DB.fxRevaluations` into one engine with one posting/reversal model; fix Engine B's exposure basis (compare booked-at-invoice-rate to current rate) or retire it.
- **Fixed-asset acquisition funding** (D-13) — allow crediting a payable (not only bank) at acquisition.
- **Immutable posting** — append-only journals instead of in-place edits.

---

## 11. Final verdict

**PASS WITH VERIFICATION PATCH REQUIRED**

Every material accounting workflow was executed and observed at runtime: all seven supplier-payment scenarios, both payroll paths (with a created employee), both recurring behaviours, both FX engines (Engine A end-to-end; Engine B up to its exposure guard), all five period-lock paths, fixed-asset acquisition and depreciation, and both opening-balance journals. Supplier-payment permissions were exercised with a viewer role; fixed-asset acquisition was verified; **>99% of statically reachable screens were opened**; console errors were captured; and state was reset between destructive scenarios.

The result is not FAIL — no material accounting workflow was unobservable. It is not a clean PASS because one reachable screen (`pricing`) hard-fails to render and seven fully-implemented screens have no NAV entry; these require the minimal, zero-business-logic patches in §9 to be fully exercised through the UI. All blocking issues are either render/navigation-only (patchable) or business-logic issues correctly routed to production.

---

## 12. No-modification confirmation

- **Original prototype unchanged** — `reference/prototype/prototype_v2.html` was only read; a runtime copy was loaded into the DOM engine. The on-disk file is byte-for-byte identical (SHA-256 `8996dd68…d7c6b2`).
- **Project unchanged** — no project file was modified.
- **Documentation unchanged** — no documentation or ADR was modified.
- **No patch applied** — all "verification-patch candidates" (§9) are described only; none were implemented.
- **Only this runtime report was created.**

---

### Evidence & confidence standard

Every workflow result cites the exact function exercised, the role used, the input, the resulting `DB` record, the derived journal lines (from `buildJournalCore`), and any toast/console output observed at runtime. Reachability and rendering were established by actually invoking `render()` and reading `#content`. "Could not be produced" statements (Engine B posting, FX double-posting) reflect that the engine's own exposure calculation yields zero on all available/perturbed data — the journal *logic* exists in code but is unreachable through the engine at runtime, and is marked NOT VERIFIABLE rather than asserted. Confidence is stated per test; all primary accounting findings are **High**, FX double-count severity and recurring-lock behaviour are **Medium**.
