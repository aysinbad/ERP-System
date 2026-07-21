# Pricing Implementation Guide

## Document Information
```
Document Name:  Pricing Implementation Guide
Version:        1.1.0 (5 صلاحيات + Cost Override Audit + Staleness Logic + Migration Note)
Status:         Approved
Classification: Source of Truth
Owner:          Solution Architecture Team
Approved-by:    Product Owner
Approved-date:  2026-07-18
Last-Updated:   2026-07-21
```

> **العلاقة بـ `Pricing.md`:** هذا الملف "إزاي يتربط بالنظام تقنياً". كل بند مبني مباشرة على قرار معتمد (ADR-017/018)، لا افتراض جديد.
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
حفظ فاتورة (أو تحديثها)
        │
        ▼
هل يوجد Cost Override في جلسة التسعير هذه؟
        │
   نعم ──► يستوجب pricing.approvePriceException
        │
        ▼
حساب الهامش الفعلي الناتج (بعد أي خصم)
        │
        ▼
belowMarginFloor(invoice)? ── لا ──► فحص exceedsLimit الحالي → حفظ عادي
        │ نعم
        ▼
هل بيانات التكلفة متجاوِزة 90 يوماً؟ (أو سعر يدوي خارج السياسة؟)
        │
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
| `fresh` | كل بنود التكلفة < 90 يوماً | Badge "محدَّث ✓" |
| `stale` | أي بند ≥ 90 يوماً | تنبيه: **"السعر الاسترشادي مبني جزئياً على بيانات تجاوزت 3 أشهر تقريباً ويحتاج مراجعة الإدارة."** |
| `no_purchase` | لا يوجد سجل شراء | "لا يوجد سعر مرجعي حديث — برجاء طلب اعتماد تسعير من الإدارة." |

**للإدارة (`viewCost`):** تفصيل كل بند قديم — اسم البند، تاريخ آخر تحديث، المصدر، القيمة.
**لمستخدم CRM (`viewSuggestedPrice` فقط):** التنبيه العام فقط — بلا تفاصيل.

**حساب الـ 90 يوم:** من `lastPurchase()` لكل بند — كما في `isCostStale()` في الـ Prototype (مصدر الحقيقة). الحساب على تاريخ **الشراء** لا تاريخ آخر بيع.

---

## Suggested Price Snapshot — Schema

يُحفَظ عند اختيار المستخدم السعر قبل تحويل الفرصة لبروفورما:

```typescript
interface PriceSnapshot {
  suggestedPrice:      number;
  priceConfidenceState: 'fresh' | 'stale' | 'no_purchase';
  costOverridePresent:  boolean;
  selectedPrice:        number;           // ما اختاره المستخدم (قد يختلف عن Suggested)
  selectedBy:           string;           // userId
  selectedAt:           DateTime;
  exceptionApproved:    boolean;          // true إذا استوجبت الحالة approvePriceException
  exceptionReason?:     string;
  pricingEngineVersion: string;           // للمستقبل — إصدار محرك التسعير وقت الحساب
}
```

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
| `pricing.price_snapshot_saved` | حفظ Snapshot عند تحويل فرصة | userId, selectedPrice, confidenceState, opportunityId |

---

## Standard Error Codes

| الكود | الحالة | HTTP |
|---|---|---|
| `PRC_NO_COST_DATA` | لا يوجد سجل شراء لحساب التكلفة | 422 |
| `PRC_COST_STALE` | بيانات التكلفة قديمة (>90 يوم) | 200 + warning |
| `PRC_MARGIN_BELOW_FLOOR` | الهامش أقل من الحد الأدنى | 409 |
| `PRC_PRICE_EXCEPTION_REQUIRED` | الحالة تستوجب `approvePriceException` | 403 |
| `PRC_OVERRIDE_SCOPE_VIOLATION` | محاولة تطبيق Override خارج الجلسة | 422 |
| `PRC_MARGIN_NEGATIVE` | هامش سالب (PINV-08 + PINV-10) | 422 |

---

## Domain Events

| الحدث | المُشغِّل | المستهلك |
|---|---|---|
| `pricing.suggested_price_viewed` | مستخدم CRM يفتح السعر الاسترشادي | Audit |
| `pricing.cost_override_created` | إنشاء Override | Audit + Notification للمدير |
| `pricing.exception_approved` | اعتماد استثناء | Audit + Notification للمدير |
| `pricing.price_snapshot_saved` | تحويل فرصة لبروفورما بسعر مختار | CRM (تحديث الفرصة) + Audit |

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
