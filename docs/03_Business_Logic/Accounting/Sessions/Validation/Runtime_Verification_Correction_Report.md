# Runtime Verification — Documentation Correction Pass

**Date:** 2026-07-25
**Evidence source (highest priority for tested scenarios):** `Prototype_Runtime_Verification_Report.md`
**Prototype:** `reference/prototype/prototype_v2.html` — SHA-256 `8996dd68a9f4eea241d4289f094f9f5088baf148b79efc6fa2f8ef5759d7c6b2` (**byte-identical / unchanged**)
**Final verdict:** `PASS WITH RUNTIME CORRECTIONS + EXPLICIT MATRIX LIMITATION`

> No application code or the prototype was modified. Only documentation `.md` files were edited.

---

## 1. Modified files (4)

| File | Change |
|---|---|
| `docs/02_Governance/Known_Issues.md` | KI-028 refuted/closed; KI-029 narrowed→Low; KI-032 reclassified→Medium; KI-034 added; permission-bypass cross-reference block; superseded appendix; index + distribution + version |
| `docs/03_Business_Logic/Accounting/Accounting_Test_Cases.md` | TC-AINV-05→PASS; TC-D07→PASS; TC-D08 replaced by D08A/B/C; TC-D11 replaced by D11A–E; TC-D19 added; totals recalculated 55→62; version |
| `docs/03_Business_Logic/Accounting/Accounting_Session6_1_Remaining_Evidence_Findings.md` | Added §12 "Post-Runtime Verification Corrections" (history preserved, not rewritten) |
| `docs/03_Business_Logic/Accounting/Accounting_Implementation_Guide.md` | JS-48/KI-029 note corrected; §2.10 opening balances corrected; FX §2.6 + duplication row reclassified latent; summary row corrected |

---

## 2. Before/after KI titles and severities

| KI | Before (title · severity) | After (title · severity) |
|---|---|---|
| KI-028 | Treasury opening balance never enters the general ledger · 🔴 Critical | **REFUTED / CLOSED** (superseded, historical) · not counted active |
| KI-029 | Fixed assets are depreciated without any acquisition entry · 🔴 Critical | Fixed-asset acquisition always defaults to bank funding and does not support payable/credit acquisition selection · 🟢 **Low** |
| KI-032 | Two parallel FX-revaluation engines; `DB.revaluations` bypasses control 1265 (violates ADR-003) · 🔴 Critical | Two parallel FX-revaluation engines with no consolidation; Engine A bypasses 1265, Engine B is inert; double recognition is a latent code risk (not runtime-reproduced) · 🟡 **Medium** |
| KI-034 | *(did not exist)* | **Pricing screen fails to render because `esc` is undefined** · 🔴 **High** (Production Frontend; Verification-patch eligible, business-logic impact = NONE) |

**Closed/refuted KIs:** KI-028 (refuted by runtime OB-01; closed, moved to superseded appendix).
**Newly-added KI number:** **KI-034** (next available; no existing KI renumbered).

**Active KI severity distribution (KI-010→034, KI-028 excluded):** Critical 9 · High 4 · Medium 9 · Low 2 = **24 active**. Net active Critical: 11 → **9**.

---

## 3. Recalculated test totals (computed directly, 55 → 62)

| | Before | After |
|---|---|---|
| Current PASS | 5 | **12** |
| Current PARTIAL | 8 | **10** |
| Current FAIL | 42 | **40** |
| **Total** | **55** | **62** |
| Target-After-Remediation: PASS | 50 | **50** (40 FAIL + 10 PARTIAL) |

- **AINV (34):** PASS 3 · PARTIAL 8 · FAIL 23 (TC-AINV-05 FAIL→PASS).
- **Scenarios (28):** PASS 9 (D04A · D07 · D08A · D08C · D09B · D11A · D11B · D12 · D19) · PARTIAL 2 (D11C · D11D) · FAIL 17.
- Net +7 cases: D07 flipped; D08→D08A/B/C; D11→D11A/B/C/D/E; +D19.

---

## 4. Exact list of runtime-refuted statements

