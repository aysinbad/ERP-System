# Accounting Engine — Implementation Guide

## Document Information
```
Document Name:  Accounting Engine Implementation Guide
Version:        0.9.0 (Runtime + source-classification corrections)
Status:         Draft — In Progress
Classification: Source of Truth
Owner:          Solution Architecture Team
Approved-by:    —
Approved-date:  —
Last-Updated:   2026-07-25 (Runtime corrections + catalog 62/0/1/63; JS-C1/JS-C2 → Confirmed)
```

> **قاعدة الفصل (ADR-014):** هذا الملف يجيب على **«كيف يُنفَّذ؟»** — أسماء الدوال، Posting Matrix، سلوك القفل والعكس والتدقيق. **«ماذا يضمن النظام؟»** في `Accounting.md`.
>
> **مصدر كل ما يلي:** `Accounting_Session6_1_Remaining_Evidence_Findings.md` (مطابَق للـPrototype) + كتالوج `Accounting.md`.
>
> ⚠️ **الأخطاء المعروفة مُعلَّمة 🔴 ولا تُقرأ كسلوك مطلوب** (`AGENTS.md` قاعدة 5). عمود **KI** يربط كل خلل بسجله.


## 0. Final Source-Boundary Reconciliation (Part 1)

> **القاعدة المعتمَدة:** يُعدّ المصدر بـ**الحدث الاقتصادي المتمايز ذي البنية المحاسبية المتمايزة**، لا بالكيان ولا بالدالة. (مستخرَجة من كتالوج جلسة 5 — مرجع `custodyTx`.)

| العنصر | الحكم | Impact on Confirmed count | المبرِّر |
|---|---|---|---|
| **JS-38b** العكس التلقائي | حدث بتاريخ منفصل ⇒ **مصدر مستقل** | +1 | كإقفال FX (حدثان) |
| **JS-41** فرع شراء صنف | فرع داخل نفس حدث الصرف | 0 | تفرّع شرطي |
| **JS-55** deposit | `kind` متمايز ⇒ مستقل | +1 | كـcustody kinds |
| **JS-56** drawing | مستقل | +1 | نفسه |
| **JS-57** bankfee | مستقل | +1 | نفسه |
| **JS-58** interest | مستقل | +1 | نفسه |
| **JS-59** closing entry | حدث اقتصادي متمايز | +1 | من DO-4 |
| **JS-C1** opening balances | **`Confirmed posting source — Opening Journal`** | **+1** (كان Conditional) | قيد `OPENING` حيّ ⇒ 3001 (runtime OB-01/OB-02) |
| **JS-C2** fixed-asset acquisition | **`Confirmed posting source — Fixed Asset Acquisition`** | **+1** (كان Conditional) | قيد اقتناء حيّ Dr 1300 / Cr bank للأصول `!opening` (runtime FA-01) |
| **JS-C3** payroll recurring | `Specialized use case of JS-19` | **−1** (يخرج من Conditional) | لا مصدر مستقل |

> **تفكيك JS-C1 / JS-C2 (مُصحَّح تشغيلياً 2026-07-25):**
> - **يصيران مصدرَي ترحيل مؤكَّدين (Confirmed)** بأدلة التشغيل الحيّة (`Prototype_Runtime_Verification_Report.md` §4 OB-01/OB-02/FA-01). لم يعودا `Conditional / Unverified candidate`.
> - **JS-C1 = مصدر واحد (Opening Journal).** الرصيد الافتتاحي للخزينة والمخزون الافتتاحي **فرعان/سطران من نفس الحدث الاقتصادي `OPENING`** — **لا يُعَدّان مصدرين منفصلين** (لم يُثبَت أنهما حدثان اقتصاديان مستقلان بموجب قاعدة حدود المصدر). **لا JS-C1b مُحتسَب.**
> - **JS-C2 = مصدر واحد (Fixed Asset Acquisition).**
> - التصنيف بعد التصحيح: **62 + 0 + 1 = 63 total catalog items**، منها **62 confirmed posting sources**، **0 Conditional**، **1 Future**.
> - **حدّ تغطية المصفوفة (Partial posting-matrix coverage):** الاعتراف بالترحيل مؤكَّد تشغيلياً للسيناريوهات المُنفَّذة؛ لم تُراجَع كل الأعمدة الـ18 لكل المصادر.

