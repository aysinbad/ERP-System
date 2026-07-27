# Accounting — Product Owner Decisions

## Document Information
```
Document Name:  Accounting Product Owner Decisions
Version:        1.0.0
Session:        Product Owner Decision Session
Status:         READY FOR PRODUCT OWNER REVIEW
Classification: Governance — decision package (does NOT modify any ADR/RFC/KI)
Owner:          Solution Architecture Team (preparer) · Product Owner (decider)
Date:           2026-07-25
Source of truth: current repository (Accounting_RFC.md §OQ · Accounting_Session6_1_Remaining_Evidence_Findings.md §5·§6.3·§9 · Known_Issues.md KI-013·KI-017·KI-030 · Decisions_Log.md)
```

This document is a Product Owner decision package only.
It does not modify business rules.
It does not supersede ADRs.
It does not supersede RFCs.
Approved decisions must later be reflected in:
- Accounting.md
- Accounting_RFC.md
- ADRs
- Accounting_Implementation_Guide.md

> **نطاق صارم:** يحتوي هذا الملف **قرارات Product Owner المفتوحة فقط** — **OQ-6.A · OQ-6.D · OQ-6.H**. لا يشمل قرارات Architecture (OQ-6.F · OQ-6.G مستبعَدة). **لم يُخترَع أي قرار PO إضافي.** لا دليل جديد · لا فحص Prototype · لا تعديل لأي وثيقة قائمة. **التوصيات** مُشتقّة من الوثائق القائمة. خيارات **OQ-6.A** و**OQ-6.D** مُشتقّة من الوثائق القائمة. خيار **OQ-6.H (A)** مُشتقّ من الوثائق القائمة. أمّا خيارا **OQ-6.H (B) و(C)** فهما نموذجان منهجيان استرشاديان لمخصص EAS 48 صاغهما المُعِدّ **فقط لدعم مقارنة Product Owner** — وليسا قرارين قائمين في الريبو، ولا يعدّلان أي قاعدة أعمال أو ADR أو RFC أو KI.
>
> **حقل «Final decision» و«Decision date» متروكان فارغين عمداً** — يملؤهما Product Owner.

**عدد القرارات: 3.**

---

## OQ-6.A — مسار استنفاد/تسوية الحساب 2017 (عمولات المبيعات المستحقة)

**1. Decision ID:** OQ-6.A

**2. Current status:** 🔴 Open — `Accounting Policy / Design Decision` · Product Owner.

**3. Business context:**
الحساب **2017 (عمولات مبيعات مستحقة)** يتراكم كالتزام عند استحقاق عمولة على المُحصَّل، لكن **لا يوجد ورك-فلو مُصمَّم لدفع/تسوية هذا الالتزام** — أي لا طريقة منضبطة لجعل 2017 مديناً عند صرف العمولة للمندوب. غياب المسار يعني التزاماً ينمو بلا استنفاد، وهو ما يحجب إصدار ضابط المراقبة `ACC-E`.

**4. Current prototype behaviour (كما هو موثَّق — S6.1 §5):**
`accrueCommission()` يُنشئ الاستحقاق **5440 ⇐ 2017** ويحفظ `basis`/`pct`/`from`/`to` (نموذج Snapshot سليم). **لا يوجد أي مسار يجعل 2017 مديناً**؛ القيد اليدوي العام قادر تقنياً لكنه **ليس مساراً مُصمَّماً** (بلا ربط بالاستحقاق ولا ضبط مبلغ ولا منع ازدواج).

**5. Current documentation position:**
`Open` · **يحجب `ACC-E` وحده** ولا يؤثر على ADR-020 (ACC-A) ولا ACC-B/C/D. `ACC-E` **غير مُنشأ**؛ فجوة الدليل مُغلَقة (`No dedicated settlement path found after scoped search`) والباقي قرار Product Owner (RFC §235 · §261 · Decisions_Log §58). مرتبط بـ **KI-017** (لا حارس تداخل فترات استحقاق + لا مسار استنفاد).

**6. Available options:**

**الخيار A — كيان دفع عمولات مستقل (`commissionPayments`) بربط إلزامي بـ`CM-XXXX`:**
- *Business impact:* مسار واضح مخصص لصرف العمولات؛ تقارير عمولة نظيفة لكل مندوب.
- *Accounting impact:* دفع العمولة = **2017 مدين / خزينة دائن** مربوطاً بالاستحقاق الأصلي؛ يصفو الالتزام بدقة.
- *Implementation impact:* كيان + شاشة + Posting Rule جديدة؛ الأعلى بناءً بين الخيارات.
- *Migration impact:* ترحيل أرصدة 2017 القائمة وربطها بالاستحقاقات؛ يحتاج seed للربط التاريخي.
- *Risks:* تعقيد إضافي؛ إن لم يُفرَض الربط الإلزامي يعود خطر الازدواج.

