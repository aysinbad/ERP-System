# CRM Implementation Guide

## Document Information
```
Document Name:  CRM Implementation Guide
Version:        1.2.0 (إضافة قسم Pricing Integration)
Status:         In Review
Classification: Source of Truth
Owner:          Solution Architecture Team
Approved-by:    —
Approved-date:  —
Last-Updated:   2026-07-21
Supersedes:     CRM_Implementation_Guide.md v1.1.0
```

> **العلاقة بـ `CRM.md`:** `CRM.md` يحدد "ماذا يجب أن يحدث". هذا الملف يحدد "إزاي يتربط بالنظام تقنياً". **لا تعديل مستقل في أحدهما بمعزل عن الآخر.**
> **لا يُعتمَد (`Approved`) قبل حسم INV-09** — بانتظار اعتماد ADR-015.

---

## State Transition Matrix

### Lead — الحالة الفعلية في الكود (غير مقيَّدة)

| من \ إلى | new | contacted | qualified | unqualified | converted |
|---|---|---|---|---|---|
| **new** | — | ✅ | ✅ | ✅ | ✅ (عبر تحويل منفصل) |
| **contacted** | ✅ | — | ✅ | ✅ | ✅ |
| **qualified** | ✅ | ✅ | — | ✅ | ✅ |
| **unqualified** | ✅ | ✅ | ✅ | — | ❌ |
| **converted** | ❌ | ❌ | ❌ | ❌ | — |

> القيد الوحيد الفعلي: `converted` نهائية تماماً.

### Opportunity — الحالة الفعلية في الكود (غير مقيَّدة)

| من \ إلى | qualification | proposal | negotiation | won | lost |
|---|---|---|---|---|---|
| **qualification** | — | ✅ | ✅ | ✅ | ✅ |
| **proposal** | ✅ | — | ✅ | ✅ | ✅ |
| **negotiation** | ✅ | ✅ | — | ✅ | ✅ |
| **won** | ✅ | ✅ | ✅ | — | ✅ |
| **lost** | ✅ | ✅ | ✅ | ✅ | — |

**🔴 قرار مطلوب:** هل يُقيَّد الخروج من `won`/`lost`؟ — موثَّق في ADR-015 (Proposed).

---

## Permissions

- **تعريف "المدير":** `role==='owner'` أو دور له `all:true` — ديناميكي وليس اسم دور ثابت.
- **فلترة كاملة** للمندوب غير المدير: يرى فقط `owner===نفسه`.
- **استثناء شاشة العملاء:** المندوب يرى كل الأسماء (لمنع التكرار)، لكن التفاصيل مقفلة.
- **إعادة التوزيع:** حصرية للمدير.
- ⚠️ **حذف Lead (`delLead`): بلا أي تحقق صلاحية حالياً** — KI-007.
- ⚠️ **كل ما سبق تحقق على الواجهة فقط** — KI-002. يُفرَض على مستوى الـ API عند البناء.

---

## Pricing Integration

> هذا القسم يصف التفاصيل التقنية لعرض السعر الاسترشادي في CRM واختياره قبل تحويل الفرصة لبروفورما. المرجع الكامل لما يرى / ما لا يرى: `CRM.md` Business Rules #12 و#13.

### فحص الصلاحية

```
can('pricing','viewSuggestedPrice') → عرض السعر + PriceConfidenceState
can('pricing','viewCost') → يتضمن viewSuggestedPrice ضمنياً
لا صلاحية → لا عرض لأي معلومات تسعير
```

### حقول العرض (للمستخدم بصلاحية viewSuggestedPrice)

```typescript
interface SuggestedPriceDisplay {
  suggestedPrice:      number;
  currency:            string;
  confidenceState:     'fresh' | 'stale' | 'no_purchase';
  badgeText:           string;    // نص للمستخدم
  // تفاصيل التكلفة: HIDDEN — للإدارة (viewCost) فقط
}
```

**نصوص الـ Badge:**
- `fresh` → "سعر مرجعي محدَّث ✓"
- `stale` → "يحتاج مراجعة الإدارة ⚠️" + tooltip: *"بيانات تجاوزت 3 أشهر تقريباً."*
- `no_purchase` → "لا يوجد سعر مرجعي حديث — برجاء طلب اعتماد تسعير."

**للإدارة (`viewCost`) إضافياً:**
- قائمة البنود القديمة: اسم البند، تاريخ آخر تحديث، المصدر، القيمة.
- إشارة إذا كان السعر يعكس Cost Override.

### Price Selector قبل تحويل الفرصة لبروفورما

