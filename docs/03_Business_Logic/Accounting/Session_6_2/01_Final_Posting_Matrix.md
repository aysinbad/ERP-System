# Accounting — Final Posting Matrix (Session 6.2, Part 1)

## Document Information
```
Document Name:  Accounting Final Posting Matrix
Version:        1.0.0
Session:        6.2 — Final Implementation Documentation
Status:         Draft — for review (implementation input)
Classification: Source of Truth (derived — no new evidence)
Owner:          Solution Architecture Team
Date:           2026-07-25
Source of truth: reference/prototype/prototype_v2.html (unchanged) · Accounting.md · Accounting_Implementation_Guide.md §2 · Prototype_Runtime_Verification_Report.md
```

> **قاعدة عدم الاختراع (AGENTS.md قاعدة 1 · ADR-000):** كل خانة أدناه مأخوذة من مصدر مُتحقَّق منه. حيث لا دليل تشغيلي: **`Not runtime-verified`**. حيث لم تُراجَع أعمدة المصدر فردياً هذه الجلسة: **`Column-review pending`**. لا قيمة مُختلَقة.
>
> **حدّ التغطية المُعلَن (موروث من `Accounting_Implementation_Guide.md` §2):** الكتالوج المعتمَد = **62 confirmed · 0 conditional · 1 future · 63 total**. المصفوفة المُطبَّعة الكاملة بالأعمدة الـ15 مكتملة لـ **35 مصدراً** (33 مراجَعة سابقاً + JS-C1 + JS-C2 المُرقَّيان تشغيلياً). الـ **27 مصدراً المؤكَّدة المتبقّية** مُدرَجة في §3 بحالة `Column-review pending` — **لا تُخترَع خاناتها**.

---

## 1. Legend (تعريف الأعمدة)

- **Snapshot/Recalc:** `SNAP` = القيمة مُجمَّدة على المستند · `RECALC` = تُحسَب لحظة بناء الدفتر · `MIXED` = الاثنان في نفس القيد.
- **Lock Check:** هل مسار الكتابة يفحص `isLocked()`؟ (`✅` = نعم · `🔴` = لا · `جزئي` = على بعض المسارات).
- **Permission Check:** الفحص على مستوى **الـHandler** وقت التشغيل (`can()`): `Handler ✅` · `Handler 🔴 (bypass)` · `Nav-gated` · `Not runtime-verified`.
- **Audit:** هل يُسجَّل حدث في `auditLog`/`securityLog`؟
- **Reversal Strategy:** سلوك التصحيح الحالي في الـProttype (مُشتقّ · محو · عكس تلقائي…) — الهدف الإنتاجي في العمود الأخير عبر ACC-C.
- **Runtime Status:** `Runtime-verified (executed)` = نُفِّذ وشوهد في `Prototype_Runtime_Verification_Report.md` §4 · `Reachable — posted lines NOT VERIFIABLE` · `Not runtime-verified (code-confirmed; screen rendered)`.
- **Production Target:** وجهة المعالجة في الـBackend الجديد + ADR الحاكم.
- كل الحسابات بأكوادها المعيارية (`Glossary.md` §10).

---

## 2. Normalized matrix — column-verified sources (35)

### 2.1 Sales & Revenue

