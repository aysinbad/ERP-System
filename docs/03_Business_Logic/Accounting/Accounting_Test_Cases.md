# Accounting Engine — Test Cases

## Document Information
```
Document Name:  Accounting Engine Test Cases
Version:        0.8.0 (Runtime verification corrections)
Status:         Draft — In Progress
Classification: Source of Truth
Owner:          Solution Architecture Team
Approved-by:    —
Approved-date:  —
Last-Updated:   2026-07-25 (Runtime verification corrections — `Prototype_Runtime_Verification_Report.md`; totals recalculated 55→62)
```

> **قاعدة العرض (`AGENTS.md` قاعدة 5):**
> - **`Expected Current Result: FAIL`** — سلوك الـPrototype الآن (خلل موثَّق).
> - **`Expected Current Result: PARTIAL`** — مطبَّق جزئياً.
> - **`Current: PASS`** — سلوك سليم حالياً.
> - كل FAIL و PARTIAL يحمل **`Target Result After Remediation: PASS`** مع الـADR المعالج.
>
> مصدر كل سيناريو: `Accounting.md` (AINV) + `Accounting_Session6_1_Remaining_Evidence_Findings.md`.

---

## 1. تغطية الثوابت (AINV-01 → AINV-34)


### TC-AINV-01 — توازن كل قيد مُرحَّل
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-01`
- **Then** يجب أن يتحقّق `AINV-01: توازن كل قيد مُرحَّل`
- **Current: PASS** ✅

### TC-AINV-02 — كل قيد قابل للتتبّع لمصدره
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-02`
- **Then** يجب أن يتحقّق `AINV-02: كل قيد قابل للتتبّع لمصدره`
- **Expected Current Result: PARTIAL** 🟡
- **Target Result After Remediation: PASS**

### TC-AINV-03 — قيمة القيد مُجمَّدة لحظة الترحيل
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-03`
- **Then** يجب أن يتحقّق `AINV-03: قيمة القيد مُجمَّدة لحظة الترحيل`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-04 — مفتاح تفرّد لكل ترحيل (لا ازدواج)
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-04`
- **Then** يجب أن يتحقّق `AINV-04: مفتاح تفرّد لكل ترحيل (لا ازدواج)`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-05 — اقتناء الأصل يسبق إهلاكه
- **Given** أصل ثابت `!opening` يُنشأ ثم يُبنى الأستاذ
- **When** يُطبَّق السيناريو الذي يختبر `AINV-05`
- **Then** `buildJournalCore` يشتقّ قيد اقتناء **مدين 1300 / دائن البنك** قبل بدء الإهلاك (runtime FA-01)
- **Current: PASS** ✅ *(scoped: مُثبَت تشغيلياً للأصول `!opening`؛ الادعاء الثابت السابق «الإهلاك بلا اقتناء» مدحوض — `Prototype_Runtime_Verification_Report.md` §4 FA-01)*

### TC-AINV-06 — القيد اليدوي يتطلب اعتماداً وتوازناً
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-06`
- **Then** يجب أن يتحقّق `AINV-06: القيد اليدوي يتطلب اعتماداً وتوازناً`
- **Current: PASS** ✅

### TC-AINV-07 — الإيراد يُعترف به بنقل المخاطر
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-07`
- **Then** يجب أن يتحقّق `AINV-07: الإيراد يُعترف به بنقل المخاطر`
- **Expected Current Result: PARTIAL** 🟡
- **Target Result After Remediation: PASS**

### TC-AINV-08 — تأجيل الإيراد للوجهة (EAS 48)
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-08`
- **Then** يجب أن يتحقّق `AINV-08: تأجيل الإيراد للوجهة (EAS 48)`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-09 — التكلفة تُقابل الإيراد في فترته
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-09`
- **Then** يجب أن يتحقّق `AINV-09: التكلفة تُقابل الإيراد في فترته`
- **Expected Current Result: PARTIAL** 🟡
- **Target Result After Remediation: PASS**