### مصفوفة المطابقة النهائية

| البند | جلسة 5 (خط أساس) | +/− | جلسة 6.2 + تصحيح تشغيلي |
|---|---|---|---|
| **Confirmed posting sources** | 54 | +JS-55/56/57/58 (+4) · +JS-38b (+1) · +JS-59 (+1) = +6 · **+JS-C1 · +JS-C2 (+2 تشغيلي)** | **62 confirmed posting sources** |
| Conditional / Unverified candidates | 3 | −JS-C3 (−1) · **−JS-C1 · −JS-C2 (−2 تشغيلي)** | **0** |
| Future | 1 | 0 | **1** |
| **63 total catalog items** | **58** | **+5** | **63 total catalog items** |

> **63 total catalog items · 62 confirmed posting sources · 0 Conditional · 1 Future.** الأعداد محسوبة حرفياً بعد تصحيح التشغيل: JS-C1/JS-C2 انتقلا من Conditional إلى Confirmed بأدلة `OPENING`/الاقتناء الحيّة؛ الرصيد الافتتاحي للخزينة والمخزون فرعان من نفس حدث `OPENING` فلا يُحتسب JS-C1b. **الإجمالي 63 ثابت.**

### تصنيف Snap/Recalc المعتمَد

| JS | التصنيف |
|---|---|
| **JS-16** | **MIXED** |
| **JS-39** | **RECALC** |
| **JS-38** | **SNAP** |
| **JS-C3** | use case of JS-19 |

RECALC+MIXED: **20 ⇒ 22** (بإضافة JS-39 وJS-16).

---

## 1. Legend

- **SNAP** = القيمة مُجمَّدة على المستند · **RECALC** = تُحسَب لحظة بناء الدفتر · **MIXED** = الاثنان في نفس القيد.
- **Lock** = هل يفحص مسار الكتابة `isLocked()`؟ · **Reversal** = ماذا يحدث عند الحذف/التعديل؟
- كل الحسابات بأكوادها المعيارية (Glossary §10).

---

## 2. Posting Matrix

> ## ⚠️ حدّ التغطية — إفصاح صريح
> **Core Posting Matrix completed for evidence-reviewed and high-risk sources; full normalized 60-source matrix remains pending.**
>
> الصفوف أدناه تغطّي **33 مصدراً** (30 مصدر ترحيل مؤكَّد مراجَع + 3 مشروطة/مقترحة) بكل الأعمدة الـ18. الـ**30 مصدراً المؤكَّدة المتبقّية** من كتالوج الـ60 (JS-02·03·04·06·13→15·17→18·20→33·40·42·49→54) **مذكورة في كتالوج `Accounting.md` لكن لم تُراجَع كل أعمدتها من الـPrototype في هذه الجلسة** — تُستكمَل في جلسة لاحقة. **لا يُدّعى اكتمال المصفوفة الكاملة.**
>
> **لماذا الفالباك لا الاختراع:** إنتاج 60 صفاً مُطبَّعاً يتطلب التحقق الفردي من الكاتب والحسابات والتصنيف لكل مصدر — ما لم يُنجَز بعد لـ30 منها. ملؤها بلا تحقق = اختراع، ممنوع بـ`AGENTS.md` قاعدة 1 وADR-000.

### المصادر المراجَعة (33)

### 2.1 المبيعات والإيراد

| JS | الحدث الاقتصادي | Collection | Writer | Trigger | Date | Debit | Credit | VAT | FX | Snap | Lock | Reversal | Audit | AINV | KI | Confidence |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **JS-01** | شحن بضاعة لعميل (إيراد+تكلفة) | `DB.invoices` | `saveInvoice` | نقل المخاطر | `inv.date` | 1100 · 5010 | 4010 · 1200 | 2100 مخرجات | `fxOf(inv)` | **MIXED** | ✅ | 🔴 KI-009 مسح | ✅ | AINV-01 | KI-009 · KI-024 | Confirmed |
| **JS-05** | مخصص مطالبات (EAS 48) | `DB.claims` (provision) | `claimsProvision` | إيراد معرّض لمطالبات | `DB.reportDate` 🔴 | 5300 | 2105 | — | — | 🔴 RECALC | جزئي | مُشتقّ | — | AINV-13 | KI-012 · KI-013 · KI-023 | Confirmed |
| **JS-51** | استحقاق عمولة مبيعات | `DB.commissionAccruals` | `accrueCommission` | استحقاق على المُحصَّل | `ca.date` | 5440 | 2017 | — | — | **SNAP** ✅ | ✅ | مُشتقّ | ✅ | AINV-25 | KI-017 | Confirmed |
| **JS-12** | مرتجع مبيعات | `DB.returns` (sales) | `saveReturn` | إرجاع عميل | `rt.date` | 4090 · 1200 | 1100 · 5010 | — | `fxOf(rt)` | 🔴 مشروط | ✅ | مُشتقّ | ✅ | AINV-31 | KI-020 | Confirmed |