| JS ID | Business Event | Module | Trigger | Entity | Posting Writer | Debit | Credit | Snap/Recalc | Lock | Permission | Audit | Reversal Strategy | Runtime Status | Production Target |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| JS-01 | شحن بضاعة لعميل (إيراد+تكلفة) | Sales/Export | نقل المخاطر (`inv.date`) | `DB.invoices` | `saveInvoice` → `buildJournalCore` | 1100 · 5010 | 4010 · 1200 · 2100 | MIXED | ✅ | Not runtime-verified | ✅ | مُشتقّ؛ حذف يمحو (KI-009/024) | Not runtime-verified (screen rendered) | Posting Service · ACC-B (snap) · ACC-C (reversal) |
| JS-05 | مخصص مطالبات (EAS 48) | Sales/Export | تعرُّض مطالبات | `DB.claims` | `claimsProvision` | 5300 | 2105 | RECALC | جزئي | Not runtime-verified | 🔴 | مُشتقّ | Not runtime-verified | Posting Service · ACC-B · ACC-A (date) |
| JS-12 | مرتجع مبيعات | Sales/Export | إرجاع عميل (`rt.date`) | `DB.returns` | `saveReturn` | 4090 · 1200 | 1100 · 5010 | مشروط (RECALC على `rt.cogsEGP`) | ✅ | Not runtime-verified | ✅ | مُشتقّ؛ عكس التكلفة مشروط (KI-020) | Not runtime-verified | Posting Service · ACC-C (عكس إلزامي) |
| JS-51 | استحقاق عمولة مبيعات | Sales/Export | استحقاق على المُحصَّل (`ca.date`) | `DB.commissionAccruals` | `accrueCommission` | 5440 | 2017 | SNAP | ✅ | Not runtime-verified | مُشتقّ | 2017 بلا مسار استنفاد (KI-017) | Not runtime-verified | Posting Service · **ACC-E (محجوب OQ-6.A)** |

### 2.2 Procurement & Inventory

| JS ID | Business Event | Module | Trigger | Entity | Posting Writer | Debit | Credit | Snap/Recalc | Lock | Permission | Audit | Reversal Strategy | Runtime Status | Production Target |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| JS-07 | استلام بضاعة (GRNI) | Procurement | `rc.date` | `DB.receipts` | `saveReceipt` | 1200 | 2016 | SNAP | ✅ | Not runtime-verified | ✅ | مُشتقّ | Not runtime-verified | Posting Service · ACC-A |
| JS-08 | فاتورة شراء (تُقفل GRNI) | Procurement | `pu.date` | `DB.purchases` | `savePurchase` | 1200(فرق) · 1130 | 2010 · 2016 · 2110 | RECALC (`unitCost`) | ✅ | Not runtime-verified | ✅ | 🔴 محو | Not runtime-verified | Posting Service · ACC-B (تجميد unitCost) · ACC-C |
| JS-09 | فروق الجرد | Inventory | `st.date` | `DB.stocktakes` | `postStocktake` | 5020 ⇄ 1200 | عكسي | RECALC | ✅ | Not runtime-verified | ✅ | مُشتقّ | Not runtime-verified | Posting Service · ACC-B |
| JS-10 | هبوط NRV (EAS 2) | Inventory | `DB.reportDate` 🔴 | `nrvWritedown()` | مُشتقّ | 5015 | 1205 | RECALC | جزئي | Not runtime-verified | 🔴 | مُشتقّ | Not runtime-verified | Posting Service · ACC-B · ACC-A (date) |
| JS-11/13 | أمر تشغيل — انحراف/هدر | Production | `wo.date` | `DB.workOrders` | `postWorkOrder` | 1200 · 1260 | عكسي · 2010 | RECALC | ✅ | Not runtime-verified | ✅ | مُشتقّ | Not runtime-verified | Posting Service · ACC-B · ACC-D (1260 dispatch) |

### 2.3 Supplier Payments (JS-16 — three branches)

