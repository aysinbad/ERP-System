# Pricing Implementation Guide

## Document Information
```
Document Name:  Pricing Implementation Guide
Version:        1.3.0 (محاذاة الاستثناءات مع Pricing_Policy + قاعدة الإنشاء الشرطية لـ PriceGuidanceRecord + توضيح PRC_NO_COST_DATA + فصل حوكمة ADR)
Status:         Approved
Classification: Source of Truth
Owner:          Solution Architecture Team
Approved-by:    Product Owner
Approved-date:  2026-07-18
Last-Updated:   2026-07-21
```

> **العلاقة بـ `Pricing.md`:** هذا الملف "إزاي يتربط بالنظام تقنياً". كل بند مبني على المعمارية الموثَّقة في ADR-017 وADR-018. التنفيذ الإنتاجي للسلوك الجديد يبقى خاضعاً لاعتماد الـ ADRs المعنية رسمياً — اعتماد الموديول (`Approved`) واعتماد الـ ADR (`Accepted`) حالتان منفصلتان (انظر `AI_CONTEXT.md` §Governance Rules).
>
> **ميزة Cost Override:** Production Design — لا مكافئ كامل في الـ Prototype. POC جزئي في `experiments/pricing-engine-poc.html` (عرض CRM بدون تكلفة + `isCostStale`).

---

## Permissions Matrix

| الصلاحية | ما تتيح | تشمل ضمنياً | الحالة الافتراضية |
|---|---|---|---|
| **`pricing.viewSuggestedPrice`** | السعر الاسترشادي + حالة الموثوقية | — | تُمنح يدوياً |
| **`pricing.viewCost`** | تفصيل التكلفة / الهدر / الهامش / الربح / بنود قديمة بالتفصيل | `viewSuggestedPrice` | مدير/مالك (`role.all`) |
| **`pricing.editCostOverride`** | تعديل مؤقت على بنود التكلفة (Production-only) | — | تُمنح يدوياً |
| **`pricing.editMargin`** | تغيير الهامش المستهدف في الحاسبة | — | مدير/مالك افتراضياً |
| **`pricing.approvePriceException`** | اعتماد حالات استثنائية (هامش منخفض / بيانات قديمة / Override / سعر يدوي) | — | مدير/مالك (`role.all`) |

**قواعد التضمين:**
- `viewCost` ⊃ `viewSuggestedPrice` — من يملك `viewCost` يرى السعر ضمنياً بلا حاجة لمنح الاثنتين.
- من لا يملك `viewSuggestedPrice` ولا `viewCost`: **لا يرى أي معلومات تسعير**.
- **لا أسماء أدوار ثابتة** — الصلاحيات أعلام ديناميكية (`role.permissions[]`) نفس نمط `crmIsManager()`.

### Migration Note — `approveLowMargin` → `approvePriceException`
> ⚠️ **للمطورين والـ AI Agents:** هجرة مطلوبة عند التنفيذ.

| القديم | الجديد |
|---|---|
| `can('pricing','approveLowMargin')` | `can('pricing','approvePriceException')` |
| حدث: `pricing.margin_floor_overridden` | حدث: `pricing.exception_approved` + `{exceptionType: 'low_margin'}` |
| سبب للهامش فقط | سبب لكل أنواع الاستثناءات |

الأدوار التي كانت تملك `approveLowMargin` تُمنح `approvePriceException` تلقائياً — لا تراجع في الصلاحيات.

---

## Approval Workflow

```
حفظ فاتورة / بروفورما (أو تحديثها)
        │
        ▼
حساب الهامش الفعلي الناتج عن unitPrice (بعد أي خصم)
        │
        ▼
belowMarginFloor(invoice)? ── لا ──► فحص exceedsLimit الحالي → حفظ عادي
        │ نعم (أو: خصم خارج الحد / تكلفة قديمة تشترط السياسة اعتمادها / Override)
        ▼
المستخدم يملك pricing.approvePriceException؟
        │
   ┌────┴────┐
   لا         نعم
   │           │
   ▼           ▼
رفض + كود   طلب سبب إجباري
PRC_PRICE_    │
EXCEPTION_    ▼
REQUIRED   حفظ + Audit: pricing.exception_approved
              + {exceptionType, reason, approvedBy, approvedAt}
```

> **ملاحظة:** اختلاف `unitPrice` عن السعر الاسترشادي **لا يُشغِّل هذا المسار** — المشغِّل هو مخالفة السياسة (هامش، خصم، قِدَم تكلفة، Override). الانحراف عن المرجع الاسترشادي بحد ذاته مسموح بحرية كاملة.

**الحد الأدنى للهامش:** إعداد عام `pricingSettings.minMarginFloor` — القيمة يحددها صاحب الصلاحية (انظر `Pricing_Policy.md §4`). القيمة الافتراضية المقترحة 10% تحتاج تأكيد Product Owner.

---