> 🔴 **JS-05:** الوعاء تراكمي في اليومية ومُفلتَر على الشاشة (KI-013)؛ التاريخ `DB.reportDate` قسراً (KI-023).
> 🔴 **JS-12:** عكس التكلفة مشروط بـ`rt.cogsEGP` المخزَّن — قد يكون فارغاً (KI-020).

### 2.2 المشتريات والمخزون

| JS | الحدث | Collection | Writer | Date | Debit | Credit | VAT | FX | Snap | Lock | Reversal | Audit | AINV | KI |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **JS-07** | استلام بضاعة (GRNI) | `DB.receipts` | `saveReceipt` | `rc.date` | 1200 | 2016 | — | `grnRate` | SNAP | ✅ | مُشتقّ | ✅ | AINV-15 | KI-014 |
| **JS-08** | فاتورة شراء (تُقفل GRNI) | `DB.purchases` | `savePurchase` | `pu.date` | 1200 (فرق) · 1130 | 2010 · 2016 · 2110 | 1130 مدخلات | `fxOf(pu)` | 🔴 RECALC (unitCost) | ✅ | 🔴 محو | ✅ | AINV-03 | KI-014 · KI-022 |
| **JS-09** | فروق الجرد | `DB.stocktakes` | `postStocktake` | `st.date` | 5020 ⇄ 1200 | عكسي | — | — | 🔴 RECALC | ✅ | مُشتقّ | ✅ | AINV-18 | KI-022 |
| **JS-10** | هبوط NRV (EAS 2) | `nrvWritedown()` | مُشتقّ | `DB.reportDate` 🔴 | 5015 | 1205 | — | — | 🔴 RECALC | جزئي | مُشتقّ | — | AINV-23 | KI-018 · KI-023 |
| **JS-11/13** | أمر تشغيل — انحراف/هدر | `DB.workOrders` | `postWorkOrder` | `wo.date` | 1200 · 1260 | عكسي · 2010 | — | — | 🔴 RECALC | ✅ | مُشتقّ | ✅ | AINV-18 | KI-021 · KI-022 |

### 2.3 سداد الموردين — مصفوفة صريحة (KI-025 · KI-026)

| فرع | الشرط | Debit | Credit | FX source | Snap | Lock | Dup guard | Ceiling | Audit |
|---|---|---|---|---|---|---|---|---|---|
| **مربوط بفاتورة** | `purchaseId` مملوء | 2010 بـ`payableEGP` | `cashAcc(treasury)` بـ`cashEGP` · (+4030/5500) | `sp.fxRate‖fxRate` للنقدية · `fxOf(pu)` للذمة | 🟡 **MIXED** | ✅ create/update · ✅ delete gate | 🔴 لا | 🔴 لا | ✅ create/update · 🔴 delete |
| **مربوط بمصروف** | `expId` مملوء | 2010 · … | نفسه · `ex.fxRate` للذمة | نفسه | 🟡 MIXED | ✅ create/update · ✅ delete gate | 🔴 لا | 🔴 لا | ✅ create/update · 🔴 delete |
| **غير مربوط** | كلاهما فارغ | 2010 بـ`payableEGP` | نفسه | `invRate = payRate` ⇒ **`fxDiff=0`** 🔴 | 🔴 يفقد الفرق | ✅ create/update · ✅ delete gate | 🔴 لا | 🔴 لا | ✅ create/update · 🔴 delete |

- **Writer:** بلوك `(DB.supplierPayments||[]).forEach` داخل `buildJournalCore()` — لا دالة مستقلة.
- **JS-16 · إعادة تصنيف معتمدة: `MIXED`.** إنشاء/تعديل السداد يفحصان القفل ويُسجَّلان في Audit؛ الفجوات المؤكدة هي الازدواج والسقف وصلاحية الإجراء وAudit الحذف. **AINV:** AINV-11 · AINV-04 · AINV-30. **KI:** KI-025 · KI-026.

