# Accounting Prototype — Independent Evidence Verification Report

```
Task type:        Clean-room verification gate (independent)
Scope:            KI-025, KI-026, KI-030, KI-032, KI-033 (+ KI-031 supporting question)
Prototype:        reference/prototype/prototype_v2.html  (24,149 lines, delivered build)
Method:           Phase A prototype-only → Phase B documentation comparison
Files modified:   NONE (see §9)
Verifier stance:  No prior report, KI, ADR, or test case was assumed correct.
```

> **Clean-room integrity note.** During Phase A only the prototype, `AGENTS.md`, and
> `DEVELOPMENT_RULES.md` were inspected. `Known_Issues.md`, the Implementation Guide,
> the Test Cases, and the Session 6.1 Findings were opened **only after** Phase A verdicts
> were fixed. ADR-021/022/023 and ADR-003 were **not provided** with the task; any claim
> that depends on their text is marked `NOT VERIFIABLE` rather than assumed.

---

## 0. Executive summary

| KI | Independent verdict | Confidence | Documented severity still justified? |
|----|--------------------|------------|--------------------------------------|
| **KI-025** — supplier-payment FX not recognised when unlinked | `CONFIRMED` | High | Yes (High) |
| **KI-026** — supplier-payment path: no lock / no dup guard / no ceiling / no audit | `PARTIALLY CONFIRMED` | High | **Narrow to Medium** — lock **and** create/update audit are present; only dedup + ceiling are truly absent |
| **KI-030** — payroll → 5400; 5410/5420/2120/2130/2140 "never used" | `PARTIALLY CONFIRMED` | High | **Narrow** — payroll-via-recurring→5400 is real, but a dedicated `postPayroll()` engine **does** post to all five accounts; "never used" is refuted |
| **KI-031** — two recurring engines; `editRecurring` shadowed | `PARTIALLY CONFIRMED` | High | Structural facts hold; the stated UI consequence is **inverted** |
| **KI-032** — two FX-revaluation engines; `DB.revaluations` bypasses 1265; double recognition | `CONFIRMED` (code facts) / `NOT VERIFIABLE` (ADR-003 conformance) | High / — | Yes (Critical) for the double-recognition mechanism |
| **KI-033** — `doClosing()` uncentralised lock path; 3099⇒3020 uncatalogued | `CONFIRMED` | Medium | Yes (Medium), with two nuances |

**Overall gate result: `PASS WITH CORRECTIONS`** (see §7). No Critical KI's core conclusion is
refuted; every source→journal trace was established reliably; several documentation *absence*
statements require narrowing or correction.

---

# PHASE A — Prototype-only findings

## Target 1 — Supplier Payment FX Behaviour (KI-025)

### A. Entity and writer
- **Entity:** `DB.supplierPayments`
- **UI writer:** `window.editSuppPay(i)` — prototype **line 13280** (create when `i<0`, edit otherwise).
- **Journal writer:** the `(DB.supplierPayments||[]).forEach(sp=>…)` block **inside `buildJournalCore()`** — **lines 3097–3113**. There is no standalone posting function.
- **UI trigger:** "＋ سداد لمورد" button → `editSuppPay(-1)` (line 13255); row pencil → `editSuppPay(i)` (13247).

### B. Posting trace
`editSuppPay → DB.supplierPayments[] → buildJournalCore() supplier-payment loop → add(sp.date, lines)`

### C. Accounting behaviour (lines 3097–3113)
```js
const payCur = sp.cur || s.cur || 'EGP';
const payRate = sp.fxRate || fxRate(payCur);        // payment-day rate
let invRate = payRate;                              // ← DEFAULT = payRate
if (sp.purchaseId){ const pu = find(sp.purchaseId); if(pu) invRate = fxOf(pu, pu.cur||payCur); }
else if (sp.expId){ const ex = find(sp.expId);      if(ex) invRate = ex.fxRate||fxRate(ex.cur||payCur); }
const cashEGP    = (sp.amount||0)*payRate;
const payableEGP = (sp.amount||0)*invRate;
const fxDiff     = payableEGP - cashEGP;
const lines = [{acc:'2010 الموردون',dr:payableEGP,cr:0},{acc:cashAcc(sp.treasury),dr:0,cr:cashEGP}];
if (Math.abs(fxDiff) > 0.01){
  if (fxDiff>0) lines.push({acc:'4030 أرباح فروق عملة',dr:0,cr:fxDiff});
  else          lines.push({acc:'5500 خسائر فروق عملة',dr:-fxDiff,cr:0});
}
```
`fxOf(obj,cur)` (line 1790) = `obj.fxRate>0 ? obj.fxRate : fxRate(cur)` — i.e. it uses the invoice's **stored** rate only if one was frozen, otherwise the **current** rate at build time.