### TC-AINV-10 — فرق العملة يُقيَّم بسعر الحدث لا الحيّ
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-10`
- **Then** يجب أن يتحقّق `AINV-10: فرق العملة يُقيَّم بسعر الحدث لا الحيّ`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-11 — فرق العملة يُعترف به على كل تسوية
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-11`
- **Then** يجب أن يتحقّق `AINV-11: فرق العملة يُعترف به على كل تسوية`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-12 — العملة الأجنبية تُقيَّد بسعر مثبَّت
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-12`
- **Then** يجب أن يتحقّق `AINV-12: العملة الأجنبية تُقيَّد بسعر مثبَّت`
- **Expected Current Result: PARTIAL** 🟡
- **Target Result After Remediation: PASS**

### TC-AINV-13 — وعاء المخصص واحد بين الدفتر والشاشة
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-13`
- **Then** يجب أن يتحقّق `AINV-13: وعاء المخصص واحد بين الدفتر والشاشة`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-14 — المخصص يُبنى على أساس مُجمَّد
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-14`
- **Then** يجب أن يتحقّق `AINV-14: المخصص يُبنى على أساس مُجمَّد`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-15 — لا فاتورة شراء مزدوجة على GRN
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-15`
- **Then** يجب أن يتحقّق `AINV-15: لا فاتورة شراء مزدوجة على GRN`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-16 — السجل بلا تاريخ لا يُرحَّل
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-16`
- **Then** يجب أن يتحقّق `AINV-16: السجل بلا تاريخ لا يُرحَّل`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-17 — لا تداخل في فترات الاستحقاق
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-17`
- **Then** يجب أن يتحقّق `AINV-17: لا تداخل في فترات الاستحقاق`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-18 — حركة المخزون منفصلة في الأستاذ
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-18`
- **Then** يجب أن يتحقّق `AINV-18: حركة المخزون منفصلة في الأستاذ`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-19 — VAT المدخلات تُعالَج باتّساق الإقرار
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-19`
- **Then** يجب أن يتحقّق `AINV-19: VAT المدخلات تُعالَج باتّساق الإقرار`
- **Expected Current Result: PARTIAL** 🟡
- **Target Result After Remediation: PASS**

### TC-AINV-20 — WHT تُحتجَز وتُورَّد
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-20`
- **Then** يجب أن يتحقّق `AINV-20: WHT تُحتجَز وتُورَّد`
- **Expected Current Result: PARTIAL** 🟡
- **Target Result After Remediation: PASS**

### TC-AINV-21 — لا تحميل مزدوج على المصروف
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-21`
- **Then** يجب أن يتحقّق `AINV-21: لا تحميل مزدوج على المصروف`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-22 — حساب المراقبة يُقفَل دورياً
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-22`
- **Then** يجب أن يتحقّق `AINV-22: حساب المراقبة يُقفَل دورياً`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-23 — NRV يشمل كل المكوّنات
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-23`
- **Then** يجب أن يتحقّق `AINV-23: NRV يشمل كل المكوّنات`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-24 — حساب المراقبة 1260 يُقفَل عند الإقفال
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-24`
- **Then** يجب أن يتحقّق `AINV-24: حساب المراقبة 1260 يُقفَل عند الإقفال`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-25 — لا التزام عمولة مزدوج
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-25`
- **Then** يجب أن يتحقّق `AINV-25: لا التزام عمولة مزدوج`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-26 — كل كتابة تمرّ بقفل الفترة
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-26`
- **Then** يجب أن يتحقّق `AINV-26: كل كتابة تمرّ بقفل الفترة`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-27 — السجل بلا تاريخ لا يتجاوز القفل
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-27`
- **Then** يجب أن يتحقّق `AINV-27: السجل بلا تاريخ لا يتجاوز القفل`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-28 — فتح الفترة بصلاحية صريحة
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-28`
- **Then** يجب أن يتحقّق `AINV-28: فتح الفترة بصلاحية صريحة`
- **Expected Current Result: PARTIAL** 🟡
- **Target Result After Remediation: PASS**

### TC-AINV-29 — ميزان المراجعة التاريخي قابل لإعادة الإنتاج
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-29`
- **Then** يجب أن يتحقّق `AINV-29: ميزان المراجعة التاريخي قابل لإعادة الإنتاج`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-30 — المُرحَّل لا يُحذَف — يُعكَس
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-30`
- **Then** يجب أن يتحقّق `AINV-30: المُرحَّل لا يُحذَف — يُعكَس`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-31 — عكس الإيراد يعكس التكلفة
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-31`
- **Then** يجب أن يتحقّق `AINV-31: عكس الإيراد يعكس التكلفة`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-32 — الإقفال يُحوِّل النتيجة للأرباح المحتجزة
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-32`
- **Then** يجب أن يتحقّق `AINV-32: الإقفال يُحوِّل النتيجة للأرباح المحتجزة`
- **Expected Current Result: PARTIAL** 🟡
- **Target Result After Remediation: PASS**

### TC-AINV-33 — الأستاذ يطابق الدفاتر المساعدة
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-33`
- **Then** يجب أن يتحقّق `AINV-33: الأستاذ يطابق الدفاتر المساعدة`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