### 2.4 إشعارات خصم الموردين — مصفوفة صريحة (KI-027)

| نوع (`reason`) | مربوط؟ | Debit | Credit | VAT | Inventory | 
|---|---|---|---|---|---|
| `price` | ✅ | 2010 بـ`tot` | **1200** بـ`base` | 1130 دائن | ✅ يخفّض المتوسط |
| `price` | ❌ | 2010 بـ`tot` | **4020** | 1130 دائن | ❌ |
| `shortage` | ✅ | 2010 بـ`tot` | **1200** | 1130 دائن | ✅ |
| `shortage` | ❌ | 2010 بـ`tot` | **4020** | 1130 دائن | ❌ |
| `quality` | — | 2010 بـ`tot` | **1260** ⚠️ | 1130 دائن | ❌ مصدر ثانٍ لتراكم 1260 |
| `late` | — | 2010 بـ`tot` | **4020** | 1130 دائن | ❌ |
| `other` | — | 2010 بـ`tot` | **4020** | 1130 دائن | ❌ |

- **Writer:** `editCreditNote` ⇒ ترحيل في بلوك `(DB.creditNotes||[]).forEach`. **قاعدة الحساب:** `cnPostingAccount(cn)` — 1200 يُحوَّل قسراً لـ4020 إن لم يكن مربوطاً.
- **Snap:** SNAP (`fxRate` مُجمَّد وقت الحفظ) · **Lock:** ✅ على الإنشاء · 🔴 على الحذف/التعديل · **Audit:** ✅ · **Dup:** 🔴 لا.
- ⚠️ **تعارض اسم 1260** بين `CN_REASONS` والأستاذ (S6.1 §4.4) — `Inferred — requires review`.

### 2.5 المصروفات والرواتب — مساران فعّالان (KI-030 · KI-031)

#### المسار A — الرواتب عبر المصروف المتكرر

| `allocType` | Debit | ملاحظة |
|---|---|---|
| `invoice` / `product` | **5100** | محمّل على شحنة |
| `general` | **5200** | تُوزَّع |
| `overhead` / افتراضي | **5400** | RC-002 «رواتب الموظفين» يقع هنا |

`DB.recurring → genRecurring() → DB.expenses → JS-19`. الحساب يتبع `allocType` لا `cat`. هذا المسار لا يفصل الرواتب أو التأمينات أو الضرائب.

#### المسار B — محرك الرواتب المتخصص

`postPayroll() → DB.manualJournals` بمرجع `PAYROLL-{month}`، ويستخدم:

- 5410 مصروف الرواتب والأجور
- 5420 حصة الشركة في التأمينات
- 2130 تأمينات مستحقة
- 2120 ضريبة كسب عمل مستحقة
- 2140 رواتب مستحقة

المحرك المتخصص يملك حارس تكرار داخل مساره، لكن لا يوجد **Cross-Guard** مع RC-002. تشغيل المسارين للشهر نفسه قد يضاعف الرواتب (KI-030).

#### المحركات المتكررة

يوجد `DB.recurring` و`DB.recurringExp`. تعريف `window.editRecurring` اللاحق الخاص بـ`DB.recurring` يظلّل التعريف الأقدم؛ شاشة `DB.recurring` تعمل بالنموذج الصحيح، بينما محرر `DB.recurringExp` الأقدم يصبح Dead Code (KI-031).

### 2.6 نظاما إعادة التقييم — مصفوفتان (KI-032)

**النظام أ — `DB.revaluations` (JS-38) — يخالف ADR-003:**

| `it.type` | `diff>0` | `diff<0` | صافي |
|---|---|---|---|
| customer | 1100 مدين | 1100 دائن | net>0 ⇒ 4030 · net<0 ⇒ 5500 |
| supplier | 2010 مدين | 2010 دائن | نفسه |
| bank/افتراضي | 1020 مدين | 1020 دائن | نفسه |

- Writer `postRevaluation` · Date `DB.reportDate` 🔴 · **SNAP** ✅ (`rates`+`diff` مُجمَّدة) · Lock 🔴 · Audit `securityLog` فقط.
- **JS-38b — العكس التلقائي:** قيد ثانٍ بتاريخ `firstOfNextMonth`، حسابات معكوسة. Reversal تلقائي مُدمَج.