Answers to the six required questions:

| # | Question | Finding |
|---|----------|---------|
| 1 | Linked to a **purchase invoice** | `invRate = fxOf(pu, …)` → invoice's frozen rate if `pu.fxRate>0`, else current rate. Real FX diff → 4030/5500. |
| 2 | Linked to an **expense** | `invRate = ex.fxRate || fxRate(ex.cur)` → expense's frozen rate if present, else current. Same behaviour. |
| 3 | **Neither** link | `invRate` stays initialised to `payRate` ⇒ `payableEGP == cashEGP` ⇒ `fxDiff = 0` ⇒ **no 4030/5500 line posted**. |
| 4 | Can `invRate == payRate`? | **Yes** — (a) no link (default), (b) linked invoice with no frozen rate + same currency/day, (c) coincidental equal rates. |
| 5 | Can a real FX difference become **zero**? | **Yes** — the unlinked case forces `fxDiff=0` regardless of actual rate movement; a linked-but-unsnapshotted invoice collapses the diff to the current-rate result. |
| 6 | Stored or recalculated? | **MIXED.** `payRate` = SNAP (`sp.fxRate` on the voucher) or current; `invRate` = **RECALC live from the invoice at journal-build time** (not frozen on the voucher). |

- **Posting date:** `sp.date`.  **Dr:** 2010.  **Cr:** cash/bank (`cashAcc(sp.treasury)`).  **FX split:** 4030 / 5500 only when `|fxDiff|>0.01`.
- **SNAP/RECALC:** **MIXED** (SNAP payment rate + RECALC invoice rate).
- **Duplicate guard / ceiling:** none in this loop (see Target 2).

### D. Direct code evidence
Lines 3099–3113 above; helper `fxOf` line 1790; `fxRate` lines 1780–1789 (returns `1` for unknown currency, warns).

### E. Independent verdict — **`CONFIRMED`** · confidence **High**
The defect exists exactly as characterised: without a link (or without a frozen invoice rate),
the genuine FX gain/loss on the foreign-currency payable is silently set to zero, and even when
linked the invoice rate is recomputed live rather than frozen on the voucher (an extension of the
KI-022 "live value" pattern). Documented **High** severity remains justified.

---

## Target 2 — Supplier Payment Controls (KI-026)

### A. Entity and writer
Same as Target 1 (`editSuppPay` line 13280; `delSuppPay` line 13346; journal loop 3097).

### B/C. Control audit of the **complete** lifecycle (UI writer + delete + journal loop + helpers)

| Control | Present? | Evidence |
|---------|----------|----------|
| **Period-lock validation** | ✅ **PRESENT** | `editSuppPay` save callback, line **13284**: `if(isLocked($('#sp_date').value)){toast('Period locked');return false;}` |
| **Duplicate-payment prevention** | ❌ absent | No dedup logic anywhere; a whole-file search for duplicate/ceiling guards returns nothing. Two vouchers on the same `purchaseId` are accepted. |
| **Invoice-balance ceiling** | ❌ absent | `suppPayBalance()` (13308) renders the outstanding balance as a **UI hint** and a "pay full" button, but the save callback never enforces `amount ≤ balance`. |
| **Overpayment prevention** | ❌ absent | Consequence of the above; 2010 can be driven debit. |
| **Uniqueness key** | ❌ absent (only sequential id) | `id = nextCode('suppPay')` → `SPV-####`; no business key on (invoice, ref, cheque). |
| **Audit logging** | ⚠️ **partial** | **Create/update audited** — line **13300**: `audit(i<0?'create':'update','سداد مورد',obj.id,…)`. **Delete NOT audited** — `delSuppPay` (13346) splices with no `audit()` call. |
| **Permission validation (action-level)** | ❌ absent at the write action | `editSuppPay`/`delSuppPay` do not call `can('supplierPayments','edit')`; access is gated only by section visibility (the KI-002 "UI-only authorization" pattern). |
| **Delete protection** | ⚠️ present-but-buggy | `delSuppPay` → `canDeleteDated(sp.date)` (3936) blocks deletion in a locked period, **but** if locked it evaluates `null` and still runs `save();render();toast('تم الحذف')` → a misleading success toast with no deletion. |
| **Reversal behaviour** | n/a by design | The journal is **derived** — `buildJournalCore()` rebuilds all lines from entities on every render — so edit/delete auto-reflect without a stored reversal entry. |