### TC-AINV-34 — تاريخ القيد من حدث اقتصادي مؤرَّخ
- **Given** نظام محاسبي قائم بمصادر قيد مؤكَّدة
- **When** يُطبَّق السيناريو الذي يختبر `AINV-34`
- **Then** يجب أن يتحقّق `AINV-34: تاريخ القيد من حدث اقتصادي مؤرَّخ`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS**

> **تغطية AINV:** PASS 3 · PARTIAL 8 · FAIL 23 = 34.
> *(تحديث تشغيلي 2026-07-25: TC-AINV-05 من FAIL→PASS بعد دليل اقتناء الأصل الحيّ FA-01.)*

---

## 2. سيناريوهات الأخلاق المكتشَفة (جلسة 6.1)


### TC-D01 — supplier payment without invoice link
- **Ref:** KI-025 · AINV-11
- **Given** سند سداد مورد بلا ربط وبعملة أجنبية
- **When** يُرحَّل
- **Then** `invRate=payRate` ⇒ `fxDiff=0` ⇒ لا اعتراف بالفرق
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — تجميد السعرين (ACC-B)

### TC-D02 — duplicate supplier payment
- **Ref:** KI-026 · AINV-04
- **Given** سندا سداد على نفس `purchaseId`
- **When** يُرحَّل الثاني
- **Then** لا حارس ازدواج ⇒ سداد مزدوج
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — مفتاح تفرّد (ACC-A/D)

### TC-D03 — supplier overpayment
- **Ref:** KI-026 · AINV-04
- **Given** سند يتجاوز رصيد الفاتورة
- **When** يُرحَّل
- **Then** لا سقف ⇒ 2010 مدين
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — سقف برصيد الفاتورة (ACC-D)

### TC-D04A — supplier payment create/update in locked period
- **Ref:** KI-026 · AINV-26
- **Given** فترة مقفلة وسند سداد مورد جديد أو قائم
- **When** يُحفظ الإنشاء أو التعديل
- **Then** `isLocked(date)` يمنع الحفظ
- **Current: PASS** ✅

### TC-D04B — supplier payment delete control and audit
- **Ref:** KI-026 · AINV-30
- **Given** سند سداد مورد في فترة مقفلة
- **When** يحاول المستخدم حذفه
- **Then** `canDeleteDated(date)` يمنع الحذف، لكن المسار لا يسجل Audit للحذف وقد يعرض رسالة نجاح مضلِّلة
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — عكس/حذف مضبوط مع Audit ورسالة صحيحة (ACC-C/D)

### TC-D05 — credit-note edit/delete after period lock
- **Ref:** KI-027 · AINV-30
- **Given** إشعار خصم مُرحَّل في فترة مقفلة
- **When** يُحذَف/يُعدَّل
- **Then** حذف نهائي بلا عكس؛ تعديل يفحص السجل الجديد
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — قيد عكسي يفحص قفل الأصل (ACC-C/D)

### TC-D06 — duplicate credit notes
- **Ref:** KI-027 · AINV-04
- **Given** إشعارا خصم على نفس `purchaseId`
- **When** يُرحَّلان
- **Then** لا منع ازدواج ⇒ خصم مزدوج
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — منع ازدواج (ACC-A)