1. "Treasury opening balance never enters the general ledger / appears on the treasury screen only." → **Refuted** (OB-01: `OPENING` journal debits treasury and credits 3001; ledger and treasury agree).
2. "Product/inventory opening balance has no ledger entry." → **Refuted** (OB-02: opening inventory Dr 1200 inside `OPENING`).
3. "Fixed assets are depreciated without any acquisition entry / acquisition entry not found." → **Refuted** (FA-01: `buildJournalCore` derives Dr 1300 / Cr bank for every `!opening` asset).
4. "FX difference is booked twice as live runtime behaviour (double posting is reproducible)." → **Refuted/refined** (FX-02/FX-03: Engine B inert — `fxExposure()` returns 0 even after a 9% move; simultaneous double-posting not reproduced; risk is latent code-level only).

---

## 5. Required confirmations

- **ACC-E and OQ-6.A remain unchanged.** ✅ (No edits to their definitions or blocked status.)
- **OQ-6.F remains open.** ✅ (🔴 Open, retained as KI-032 target and in FX test cases.)
- **Source counts:** JS-C1/JS-C2 reclassified from `Conditional` to **Confirmed posting source** by runtime evidence (see §6 below). Catalog composition: **62 confirmed · 0 Conditional · 1 Future · 63 total** (total preserved).
- **No prototype or application code was modified.** ✅ (Prototype SHA-256 byte-identical; `experiments/pricing-engine-poc.html` untouched; only `.md` docs edited.)

**Explicit matrix limitation:** ledger ↔ subledger agreement is confirmed **only for tested seed/runtime data** (treasury + product opening); the broad AINV-33 invariant remains open for other subledgers. Engine B's 1265/3020 posting lines are **NOT VERIFIABLE** end-to-end (present in code, unreachable via the engine's zero-exposure calc).

---

## 6. Source-classification consistency correction (2026-07-25)

### Before/after source classification

| Source | Before | After |
|---|---|---|
| JS-C1 opening balances | `Conditional / Unverified candidate — likely future` | **Confirmed posting source — Opening Journal** (treasury + product opening are **branches/lines of the same `OPENING` economic event**; **not** separate sources — no JS-C1b counted) |
| JS-C2 fixed-asset acquisition | `Conditional / Unverified candidate — likely future` | **Confirmed posting source — Fixed Asset Acquisition** (Dr 1300 / Cr bank for `!opening` assets) |
| JS-C3 payroll recurring | use case of JS-19 | unchanged — not an independent source |

### Exact final catalog counts

| | Before | After |
|---|---|---|
| Confirmed posting sources | 60 | **62** |
| Conditional | 2 | **0** |
| Future | 1 | **1** |
| **Total catalog items** | **63** | **63** |

Composition arithmetic: **62 + 0 + 1 = 63** (total preserved). RECALC/MIXED count stays **22** (JS-C1/JS-C2 are SNAP); ratio over the updated denominator ≈ 22/62 ≈ 35.5%; the session-6.2 36.7% figure is retained as the 60-source analysis baseline (RFC reconciliation note).

### Files changed in this correction

1. `Accounting_Implementation_Guide.md` — §0 catalog rows + reconciliation matrix (60→62, Conditional 2→0); §2.10 JS-C1b folded into JS-C1; posting-matrix TODO.
2. `Accounting.md` — §2.9 JS-C1/JS-C2 → Confirmed; §2.10 count rows; catalog intro annotation.
3. `Accounting_Session6_1_Remaining_Evidence_Findings.md` — §12.3/§12.4 reclassification; supersede banner on historical §6.
4. `Accounting_RFC.md` — catalog cross-ref 60→62; RECALC reconciliation footnote; header 60/63→62/63.
5. `Runtime_Verification_Correction_Report.md` — this section.
6. `Accounting_Test_Cases.md` — **no change** (no JS-C1/JS-C2 conditional wording present).

### Preserved / confirmed

63 total catalog items ✅ · KI-028 closed/refuted ✅ · KI-029 Low ✅ · KI-032 Medium ✅ · KI-034 High ✅ · test totals PASS 12 · PARTIAL 10 · FAIL 40 · Total 62 ✅ · ACC-E and OQ-6.A unchanged ✅ · OQ-6.F open ✅ · no prototype/application changes ✅ · **no source double-counted** (product opening is a line of JS-C1, not a separate JS-C1b) ✅

**Final verdict:** `PASS WITH RUNTIME CORRECTIONS + PARTIAL POSTING-MATRIX COVERAGE`