### D. Direct code evidence
`isLocked` 3931–3934; `canDeleteDated` 3936–3944; `audit` 1102+; writer 13280–13305; delete 13346.

### E. Independent verdict — **`PARTIALLY CONFIRMED`** · confidence **High**
The **duplicate-guard**, **ceiling/overpayment**, **action-level permission**, and **delete-audit**
gaps are real and confirmed. But two of the KI's four headline claims — **"no `isLocked()` observed"**
and **"no `audit()` observed"** — are **contradicted**: the create/edit path performs both. Documented
**High** severity is **overstated**; recommend **narrowing to Medium** and rewording to the genuinely
missing controls.

---

## Target 3 — Payroll and Recurring Expenses (KI-030 / KI-031)

### A. Entities and writers — there are **three** relevant paths
1. **Engine A — `DB.recurringExp`** → `generateRecurring()` (line **2298**); UI `editRecurring`/`delRecurring` at **8104 / 8132**; due-date helper `recurringDue()` (2274). Auto-run only via `autoGenerateRecurring()` (2631) **if `DB.autoRecurring` is set** (startup hook line 5335; default off, toggle 13056).
2. **Engine B — `DB.recurring`** → `genRecurring(rc,monthStr)` (line **11458**); UI `runRecurring`/`runAllRecurring` (11498/11505), `editRecurring`/`delRecurring` at **11510 / 11530**. **Purely manual** (buttons). Seed **RC-002 "رواتب الموظفين" 80000, cat:'رواتب', allocType:'overhead'** (line 649).
3. **Dedicated payroll engine — `postPayroll()`** (line **12408**), wired to a **"📥 ترحيل / Post" button** (line **12343**) in the payroll view. Posts a balanced **manual journal** with `ref='PAYROLL-{month}'`.

### B. Posting traces
- Engines A & B: `→ DB.expenses[] → buildJournalCore() expense loop (line 3032) → add()`
- Payroll engine: `postPayroll() → DB.manualJournals[] (ref PAYROLL-month) → ledger`

### C. Accounting behaviour

**Expense loop account selection is driven by `allocType`, NOT `cat`** (line **3036**):
```js
const drAcc = e.allocType==='invoice' ? '5100 …'
            : e.allocType==='product' ? '5100 …'
            : e.allocType==='general' ? '5200 مصاريف تشغيلية'
            :                           '5400 مصاريف إدارية وعمومية';
```
RC-002 payroll (`allocType:'overhead'`) therefore lands in **5400 مصاريف إدارية وعمومية**, mixed with rent/utilities — no withholding, no employee/company split. The `cat:'رواتب'` label is stored but **never used to choose an account**.

**But `postPayroll()` (12419–12433) posts a proper payroll journal that uses every "unused" account:**
```js
lines.push({acc:'5410 مصروف الرواتب والأجور', dr:totGross, cr:0});
if(totInsCo>0) lines.push({acc:'5420 مصروف تأمينات اجتماعية (حصة الشركة)', dr:totInsCo, cr:0});
if(totIns>0)   lines.push({acc:'2130 تأمينات اجتماعية مستحقة', dr:0, cr:totIns});
if(totTax>0)   lines.push({acc:'2120 ضريبة كسب عمل مستحقة',   dr:0, cr:totTax});
if(totOtherDed>0) lines.push({acc:'5400 مصاريف إدارية وعمومية', dr:0, cr:totOtherDed});
lines.push({acc:'2140 رواتب مستحقة', dr:0, cr:totNet});
```
It has its **own duplicate guard** (`ref='PAYROLL-'+payMonth`; "Already posted for this month"), its own **audit** call, and a balance check.