**الخيار B — توسعة `supplierPayments` لتشمل الموظفين/الوكلاء:**
- *Business impact:* إعادة استخدام مسار قائم؛ أسرع للسوق.
- *Accounting impact:* 2017 مدين عبر مسار السداد نفسه؛ لكن يخلط ذمم الموردين بعمولات الموظفين في كيان واحد.
- *Implementation impact:* متوسط — توسعة كيان قائم لا بناء جديد.
- *Migration impact:* أقل من A؛ لكن يرث فجوات `supplierPayments` (لا حارس ازدواج/سقف — KI-026) ما لم تُعالَج.
- *Risks:* تلوّث دلالي (مورد ≠ مندوب)؛ صعوبة تقارير العمولة المنفصلة؛ يورّث ثغرات السداد.

**الخيار C — قيد يدوي مُوجَّه بربط إلزامي + ضبط مبلغ + منع ازدواج + أثر تدقيقي:**
- *Business impact:* أخف حلاً؛ يعتمد على انضباط المستخدم.
- *Accounting impact:* 2017 مدين عبر قيد موجَّه مربوط بالاستحقاق؛ صحيح إن فُرضت القيود.
- *Implementation impact:* الأقل — قيد يدوي مقيَّد بقواعد تحقق.
- *Migration impact:* الأدنى.
- *Risks:* الأضعف ضماناً؛ القيد اليدوي يظل عرضة للخطأ ما لم تُفرَض كل القيود على الخادم.

**7. Open dependencies:**
`ACC-E` مبني فوق **ACC-A + ACC-D** معاً؛ هذا القرار **يحجب ACC-E فقط**. مرتبط بـ **KI-017**. معيار فكّ الحجب (من RFC): أي خيار يجب أن يوفّر — مصدر كيان · ربط بالاستحقاق · ضبط مبلغ · منع ازدواج · أثر تدقيقي · قفل فترة.

**8. Recommendation:**
**الخيار A** — يستوفي معايير فكّ الحجب الستة كاملةً وهو الاتجاه المُلمَّح إليه في `Accounting_Session6_1…` §786 (كيان دفع عمولات مستقل بربط إلزامي بـ`CM-XXXX`). الخيار C مقبول كحل انتقالي منخفض التكلفة إن فُرضت قيوده على الخادم.

**9. Final decision:** _______________________ (يملؤه Product Owner)

**Implementation Reference:** ______________

**10. Decision owner:** Product Owner

**11. Decision date:** ______________

**12. Blocking scope:** **Blocks ACC-E only**

---

## OQ-6.D — اعتماد مسار الرواتب الرسمي ومعالجة RC-002 المتكرر

**1. Decision ID:** OQ-6.D

**2. Current status:** 🔴 Open — `Accounting Policy Decision` · Product Owner.

**3. Business context:**
يوجد **مساران فعّالان للرواتب** بلا تنسيق، ما يسمح بإثبات الرواتب **مرتين** لنفس الشهر ويشوّه عرض المصروفات (EAS 1). القرار المطلوب: **أي مسار يصبح Canonical**، وكيف يُعالَج المسار المتكرر `RC-002` لمنع الازدواج.

**4. Current prototype behaviour (موثَّق — KI-030 · runtime PAY-04):**
- **المسار المتكرر:** `DB.recurring → genRecurring() → DB.expenses (JS-19)` بـ`allocType:'overhead'` ⇒ يقع في **5400** بلا فصل الرواتب/التأمينات/الضرائب.
- **المسار المتخصص:** `postPayroll() → DB.manualJournals (PAYROLL-{month})` ⇒ **5410 · 5420 · 2120 · 2130 · 2140**.
- **لا Cross-Guard** بين المسارين ⇒ تشغيلهما للشهر نفسه يضاعف الرواتب (مؤكَّد تشغيلياً PAY-04).

**5. Current documentation position:**
`Open` · **KI-030 🔴 Critical**. **تصحيح المراجعة المستقلة:** `postPayroll()` موجود فعلاً؛ السؤال لم يعد «كيف نبني قيد الرواتب من الصفر» بل **أي مسار Canonical وكيف نمنع الازدواج مع RC-002**. الهدف المُوثَّق (KI-030): مسار رواتب إنتاجي واحد **أو** مفتاح تفرّد مشترك على الشهر/الدورة، مع **تعطيل أو إعادة تصنيف RC-002**. Target ADR: **ACC-A** (تفرّد) · **ACC-B** (مدخلات التصنيف).