**النظام ب — `DB.fxRevaluations` (JS-34→37) — متوافق ADR-003:**

| JS | الشرط | Debit | Credit | Date |
|---|---|---|---|---|
| JS-34 | `net>0` | 1265 | 4030 | `v.date‖reportDate` |
| JS-35 | `net<0` | 5500 | 1265 | نفسه |
| JS-36 | `closed & net>0` | 3020 | 1265 | `closedDate` |
| JS-37 | `closed & net<0` | 1265 | 3020 | نفسه |

- Writer `postFxRevaluation`/`closeFxRevaluation` · **SNAP** · **Lock ✅** · **Audit ✅ كامل**.
- 🟡 **الازدواج (خطر كامن latent — لم يُعَد إنتاجه تشغيلياً):** كلاهما يملك مُصدِّر 4030/5500 بلا Cross-Guard (حقيقة كود)، لكن المحرك B **خامل** (`fxExposure()`=صفر) فلم يُنتَج قيد مزدوج تشغيلياً. المحرك A فقط يُرحِّل (runtime FX-02/FX-03). التوحيد يبقى مفتوحاً (KI-032 · **OQ-6.F 🔴 Open**).

### 2.7 حركات الخزينة — خمسة `kind` (S6.1 §7.6)

| JS | `kind` | Debit | Credit | Snap | AINV |
|---|---|---|---|---|---|
| **JS-39** | `transfer` | خزينة الوجهة بـ`toEgp` · (+5500) | خزينة المصدر بـ`srcEgp` · (+4030) | 🔴 **RECALC** (`fxRate` حيّ لكل طرف) | AINV-10 |
| **JS-55** | `deposit` | حساب الخزينة | 3010 رأس المال | SNAP | — |
| **JS-56** | `drawing` | 3030 مسحوبات | حساب الخزينة | SNAP | — |
| **JS-57** | `bankfee` | 5400 | حساب الخزينة | SNAP | — |
| **JS-58** | `interest` | حساب الخزينة | 4020 | SNAP | — |

- Writer: بلوك `(DB.treasuryTx||[]).forEach` · Date `tx.date` · Lock حسب النموذج.
- **JS-39 · إعادة تصنيف معتمدة: `RECALC`** (S6.1 §10.2 · ACC-B يعالجه).

### 2.8 العهدة — أربعة فروع (JS-41)

| فرع | الشرط | Debit | Credit |
|---|---|---|---|
| issue | `kind='issue'` | 1120 عهد | خزينة |
| expense · على شحنة | `invId` مملوء | **5100** | 1120 |
| expense · تشغيلية | `cat='تشغيلية'` | **5200** | 1120 |
| expense · مباشرة | `cat='مباشرة'` | **5100** | 1120 |
| expense · افتراضي | — | **5400** | 1120 |
| **expense · شراء صنف** | `prodCode && qty>0` | **1200** | 1120 |
| settle | `kind='settle'` | خزينة/1010 | 1120 |

- 🔴 **لا معالجة VAT إطلاقاً** على مصروفات العهدة · حارس الرصيد تحذير لا منع · Lock ✅.

### 2.9 قيود الفترة والإقفال

| JS | الحدث | Debit | Credit | Date | Snap | AINV | KI |
|---|---|---|---|---|---|---|---|
| JS-43 | مخصص مطالبات (سنوي) | 5300 | 2105 | `DB.reportDate` 🔴 | RECALC | AINV-34 | KI-023 |
| JS-44 | مخصص ديون مشكوك فيها | 5400 | 1110 | `DB.reportDate` 🔴 | RECALC | AINV-34 | KI-023 |
| JS-45 | مخصص نهاية الخدمة | 5430 | 2160 | `DB.reportDate` 🔴 | RECALC | AINV-34 | KI-023 |
| JS-46 | مخصص ضريبة الدخل | 5900 | 2150 | `DB.reportDate` 🔴 | RECALC | AINV-34 | KI-022 · KI-023 |
| JS-47 | أصل ضريبي مؤجل (EAS 24) | 1340 | 5910 | `DB.reportDate` 🔴 | RECALC | AINV-34 | KI-022 · KI-023 |
| JS-48 | قسط إهلاك شهري | 5400 | 1310 | شهري | RECALC | AINV-05 ✅ | KI-029 🟢 (تمويل اقتناء افتراضي بنك — الاقتناء **مُرحَّل** JS-C2) |
| **JS-59** | 🆕 قيد إقفال (مقترح) | 3099 | 3020 | `lockDate` | — | AINV-26 | KI-033 |