Answers to the eight required questions:

| # | Question | Finding |
|---|----------|---------|
| 1 | Auto or manual generation? | **Manual by default.** Only Engine A auto-runs, and only if `DB.autoRecurring` is on. Engine B (which holds payroll) is manual. |
| 2 | Which recurring entity creates payroll expenses? | **`DB.recurring`** (Engine B / `genRecurring`), via seed RC-002. |
| 3 | Does the generated record enter `DB.expenses`? | **Yes** — both engines push to `DB.expenses` with `recurringId`. |
| 4 | Which field selects the expense account? | **`allocType`** (line 3036) — **not** `cat`. |
| 5 | Which account receives payroll? | Via recurring RC-002 → **5400**. Via `postPayroll()` → **5410** (+5420). |
| 6 | Are 5410/5420/2120/2130/2140 used by any posting path? | **YES — all five by `postPayroll()`** (5410/5420 debited, 2130/2120/2140 credited). 2120/2130 are *also* debited on remittance via `DB.taxPayments` (`_TAXACC`, line 3290). They are **not** used by the recurring engines. |
| 7 | Two recurring engines? | **Yes** — `generateRecurring` (`recurringExp`) and `genRecurring` (`recurring`). |
| 8 | Is one `editRecurring` shadowed? | **Yes** — `window.editRecurring` defined at **8104** (recurringExp) and **11510** (recurring); the later (11510) wins. A code comment at 8098 itself admits a "dead code shadowed by a later definition". |

### D. Direct code evidence
Expense loop 3032–3060; account selector 3036; RC-002 seed 649; COA payroll accounts 759–761, 785–786; `postPayroll` 12408–12435; payroll button 12343; shadow defs 8104 & 11510; auto flag 2631/5335.

### E. Independent verdicts
- **KI-030 — `PARTIALLY CONFIRMED`** · confidence **High.** The real defect (payroll routed through the recurring engine lands in **5400** with no separation) is **confirmed**. The sweeping claim **"5410/5420/2120/2130/2140 … never used / no journal side for any"** is **REFUTED** — a dedicated, button-wired `postPayroll()` posts to all of them. The two payroll paths (RC-002 → 5400 and `postPayroll` → 5410) have **no cross-guard**, so running both for the same month **double-counts payroll** — arguably a *more* serious finding than the KI states, but different from what it states. **Critical** severity is defensible for the double-count/mis-post, but the KI text must be rewritten.
- **KI-031 — `PARTIALLY CONFIRMED`** · confidence **High.** Two parallel engines: confirmed. `editRecurring` shadowed: confirmed. **However the stated consequence is inverted.** The only callers of `editRecurring` are in `viewRecurring` (11481/11492), which lists **`DB.recurring`** rows; because the later definition (11510, the `DB.recurring` form) wins, that screen opens the **correct** form. It is the **`recurringExp` editor (8104) that is unreachable dead code** — there is no view that lists `recurringExp` rows with an edit button. So "the edit button on the `DB.recurring` screen opens the `recurringExp` form" is **contradicted**; the shadow makes the *other* editor dead. Structural **Medium** severity stands; wording needs correction.

---

## Target 4 — FX Revaluation Engines (KI-032)

### A. Entities and writers
- **Engine 1 — `DB.revaluations`**: writer `postRevaluation()` (**8549**); journal block **3187–3213**; delete `delRevaluation` (8546).
- **Engine 2 — `DB.fxRevaluations`**: writer `postFxRevaluation()` (**22675**), `closeFxRevaluation()` (**22693**); journal block **3374–3402**; delete `delFxRevaluation` (22707).
- **Shared input:** Engine 1 reads `revaluation()` → `foreignBalances()` = customers+suppliers+banks (2548/2495). Engine 2 reads `fxExposure()`/`fxRevalNet()` (22614/22655), which iterate **the same** `DB.suppliers`, `DB.customers`, and banks.

### B. Posting traces
- E1: `postRevaluation() → DB.revaluations[] → block 3187 → party accounts + 4030/5500 → auto-reverse next month`
- E2: `postFxRevaluation() → DB.fxRevaluations[] → block 3374 → 1265 + 4030/5500`; then `closeFxRevaluation() → 1265 ⇄ 3020`

### C. Accounting behaviour