| JS ID | Business Event | Module | Trigger | Entity | Posting Writer | Debit | Credit | Snap/Recalc | Lock | Permission | Audit | Reversal Strategy | Runtime Status | Production Target |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| JS-16 (linked-invoice) | سداد مورد مربوط بفاتورة | Treasury/AP | حفظ سداد (`purchaseId`) | `DB.supplierPayments` | بلوك `forEach` داخل `buildJournalCore` | 2010 بـ`payableEGP` (+5500) | `cashAcc(treasury)` (+4030) | MIXED | ✅ create/update · ✅ delete gate | **Handler 🔴 (bypass — viewer created SPV, SP-07)** | ✅ create/update · 🔴 delete | مُشتقّ؛ delete يعطي «نجاح كاذب» في فترة مقفلة (SP-06) | **Runtime-verified (executed)** SP-01/05/06/07 | Posting Service · **ACC-B** (تجميد السعرين) · **ACC-C** (عكس) · **ACC-D** (صلاحية+قفل مركزي) · KI-002 backend perms |
| JS-16 (linked-expense) | سداد مورد مربوط بمصروف | Treasury/AP | حفظ سداد (`expId`) | `DB.supplierPayments` | نفسه | 2010 | نفسه · `ex.fxRate` | MIXED | ✅ create/update | Handler 🔴 (bypass) | ✅ create/update · 🔴 delete | مُشتقّ | Not runtime-verified (فرع لم يُنفَّذ منفصلاً) | نفس أعلاه |
| JS-16 (unlinked) | سداد مورد غير مربوط | Treasury/AP | كلا الحقلين فارغ | `DB.supplierPayments` | نفسه | 2010 بـ`payableEGP` | `cashAcc` | 🔴 **يفقد الفرق** (`invRate=payRate ⇒ fxDiff=0`) | ✅ create/update | Handler 🔴 (bypass) | ✅ create/update · 🔴 delete | مُشتقّ | **Runtime-verified (executed)** SP-02 (لا 4030/5500) | Posting Service · **ACC-B** (KI-025) |

> **JS-16 defects confirmed:** لا حارس ازدواج (SP-03) · لا سقف/رفض سداد زائد (SP-04) · حذف في فترة مقفلة يُظهر «تم الحذف» بلا حذف فعلي (SP-06) · viewer ينشئ سنداً (SP-07). **AINV:** AINV-11 · AINV-04 · AINV-30. **KI:** KI-025 · KI-026 · KI-002.

### 2.4 Supplier Credit Notes (`cnPostingAccount`)

| JS ID | Business Event | Module | Trigger | Entity | Posting Writer | Debit | Credit | Snap/Recalc | Lock | Permission | Audit | Reversal Strategy | Runtime Status | Production Target |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| JS (credit-note) | إشعار خصم مورد | Procurement/AP | حفظ إشعار | `DB.creditNotes` | `editCreditNote` → بلوك `forEach` | 2010 بـ`tot` | `cnPostingAccount(cn)`: 1200 (مربوط price/shortage) · 4020 (غير مربوط) · **1260** (quality ⚠️) · 1130 دائن | SNAP (`fxRate` مُجمَّد) | ✅ إنشاء · 🔴 حذف/تعديل | Not runtime-verified | ✅ | 🔴 حذف نهائي بلا عكس؛ تعديل يعيد كتابة التاريخ (KI-027) | Not runtime-verified | Posting Service · **ACC-C** (عكس مؤرَّخ) · **ACC-D** (قفل الحذف) |

> ⚠️ **تعارض اسم 1260** بين `CN_REASONS('quality')` والأستاذ (S6.1 §4.4) — `Inferred — requires review`، لا يُحسَم هنا.

### 2.5 Payroll & Recurring (two active paths — KI-030/031)

| JS ID | Business Event | Module | Trigger | Entity | Posting Writer | Debit | Credit | Snap/Recalc | Lock | Permission | Audit | Reversal Strategy | Runtime Status | Production Target |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| JS-19 (recurring salary — Path A) | راتب عبر مصروف متكرر | Expenses/HR | `genRecurring()` شهري | `DB.recurring` → `DB.expenses` | `genRecurring` → JS-19 | **5400** (allocType overhead) | 2015 | SNAP (per row) | 🔴 توليد بلا `isLocked` (REC-04) | **Handler 🔴 (bypass — editRecurring)** | جزئي | مُشتقّ | **Runtime-verified (executed)** PAY-03 (5400/2015) | Posting Service · **ACC-B** (تصنيف) · **ACC-D** (قفل التوليد) · KI-030 cross-guard |
| Payroll (Path B — dedicated) | ترحيل رواتب متخصص | HR/Payroll | `postPayroll()` | `DB.manualJournals` (`PAYROLL-{month}`) | `postPayroll` | 5410 · 5420 | 2130 · 2120 · 2140 | SNAP | ✅ (guard `ref` تكرار) | **Handler 🔴 (bypass — accountant posted, PAY-05)** | ✅ | مُشتقّ (manual journal) | **Runtime-verified (executed)** PAY-02/04/05 | Posting Service · **ACC-A** (uniqueness key) · KI-030 (single source) · KI-002 backend perms |