### TC-D07 — opening treasury balance vs ledger
- **Ref:** KI-028 (Refuted/Closed) · AINV-33 (scoped)
- **Given** خزينة برصيد افتتاحي (بيانات seed)
- **When** يُبنى الأستاذ
- **Then** قيد `ref:'OPENING'` يدين حسابات الخزينة (1010/1021/1022) ويقيّد **3001** = 2,403,839.5؛ **حساب 3001 حاضر**، والأستاذ والخزينة **متطابقان** لبيانات seed/runtime المختبَرة
- **Current: PASS** ✅ *(runtime OB-01 — الادعاء الثابت السابق «الرصيد الافتتاحي لا يدخل الأستاذ» مدحوض. مطابقة الأستاذ/الدفتر المساعد هنا **مقصورة على بيانات seed/runtime المختبَرة**؛ ثابت AINV-33 العام يبقى مفتوحاً لبقية الدفاتر المساعدة.)*

### TC-D19 — product opening inventory journal
- **Ref:** KI-028 (Refuted/Closed) · AINV-33 (scoped) — 🆕 مُضاف بالمراجعة التشغيلية
- **Given** كميات/تكاليف افتتاحية للأصناف (بيانات seed)
- **When** يُبنى الأستاذ
- **Then** المخزون الافتتاحي 1,371,339.5 يُقيَّد **مدين 1200** داخل نفس قيد `OPENING`؛ افتتاح مخزون الأستاذ يربط تقييم المخزون
- **Current: PASS** ✅ *(runtime OB-02 — مقصور على بيانات seed/runtime المختبَرة.)*

### TC-D08A — new non-opening fixed asset posts acquisition
- **Ref:** KI-029 (narrowed) · AINV-05 — 🆕 يحلّ محلّ TC-D08 السابق
- **Given** أصل ثابت `!opening` جديد (تكلفة 120,000)
- **When** يُبنى الأستاذ
- **Then** يُرحَّل قيد اقتناء **مدين 1300 = 120,000 / دائن البنك الافتراضي 1021 = 120,000** («شراء أصل ثابت»)
- **Current: PASS** ✅ *(runtime FA-01)*

### TC-D08B — acquisition funding cannot be selected as supplier/payable
- **Ref:** KI-029 (narrowed) — 🆕
- **Given** أصل يُشترى بالأجل
- **When** يُنشأ عبر `editAsset` (لا يضبط `treasury`)
- **Then** الطرف الدائن **يفترض البنك 1021 دائماً**؛ **لا يمكن** اختيار ذمة دائنة/2010 كمصدر تمويل
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — إتاحة اختيار تمويل بنك أو ذمة دائنة (ACC-A — بند التمويل فقط)

### TC-D08C — depreciation posts 5400 / 1310
- **Ref:** AINV-05 · JS-48 — 🆕
- **Given** أصل ثابت
- **When** يُرحَّل قسط الإهلاك الشهري
- **Then** **مدين 5400 / دائن 1310** لكل شهر (مثال FA-001 ≈ 6,666.67/شهر؛ مجمع 1310 = 39,583.33)
- **Current: PASS** ✅ *(runtime FA-02 — ملاحظة: يستخدم 5400 بدل حساب إهلاك مخصص — فرق عرض طفيف، لا خلل قيد)*

### TC-D09A — recurring payroll posts to 5400
- **Ref:** KI-030 · EAS 1
- **Given** RC-002 «رواتب الموظفين» بـ`allocType:'overhead'`
- **When** يُولَّد عبر `genRecurring()` ويُرحَّل كـJS-19
- **Then** يقع في 5400 بلا فصل الرواتب والتأمينات والضرائب
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — إزالة/إعادة تصنيف مسار الرواتب المتكرر

### TC-D09B — dedicated payroll posting uses payroll accounts
- **Ref:** KI-030
- **Given** بيانات Payroll محسوبة لشهر
- **When** يُشغَّل `postPayroll()`
- **Then** يستخدم 5410/5420/2120/2130/2140 ويمنع تكرار `PAYROLL-{month}` داخل مساره
- **Current: PASS** ✅

### TC-D09C — payroll cross-path duplicate
- **Ref:** KI-030 · AINV-04 · AINV-21
- **Given** تم توليد RC-002 للشهر نفسه ثم تشغيل `postPayroll()`
- **When** يُبنى الأستاذ
- **Then** لا يوجد Cross-Guard بين المسارين ⇒ يمكن إثبات الرواتب مرتين
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — مفتاح تفرّد مشترك ومسار رواتب واحد (ACC-A)