### 2.10 الأرصدة الافتتاحية — ✅ Confirmed posting source (Opening Journal) (مُصحَّح تشغيلياً 2026-07-25)

| البند | الحالة |
|---|---|
| **JS-C1** (Opening Journal) | **`Confirmed posting source`.** قيد `ref:'OPENING'` واحد يضمّ **فرعين/سطرين من نفس الحدث الاقتصادي**: (1) الخزينة تدخل الأستاذ مقابل **3001** (runtime OB-01)؛ (2) المخزون الافتتاحي **مدين 1200** داخل نفس القيد (runtime OB-02). الأستاذ والخزينة متطابقان. ~~KI-028~~ **مدحوض/مغلق**. **لا يُحتسب فرع المخزون كمصدر مستقل (JS-C1b) — هو سطر من حدث `OPENING`.** |

---

## 3. الأخطاء المعروفة المؤثّرة على التنفيذ

| المجال | الخلل | KI | ADR المعالج |
|---|---|---|---|
| القيمة الحيّة | `unitCost()` · `fxRate()` · `invRate` تُحسَب وقت البناء | KI-022 · KI-025 | ACC-B |
| التاريخ | `DB.reportDate` كتاريخ قيد | KI-023 | ACC-A · ACC-B |
| الحذف | يمحو التاريخ بلا عكس | KI-008 · KI-024 · KI-027 | ACC-C |
| القفل | موزَّع · مسارات تتجاوزه | KI-015 · KI-016 · KI-026 · KI-033 | ACC-D |
| الافتتاحي/الأصول | ~~لا قيد افتتاحي · لا قيد اقتناء~~ **مدحوض تشغيلياً** — الافتتاحي والاقتناء **مُرحَّلان** (OB-01/OB-02/FA-01)؛ يبقى فقط: تمويل الاقتناء يفترض البنك دائماً | ~~KI-028~~ (مغلق) · KI-029 🟢 | ACC-A (تمويل فقط) |
| الرواتب | مساران بلا Cross-Guard؛ المتكرر إلى 5400 والمتخصص إلى حسابات الرواتب | KI-030 | ACC-A · ACC-B · OQ-6.D |
| ازدواج | نظاما تقييم (ازدواج FX **كامن latent** لا مؤكَّد تشغيلياً؛ محرك B خامل) · محركان متكرران · مسارا رواتب | KI-030 · KI-031 · KI-032 🟡 | OQ-6.F 🔴 Open |
| 2017 | لا مسار استنفاد | KI-017 | ACC-E (محجوب OQ-6.A) |

---

## 4. Permissions · Locking · Audit (إطار عام)

- **Permissions:** نمط `can('accounts','edit')` · `can('journal','edit')` · `can('purchases','edit')` — دوال ديناميكية لا أسماء أدوار.
- **Locking (مطلوب):** خدمة `PeriodLockService` مركزية (ACC-D) تحلّ محل فحوص `isLocked` الموزَّعة.
- **Audit (مطلوب):** كل حدث محاسبي يمرّ بـ`audit()` — إنشاء/تعديل سداد الموردين مدقَّقان، لكن حذف السداد وحذف إعادة التقييم يحتاجان تغطية صحيحة.

---

## 5. المتبقّي لجلسات لاحقة

- **إكمال المصفوفة المُطبَّعة للـ30 مصدراً المؤكَّدة المتبقّية** (JS-02·03·04·06·13→15·17→18·20→33·40·42·49→54) بمراجعة أعمدتها من الـPrototype.
- **Posting Matrix (الأعمدة الـ18) لـ JS-C1/JS-C2** — أصبحا **Confirmed posting sources** (Opening Journal · Fixed Asset Acquisition)؛ تُستكمَل صفوفهما الكاملة في المصفوفة المُطبَّعة (تغطية جزئية حالياً — Partial posting-matrix coverage).
- **Domain Events** لكل مصدر مؤكَّد (بعد ACC-A).
- **Standard Error Codes** و**Reconciliation Service** (تفصيل ACC-A).