```
المستخدم يضغط "تحويل لبروفورما" على فرصة ناجحة
        │
        ▼
هل المستخدم يملك pricing.viewSuggestedPrice؟
        │
   لا ──► تحويل مباشر (Prototype behavior — fallback)
        │
   نعم  ▼
جلب SuggestedPrice + PriceConfidenceState لكل منتج في الفرصة
        │
        ▼
عرض Price Selector: السعر الاسترشادي (مع badge) + حقل سعر يدوي
        │
        ▼
هل البيانات قديمة (stale) أو Override أو سعر يدوي خارج السياسة؟
        │
   نعم ──► فحص pricing.approvePriceException
           └─ لا صلاحية → رفض + PRC_PRICE_EXCEPTION_REQUIRED
           └─ يملك → سبب إجباري → متابعة
        │
   لا   ▼
حفظ PriceSnapshot على البروفورما → تنفيذ التحويل
```

### PriceSnapshot Schema (يُحفَظ على البروفورما)

```typescript
proforma.priceSnapshot = {
  suggestedPrice:       number,
  confidenceState:      'fresh' | 'stale' | 'no_purchase',
  costOverridePresent:  boolean,
  selectedPrice:        number,      // ما اختاره المستخدم
  selectedBy:           string,      // userId
  selectedAt:           DateTime,
  exceptionApproved:    boolean,
  exceptionReason?:     string,
}
```

### Error Codes الخاصة بـ Pricing Integration في CRM

| الكود | السياق | HTTP |
|---|---|---|
| `PRC_PRICE_EXCEPTION_REQUIRED` | تحويل فرصة بسعر يستوجب اعتماد استثناء | 403 |
| `PRC_NO_COST_DATA` | لا يوجد سعر استرشادي للمنتج | 422 |
| `CRM_PROFORMA_PRICE_REQUIRED` | المستخدم لم يختر سعراً قبل التحويل | 422 |

---

## Notifications

- **الحالة الحالية:** تنبيهات داخل التطبيق فقط — لا بريد إلكتروني ولا واتساب.
- أمثلة موجودة: مهمة متأخرة، عميل محتمل يبرد (>14 يوم)، فرصة تجاوزت موعد الإغلاق، عميل بمخاطر فقد مرتفعة.
- **فجوة موثَّقة:** "Notification Center (Email/WhatsApp)" — تحتاج قرار تصميم منفصل.
- أي قناة إشعار جديدة تُبنى فوق Domain Events أدناه، لا منطق منفصل.

---

## Audit Rules

**يُسجَّل حالياً (عبر `audit()`):**
- إنشاء/تعديل عميل (قبل/بعد التعديل).
- تحويل Lead إلى عميل.
- إعادة توزيع ملكية عميل.
- كل نشاط CRM مسجَّل.

**⚠️ لا يُسجَّل حالياً:**
- حذف Lead (`delLead`) — KI-007.
- تغييرات مراحل الفرصة.
- اختيار السعر الاسترشادي وحفظ PriceSnapshot *(يُسجَّل عبر `pricing.price_snapshot_saved` في Pricing Audit — انظر `Pricing_Implementation_Guide.md`)*.

سجل التدقيق قابل للتلاعب في Prototype (KI-004) — لا يُعتمَد عليه قبل بناء Backend غير قابل للتعديل.

---

## Domain Events

| الحدث | يُطلَق عند | الحمولة الأساسية |
|---|---|---|
| `crm.lead.created` | إنشاء Lead جديد | `leadId, owner, source, estValue, cur` |
| `crm.lead.stage_changed` | تغيّر `stage` لأي Lead | `leadId, fromStage, toStage, owner` |
| `crm.lead.converted` | نجاح `convertLeadToCustomer` | `leadId, customerCode, owner` |
| `crm.opportunity.created` | إنشاء فرصة | `oppId, custCode, owner, value, cur, stage` |
| `crm.opportunity.stage_changed` | أي تغيّر مرحلة | `oppId, fromStage, toStage, owner` |
| `crm.opportunity.won` | الوصول لـ `won` | `oppId, custCode, value, cur, owner` |
| `crm.opportunity.lost` | الوصول لـ `lost` | `oppId, custCode, value, cur, lostReason, owner` |
| `crm.opportunity.converted_to_proforma` | نجاح التحويل لبروفورما مع PriceSnapshot | `oppId, proformaId, custCode, selectedPrice, confidenceState` |
| `crm.customer.duplicate_blocked` | رفض حفظ بسبب INV-01 | `attemptedName, country, existingCode, existingOwner, blockedBy` |
| `crm.customer.ownership_reassigned` | نجاح `crmReassign` | `customerCode, fromOwner, toOwner, reassignedBy` |
| `crm.activity.logged` | أي `crmLog()` ناجح | `activityId, refType, refId, type, by` |
| `crm.task.created` | إنشاء مهمة | `taskId, refType, refId, due, priority, owner` |
| `crm.task.completed` | إتمام مهمة | `taskId, owner` |
| `crm.lead.deleted` | نجاح `delLead` | `leadId, deletedBy, wasConverted` |
| `crm.customer.health_band_changed` *(مقترح — يحتاج Job مجدوَل)* | عبور نطاق الصحة | `customerCode, fromBand, toBand, score` |
| `crm.customer.churn_risk_escalated` *(مقترح — يحتاج Job مجدوَل)* | عبور `churnRisk.level` لـ `high` | `customerCode, risk, level` |