### TC-D10 — duplicate recurring engines and shadowed legacy editor
- **Ref:** KI-031
- **Given** وجود `DB.recurring` و`DB.recurringExp` وتعريفين لـ`window.editRecurring`
- **When** تُحمَّل الصفحة
- **Then** التعريف اللاحق الخاص بـ`DB.recurring` يظلّل التعريف الأقدم؛ شاشة `DB.recurring` تعمل، لكن محرر `DB.recurringExp` يصبح Dead Code غير قابل للوصول
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — محرك واحد أو أسماء وواجهات مستقلة

### TC-D11A — Engine A posts successfully
- **Ref:** KI-032 · JS-38
- **Given** أرصدة أجنبية مفتوحة وأسعار إقفال أعلى من الدفتري
- **When** يُشغَّل المحرك A `postRevaluation()`
- **Then** يُرحَّل قيد على حسابات الأطراف/البنك مقابل **4030** مع عكس تلقائي (`REV-001 net=27,621`)
- **Current: PASS** ✅ *(runtime FX-01 — end-to-end)*

### TC-D11B — Engine B screen and handler are reachable
- **Ref:** KI-032
- **Given** دور يملك الوصول (owner/`accounts:edit`)
- **When** تُفتح شاشة `fxReval` ويُستدعى `postFxRevaluation`
- **Then** الشاشة تُرسَم والـhandler يعمل ويفحص `can('accounts','edit')` و`isLocked()` (مُقيَّد صحيحاً على المحاسب)
- **Current: PASS** ✅ *(runtime FX-02/FX-04 — قابلية الوصول والحماية)*

### TC-D11C — Engine B produces zero exposure in the tested runtime
- **Ref:** KI-032 · AINV-10
- **Given** تحرُّك سعر 9% (EUR 55.2→60 · USD 50.5→53)
- **When** يُحسب `fxExposure()`
- **Then** يعيد **صفراً** لأن «المبني بالجنيه» يُعاد حسابه بالسعر الحالي ⇒ `diff≈0`؛ الـhandler يرفض الترحيل («لا يوجد فرق تقييم») — المحرك B **خامل فعلياً**
- **Expected Current Result: PARTIAL** 🟡 *(reachable-but-inert / data limitation؛ سطور 1265/3020 **NOT VERIFIABLE** end-to-end — runtime FX-02)*
- **Target Result After Remediation: PASS** — إصلاح أساس التعرُّض (سعر الفاتورة مقابل الحالي) أو إلغاء المحرك (OQ-6.F)

### TC-D11D — simultaneous double posting is NOT runtime-reproduced
- **Ref:** KI-032 · AINV-10
- **Given** كلا المحركين لهما مُصدِّر 4030/5500 بلا Cross-Guard (حقيقة كود)
- **When** يُشغَّل المحركان معاً على نفس الأرصدة
- **Then** المحرك A فقط يُرحِّل؛ المحرك B يُنتِج صفر تعرُّض ⇒ **الازدواج المزدوج لم يُعَد إنتاجه تشغيلياً**
- **Expected Current Result: PARTIAL** 🟡 *(latent code risk — خطر كامن غير مؤكَّد تشغيلياً؛ الادعاء الثابت السابق «الفرق يُثبَت مرتين فعلياً» مُنقَّح — runtime FX-03)*
- **Target Result After Remediation: PASS** — Cross-Guard/محرك واحد (OQ-6.F)

### TC-D11E — two-engine architecture remains unresolved
- **Ref:** KI-032 · OQ-6.F
- **Given** محرّكان متوازيان؛ المحرك A يتجاوز 1265 والمحرك B يستخدم 1265
- **When** يُبحَث عن قرار توحيد
- **Then** **لا قرار توحيد**؛ نموذجان متعارضان لإعادة التقييم
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — نظام واحد متسق (OQ-6.F — يبقى 🔴 Open)

### TC-D12 — revaluation automatic reversal
- **Ref:** JS-38b
- **Given** إعادة تقييم بـ`reverse=true`
- **When** يمرّ أول الشهر التالي
- **Then** قيد عكس تلقائي كامل بتاريخ `firstOfNextMonth`
- **Current: PASS** ✅ — سلوك سليم — IAS 21