> **Cross-path duplicate (PAY-04):** Path A (5400) + Path B (5410) للشهر نفسه بلا Cross-Guard ⇒ ازدواج راتب. **Target:** مصدر راتب واحد + مفتاح تفرّد (ACC-A · OQ-6.D).

### 2.6 FX Revaluation — two engines (KI-032)

| JS ID | Business Event | Module | Trigger | Entity | Posting Writer | Debit | Credit | Snap/Recalc | Lock | Permission | Audit | Reversal Strategy | Runtime Status | Production Target |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| JS-38 (Engine A) | إعادة تقييم أرصدة | Accounting/FX | `postRevaluation()` (`DB.reportDate`) | `DB.revaluations` | `postRevaluation` | 1100/2010/1020 (حسب النوع) | net>0 ⇒ 4030 · net<0 ⇒ 5500 | SNAP ✅ | 🔴 | **Handler 🔴 (bypass — viewer posted, FX-01/§5)** | `securityLog` فقط | **عكس تلقائي (JS-38b)** بتاريخ `firstOfNextMonth` | **Runtime-verified (executed)** FX-01 (`REV-001 net=27,621`) | Posting Service · **يخالف 1265** ⇒ **OQ-6.F (توحيد)** · ACC-D (perm/lock) |
| JS-38b (Engine A reversal) | العكس التلقائي | Accounting/FX | أول الشهر التالي | `DB.revaluations` | مُدمَج | حسابات معكوسة | معكوسة | SNAP | 🔴 | تابع JS-38 | تابع | عكس تلقائي مُدمَج | **Runtime-verified (executed)** FX-01 | نفس أعلاه |
| JS-34 (Engine B) | تقييم موجب مُرحَّل | Accounting/FX | `postFxRevaluation` (`net>0`) | `DB.fxRevaluations` | `postFxRevaluation` | 1265 | 4030 | SNAP | ✅ | **Handler ✅ (`accounts:edit`)** | ✅ كامل | عبر close (JS-36/37) | **Reachable — posted lines NOT VERIFIABLE** (FX-02: `fxExposure()`=0 ⇒ لا ترحيل؛ المحرك خامل) | Posting Service · **OQ-6.F** (إصلاح أساس التعرُّض أو إلغاء) |
| JS-35 (Engine B) | تقييم سالب مُرحَّل | Accounting/FX | `postFxRevaluation` (`net<0`) | `DB.fxRevaluations` | `postFxRevaluation` | 5500 | 1265 | SNAP | ✅ | Handler ✅ | ✅ | عبر close | **Reachable — NOT VERIFIABLE** (FX-02) | نفس أعلاه |
| JS-36 (Engine B) | إقفال تقييم موجب | Accounting/FX | `closeFxRevaluation` | `DB.fxRevaluations` | `closeFxRevaluation` | 3020 | 1265 | SNAP | ✅ | Handler ✅ | ✅ | — | **Reachable — NOT VERIFIABLE** (FX-02) | نفس أعلاه |
| JS-37 (Engine B) | إقفال تقييم سالب | Accounting/FX | `closeFxRevaluation` | `DB.fxRevaluations` | `closeFxRevaluation` | 1265 | 3020 | SNAP | ✅ | Handler ✅ | ✅ | — | **Reachable — NOT VERIFIABLE** (FX-02) | نفس أعلاه |

