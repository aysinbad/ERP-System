# Accounting Engine — RFCs

## Document Information
```
Document Name:  Accounting Engine RFCs
Version:        1.0.0 (RFC-ACC-001 محسوم — Immutable Posting + Reconciliation Engine)
Status:         Active
Classification: Reference
Owner:          Solution Architecture Team
Last-Updated:   2026-07-23 (Session 5 merge)
```

> RFCs الخاصة بمحرك المحاسبة. النقاش التفصيلي (خيارات/تريد أوف) يُفتَح عند نضوج السؤال؛ لو استقر، يُرقَّى لـ ADR رسمي في `02_Governance/ADR/` ويُعلَّم هنا كمكتمل (لا يُحذَف — للتتبّع التاريخي).

---

## RFC-ACC-001 — Snapshot vs Recalculation (Cross-Module)

```
RFC:              RFC-ACC-001
Title:            Snapshot vs Recalculation of Derived Values
Type:             Cross-Module
Status:           ✅ RESOLVED (2026-07-23, Session 5)
Decision:         Immutable Posting + Reconciliation Engine
Assessment:       Well supported
Scope:            Accounting · Pricing · Inventory
Owner:            Solution Architecture Team
Opened:           2026-07-21
Resolved:         2026-07-23
Referenced-by:    Pricing.md · Inventory_Production.md
Expected-outcome: Cross-Module ADR
Promoted-to:      ADR-020 (ACC-A) — Proposed
                  + ACC-B · ACC-C · ACC-D · ACC-E (مرشّحون — §ADR Dependency Map)
```

> **هذا هو المصدر القانوني الوحيد (Canonical) لهذا السؤال.** أي موديول آخر يمسّه يُشير إليه هنا عبر cross-reference، ولا يفتح نقاشاً موازياً.
>
> ⚠️ **الأدلة أُعيد بناؤها من الصفر في جلسة 5.** النص التحليلي السابق (نسخة 0.5.0) **لا يُعتمَد** لأن قرينة العمولة سقطت بالكامل — `accrueCommission()` تبيّن أنها **نموذج Snapshot سليم** لا مثال على إعادة الحساب (C-01). الجدول القديم لحالة الأبعاد أُبطل واستُبدل بما أدناه.

### Problem (المشكلة الجذرية)

هل يُجمِّد النظام القيمة المشتقّة **لحظة الحدث** (Snapshot)، أم يعيد حسابها من **الحالة الحالية للنظام** كلما طُلبت (Recalculation)؟

### Resolution Status by Dimension — نهائي

| البُعد | الموديول | الحالة | القرار |
|---|---|---|---|
| **Pricing Guidance at Proforma Decision** | Pricing → CRM | ✅ **محسوم (2026-07-21)** | `proforma.items[].unitPrice` هو السعر التجاري الفعلي، مستقل. السعر الاسترشادي **metadata اختيارية للتدقيق فقط** (`PriceGuidanceRecord`) |
| **Commission Recalculation** | Pricing (PINV-12) | ✅ **محسوم — القرينة أُبطلت** | **C-01:** `accrueCommission()` تتبع `Calculate → Review → Snapshot → Post → Lock → Audit` وتحفظ `basis`/`pct`/`from`/`to`. **نموذج Snapshot يُحتذى، لا فجوة.** الفجوة المتبقية الوحيدة: غياب حارس تداخل الفترات ⇒ KI-017 · AINV-25 |
| **Inventory Unit Cost** | Inventory | ✅ **محسوم — Snapshot إلزامي** | `unitCost()` حيّة داخل بناء الدفتر ⇒ **12 مصدراً مباشراً + 2 غير مباشر = 14 في سلسلة التبعية**. تُجمَّد كـ`UnitCostAtPosting` ⇒ ACC-B · KI-022 |
| **Accounting Derived Values** | Accounting | ✅ **محسوم — Snapshot إلزامي** | المخصصات وأساس الاحتساب وتاريخ القيد تُجمَّد لحظة الترحيل ⇒ AINV-03 · AINV-34 · KI-023 |

## 6.1 Snapshot patterns already implemented