| # | Question | Finding |
|---|----------|---------|
| 1 | Two distinct entities? | **Yes** — `DB.revaluations` and `DB.fxRevaluations`. |
| 2 | Each produces journal entries? | **Yes** — both have posting blocks in `buildJournalCore()`. |
| 3 | Which accounts? | **E1:** 1100 / 2010 / 1020 (party) + **4030 / 5500** (P&L) — **no 1265**. **E2:** **1265** (control) + 4030 / 5500 at post; **3020** (retained earnings) at close. |
| 4 | Does one bypass 1265? | **Yes — E1 bypasses 1265 entirely**; E2 routes through it. The COA defines 1265 "فروق تقييم عملة معلّقة (وسيط)" (804) as the designed suspense/control account, which only E2 honours. |
| 5 | Both consume the same exposure? | **Yes** — both revalue the same foreign-currency AR/AP/bank balances at the closing/current rate. Overlapping input population **demonstrated** (foreignBalances vs fxExposure iterate the same customers/suppliers/banks). |
| 6 | Double recognition technically possible? | **Yes** — no cross-guard exists; `closeReadiness`/alerts only check `DB.revaluations` (15910/15947) and are blind to `DB.fxRevaluations`. Running both posts the same unrealised gain/loss to 4030/5500 **twice**. |
| 7 | Automatic reversal? | **E1: yes** — reverses on the first of next month (`rv.reverse!==false`, block 3208–3212). **E2: no** auto-reversal — it is manually closed into retained earnings instead. |
| 8 | Which engine is consistent with ADR-003? | **`NOT VERIFIABLE`** — ADR-003 was not supplied. *Circumstantial:* the COA's dedicated 1265 control account matches E2's method, so E2 is the design-consistent engine, but this cannot be confirmed against the ADR text. |

- **FX source:** current/closing system rate on open foreign balances.  **SNAP/RECALC:** RECALC (revalued at closing rate vs booked value).

### D. Direct code evidence
`revaluation` 2548; `foreignBalances` 2495; `fxExposure` 22614; E1 block 3187–3213 (auto-reverse 3208–3212); E2 block 3374–3402; writers 8549 & 22675; COA 1265 line 804.

### E. Independent verdict — **`CONFIRMED`** (code facts) · confidence **High**; **`NOT VERIFIABLE`** for the ADR-003-conformance sub-claim.
Two engines, shared exposure, E1 bypasses 1265, E1 auto-reverses while E2 does not, and double
recognition is technically possible with no guard — all directly verified. The "**violates ADR-003**"
label cannot be checked because the ADR is absent. Documented **Critical** severity remains justified
for the double-recognition mechanism.

---

## Target 5 — Closing and Period Lock (KI-033)

### A. Entities and writers
- **`doClosing()`** — line **11272**; records `DB.closings` and a `DB.manualJournals` entry (ref `'إقفال'`); reversible via `reopenLastClosing()` (11301).
- **`doMonthEndClose()`** — line **8281**.
- **`lockPeriodNow()` / `unlockPeriodNow()`** — lines **14212 / 14213**.
- **Settings inline lock** — line 7844 (`onchange` writes `DB.lockDate`); **fiscal-year** lock — line 18382.
- **`closeReadiness()`** — line 14181 (checklist only).  **`DB.lockDate`** is the single shared lock flag read by `isLocked()` (3931).

### B. Posting trace
`doClosing() → DB.manualJournals.push({date:cp.to, ref:'إقفال', lines:[3099⇄3020]}) ; DB.closings.push(...) ; DB.lockDate = cp.to`

### C. Accounting behaviour (11272–11296)