> ⚠️ آخر بندين غير موجودَين كأحداث فعلية في الـ Prototype — الحساب "عند الطلب" فقط، يحتاجان Job مجدوَل عند البناء.

---

## Standard Error Codes

| الكود | الوصف | الموقف في Prototype | HTTP |
|---|---|---|---|
| `CRM_CUSTOMER_DUPLICATE_OWNED` | عميل بنفس الاسم+الدولة محجوز لمندوب آخر | `editCustomer` → "محجوز لـ..." | 409 |
| `CRM_CUSTOMER_DUPLICATE_UNOWNED` | تحذير تكرار غير حاسم | `editCustomer` → "يوجد عميل بنفس الاسم" | 200 + warning |
| `CRM_LEAD_ALREADY_CONVERTED` | محاولة تحويل Lead محوَّل | `convertLeadToCustomer` → "تم تحويله مسبقاً" | 409 |
| `CRM_CUSTOMER_NOT_OWNED` | مندوب يحاول تعديل عميل غير مملوك له | `editCustomer` (حارس `customerSaveGuard`) | 403 |
| `CRM_REASSIGN_MANAGER_ONLY` | محاولة إعادة توزيع من غير مدير | `crmReassign` → "للمدير فقط" | 403 |
| `CRM_LEAD_NAME_REQUIRED` | حفظ Lead بدون اسم | `editLead` → "أدخل الاسم" | 422 |
| `CRM_CUSTOMER_NAME_REQUIRED` | حفظ عميل بدون اسم | `editCustomer` → "أدخل اسم العميل" | 422 |
| `CRM_ENTITY_NOT_FOUND` | Lead/Opportunity غير موجود | فحوصات `if(!o)return` المتكررة | 404 |
| `CRM_PROFORMA_PRODUCT_UNMATCHED` | تحويل فرصة بدون تطابق كود منتج | `crmOppToProforma` → تنبيه | 200 + warning |
| `CRM_PROFORMA_PRICE_REQUIRED` | لم يختر المستخدم سعراً قبل التحويل | (Production-only) | 422 |
| `PRC_PRICE_EXCEPTION_REQUIRED` | سعر يستوجب `approvePriceException` | (Production-only — Pricing Integration) | 403 |
| `CRM_LEAD_DELETE_CONVERTED_BLOCKED` *(مقترح — OQ-5)* | حذف Lead بحالة `converted` | لا يوجد مكافئ حالياً | 409 |
| `CRM_OPPORTUNITY_STAGE_TRANSITION_BLOCKED` *(مقترح — ADR-015)* | انتقال مرحلة غير مسموح | لا يوجد مكافئ حالياً | 409 |

---

## Open Questions (خاصة بالدمج التقني)

1. هل نطاق Audit يشمل `crm.opportunity.stage_changed` عموماً، أم فقط عند تجاوز ADR-015؟
2. هل نعتمد `health_band_changed` و`churn_risk_escalated` (يحتاجان Job مجدوَل)؟

---

## Cross-references

### Related ADRs
- `docs/02_Governance/ADR/ADR-015.md` — تقييد الانتقالات (Proposed)
- `docs/02_Governance/ADR/ADR-018.md` — نموذج صلاحيات التسعير (صلاحيات Pricing Integration)
- `docs/02_Governance/ADR/ADR-019.md` — Domain Event (يؤثر على `crm.opportunity.converted_to_proforma`)

### Depends On
- `docs/03_Business_Logic/CRM/CRM.md` — منطق الأعمال (المرجع الأعلى)
- `docs/03_Business_Logic/Pricing/Pricing_Implementation_Guide.md` — PriceSnapshot Schema، Staleness Logic
- `docs/01_Standards/API_Standards.md` — اصطلاح أكواد الخطأ
- `docs/02_Governance/Known_Issues.md` — KI-002, KI-004, KI-007