| النمط | الدليل | الجودة |
|---|---|---|
| `accrueCommission()` | S4.5 T4 — `Calculate → Review → Snapshot → Post → Lock → Audit` مع حفظ `basis`/`pct`/`from`/`to` | ✅ **نموذجي** |
| `fxOf()` — سعر مثبَّت على المستند | JS-01/09/13/14/17/31 · ADR-007 | ✅ قوي (خرق واحد: JS-08) |
| `st.avgCost` مخزَّن على سطر الجرد | S4 §4.5.3.2 | ✅ |
| `rt.cogsEGP` مخزَّن على المرتجع | S3 §3.10.1.4 | ⚠️ قد يكون فارغاً |
| `rc.fxRate` مخزَّن على إذن الاستلام | S4 §4.2.5 | ✅ |
| `pm.fxRate` مثبَّت وقت التحصيل | S3 §3.9.1 — تعليق الكود صريح | ✅ |
| `puEntersStock()` — قرار أهلية واحد | S4 §4.1.1 | ✅ **نموذجي** |

**⇒ النمط الصحيح موجود داخل النظام، ومطبَّق جيداً، في سبعة مواضع على الأقل.**

## 6.2 Recalculation patterns

> **الحجم الكمّي:** **20 من 54 مصدراً مؤكَّداً** (19 RECALC + 1 MIXED) ≈ **37%**. منها **12 بتبعية مباشرة** لـ`unitCost()` و**14 ضمن سلسلة التبعية الكاملة**. راجع §2.10.

| النمط | الدليل | الخطورة |
|---|---|---|
| **`unitCost()` داخل بناء الدفتر** | S4 §4.4.3 — عبر JS-02/20/21/22/23/26/27/28/53 | 🔴🔴 **الأخطر** |
| `claimsProvision()` | S4.5 T2 — مبلغ مطلق على وعاء متغيّر | 🔴 |
| `nrvWritedown()` | S4.5 T3 — يعتمد `unitCost()` الحيّة | 🔴 |
| قيود التأجيل تختفي وتظهر | S3 §3.2.2 — بلا حركة عكسية | 🔴 |
| `productionMode()` يغيّر معالجة أوامر تاريخية | S4 §4.6.4 — إعداد عام لا على المستند | 🔴 |
| `doubtfulProvision` · `endOfServiceProvision` · `incomeTaxProvision` · `deferredTaxAsset` | JS-44/45/46/47 | 🟡 |
| الإهلاك يُعاد اشتقاقه من جدول الأصل | JS-48 | 🟡 |

## 6.3 Context-dependent calculation

**نمط ثالث لم يكن معروفاً قبل جلسة 4.6.**

`fInvoices()` تعتمد `inPeriod()` ⇒ `window._period` — فلتر واجهة. و`buildJournal()` **تحيّده قسراً** أثناء بناء الدفتر (إصلاح موثَّق).

| السياق | مجتمع الفواتير |
|---|---|
| قيد اليومية (JS-43) | **كل الفواتير — تراكمي أبدي** |
| لوحة EAS وقائمة الدخل | فواتير المدى المُرشَّح |

⇒ **نفس الدالة تُنتج وعاءين مختلفين.** المستخدم يضبط النسبة وهو يرى وعاء الفترة، والقيد يُرحَّل على الوعاء الكامل. **والوسم في الواجهة يقول «من إيراد الفترة».**

هذا ليس Snapshot ولا Recalculation — إنه **حساب معتمد على السياق**، وهو **أخطر الثلاثة** لأنه غير مرئي: الرقمان صحيحان كلٌّ في سياقه، ولا شيء ينبّه للاختلاف.

## 6.4 Immutable-posting gap

| الفجوة | الدليل |
|---|---|
| الحذف أو التعديل يعيد كتابة الدفاتر التاريخية | S3 §3.10.3 · AINV-30 |
| ميزان مراجعة تاريخي غير قابل لإعادة الإنتاج | AINV-29 |
| **لا يوجد دفتر مُرحَّل مُخزَّن كمصدر حقيقة** | `buildJournalCore()` تبنيه من الصفر كل استدعاء |
| ثلاثة قيود على الأقل مؤرَّخة بـ`DB.reportDate` المتغيّر | JS-26/43/44 |
| لا مفتاح تفرّد ⇒ ازدواج ممكن (KI-009 · تعدّد فواتير GRN) | AINV-04 · AINV-15 |

## 6.5 مقارنة البدائل