| # | Question | Finding |
|---|----------|---------|
| 1 | Does `doClosing()` create a journal entry? | **Yes** — a balanced closing entry into `DB.manualJournals`. |
| 2 | Are 3099 and 3020 actually posted? | **Yes** — profit: `Dr 3099 / Cr 3020`; loss: the reverse. |
| 3 | What date? | `cp.to` (period-end from `closingPreview()`); the same date sets the lock. |
| 4 | Does it set `DB.lockDate`? | **Yes** — `DB.lockDate = cp.to`. |
| 5 | Distinct closing/lock path? | **Yes** — and it is **not** the only one: `doMonthEndClose` (lock only, **no** journal), `lockPeriodNow`, the settings-inline `onchange`, and fiscal-year all also write `DB.lockDate`. So it is *an* uncentralised path, but calling it the "third" **undercounts** the number of lock paths. |
| 6 | Catalogued elsewhere? | The closing entry has no visible JS-catalog id in the prototype (documentation proposes JS-59). Cannot be fully verified in-prototype. |
| 7 | Same period-lock control as others? | **Yes** — all paths share `DB.lockDate` / `isLocked()`. |
| 8 | Permission / audit validation? | **Present on `doClosing`** — `can('accounts','edit')` gate, `audit('create','إقفال فترة',…)`, and a repeat-close guard (`cp.alreadyClosed`). By contrast `doMonthEndClose` and the settings-inline path have **no** audit/permission and post **no** closing entry. |

### D. Direct code evidence
`doClosing` 11272–11296 (3099/3020 lines 11287–11288; lock 11294; audit 11295); `doMonthEndClose` 8281–8285; `lockPeriodNow`/`unlockPeriodNow` 14212–14213; `isLocked` 3931.

### E. Independent verdict — **`CONFIRMED`** · confidence **Medium**
`doClosing()` is indeed an uncentralised path that posts an (un-catalogued) `3099⇒3020` entry and
sets the lock. Two nuances for correction: (i) it is **guarded** (permission + audit + repeat-close),
so it is not "unguarded"; (ii) it is not the *third* — there are several lock paths, of which the
**lock-only `doMonthEndClose`** (locks without ever posting the closing entry) is the more concerning
sibling. Documented **Medium** severity remains justified.

---

# PHASE B — Documentation comparison

Sources compared: `Known_Issues.md`, `Accounting_Implementation_Guide.md`, `Accounting_Test_Cases.md`,
`Accounting_Session6_1_Remaining_Evidence_Findings_FINAL_v2_0_0.md`.
**ADR-021 / ADR-022 / ADR-023 / ADR-003 were not provided** → any ADR-conformance claim is `Not Verifiable`.

### KI-025

| Claim | Prototype finding | Documentation statement | Match status |
|-------|-------------------|-------------------------|--------------|
| Unlinked payment ⇒ `invRate=payRate` ⇒ `fxDiff=0` | Confirmed (line 3101 default; 3106) | KI-025 / Findings §3.4 G-1 | **Exact Match** |
| Linked ⇒ `invRate` derived **live** from invoice at build time | Confirmed (`fxOf(pu…)` 3102, 1790) | Findings §3.4 G-2 ("امتداد KI-022") | **Exact Match** |
| 4030/5500 understated; payable not cleared | Confirmed | KI-025 impact | **Exact Match** |
| Classification MIXED | Confirmed (SNAP pay + RECALC inv) | Impl Guide §2.3 JS-16 = MIXED | **Exact Match** |

### KI-026

| Claim | Prototype finding | Documentation statement | Match status |
|-------|-------------------|-------------------------|--------------|
| No `isLocked()` on the payment form | **`isLocked` IS present** (line 13284) | KI-026 & Findings §3.3 line 157: "لا `isLocked()` مُشاهَد" | **Contradicted** |
| No `audit()` on the payment path | **`audit` create/update IS present** (13300); only *delete* is unaudited | KI-026 & Findings line 159: "لم يُرَ `audit()`" | **Contradicted** (delete-only is accurate) |
| No duplicate guard | Confirmed absent | KI-026 / §3.4 G-3 | **Exact Match** |
| No invoice-balance ceiling / overpayment guard | Confirmed absent | KI-026 / G-3 | **Exact Match** |
| *Impl Guide side-note* | Lock ✅ on form / 🔴 on recurring gen | Impl Guide §2.5 line 148 | **(more accurate than the KI)** |

### KI-030

| Claim | Prototype finding | Documentation statement | Match status |
|-------|-------------------|-------------------------|--------------|
| Recurring payroll (allocType overhead) → 5400 | Confirmed (selector line 3036; RC-002 line 649) | KI-030 / TC-D09 / Impl Guide §2.5 | **Exact Match** |
| Account chosen by `allocType`, not `cat` | Confirmed | Impl Guide §2.5 matrix | **Exact Match** |
| 5410/5420/2120/2130/2140 **never used / no journal side** | **Refuted** — `postPayroll()` posts all five (12419–12433), button-wired (12343) | KI-030; Findings line 481; TC-D09 line 339 | **Contradicted** |
| Recurring-gen path lacks `isLocked()` | Confirmed (genRecurring/generateRecurring have no lock) | Findings §6.3 line 427/456 | **Exact Match** |