### 2.7 Treasury movements (`DB.treasuryTx` — five kinds)

| JS ID | Business Event | Module | Trigger | Entity | Posting Writer | Debit | Credit | Snap/Recalc | Lock | Permission | Audit | Reversal Strategy | Runtime Status | Production Target |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| JS-39 | تحويل بين خزينتين | Treasury | `kind='transfer'` (`tx.date`) | `DB.treasuryTx` | بلوك `forEach` | خزينة الوجهة بـ`toEgp` (+5500) | خزينة المصدر بـ`srcEgp` (+4030) | **RECALC** (`fxRate` حيّ لكل طرف) | حسب النموذج | Not runtime-verified | حسب النموذج | مُشتقّ | Not runtime-verified (screen rendered) | Posting Service · **ACC-B** (تجميد سعري الطرفين) |
| JS-55 | إيداع رأس مال | Treasury | `kind='deposit'` | `DB.treasuryTx` | نفسه | حساب الخزينة | 3010 | SNAP | حسب النموذج | Not runtime-verified | حسب النموذج | مُشتقّ | Not runtime-verified | Posting Service · ACC-C |
| JS-56 | مسحوبات | Treasury | `kind='drawing'` | `DB.treasuryTx` | نفسه | 3030 | حساب الخزينة | SNAP | حسب النموذج | Not runtime-verified | حسب النموذج | مُشتقّ | Not runtime-verified | Posting Service · ACC-C |
| JS-57 | رسوم بنكية | Treasury | `kind='bankfee'` | `DB.treasuryTx` | نفسه | 5400 | حساب الخزينة | SNAP | حسب النموذج | Not runtime-verified | حسب النموذج | مُشتقّ | Not runtime-verified | Posting Service · ACC-C |
| JS-58 | فوائد دائنة | Treasury | `kind='interest'` | `DB.treasuryTx` | نفسه | حساب الخزينة | 4020 | SNAP | حسب النموذج | Not runtime-verified | حسب النموذج | مُشتقّ | Not runtime-verified | Posting Service · ACC-C |

### 2.8 Custody (JS-41 — branches)

