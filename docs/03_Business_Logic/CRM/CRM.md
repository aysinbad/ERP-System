# CRM Business Logic

## Document Information
```
Document Name:  CRM Business Logic
Version:        2.2.0 (إضافة Business Rules #12 و#13 — Pricing Integration)
Status:         In Review
Classification: Source of Truth
Owner:          Product Owner
Approved-by:    —
Approved-date:  —
Last-Updated:   2026-07-21
Supersedes:     CRM.md v2.1.0
```

> **المصدر:** مراجعة مباشرة لكود `reference/prototype/prototype_v2.html` — لا افتراض. هذه الوثيقة تغطي "ماذا يجب أن يحدث" فقط (منطق الأعمال الخالص). تفاصيل الدمج التقني (State Matrix, Permissions, Notifications, Audit, Domain Events, Error Codes) انتقلت إلى `CRM_Implementation_Guide.md`.
> **لا تُعتمَد (`Approved`) قبل حسم INV-09 (موثّق بالتفصيل في `CRM_Implementation_Guide.md` — State Transition Matrix)** — يكشف سلوكاً غير مقيَّد في الكود الحالي يحتاج قرار صريح من الفريق.

---

## Module Dependencies

**Depends On (Business Modules):**
- لا اعتماد مباشر على موديولات أخرى في منطق CRM الخالص — CRM هو منتج أولي لا يستهلك مخرجات موديولات أخرى في القرارات الداخلية.
- **Pricing** — يُستهلَك منه السعر الاسترشادي (`SuggestedPrice`) كمدخل اختياري قبل تحويل الفرصة لبروفورما (Business Rule #13). CRM لا يستدعي محرك التسعير مباشرةً — يعرض ما يُرسَل له من Pricing فقط.

**Depends On (Shared Utilities):**
- لا يوجد اعتماد مباشر مؤكَّد على `fxRate()` داخل منطق CRM الخالص.

**Produces (يُستهلَك من موديولات أخرى):**
- **`crm.opportunity.won`** (Domain Event) → Sales/Export ينشئ البروفورما عند استهلاك هذا الحدث (ADR-019).
- **سلسلة اشتقاق ملكية العميل** → Pricing (`commissionRows` تعتمد عليها لحساب العمولة — Business Rules #7).
- **`custCode`** على كل فرصة → ينتقل عبر السلسلة لـ Sales/Export ثم Accounting.

---

## Purpose
تمكين مندوبي المبيعات من إدارة دورة حياة العميل الكاملة (من أول تواصل محتمل حتى فرصة بيعية مغلقة) مع رؤية تلقائية لصحة العلاقة ومخاطر فقدها، وربط ذلك بدورة المبيعات والتصدير القائمة (بروفورما → عقد → فاتورة تجارية) دون ازدواج أو تعارض بيانات بين المندوبين.

---

## Scope

**داخل النطاق:**
- العملاء المحتملون (Leads) ودورة تأهيلهم
- الفرص البيعية (Opportunities) والمسار (Pipeline)
- الأنشطة (Activities) والمهام (Tasks)
- محركات ذكاء العملاء: نقاط الصحة، مخاطر الفقد، تصنيف RFM، أفضل إجراء تالٍ
- قواعد الملكية ومنع تضارب المندوبين
- عرض السعر الاسترشادي من Pricing واختيار سعر البروفورما (Business Rules #12, #13)

**خارج النطاق (تُغطّى في وثائق أخرى):**
- تصميم قاعدة البيانات والـ API — Technical Design
- تفاصيل الدمج التقني — `CRM_Implementation_Guide.md`
- حالات الاختبار — `CRM_Test_Cases.md`
- منطق دورة البيع بعد البروفورما — وحدة Sales/Export
- منطق المحاسبة — لا علاقة له بـ CRM مباشرة
- حساب السعر الاسترشادي نفسه أو تفاصيل التكلفة — وحدة Pricing (Black Box)

---

## Actors

| الفاعل | الوصف |
|---|---|
| **مندوب مبيعات (Sales Rep)** | يرى ويدير فقط ما يملكه (`owner === نفسه`) من محتملين/فرص/مهام/أنشطة، ويرى أسماء كل العملاء بدون تفاصيل غير المملوك له |
| **مدير/مالك (Manager/Owner)** | أي مستخدم بدور `owner` أو دور له صلاحية `all` — يرى كل شيء بلا فلترة، وهو الوحيد المخوَّل لإعادة توزيع ملكية العملاء |
| **النظام (محركات آلية)** | يحسب نقاط الصحة/المخاطر/التوصيات تلقائياً بلا تدخل بشري، ويُنشئ مهام تلقائية عند أحداث معينة (مثال: فوز فرصة) |

---

## Entities

| الكيان | الوصف | الحقول الحاكمة للمنطق |
|---|---|---|
| **Lead** | جهة اهتمام أولية قبل التأهيل | `stage`, `owner`, `source`, `estValue`, `lastActivity`, `convertedCustCode` |
| **Opportunity** | صفقة محتملة قيد التفاوض | `stage`, `probability`, `value`, `cur`, `owner`, `expectedClose`, `proformaId`, `leadId` |
| **Activity** | سجل تفاعل (مكالمة/بريد/اجتماع/واتساب/زيارة/ملاحظة) | `refType`, `refId`, `type`, `date`, `by` |
| **Task** | متابعة بموعد استحقاق | `refType`, `refId`, `due`, `priority`, `status`, `owner` |
| **Customer (حقل الملكية فقط)** | العميل الفعلي — CRM يضيف حقل `owner` عليه فقط | `owner` |

---

## Status Flow

### Lead
```
new → contacted → qualified → converted (نهائية، عبر إجراء منفصل convertLeadToCustomer)
                → unqualified (نهائية)
```
لا رجوع من `converted` أو `unqualified`.

### Opportunity
```
qualification (20%) → proposal (45%) → negotiation (70%) → won (100%)
                                                          → lost (0%)
```
**عند الوصول لـ `won`:** تُنشأ مهمة تلقائية "إصدار بروفورما/فاتورة تجارية"، ويبدأ مسار التحويل للبروفورما.

---

## Business Rules

### 1. تسجيل العملاء الجدد — بمنطق "المندوب يسجّل بنفسه"
جديد العملاء يُسجَّل باسم المندوب الذي أدخله مباشرة. لا يوجد "مستودع مشترك" بلا مالك — كل عميل له مالك أو بلا مالك (unowned) حتى يُعيَّن.

### 2. منع ازدواج العملاء المُملوكَين
الشرط: عميلان بنفس (الاسم + الدولة) لا يمكن أن يكونا مملوكَين لمندوبَين مختلفَين في نفس الوقت. يُطبَّق بشكل صارم (حارس `customerSaveGuard`). إذا كان العميل الموجود بلا مالك، تُعرَض تحذير غير مانع.

### 3. تحويل الفرصة الناجحة لبروفورما

**الآلية الحالية في الـ Prototype:** `crmOppToProforma` يستدعي مباشرة كود إنشاء بروفورما بالسعر الثابت `Product.price`.

**الآلية المعتمدة للبناء الإنتاجي (2026-07-21 — ميزة السعر الاسترشادي):**
يُعرَض السعر الاسترشادي (بصلاحية `pricing.viewSuggestedPrice`) في **Price Guidance Panel** — لوحة معلومات مصاحبة غير ملزِمة. المستخدم يُدخَل `unitPrice` الفعلي لكل بند بشكل مستقل بناءً على التفاوض مع العميل. البروفورما لا تُمنَع ولا تُلزَم باستخدام السعر الاسترشادي. (تفاصيل تقنية في `CRM_Implementation_Guide.md` قسم Pricing Integration.)

### 4. قواعد الملكية الهرمية
الملكية تُحدَّد بترتيب: (1) `owner` على العميل مباشرةً، (2) فرصة مرتبطة بالعميل ولها مالك، (3) Lead تحوّل للعميل وله مالك، (4) آخر نشاط مسجَّل وله `by`.

### 5. إعادة توزيع الملكية — صلاحية المدير حصراً
فقط المدير/المالك (`crmIsManager()`) يستطيع إعادة توزيع ملكية عميل. تُحدَّث `owner` على العميل مباشرةً (مصدر الحقيقة الأساسي).

### 6. حساب نقاط الصحة والمخاطر
نقاط الصحة: تكرار الشراء (30) + حجم المبيعات (25) + حداثة التواصل (25) + الالتزام بالسداد (20) — المجموع محصور [0,100].

### 7. اشتقاق مالك العميل لحساب العمولة
نفس سلسلة الاشتقاق في قاعدة #4 — هذه القاعدة تُستهلَك من Pricing (`commissionRows`) لتحديد مستحق العمولة.

### 8. تصنيف RFM — شرطي متتابع
بالترتيب (أول شرط يتحقق يفوز): `freq=0` → جديد/محتمل؛ تواصل ≤45ي + طلبات≥3 + مبيعات≥300K → مميّز؛ تواصل ≤60ي + طلبات≥2 → وفيّ؛ >120ي → خامل؛ >75ي → معرّض للفقد؛ غير ذلك → واعد.

### 9. أفضل إجراء تالٍ (Next Best Action)
لكل كيان، تُبنى قائمة إجراءات مرشّحة بدرجة أولوية، ويُعرَض الأعلى فقط.

### 10. التوقّع المرجّح للمسار (Pipeline Forecast)
Gross = مجموع قيم الفرص المفتوحة. Weighted = `القيمة × احتمالية المرحلة`. Win Rate = `ناجحة ÷ (ناجحة+خاسرة)`.

### 11. حذف العميل المحتمل (Lead Deletion) — حذف نهائي، بلا قيد
الكود يدعم حذفاً نهائياً حقيقياً للـ Lead بلا أي فحص لحالته.
⚠️ **خطر تكامل بيانات:** حذف Lead محوَّل يكسر سلسلة اشتقاق مالك العميل (قاعدة 4 وقاعدة 7).

### 12. رؤية السعر الاسترشادي في CRM (Pricing Visibility)

> هذه القاعدة تحدد ما يرى / ما لا يرى مستخدم CRM من بيانات التسعير. التفاصيل التقنية في `CRM_Implementation_Guide.md` قسم Pricing Integration.

مستخدم CRM بصلاحية `pricing.viewSuggestedPrice` يرى **فقط:**
- ✅ السعر الاسترشادي النهائي للمنتج.
- ✅ حالة الموثوقية: "محدَّث" أو "يحتاج مراجعة الإدارة".
- ✅ تنبيه مبسَّط عند وجود بيانات قديمة: *"السعر الاسترشادي مبني جزئياً على بيانات تجاوزت 3 أشهر تقريباً ويحتاج مراجعة الإدارة."*

مستخدم CRM **لا يرى في أي حال:**
- ❌ تكلفة المنتج أو الخامات أو الهدر.
- ❌ هامش الربح أو نسبته أو قيمة الربح.
- ❌ نصيب المصاريف العامة أو تفاصيل بنود التكلفة.
- ❌ أي تعديل يدوي على التكلفة (Cost Override) أجرته الإدارة — يرى السعر الناتج بعده فقط.
- ❌ معادلة الحساب أو تفاصيل محرك التسعير.
- ❌ تفاصيل البنود القديمة (أي بند، تاريخه، قيمته) — التنبيه العام فقط.

مستخدم CRM بلا صلاحية `viewSuggestedPrice` أو `viewCost`: **لا يرى أي معلومات تسعير**.

### 13. السعر الاسترشادي كمرجع داخلي غير ملزم (Price Guidance Panel)

عند إنشاء بروفورما من فرصة، يُعرَض السعر الاسترشادي (بصلاحية `pricing.viewSuggestedPrice`) في **Price Guidance Panel** — لوحة معلومات مصاحبة غير ملزِمة. المستخدم يُدخَل `unitPrice` الفعلي لكل بند بشكل مستقل.

**قواعد:**
1. `unitPrice` الفعلي المُدخَل هو سعر البروفورما — يمكن أن يكون مساوياً أو أعلى أو أقل من السعر الاسترشادي.
2. لا يُمنَع إنشاء البروفورما لمجرد اختلاف `unitPrice` عن السعر الاسترشادي.
3. `pricing.viewSuggestedPrice` تتحكم في رؤية لوحة المعلومات فقط — ليست شرطاً لإنشاء البروفورما.
4. `pricing.approvePriceException` تُطبَّق فقط عندما يُخالف `unitPrice` الفعلي سياسةً (هامش أدنى، خصم خارج الحد، تكلفة قديمة تشترط السياسة اعتمادها، Override).
5. يُحفَظ `suggestedPriceAtDecision` داخلياً كـ metadata للتدقيق والمقارنة فقط — لا يُعرَض للعميل ولا يُعتبَر سعر المستند.

**⚠️ ملاحظة الـ Prototype:** `crmOppToProforma` يستخدم `Product.price` الثابت. قاعدة #13 تحل محله في البناء الإنتاجي.

---

## Business Invariants

> قواعد لا يمكن كسرها بغض النظر عن أي تنفيذ تقني.

| # | القاعدة الثابتة | المصدر/الدليل |
|---|---|---|
| INV-01 | لا يوجد عميلان بنفس (الاسم + الدولة) مملوكان لمندوبين مختلفين في نفس الوقت | `crmCheckDuplicate` + حارس `editCustomer` |
| INV-02 | فرصة بيعية لا تتحول لفاتورة تجارية مباشرة — لازم تمر ببروفورما أولاً | ADR-001 |
| INV-03 | `Lead.stage` لا يتحول لـ `converted` إلا عبر `convertLeadToCustomer`، ولا رجوع منها | `convertLeadToCustomer` (يرفض لو محوَّل مسبقاً) |
| INV-04 | نقاط Health/Lead Score/Churn محصورة دائماً بين 0 و100 | `Math.max(0,Math.min(100,...))` |
| INV-05 | إعادة توزيع ملكية عميل فعل حصري للمدير/المالك فقط | `crmReassign` يرفض لغير المدير |
| INV-06 | تعريف "المدير" مشتق من الصلاحيات (`role.all` أو `owner`) وليس اسم دور ثابت | `crmIsManager()` |
| INV-07 | ترتيب تقييم شرائح RFM ثابت ومتتابع (أول تطابق يفوز) | قاعدة #8 |
| INV-08 | كل نشاط CRM مسجَّل يُحدِّث `lastActivity` على الكيان المرتبط تلقائياً | `crmLog()` |
| **INV-09 (قرار مطلوب)** | هل تُقيَّد انتقالات مراحل الفرصة/العميل المحتمل؟ — **بانتظار ADR-015** | تفصيل في `CRM_Implementation_Guide.md` |

---

## Reports

- **لوحة CRM الرئيسية:** المسار المرجّح، معدّل الفوز، عدد المحتملين النشطين والواعدين، المهام المفتوحة/المتأخرة، عدد العملاء المعرّضين لخطر الفقد.
- **لوحة صحة العملاء:** ترتيب تصاعدي (الأضعف أولاً) بالنقاط، الشريحة (RFM)، الذمم القائمة، الإجراء التالي المقترح.
- **أفضل المحتملين:** أعلى 6 بالنقاط.
- **جدول زمني للأنشطة الأخيرة.**

---

## Open Questions

1. **🔴 الأهم (INV-09):** هل تُقيَّد انتقالات مراحل الفرصة/العميل المحتمل؟ **بانتظار اعتماد ADR-015.**
2. هل مطلوب Fuzzy matching لأسماء العملاء (بدل التطابق الحرفي الحالي) في v1؟
3. ما القاعدة الإلزامية (إن وُجدت) لحقل `lostReason` عند خسارة فرصة؟
4. هل يُمنع أو يُسمح (بتحذير) إنشاء فرصة لعميل محجوز لمندوب آخر؟
5. **🔴 هل يُمنع حذف Lead بحالة `converted` تحديداً؟** (انظر Business Rules #11 — KI-007).

---

## Future Enhancements

- قناة تنبيهات بريد/واتساب بدل In-app فقط (تفاصيل في `CRM_Implementation_Guide.md` قسم Notifications).

---

## Cross-references

### Related ADRs
- `docs/02_Governance/ADR/ADR-001.md` — فرصة→بروفورما (Accepted)
- `docs/02_Governance/ADR/ADR-004.md` — ملكية العملاء: المندوب يسجّل بنفسه (Accepted)
- `docs/02_Governance/ADR/ADR-015.md` — تقييد انتقالات Won/Lost مع تجاوز مبرَّر (Proposed)
- `docs/02_Governance/ADR/ADR-016.md` — CRM Enterprise Enhancements (Proposed)
- `docs/02_Governance/ADR/ADR-018.md` — نموذج صلاحيات التسعير (Proposed) — صلاحيات `viewSuggestedPrice`/`approvePriceException` المُستهلَكة في #12 و#13
- `docs/02_Governance/ADR/ADR-019.md` — استبدال استدعاء CRM→Sales بـ Domain Event (Proposed)

### Related RFCs
- `docs/03_Business_Logic/CRM/CRM_RFC.md` — RFC-CRM-001 ✅ (ADR-015) · RFC-CRM-002 ✅ (ADR-016)

### Depends On
- `docs/01_Standards/Glossary.md` §1 — مصطلحات دورة البيع
- `docs/02_Governance/Known_Issues.md` — KI-007 (delLead بلا صلاحية)
- `docs/00_Project/Current_State.md` — فجوة Notification Center

### Referenced Modules
- `docs/03_Business_Logic/Pricing/Pricing.md` — Business Rules #7 (ملكية العميل→العمولة) · Business Rules §Suggested Price (السعر الاسترشادي المعروض في #12, #13)
- `docs/03_Business_Logic/Sales_Export/Sales_Export.md` — ADR-019 (Domain Event)

### Implementation Guide
- `docs/03_Business_Logic/CRM/CRM_Implementation_Guide.md` — State Matrix · Permissions · Pricing Integration · Notifications · Audit · Domain Events · Error Codes

### Test Cases
- `docs/03_Business_Logic/CRM/CRM_Test_Cases.md` _(Placeholder — بانتظار اعتماد ADR-015)_