**6. Available options:**

**الخيار A — اعتماد `postPayroll` كمسار Canonical وإلغاء RC-002 كمسار رواتب (إعادة تصنيفه لغير-راتب):**
- *Business impact:* مصدر راتب واحد واضح؛ يزيل الالتباس التشغيلي.
- *Accounting impact:* الرواتب تُرحَّل دائماً إلى 5410/5420 والتزاماتها 2120/2130/2140 — **متوافق مع عرض EAS 1**؛ يزول تلوّث 5400 بالرواتب.
- *Implementation impact:* متوسط — اعتماد `postPayroll` + منع RC-002 من التصنيف كراتب + مفتاح تفرّد شهري (ACC-A).
- *Migration impact:* مراجعة أشهر سابقة أُثبتت عبر RC-002 وإعادة تصنيفها/عكسها (ACC-C) عند اللزوم.
- *Risks:* لو استُخدم RC-002 لأغراض أخرى مشروعة، إلغاؤه كراتب يحتاج بديلاً لتلك الحالات.

**الخيار B — إبقاء كلا المسارين مع مفتاح تفرّد مشترك (Cross-Guard) شهري + إعادة تصنيف RC-002 خارج overhead:**
- *Business impact:* مرونة أكبر؛ لكن يبقى مساران للصيانة.
- *Accounting impact:* يمنع الازدواج بمفتاح تفرّد؛ لكن ما لم يُفصَل RC-002 يظل عرض 5400 مخلوطاً حتى تُعاد تصنيفه.
- *Implementation impact:* أعلى — Cross-Guard مشترك عبر كيانين مختلفين (`DB.recurring` و`manualJournals`) + إعادة تصنيف.
- *Migration impact:* مماثل لـA مع تعقيد التزامن بين المسارين.
- *Risks:* تعقيد صيانة دائم؛ مصدر حقيقة مزدوج يظل خطراً بنيوياً (نمط KI-031).

**7. Open dependencies:**
**ACC-A** (مفتاح تفرّد الترحيل) · **ACC-B** (تجميد مدخلات التصنيف) · **KI-030** · عرض **EAS 1**. لا يعتمد على ACC-E.

**8. Recommendation:**
**الخيار A** — مصدر راتب واحد Canonical (`postPayroll`) + مفتاح تفرّد شهري (ACC-A) + إعادة تصنيف RC-002 بحيث لا يمر كراتب overhead. يحسم الازدواج من الجذر ويحقّق عرض EAS 1 بأقل دَين بنيوي، متسقاً مع اتجاه RFC OQ-6.D («اعتماد مسار الرواتب الرسمي وإلغاء/تعطيل RC-002 كمسار موازٍ»).

**9. Final decision:** _______________________ (يملؤه Product Owner)

**Implementation Reference:** ______________

**10. Decision owner:** Product Owner

**11. Decision date:** ______________

**12. Blocking scope:** **Does not block implementation**
> *(توضيح نطاق: لا يحجب Phase 1 أو Phase 2 (أساس محرك الترحيل/المطابقة)؛ لكنه **مطلوب قبل تثبيت Posting Rule للرواتب** — يقع في بناء وحدة الرواتب لاحقاً. مفتاح التفرّد العام (Phase 1 · مهمة 7.5) يُبنى بمعزل عن هذا القرار.)*

---

## OQ-6.H — منهجية اشتقاق نسبة مخصص المطالبات (`suggestedClaimsRate`)

**1. Decision ID:** OQ-6.H

**2. Current status:** 🔴 Open — `Accounting Policy Decision` · Product Owner.

**3. Business context:**
مخصص المطالبات (EAS 48) يُبنى على **نسبة** تُطبَّق على وعاء الإيراد المعرَّض. الأساس الذي تُشتقّ منه هذه النسبة **غير محدَّد سياسةً** — ما يترك تقدير المخصص بلا منهجية معتمَدة قابلة للتدقيق.

**4. Current prototype behaviour (موثَّق — S4.6 DO-6):**
`suggestedClaimsRate()` **تشتقّ نسبة من تاريخ المطالبات**، لكن **أساس الاشتقاق لم يُفحَص** (`Not examined — S4.6 DO-6`). قيد المخصص يقع **5300 ⇐ 2105** (JS-05/JS-43). *(لا فحص Prototype إضافي في هذه الجلسة — يُنقَل ما وثّقه الريبو فقط.)*