### KI-031

| Claim | Prototype finding | Documentation statement | Match status |
|-------|-------------------|-------------------------|--------------|
| Two parallel recurring engines | Confirmed | KI-031 | **Exact Match** |
| `editRecurring` defined twice, later shadows earlier | Confirmed (8104 vs 11510) | KI-031 (+ code comment 8098) | **Exact Match** |
| Consequence: **recurring screen opens the recurringExp form** | Inverted — recurring screen opens the correct form; the **recurringExp editor is dead code** | KI-031 observed behaviour | **Contradicted (direction)** |

### KI-032

| Claim | Prototype finding | Documentation statement | Match status |
|-------|-------------------|-------------------------|--------------|
| Two parallel FX-revaluation engines, same balances | Confirmed | KI-032 / §7.2 / TC-D11 | **Exact Match** |
| `DB.revaluations` posts party + 4030/5500 with auto-reversal | Confirmed (3187–3213) | KI-032 / TC-D12 | **Exact Match** |
| `DB.fxRevaluations` routes through 1265 | Confirmed (3374–3402) | KI-032 / Impl Guide §2.6 | **Exact Match** |
| Running both ⇒ diff recognised twice | Confirmed (no cross-guard) | KI-032 impact | **Exact Match** |
| `DB.revaluations` **violates ADR-003** | Cannot check — ADR-003 not provided | KI-032 title / OQ-6.F | **Not Verifiable** |

### KI-033

| Claim | Prototype finding | Documentation statement | Match status |
|-------|-------------------|-------------------------|--------------|
| `doClosing()` posts 3099⇒3020 and sets `DB.lockDate` | Confirmed (11287–11294) | KI-033 / TC-D14 | **Exact Match** |
| Uncentralised lock path | Confirmed (shares DB.lockDate; several sibling paths) | KI-033 | **Supported but Incomplete** (it is one of *several*, not the "third") |
| Closing entry uncatalogued (JS-59 proposed) | Entry exists; no in-prototype catalog id | KI-033 / Impl Guide JS-59 | **Supported but Incomplete** |
| (Implicit) unguarded path | `doClosing` **is** permission-gated + audited + repeat-guarded | — | **Supported but Overstated** if read as "unguarded" |

---

# §5 — Correction list (documentation statements needing narrowing/correction)

1. **KI-026 (and Findings §3.3, TC line 296–299): remove the "no `isLocked()`" claim.** `editSuppPay` enforces `isLocked($('#sp_date').value)` (line 13284). Reword to: lock present on create/edit; **missing** guards are duplicate-payment, invoice-balance ceiling/overpayment, and action-level permission.
2. **KI-026 (and Findings line 159): narrow the audit claim.** Create/update **are** audited (line 13300); only **`delSuppPay` is unaudited** (and has a "success toast on blocked delete" bug). Recommend severity **High → Medium**.
3. **KI-030 (and Findings line 481, TC-D09 line 339): retract "5410/5420/2120/2130/2140 never used".** `postPayroll()` (12408, button 12343) posts to all five. Restate the defect as: *payroll generated through the recurring engine mis-posts to 5400, and the recurring path and `postPayroll()` have no cross-guard → double-count risk.*
4. **KI-031: correct the direction.** The `DB.recurring` screen opens the correct form; the shadow renders the **`recurringExp` editor (8104) unreachable dead code**. There is no screen where the wrong form opens.
5. **KI-032: mark the "violates ADR-003" sub-claim `Not Verifiable`** until ADR-003 is supplied. All code-level facts stand.
6. **KI-033: (a) it is one of several lock paths, not "the third"; (b) note `doClosing` is permission-gated, audited, and repeat-guarded; (c) flag the lock-only `doMonthEndClose` (no closing entry, no audit) as the sibling of concern.**
7. **General provenance note.** The contradicted *absence* claims (supplier-payment lock/audit; payroll accounts) all trace to `Accounting_Session6_1_Remaining_Evidence_Findings` written under a strict "`Not found after scoped search`" convention. The delivered `prototype_v2.html` contains a more complete implementation than those findings describe, suggesting the prototype was patched **after** the findings were authored. The findings' honesty convention is sound; the derived KIs simply need re-syncing to the current build.

