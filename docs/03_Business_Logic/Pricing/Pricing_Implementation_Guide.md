# Pricing Implementation Guide

## Document Information
```
Document Name:  Pricing Implementation Guide
Version:        1.0.0 (مبني على ADR-017 وADR-018)
Status:         Approved
Classification: Source of Truth
Owner:          Solution Architecture Team
Approved-by:    Product Owner
Approved-date:  2026-07-18
Last-Updated:   2026-07-18
```

> **العلاقة بـ `Pricing.md`:** نفس مبدأ `CRM/CRM_Implementation_Guide.md` — هذا الملف "إزاي يتربط بالنظام تقنياً". كل بند هنا مبني مباشرة على قرار معتمد (ADR-017 أو ADR-018)، لا افتراض جديد.

---

## Permissions Matrix

| الصلاحية | الوصف | من يملكها افتراضياً |
|---|---|---|
| **`pricing.viewCost`** | رؤية تفصيل التكلفة/الهدر/الهامش/الربح كاملاً في حاسبة التسعير وسلّة التسعير | مدير/مالك (`role.all`) — قابل للمنح لأي دور آخر عبر مصفوفة الصلاحيات |
| **`pricing.viewSuggestedOnly`** | (الحالة الافتراضية لغياب `pricing.viewCost`) رؤية السعر المقترح/سعر البيع النهائي فقط | أي دور بلا `pricing.viewCost` صراحة |
| **`pricing.approveLowMargin`** | تجاوز حد الهامش الأدنى عند حفظ فاتورة، بشرط سبب إجباري | مدير/مالك افتراضياً — نفس منطق `crmIsManager()` المُعاد استخدامه هنا، وليس اسم دور جديد |

⚠️ **لا تُستحدَث أسماء أدوار جديدة** (لا "Finance"، لا "Sales Manager") — الصلاحيات أعلام منطقية تُمنح عبر نظام الأدوار الحالي (ADR-018، رفض صريح لتصنيف الأدوار الثابت).

## Approval Workflow

```
حفظ فاتورة (أو تحديثها)
        │
        ▼
حساب الهامش الفعلي الناتج (بعد أي خصم مطبَّق)
        │
        ▼
belowMarginFloor(invoice)? ──── لا ───► حفظ عادي (مروراً بفحص exceedsLimit الحالي كما هو)
        │ نعم
        ▼
المستخدم يملك pricing.approveLowMargin؟
        │
   ┌────┴────┐
   لا         نعم
   │           │
   ▼           ▼
رفض الحفظ    طلب سبب إجباري (نفس نمط ADR-015)
+ رسالة       │
PRC_MARGIN_    ▼
BELOW_FLOOR   حفظ + تسجيل تدقيق إلزامي (Audit)
              + حدث pricing.margin_floor_overridden
```

**الحد الأدنى للهامش (Minimum Margin Floor):** إعداد عام جديد على مستوى النظام — **القيمة الافتراضية المقترحة 10%، تحتاج تأكيد نهائي من الفريق قبل التنفيذ الفعلي** (ليست قراراً مغلقاً في ADR-018 نفسه).

## Notifications
- تنبيه فوري للمدير عند أي تجاوز لحد الهامش الأدنى (`pricing.margin_floor_overridden`).
- لا توجد قناة بريد/واتساب حالياً (نفس الفجوة العامة الموثَّقة في `CRM/CRM_Implementation_Guide.md`) — تنبيهات داخل التطبيق فقط.

## Domain Events

| الحدث | يُطلَق عند | الحمولة الأساسية |
|---|---|---|
| `pricing.suggestion.calculated` | كل استدعاء ناجح لـ `pricingCalc` | `prodCode, qty, unitCost, wastePct, overheadRate, marginPct, suggestedUnitPrice` |
| `pricing.override.applied` | تعديل يدوي للسعر في سلّة التسعير | `prodCode, calculatedPrice, overridePrice` |
| `pricing.margin_floor_overridden` | تجاوز مدير لحد الهامش الأدنى (بسبب إجباري) | `invoiceId, resultingMargin, floorPct, reason, overriddenBy` |
| `pricing.cost_component.category_used` *(مقترح، مرتبط بـ ADR-017)* | استخدام متوسط تاريخي من `DB.expenses` لأي مكوّن (شحن/تعبئة/تأمين/بنكية/فحص/شهادات) | `category, averageValueEGP, sampleSizeInvoices, windowDays:90` |

## Standard Error Codes

| الكود | الوصف | HTTP المقترح |
|---|---|---|
| `PRC_MARGIN_BELOW_FLOOR` | الهامش الناتج أقل من الحد الأدنى، والمستخدم بلا صلاحية تجاوز | 403 Forbidden |
| `PRC_MARGIN_OVERRIDE_REASON_REQUIRED` | محاولة تجاوز الحد الأدنى بلا إدخال سبب | 422 Unprocessable Entity |
| `PRC_COST_HIDDEN` | محاولة الوصول لبيانات تكلفة من مستخدم بلا `pricing.viewCost` | 403 Forbidden |
| `PRC_NO_RECENT_REFERENCE_COST` | لا يوجد سعر مرجعي حديث (منتج بلا شراء خلال 90 يوم أو بلا شراء إطلاقاً) | 200 OK + `isSuggested:false` (ليس خطأ مانع) |

## API Mapping (توضيحي، غير نهائي — ينتظر بناء أول نسخة فعلية من عقد الـ API)

| قاعدة العمل | نقطة API المقترحة | ملاحظة |
|---|---|---|
| حساب سعر مقترح لمنتج واحد | `POST /api/v1/pricing/suggest` | يتبع اصطلاح `docs/01_Standards/API_Standards.md` |
| سلّة تسعير متعددة المنتجات | `POST /api/v1/pricing/basket/calculate` | — |
| حفظ فاتورة (يشمل فحص `belowMarginFloor`) | يبقى ضمن `POST /api/v1/sales/invoices` الموجود أصلاً في وحدة Sales — **ليس endpoint منفصل للتسعير** | فحص الهامش جزء من حفظ الفاتورة، لا عملية منفصلة |

## Audit Rules

يُسجَّل إلزامياً (بناءً على ADR-018):
- كل تجاوز لحد الهامش الأدنى: المستخدم، الفاتورة، الهامش الناتج، السبب المُدخَل، التاريخ/الوقت.

⚠️ **لا يُسجَّل حالياً (فجوة موروثة من الوضع الحالي، خارج نطاق ADR-018 المباشر):** تعديل السعر يدوياً (`overridePrice`) في سلّة التسعير نفسها (قبل ما يتحول لفاتورة) — هذا لسه غير مغطى بتصميم تدقيق مخصَّص، لأنه سابق على مرحلة "حفظ الفاتورة" التي عندها فقط يُفحَص `belowMarginFloor`.

---

## Cross-references
- `docs/03_Business_Logic/Pricing/Pricing.md`: منطق الأعمال الأساسي (المرجع الأعلى لأي تعارض)
- `docs/02_Governance/ADR/ADR-017.md`: مصادر مكوّنات التكلفة
- `docs/02_Governance/ADR/ADR-018.md`: صلاحية رؤية التكلفة وحد الهامش الأدنى
- `docs/02_Governance/ADR/ADR-015.md`: نمط "سبب إجباري" المُعاد استخدامه هنا
- `docs/01_Standards/API_Standards.md`: اصطلاح أكواد الخطأ ومسارات الـ API