**5. Current documentation position:**
`Open` · مرتبط بـ **KI-013 🔴 Critical** — وعاء مخصص المطالبات **تراكمي في اليومية ومُفلتَر على الشاشة** (`Displayed Basis ≠ Posted Basis`، ويشمل proforma والمؤجَّل خطأً). `DO-6` حُوِّل إلى **OQ-6.H** (S6.1 §9). Target: `Accounting.md` (سياسة).

**6. Available options:**

**الخيار A — منهجية النسبة التاريخية مُشكْلنة (نسبة مطالبات/إيراد على نافذة زمنية معرَّفة) مع تجميد الأساس على القيد:**
- *Business impact:* تقدير مبني على بيانات فعلية؛ يعكس سلوك المطالبات الحقيقي.
- *Accounting impact:* يتطلب تعريف **وعاء المخصص الموحَّد** (يستبعد proforma والمؤجَّل — يحل KI-013)؛ النسبة والأساس **يُجمَّدان على القيد** (ACC-B البند 7).
- *Implementation impact:* متوسط — تعريف النافذة والوعاء + تجميد `ProvisionBase`/`pct`.
- *Migration impact:* إعادة احتساب المخصصات التاريخية على الوعاء الموحَّد.
- *Risks:* حساسية للنافذة المختارة؛ تذبذب النسبة بين الفترات إن لم تُنعَّم.

**الخيار B — نسبة سياسة ثابتة تحدّدها الإدارة/PO (متوافقة EAS 48) تُراجَع دورياً:**
- *Business impact:* أبسط وأثبت؛ سهل الشرح والتدقيق.
- *Accounting impact:* نسبة واحدة على الوعاء الموحَّد؛ تُجمَّد على القيد (ACC-B).
- *Implementation impact:* الأدنى — قيمة سياسة قابلة للإعداد + وعاء موحَّد.
- *Migration impact:* الأدنى.
- *Risks:* أقل حساسية للواقع؛ قد لا يعكس تغيّر مخاطر المطالبات ما لم تُراجَع.

**الخيار C — نسبة قائمة على المخاطر لكل عقد/incoterm:**
- *Business impact:* الأدق لمحفظة تصدير متنوعة المخاطر.
- *Accounting impact:* مخصص أدق لكل شحنة؛ يتطلب تصنيف مخاطر لكل عقد.
- *Implementation impact:* الأعلى — نموذج مخاطر + بيانات تصنيف لكل عقد.
- *Migration impact:* الأعلى — يحتاج تصنيف المخاطر التاريخي.
- *Risks:* تعقيد بيانات وصيانة؛ قد يكون فوق حاجة المرحلة الحالية.

**7. Open dependencies:**
**KI-013** (توحيد وعاء المخصص — استبعاد proforma/المؤجَّل، وتطابق الدفتر مع الشاشة) · **ACC-B** (تجميد `ProvisionBase`/`pct` على القيد — ADR-021 البند 7). لا يعتمد على ACC-E. يمسّ JS-05 / JS-43 (EAS 48).

**8. Recommendation:**
**الخيار A** مع تجميد الأساس (ACC-B) وتوحيد الوعاء (KI-013) كحل يعكس الواقع ويظل قابلاً للتدقيق؛ **الخيار B** بديل منخفض التكلفة إن فضّل PO البساطة والثبات. أياً كان الخيار، **توحيد الوعاء (KI-013) شرط مسبق** لأي منهجية.

**9. Final decision:** _______________________ (يملؤه Product Owner)

**Implementation Reference:** ______________

**10. Decision owner:** Product Owner

**11. Decision date:** ______________

**12. Blocking scope:** **Does not block implementation**
> *(توضيح نطاق: لا يحجب Phase 1/2؛ مطلوب قبل تثبيت Posting Rule لمخصص المطالبات (JS-05/JS-43) في بناء وحدة نهاية الفترة.)*

---

## ملخّص القرارات

| Decision ID | العنوان | Owner | Status | Blocking scope |
|---|---|---|---|---|
| OQ-6.A | مسار تسوية الحساب 2017 (عمولات) | Product Owner | 🔴 Open | Blocks ACC-E only |
| OQ-6.D | مسار الرواتب الرسمي + RC-002 | Product Owner | 🔴 Open | Does not block implementation |
| OQ-6.H | منهجية نسبة مخصص المطالبات | Product Owner | 🔴 Open | Does not block implementation |

## Decision Status Workflow

```
Draft
↓
Product Owner Review
↓
Approved
↓
Architecture Update
↓
Documentation Update
↓
Implementation
```

**Final status: READY FOR PRODUCT OWNER REVIEW**