---

# §6 — Confidence matrix

| KI | Verdict | Confidence | Key limitation |
|----|---------|------------|----------------|
| KI-025 | CONFIRMED | High | none |
| KI-026 | PARTIALLY CONFIRMED | High | none (contradiction is clear-cut) |
| KI-030 | PARTIALLY CONFIRMED | High | none (contradiction is clear-cut) |
| KI-031 | PARTIALLY CONFIRMED | High | none |
| KI-032 | CONFIRMED / (ADR) NOT VERIFIABLE | High / — | ADR-003 text not provided |
| KI-033 | CONFIRMED | Medium | JS-catalog membership not checkable in-prototype |

---

# §7 — Acceptance gate

- Fully **CONFIRMED**: KI-025, KI-032 (code facts), KI-033 → **3**.
- **PARTIALLY CONFIRMED**: KI-026, KI-030, KI-031 → **3**.
- **REFUTED (whole KI)**: **none**. (Refuted *sub-claims* exist inside KI-026 and KI-030; the **core** conclusion of each KI stands.)
- Every source→journal trace was established reliably.
- No Critical KI's **core** conclusion is refuted (KI-030 core = payroll mis-posted/uncontrolled → confirmed; KI-032 → confirmed).
- Documentation statements needing narrowing/correction: several (see §5).

**Result: `PASS WITH CORRECTIONS`.**

*Why not PASS:* fewer than 4 KIs are fully CONFIRMED, and material *absence* statements (supplier-payment
lock/audit; payroll-account usage) are contradicted — these require correction before the docs are ported.

*Why not FAIL:* no Critical KI's core conclusion is refuted; the source→journal traces are all reliable;
the contradictions narrow secondary claims rather than overturning any substantive accounting defect
(the FX, account-routing, double-recognition, and lock-path defects all stand).

> Per `AGENTS.md` rule 6 / `DEVELOPMENT_RULES.md` §4.3, these code-vs-documentation conflicts are
> **reported for human governance review, not resolved unilaterally**. No documentation, KI, or ADR was
> edited by this verification.

---

# §8 — Direct code evidence appendix (line index)

| Item | Prototype lines |
|------|-----------------|
| Supplier-payment journal loop (KI-025/026) | 3097–3113 |
| `editSuppPay` writer — `isLocked` 13284, `audit` 13300 | 13280–13305 |
| `delSuppPay` (unaudited; canDeleteDated) | 13346 |
| `suppPayBalance` (UI-only ceiling hint) | 13308–13344 |
| `fxOf` / `fxRate` / `cashAcc` / `isLocked` / `canDeleteDated` | 1790 / 1780 / 1625 / 3931 / 3936 |
| Expense loop + `allocType` account selector (KI-030) | 3032–3060 (selector 3036) |
| `generateRecurring` (Engine A) | 2298–2310 |
| `genRecurring` (Engine B) + RC-002 seed | 11458–11470 / 649 |
| `editRecurring` shadow pair | 8104 / 11510 |
| `postPayroll` + button (refutes "never used") | 12408–12435 / 12343 |
| Payroll COA accounts | 759–761, 785–786 |
| `postRevaluation` / Engine-1 block / auto-reverse | 8549 / 3187–3213 / 3208–3212 |
| `postFxRevaluation` / `closeFxRevaluation` / Engine-2 block | 22675 / 22693 / 3374–3402 |
| Shared exposure sources | 2548 / 2495 / 22614 |
| COA 1265 control | 804 |
| `doClosing` (3099/3020, lock, audit, guard) | 11272–11296 |
| `doMonthEndClose` (lock only) | 8281–8285 |
| `lockPeriodNow` / `unlockPeriodNow` | 14212 / 14213 |

---

# §9 — No-modification confirmation

No repository, documentation, application, prototype, or ADR file was created, edited, or deleted
during this verification. The prototype and all governance documents were opened **read-only**. The
only artifact produced is this report (`Accounting_Independent_Verification_Report.md`) in the outputs
folder. No ADR was created or modified; no documentation was updated.