| البديل | الوصف | المزايا | العيوب | الحكم |
|---|---|---|---|---|
| **1. Full Recalculation** | إبقاء الدفتر مُشتقّاً بالكامل، مع تجميد المدخلات فقط | لا هجرة معمارية · استحالة تناقض الدفتر مع المستندات · يحفظ ما بُني | ❌ لا يحل AINV-29 ولا AINV-30 · الحذف يبقى محو تاريخ · **غير مقبول لأي مراجع خارجي** | **مرفوض** |
| **2. Hybrid Ledger Authority** | دفتر مُخزَّن للفترات المقفلة، مُشتقّ للمفتوحة — **أي أن سلطة الدفتر تنتقل عند لحظة الإقفال** | هجرة أخف · مرونة في الفترة الجارية | ❌ سلوكان لنفس القيد · لحظة القفل تصير تحويلاً معمارياً · **يضاعف التعقيد بدل تقليله** · الفترة المفتوحة تبقى بلا أثر تدقيقي | **مرفوض** |
| **3. Immutable Posting + Reconciliation Engine** | قيد مُخزَّن غير قابل للتعديل؛ ومحرك الاشتقاق يتحول لأداة مطابقة | ✅ يحل AINV-02/03/04/29/30 · أثر تدقيقي كامل · **يحفظ قيمة `buildJournalCore` كرقابة داخلية** · متوافق مع الممارسة المحاسبية المعيارية | تكلفة هجرة أعلى · يتطلب إعادة كتابة طبقة الترحيل | **✅ معتمد** |

> **حدود رفض `Hybrid Ledger Authority`:**
>
> المرفوض هو **انتقال سلطة الدفتر من دفتر مشتق إلى دفتر ثابت عند لحظة الإقفال**. لا يُرفض وجود **دفتر مُرحَّل ثابت** مع **تقارير تشغيلية مشتقة** و**محرك مطابقة** — فهذا هو النموذج المعتمد نفسه.

## 6.6 القرار

> **Production accounting uses immutable posted journals with frozen snapshots. Recalculation engines remain reconciliation tools, not the legal ledger.**
>
> **معتمد.**
>
> **Assessment: `Well supported`** — بعد تصحيح الأدلة العددية (20 RECALC/MIXED · 12 direct · 14 chain · 18/34 violated) وتوضيح حدود رفض `Hybrid Ledger Authority` وإضافة تقدير تكلفة الهجرة.

**الأسباب (بالترتيب):**
1. **AINV-29 غير قابل للتحقيق بدونه.** ميزان تاريخي غير قابل لإعادة الإنتاج يُبطل الدفاتر أمام المراجعة — لا لأن الأرقام خاطئة، بل لأنها غير قابلة للإثبات.
2. **النمط مُثبَت داخل النظام نفسه** — `accrueCommission()` و`puEntersStock()` نموذجان ناجحان. التعميم لا الاستيراد.
3. **البديلان الآخران لا يحلّان مشكلة الحذف** — وهي أخطر فجوة تدقيقية.
4. **يحفظ ما بُني**: `buildJournalCore()` تصير **خدمة مطابقة ليلية** تقارن الدفتر المُخزَّن بالمستندات وتُخرج تقرير انحراف. الانحراف = خطأ ترحيل أو تلاعب. **نقطة الضعف تصير أقوى رقابة داخلية.**

**Trade-offs المقبولة صراحةً:**
- ❌ الدفتر قد يتناقض مع المستندات — وهذا **بالضبط** ما تكشفه خدمة المطابقة.
- ❌ التصحيح يحتاج قيداً عكسياً بدل تعديل — تكلفة تشغيلية مقبولة مقابل الأثر التدقيقي.
- ❌ حجم تخزين أكبر — غير جوهري.
- ❌ تعقيد أعلى في طبقة الترحيل — مركَّز في مكان واحد لا موزَّع.

**Migration impact:**
- **Posting Service يُبنى أولاً، قبل أي وحدة أعمال.** بناء المبيعات أو المخزون أولاً يعيد إنتاج المشكلة في تقنية أحدث.
- كل مصدر من الـ**54 مصدراً المؤكداً** يُترجَم إلى **Domain Event ⇒ Posting Rule**.
- الـ**20 مصدر RECALC/MIXED** تحتاج **تحديد نقطة التجميد** لكل قيمة (`UnitCostAtPosting`, `ExchangeRate`, `ProvisionBase`).
- منها **14 مصدراً** تقع ضمن سلسلة التبعية الكاملة لـ`unitCost()`، تشمل **12 اعتماداً مباشراً** و**اعتمادين غير مباشرين**.
- الأرصدة الافتتاحية للإنتاج تُشتقّ من الـprototype مرة واحدة، ثم لا يُشتقّ شيء بعدها.

**Prototype impact:** **لا شيء.** الـprototype مرجع لا يُعدَّل (ADR-000 · قاعدة 9). القرار يحكم **الإنتاج** حصراً.

**تقدير نوعي لتكلفة الهجرة:**