## Cost Override — Entity Schema

> **Production Design — لا مكافئ في الـ Prototype.**

```typescript
interface CostOverride {
  id:              string;          // UUID
  pricingSessionId: string;         // مرجع لجلسة التسعير
  costComponent:   CostComponent;   // البند المعدَّل
  originalValue:   number;          // القيمة الأصلية (بعملة المنتج)
  overrideValue:   number;          // القيمة المعدَّلة
  reason:          string;          // سبب التعديل (إجباري، min 10 chars)
  overriddenBy:    string;          // userId
  overriddenAt:    DateTime;        // UTC timestamp
  scope:           'single_session'; // ثابت — لا خيار آخر حالياً
  companyId:       string;          // Multi-tenant
}

type CostComponent =
  | 'raw_material'   // مواد خام
  | 'packaging'      // تعبئة وتغليف
  | 'freight'        // شحن
  | 'manufacturing'  // تصنيع/تشغيل
  | 'waste'          // هدر
  | 'commission'     // عمولات
  | 'direct_expense' // مصروفات مباشرة
  | 'other';         // أي بند آخر في محرك التكلفة
```

**قواعد العزل (لا استثناء):**
- Override لا يكتب في `DB.products`, `DB.purchases`, `DB.inventory`, أو أي جدول محاسبي.
- Override يُقرأ فقط داخل `pricingCalc` خلال جلسة التسعير الحالية.
- عند انتهاء الجلسة: Override يُحفَظ في Audit فقط — لا يُطبَّق على جلسات لاحقة.

---

## Staleness Display Logic

| الحالة | الشرط (Prototype: `isCostStale`) | عرض المستخدم |
|---|---|---|
| `fresh` | كل بنود التكلفة ≤ 90 يوماً مكتملاً (اليوم 90 = محدَّث) | Badge "محدَّث ✓" |
| `stale` | أي بند > 90 يوماً مكتملاً (اليوم 91 فأكثر = قديم) | تنبيه: **"السعر الاسترشادي مبني جزئياً على بيانات تجاوزت 3 أشهر تقريباً ويحتاج مراجعة الإدارة."** |
| `no_purchase` | لا يوجد سجل شراء | "لا يوجد سعر مرجعي حديث — برجاء طلب اعتماد تسعير من الإدارة." |

**للإدارة (`viewCost`):** تفصيل كل بند قديم — اسم البند، تاريخ آخر تحديث، المصدر، القيمة.
**لمستخدم CRM (`viewSuggestedPrice` فقط):** التنبيه العام فقط — بلا تفاصيل.

**حساب الـ 90 يوم:** من `lastPurchase()` لكل بند — كما في `isCostStale()` في الـ Prototype (مصدر الحقيقة). الحساب على تاريخ **الشراء** لا تاريخ آخر بيع.

---

## Price Guidance Record — Schema (Audit Metadata، على مستوى كل بند)

> **قاعدة الإنشاء الشرطية:** `PriceGuidanceRecord` **ليس إلزامياً لكل بروفورما**. إذا حُسِب السعر الاسترشادي وعُرِض بنجاح للمستخدم، يجب تسجيل سجل واحد لكل بند مَعروض. إذا لم يملك المستخدم صلاحية `viewSuggestedPrice`، أو كان السعر الاسترشادي غير متاح (`PRC_NO_COST_DATA`)، **لا يُنشَأ أي سجل** وتستمر عملية إنشاء البروفورما بشكل طبيعي بسعر يدوي.
>
> اختياري لسير عمل البروفورما ككل · إلزامي عند عرض السعر الاسترشادي فعلياً · سجل واحد لكل بند · لا يُعيق إنشاء البروفورما أبداً · لا يُعتبَر السعر التجاري الفعلي بأي حال.

> يُحفَظ كـ metadata **لكل بند على حدة** على البروفورما لأغراض التدقيق والمقارنة التحليلية. **ليس** سعر البند — `unitPrice` الفعلي على `proforma.items[]` هو السعر الحقيقي.

```typescript
interface PriceGuidanceRecord {
  lineId:                   string;   // مرجع لبند البروفورما
  productCode:               string;
  quantity:                  number;
  currency:                  string;
  suggestedPriceAtDecision:  number;  // السعر الاسترشادي وقت إنشاء البروفورما
  priceConfidenceState:      'fresh' | 'stale' | 'no_purchase';
  costOverridePresent:       boolean;
  recordedBy:                string;  // userId
  recordedAt:                DateTime;
}
```

**التخزين:**
```typescript
proforma.priceGuidanceRecords: PriceGuidanceRecord[]   // مصفوفة — سجل واحد لكل بند
```