### TC-D13 — treasury transfer rate recalculation
- **Ref:** JS-39 · AINV-10
- **Given** تحويل بين خزينتين بعملتين
- **When** يُبنى الأستاذ بعد تغيّر الأسعار
- **Then** `fxRate()` حيّ لكلا الطرفين ⇒ قيمة تاريخية تتغيّر
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — تجميد سعري الطرفين (ACC-B)

### TC-D14 — recurring generation in locked period
- **Ref:** KI-030 · AINV-26
- **Given** قيد متكرر في فترة مقفلة
- **When** يُطلَق التوليد يدوياً
- **Then** مسار التوليد اليدوي بلا `isLocked()`
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — قفل مركزي على التوليد (ACC-D)

### TC-D15 — closing through multiple paths
- **Ref:** KI-033 · AINV-26
- **Given** فترة جاهزة للإقفال
- **When** يُقفَل عبر `doClosing`/`doMonthEndClose`
- **Then** `doClosing()` يرحّل 3099⇄3020 مع Permission/Audit، بينما `doMonthEndClose()` ومسارات أخرى قد تقفل بلا نفس القيد والضوابط
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — خدمة قفل واحدة (ACC-D)

### TC-D16 — records without dates
- **Ref:** KI-016 · AINV-16 · AINV-27
- **Given** سجل بتاريخ فارغ
- **When** يُبنى الأستاذ ويُقفَل
- **Then** `inPeriod()` تُمرِّره ⇒ يتجاوز القفل
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — رفض السجل بلا تاريخ (ACC-D)

### TC-D17 — account 2017 accumulation
- **Ref:** KI-017 · AINV-25
- **Given** استحقاقات عمولة متتالية
- **When** تُرحَّل
- **Then** 2017 يتراكم أبدياً؛ فترات متداخلة تُضاعفه
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — حارس تداخل + استنفاد (ACC-E محجوب OQ-6.A)

### TC-D18 — no dedicated settlement workflow
- **Ref:** KI-017 · OQ-6.A
- **Given** رصيد قائم في 2017
- **When** يُبحَث عن مسار تسوية
- **Then** لا مسار يجعل 2017 مديناً؛ القيد اليدوي قادر تقنياً لا تصميماً
- **Expected Current Result: FAIL** 🔴
- **Target Result After Remediation: PASS** — مسار مُصمَّم — PO (OQ-6.A)

---

## 3. ملخّص التغطية — محسوب مباشرةً من 62 حالة *(بعد المراجعة التشغيلية 2026-07-25)*

| الحالة | العدد |
|---|---|
| **Current FAIL** | **40** |
| **Current PARTIAL** | **10** |
| **Current PASS** | **12** |
| **Total** | **62** |
| **`Target Result After Remediation: PASS`** | **50** (= 40 FAIL + 10 PARTIAL) |

- AINV (34): PASS **3** · PARTIAL 8 · FAIL **23**
- سيناريوهات (28): PASS **9** (TC-D04A · TC-D07 · TC-D08A · TC-D08C · TC-D09B · TC-D11A · TC-D11B · TC-D12 · TC-D19) · PARTIAL **2** (TC-D11C · TC-D11D) · FAIL **17**

> **إعادة الحساب التشغيلية (من 55→62 حالة):**
> - AINV: TC-AINV-05 من FAIL→PASS (اقتناء مؤكَّد FA-01) ⇒ PASS 2→3 · FAIL 24→23.
> - السيناريوهات: TC-D07 من FAIL→PASS (OB-01)؛ TC-D08 (FAIL) استُبدل بـ TC-D08A(PASS)·TC-D08B(FAIL)·TC-D08C(PASS)؛ TC-D11 (FAIL) استُبدل بـ TC-D11A(PASS)·TC-D11B(PASS)·TC-D11C(PARTIAL)·TC-D11D(PARTIAL)·TC-D11E(FAIL)؛ أُضيف TC-D19 (PASS). صافي +7 حالات.
> - **الأعداد محسوبة مباشرة من الحالات النهائية**، لم تُحافَظ الأعداد السابقة شكلياً. صادر التقييم: `Prototype_Runtime_Verification_Report.md`.