| البُعد | التقدير | التعليل |
|---|---|---|
| Architecture effort | **High** | طبقة ترحيل جديدة كلياً + Domain Events لكل مصدر من الـ54 |
| Data migration effort | **High** | ترحيل الأرصدة الافتتاحية مرة واحدة + تجميد قيم 20 مصدر RECALC/MIXED |
| Operational transition risk | **Medium–High** | التصحيح بالعكس بدل التعديل يغيّر عادات المستخدمين ويحتاج تدريباً |
| Incremental rollout feasibility | **Medium** | Posting Service يسبق كل وحدة أعمال ⇒ لا تجزئة أفقية، والتجزئة الرأسية ممكنة |
| Prototype reuse | **High** لمنطق المطابقة · **Low** لسلطة الدفتر | `buildJournalCore()` تُعاد استخدامها كخدمة مطابقة؛ ودورها كمصدر حقيقة يسقط |

**الأدلة العددية المصحَّحة المستنَد إليها:** 20 مصدر RECALC/MIXED · 12 تبعية مباشرة لـ`unitCost()` · 14 ضمن السلسلة الكاملة · **18 من 34 ثابتاً مخروقاً** بعد إعادة التصنيف وإضافة AINV-33/34.

**Required ADRs:** الخمسة في §7 — و`ACC-A` هو التجسيد المباشر لهذا القرار.

---

## ADR Dependency Map

```
ACC-A — Immutable Posting Architecture        [الأساس — لا تبعية]
│
├── ACC-B — Snapshot and Valuation Inputs     [يعتمد: A]
│   │
│   └── ACC-C — Reversal Instead of Delete    [يعتمد: A + B]
│
├── ACC-D — Centralized Period Lock           [يعتمد: A]
│   │
│   └── ACC-E — Control Account Settlement    [يعتمد: A + D]  🔴 Blocked
│
└──────────────────────────────────────────►  ACC-E
        (ACC-E يعتمد على ACC-A مباشرةً أيضاً، لا عبر ACC-D فقط)
```

| ADR | يعتمد على | الجاهزية |
|---|---|---|
| ACC-A | — | ✅ جاهز للكتابة الكاملة |
| ACC-B | A | ✅ جاهز (يستحسن حسم OQ-3.A و OQ-3.D أولاً) |
| ACC-C | **A + B** | ✅ محسوم مبدئياً — بعد A و B |
| ACC-D | A | ✅ محسوم مبدئياً — بعد A |
| ACC-E | **A + D** | 🔴 **Blocked** — استنفاد 2017 |

### ACC-A — Immutable Posting Architecture
- **Scope:** الدفتر سجل مُخزَّن غير قابل للتعديل؛ الحدث يُطلَق مرة؛ مفتاح تفرّد لكل ترحيل؛ Read Models مشتقّة من الدفتر لا من المستندات.
- **Dependency:** لا شيء — **الأساس**.
- **Evidence:** §6 بالكامل · AINV-02/04/05/13/29.
- **OQs resolved:** OQ-3.K · OQ-4.D · OQ-4.M *(الشق المعماري)*.
- **لماذا ADR منفصل:** قرار معماري جذري يحكم كل ما بعده؛ خلطه بغيره يجعله غير قابل للمراجعة.
- **Consequences:** إعادة بناء طبقة الترحيل · `buildJournalCore` ⇒ خدمة مطابقة · جدول `JournalEntry`/`JournalLine` بلا UPDATE/DELETE.
- **الحالة:** ✅ **محسوم — جاهز للكتابة الكاملة.**

### ACC-B — Snapshot and Valuation Inputs
- **Scope:** أي قيمة تدخل الدفتر تُجمَّد لحظة الترحيل: التكلفة · سعر الصرف · وضع الإنتاج · أساس المخصص.
- **Dependency:** **ACC-A** (لا معنى للتجميد بلا قيد ثابت).
- **Evidence:** AINV-03/10/11/18 · §6.2 · S4 §4.4.3.
- **OQs resolved:** OQ-3.A · OQ-3.D · OQ-3.H · OQ-4.E · OQ-4.L.
- **لماذا منفصل:** يمسّ Pricing وInventory وAccounting ⇒ **Cross-Module ADR**.
- **Consequences:** كل سطر يحمل `UnitCostAtPosting` و`ExchangeRate` و`AmountCurrency`.
- **الحالة:** ✅ **محسوم — جاهز للكتابة.**

### ACC-C — Reversal Instead of Delete
- **Scope:** لا حذف لمستند مُرحَّل؛ التصحيح قيد عكسي بمرجع وسبب.
  > **بند إلزامي:** القيد العكسي يستخدم **القيم المُجمَّدة من القيد الأصلي** — بما يشمل المبلغ وسعر الصرف والتكلفة وأساس التقييم — **ولا يعيد التقييم باستخدام قيم تاريخ العكس**.