**قاعدة العزل:**
- `priceGuidanceRecords` لأغراض التدقيق فقط — لا يُعرَض للعميل ولا يُطبَع على المستند.
- `proforma.items[].unitPrice` هو القيمة التجارية الحقيقية التي تدخل كل الحسابات اللاحقة (ضريبة، COGS، عمولة).
- `exceptionApproved`/`exceptionReason` (إذا خالف `unitPrice` سياسةً) تُسجَّل في Audit Log العام (`pricing.exception_approved`) — لا داخل `PriceGuidanceRecord` نفسه، لتفادي ازدواج مصدر الحقيقة.

---

## Notifications
- تنبيه فوري للمدير عند أي `pricing.exception_approved`.
- لا قناة بريد/واتساب حالياً — تنبيهات داخل التطبيق فقط (فجوة موثَّقة في `CRM_Implementation_Guide.md`).

---

## Audit Rules

كل الأحداث التالية تُسجَّل إلزامياً في `auditLog`:

| الحدث | يُسجَّل عند | الحقول الإلزامية |
|---|---|---|
| `pricing.margin_changed` | تغيير الهامش بصلاحية `editMargin` | userId, oldMargin, newMargin, productCode |
| `pricing.cost_override_created` | إنشاء Cost Override | userId, costComponent, originalValue, overrideValue, reason |
| `pricing.exception_approved` | اعتماد استثناء | userId, exceptionType, reason, invoiceId/sessionId |
| `pricing.price_guidance_recorded` | تسجيل `PriceGuidanceRecord` لبند واحد بعد عرض السعر الاسترشادي بنجاح | `userId, opportunityId, proformaId, lineId, productCode, quantity, currency, suggestedPriceAtDecision, priceConfidenceState, costOverridePresent, recordedAt` |

> **الحدث يُصدَر مرة واحدة لكل بند مُسجَّل.** يُطلَق **فقط** بعد نجاح عرض السعر الاسترشادي وحفظ `PriceGuidanceRecord` الخاص به على مستوى البند. **لا يُصدَر** في أي من الحالات التالية: السعر الاسترشادي غير متاح (`PRC_NO_COST_DATA`)، المستخدم بلا صلاحية `viewSuggestedPrice`، أو لم يُعرَض أي سعر استرشادي أصلاً.

---

## Standard Error Codes

| الكود | الحالة | HTTP |
|---|---|---|
| `PRC_NO_COST_DATA` | لا يوجد سجل شراء لحساب التكلفة — نتيجة/خطأ داخلي لخدمة التسعير | 422 (من واجهة برمجة التسعير فقط) |
| `PRC_COST_STALE` | بيانات التكلفة قديمة (>90 يوماً مكتملاً) | 200 + warning |
| `PRC_MARGIN_BELOW_FLOOR` | الهامش أقل من الحد الأدنى | 409 |
| `PRC_PRICE_EXCEPTION_REQUIRED` | الحالة تستوجب `approvePriceException` | 403 |
| `PRC_OVERRIDE_SCOPE_VIOLATION` | محاولة تطبيق Override خارج الجلسة | 422 |
| `PRC_MARGIN_NEGATIVE` | هامش سالب (PINV-08 + PINV-10) | 422 |

> **⚠️ توضيح حاسم:** `PRC_NO_COST_DATA` هو **نتيجة داخلية لخدمة التسعير فقط** — يعني عدم القدرة على حساب سعر استرشادي. **هذا لا يعني تلقائياً وجوب حجب المستند التجاري (البروفورما/الفاتورة).** الموديول المستهلك (CRM) مسؤول عن التقاط هذه النتيجة وتحويلها لسلوك غير حاجب خاص به — انظر `CRM_Implementation_Guide.md` قسم Pricing Integration (`CRM_PROFORMA_GUIDANCE_UNAVAILABLE`, HTTP 200 + warning).

---

## Domain Events

| الحدث | المُشغِّل | المستهلك |
|---|---|---|
| `pricing.suggested_price_viewed` | مستخدم CRM يفتح السعر الاسترشادي | Audit |
| `pricing.cost_override_created` | إنشاء Override | Audit + Notification للمدير |
| `pricing.exception_approved` | اعتماد استثناء | Audit + Notification للمدير |
| `pricing.price_guidance_recorded` | حفظ `PriceGuidanceRecord` لبند عند إنشاء البروفورما | CRM (تحديث الفرصة) + Audit |

---

## Cross-references

### Related ADRs
- `docs/02_Governance/ADR/ADR-017.md` — مصادر مكوّنات التكلفة
- `docs/02_Governance/ADR/ADR-018.md` — نموذج الصلاحيات الكامل (5 صلاحيات)

### Related RFCs
- `docs/03_Business_Logic/Pricing/Pricing_RFC.md` — RFC-PRC-001 ✅ · RFC-PRC-002 ✅ · RFC-PRC-004 ⬜
- `docs/03_Business_Logic/Accounting/Accounting_RFC.md → RFC-ACC-001`

### Business Logic Source
- `docs/03_Business_Logic/Pricing/Pricing.md`

### Policy
- `docs/04_Policies/Pricing_Policy.md`