| JS ID | Business Event | Module | Trigger | Entity | Posting Writer | Debit | Credit | Snap/Recalc | Lock | Permission | Audit | Reversal Strategy | Runtime Status | Production Target |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| JS-41 (issue) | صرف عهدة | Treasury/Custody | `kind='issue'` | `DB.custody` | بلوك `forEach` | 1120 | خزينة | SNAP | ✅ | Not runtime-verified | Not runtime-verified | مُشتقّ | Not runtime-verified | Posting Service · ACC-D |
| JS-41 (expense) | مصروف من عهدة | Treasury/Custody | `kind='expense'` | `DB.custody` | نفسه | 5100/5200/5400/**1200** (حسب `cat`/`prodCode`) | 1120 | SNAP | ✅ | Not runtime-verified | Not runtime-verified | مُشتقّ | Not runtime-verified | Posting Service · ACC-B · **VAT gap** |
| JS-41 (settle) | تسوية عهدة | Treasury/Custody | `kind='settle'` | `DB.custody` | نفسه | خزينة/1010 | 1120 | SNAP | ✅ | Not runtime-verified | Not runtime-verified | مُشتقّ | Not runtime-verified | Posting Service · ACC-D |

> 🔴 **لا معالجة VAT** على مصروفات العهدة · حارس الرصيد تحذير لا منع (`Not runtime-verified` — لم يُنفَّذ سيناريو عهدة تشغيلياً).

### 2.9 Period-end provisions, depreciation & closing

| JS ID | Business Event | Module | Trigger | Entity | Posting Writer | Debit | Credit | Snap/Recalc | Lock | Permission | Audit | Reversal Strategy | Runtime Status | Production Target |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| JS-43 | مخصص مطالبات سنوي | Accounting | إقفال (`DB.reportDate` 🔴) | derived | مُشتقّ | 5300 | 2105 | RECALC | حسب المسار | Not runtime-verified | Not runtime-verified | مُشتقّ | Not runtime-verified | Posting Service · ACC-B · ACC-A (date) |
| JS-44 | مخصص ديون مشكوك فيها | Accounting | إقفال (`DB.reportDate` 🔴) | derived | مُشتقّ | 5400 | 1110 | RECALC | حسب المسار | Not runtime-verified | Not runtime-verified | مُشتقّ | Not runtime-verified | Posting Service · ACC-B · ACC-A |
| JS-45 | مخصص نهاية الخدمة | Accounting/HR | إقفال (`DB.reportDate` 🔴) | derived | مُشتقّ | 5430 | 2160 | RECALC | حسب المسار | Not runtime-verified | Not runtime-verified | مُشتقّ | Not runtime-verified | Posting Service · ACC-B · ACC-A |
| JS-46 | مخصص ضريبة الدخل | Accounting | إقفال (`DB.reportDate` 🔴) | derived | مُشتقّ | 5900 | 2150 | RECALC | حسب المسار | Not runtime-verified | Not runtime-verified | مُشتقّ | Not runtime-verified | Posting Service · ACC-B · ACC-A |
| JS-47 | أصل ضريبي مؤجل (EAS 24) | Accounting | إقفال (`DB.reportDate` 🔴) | derived | مُشتقّ | 1340 | 5910 | RECALC | حسب المسار | Not runtime-verified | Not runtime-verified | مُشتقّ | Not runtime-verified | Posting Service · ACC-B · ACC-A |
| JS-48 | قسط إهلاك شهري | Fixed Assets | شهري | `DB.assets` | `assetDepreciation()` → `buildJournalCore` | 5400 | 1310 | RECALC | — | **Handler ✅ (`editAsset` — accounts:edit)** | Not runtime-verified | مُشتقّ | **Runtime-verified (executed)** FA-02 (5400/1310) | Posting Service · حساب إهلاك مخصص (تحسين) |
| JS-59 | قيد إقفال (مُرحَّل) | Accounting | `doClosing()` (`lockDate`) | `DB.manualJournals` · `DB.closings` | `doClosing` | 3099 | 3020 | — | ✅ (guard تكرار) | **Handler ✅ (`accounts:edit`)** | ✅ | — (قيد ختامي) | **Runtime-verified (executed)** CLOSE-01 (`net 544,014.33`) | Posting Service · **ACC-D** (توحيد القفل) |

### 2.10 Opening balances & fixed-asset acquisition (Confirmed — runtime)

| JS ID | Business Event | Module | Trigger | Entity | Posting Writer | Debit | Credit | Snap/Recalc | Lock | Permission | Audit | Reversal Strategy | Runtime Status | Production Target |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| JS-C1 | القيد الافتتاحي (Opening Journal) — الخزينة + المخزون فرعان من نفس الحدث | Accounting | بدء التشغيل (`ref:'OPENING'`) | `tr.opening` · product opening | `buildJournalCore` (OPENING block) | 1010/1021/1022 (خزينة) · **1200** (مخزون افتتاحي) | **3001** أرصدة افتتاحية | SNAP | — | Not runtime-verified (perm) | Not runtime-verified | لا عكس (قيد تأسيس) | **Runtime-verified (executed)** OB-01 (3001=2,403,839.5) · OB-02 (1200) | Posting Service · Migration seed (ACC-A) |
| JS-C2 | شراء أصل ثابت (Fixed Asset Acquisition) | Fixed Assets | إنشاء أصل `!opening` | `DB.assets` | `buildJournalCore` (2921–2928) | **1300** بالتكلفة | البنك (افتراضي **1021**) | SNAP | — | **Handler ✅ (`editAsset` — accounts:edit)** | Not runtime-verified | مُشتقّ | **Runtime-verified (executed)** FA-01 (Dr 1300 120,000 / Cr 1021) | Posting Service · **KI-029** (إتاحة تمويل ذمة دائنة) |

---

## 3. Confirmed sources pending individual column-review (27)

> هذه مصادر **مؤكَّدة** في كتالوج `Accounting.md`، لكن **لم تُراجَع أعمدتها الـ15 فردياً** في هذه الجلسة. **لا تُخترَع خاناتها.** الحالة لكلٍّ: Snap/Lock/Permission/Audit/Reversal/Runtime = `Column-review pending` ما لم يُذكر خلاف ذلك في §2. **Runtime Status = Not runtime-verified** (لم تُنفَّذ كسيناريو ترحيل في `Prototype_Runtime_Verification_Report.md` §4).

| JS ID | Business Event (من الكتالوج) | Module | الحالة |
|---|---|---|---|
| JS-02 · JS-03 · JS-04 · JS-06 | إيراد/ذمة/ضريبة/تكلفة (فروع البيع) | Sales/Export | Column-review pending · Not runtime-verified |
| JS-13 · JS-14 · JS-15 | مشتريات/GRNI/مدخلات (فروع الشراء) | Procurement | Column-review pending · Not runtime-verified |
| JS-17 · JS-18 | مخزون/تكلفة (فروع) | Inventory | Column-review pending · Not runtime-verified |
| JS-20 · JS-21 · JS-22 · JS-23 · JS-24 · JS-25 · JS-26 | مخزون/جرد/تكلفة/تأجيل | Inventory | Column-review pending · Not runtime-verified |
| JS-27 · JS-28 · JS-29 · JS-30 | إنتاج/أوامر تشغيل | Production | Column-review pending · Not runtime-verified |
| JS-31 · JS-32 · JS-33 | تمويل تجاري/اعتمادات (LC) | Trade Finance | Column-review pending · Not runtime-verified |
| JS-40 · JS-42 | خزينة/عهدة (فروع) | Treasury | Column-review pending · Not runtime-verified |
| JS-49 · JS-50 | مخصصات/تسويات | Accounting | Column-review pending · Not runtime-verified |
| JS-52 · JS-53 · JS-54 | قيود يدوية/تسويات ختامية | Accounting | Column-review pending · Not runtime-verified |

> **الإجمالي:** 35 مصدراً مُطبَّعاً كاملاً (§2) + 27 مصدراً `Column-review pending` (§3) = **62 confirmed** · **0 conditional** · **1 future** (JS-F1 إشعار دائن للعميل — `Requires future implementation`) = **63 total catalog items**. مطابق للكتالوج المعتمَد.

---

## 4. Runtime-verified summary (from Prototype_Runtime_Verification_Report.md §4)

| المصدر | سيناريو التشغيل | النتيجة |
|---|---|---|
| JS-16 supplier payments | SP-01 → SP-07 | Executed (7 فروع) — دفاع مطلوب: perm · dedup · ceiling · delete-audit |
| Payroll Path B | PAY-02/04/05 | Executed — 5410/5420/2120/2130/2140؛ **bypass** (accountant) |
| JS-19 recurring salary | PAY-03 | Executed — 5400/2015 |
| JS-38 Engine A + JS-38b | FX-01 | Executed — 4030 + عكس تلقائي؛ **bypass** (viewer) |
| JS-34→37 Engine B | FX-02 | Reachable؛ **posted lines NOT VERIFIABLE** (zero exposure) |
| JS-59 closing | CLOSE-01 | Executed — 3099/3020 |
| JS-48 depreciation | FA-02 | Executed — 5400/1310 |
| JS-C2 acquisition | FA-01 | Executed — 1300 / bank |
| JS-C1 opening | OB-01/OB-02 | Executed — 3001 · 1200 |

**كل ما عدا ذلك:** `Not runtime-verified` (code-confirmed؛ الشاشة تُرسَم) — لا يُدّعى تنفيذ لم يحدث.