- **Dependency:** **ACC-A و ACC-B معاً.** ACC-A لأن الدفتر المُشتقّ لا يُحذف منه شيء أصلاً؛ و**ACC-B** لأن العكس بالقيم المُجمَّدة يستلزم وجود تلك القيم مخزَّنة على القيد الأصلي.
- **Evidence:** AINV-30/31/32 · S3 §3.10.3 · KI-008.
- **OQs resolved:** جزئياً OQ-4.I *(التحفّظ الباقي)*.
- **لماذا منفصل:** يمسّ كل وحدة لها مستندات، لا المحاسبة وحدها.
- **Consequences:** Soft-delete · `ReversalOfId` · `ReasonCode` · مراجعة كل `delXxx`.
- **الحالة:** ✅ **محسوم مبدئياً** — يُكتب بعد ACC-A.

### ACC-D — Centralized Period Lock
- **Scope:** فحص القفل في نقطة الترحيل المركزية؛ رفض السجل بلا تاريخ؛ إعادة الفتح بصلاحية وسبب وأثر.
- **Dependency:** **ACC-A** (نقطة ترحيل مركزية شرط وجودها).
- **Evidence:** AINV-26/27/28 · C-06 · C-07.
- **OQs resolved:** OQ-3.I · OQ-3.E.
- **لماذا منفصل:** ضابط رقابي بأثر تشغيلي مباشر (فترات · صلاحيات · إعادة فتح).
- **Consequences:** توحيد مسارات القفل الأربعة · حارس داخل الدالة لا في الـHTML · فترات محاسبية مستقلة بدل تاريخ واحد.
- **الحالة:** ✅ **محسوم مبدئياً** — يُكتب بعد ACC-A.

### ACC-E — Control Account Settlement and Month Close
- **Scope:** كل حساب مراقبة له إجراء تصفية إلزامي؛ المخصصات فروق تسوية؛ رصيد غير مفسَّر يمنع الإقفال.
- **Dependency:** **ACC-D** (المانع يحتاج نقطة قفل موحَّدة) · **ACC-A**.
- **Evidence:** AINV-20/21/22/23/24 · C-04 · C-05 · **C-07** · OQ-4.K.
- **OQs resolved:** OQ-3.F · OQ-4.F · OQ-4.K.
- **لماذا منفصل:** يجمع سياسة محاسبية (نمط المخصص) مع ضابط تشغيلي (مانع الإقفال) — نطاق متماسك مستقل.
- **Consequences:** لوحة حسابات مراقبة (1260 · 2016 · 1265 · 2105 · 1205 · 1190 · 2017) · `closeReadiness()` موسَّعة · إلغاء `Monthly_Close_1260_Temporary_Control.md` عند التطبيق.
- **Dependency (صريحة):** **ACC-A** (نقطة ترحيل ودفتر ثابت) **و ACC-D** (نقطة قفل موحَّدة يُعلَّق عليها المانع).
- **الحالة:** 🔴 **`Blocked pending evidence on settlement/consumption of account 2017`**
  > 2017 حساب مراقبة يقع ضمن نطاق هذا الـADR، ولم يُفحَص إن كان له مسار استنفاد. كتابة ضابط تصفية لحساب مجهول السلوك غير جائزة.
  > **لا يؤثر هذا الحجب على ACC-A ولا ACC-B ولا ACC-C ولا ACC-D.**

> **قيد إلزامي:** لا يُصدَر `ACC-C` ولا `ACC-D` قبل `ACC-A`. و`ACC-C` بعد `ACC-B`. و`ACC-E` بعد `ACC-A` و`ACC-D` معاً، **ومحجوب حتى حسم استنفاد 2017**.

---

## Cross-references

- `docs/02_Governance/ADR/ADR-020.md` — **ACC-A** · `Proposed` — التجسيد المباشر لقرار هذا الـRFC
- `docs/03_Business_Logic/Accounting/Accounting.md` — Business Invariants (AINV-01→34) · Journal Entry Source Catalog (58 مصدراً)
- `docs/02_Governance/Known_Issues.md` — KI-010 → KI-024 · KI-008 · KI-009
- `docs/03_Business_Logic/Pricing/Pricing.md` — PINV-12 *(القرينة أُبطلت — انظر C-01 أعلاه؛ يحتاج patch مستقل)*
- `docs/03_Business_Logic/Inventory_Production/Inventory_Production.md` — تكلفة الوحدة ⇒ ACC-B

---

## مرشّحات أخرى

_(لا شيء بعد.)_
