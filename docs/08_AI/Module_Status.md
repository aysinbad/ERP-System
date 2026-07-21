# Module Status — Detailed History

## Document Information
```
Document Name:  Module Status (Detailed)
Version:        1.1.0 (تحديث Pricing + CRM — ميزة السعر الاسترشادي)
Status:         Active
Classification: Reference
Owner:          Solution Architecture Team
Last-Updated:   2026-07-21
```

> **الغرض:** سجلّ تفصيلي لحالة كل موديول وأبرز اكتشافاته. يُقرأ عند الحاجة للسياق التاريخي.
> للحالة الحالية السريعة: انظر `AI_CONTEXT.md`.

---

## Documentation Status Dashboard

| الموديول | حالة التوثيق | ملاحظة |
|---|---|---|
| **CRM** | 🟡 In Review | لا يُعتمد قبل حسم ADR-015 · **Pricing Integration مضاف (#12, #13)** |
| **Pricing** | ✅ Approved | OQ-1 مغلق · ميزة السعر الاسترشادي · `docs/04_Policies/Pricing_Policy.md` |
| **Sales / Export** | 🟢 Complete — Pending Approval | المحتوى كامل، لم يُعتمد رسمياً بعد |
| **Inventory & Production** | 🟢 Complete — Pending Approval | المرحلتان مكتملتان، لم يُعتمد رسمياً بعد |
| **Accounting** | 🔵 In Progress — Session 2/6 | جلستان مكتملتان (2026-07-21) |
| **HR** | ⬜ Not Started | آخر موديول في الطابور |

---

## Pricing (Approved)

**الجلسات:** 6/6 مكتملة. الحالة: `Approved`.

**تحديث 2026-07-21 — ميزة السعر الاسترشادي السري:**
- `Pricing.md` v1.3.0: Entities #8 (SuggestedPrice) · #9 (PriceConfidenceState) · #10 (CostOverride) · Business Rules §Suggested Price & CRM Integration (6 قواعد).
- **OQ-1 مغلق** — بالصياغة الصحيحة: CRM يعرض السعر الاسترشادي كمرجع اختياري مستقل في **Price Guidance Panel**، البروفورما حرة في استخدام `unitPrice` مختلف.
- `Pricing_Implementation_Guide.md` v1.1.0: Permissions Matrix (5 صلاحيات) · Cost Override Audit Entity · Staleness Display Logic · Price Guidance Record Schema.
- `Pricing_RFC.md` v1.1.0: RFC-PRC-004 (Future Enhancement) · RFC-ACC-001 بُعد Pricing→Proforma محسوم.
- `docs/04_Policies/Pricing_Policy.md`: جديد — سياسة إدارية بلا أرقام ثابتة ولا أسماء برمجية.
- **ADR-018 موسَّع:** 5 صلاحيات · `approvePriceException` بدل `approveLowMargin` · Migration Note.

**اكتشافات جوهرية سابقة (لا تزال صالحة):**
- باگ مؤكَّد في BOM متداخل (PINV-07) — غير مُفعَّل بالبيانات الحالية.
- ADR-017/018: `Proposed` — لم يُعتمدا رسمياً (قاعدة مستويات الاعتماد).
- RFC-PRC-003 رُقِّي إلى RFC-ACC-001.

**مفاهيم جوهرية للحفظ:**
- السعر الاسترشادي = مرجع غير ملزم (`Price Guidance Panel`) — لا يُنسَخ لسعر البروفورما.
- `approvePriceException` تُطبَّق على مخالفة السياسة (هامش، خصم، قِدَم، Override) — لا على الانحراف عن السعر الاسترشادي.
- `suggestedPriceAtDecision` = metadata للتدقيق فقط — لا يُعرَض للعميل ولا يؤثر على حسابات.

---

## CRM (In Review)

الحالة: `In Review`. لا يُعتمد قبل حسم ADR-015 (وADR-016 اختيارياً).

**تحديث 2026-07-21:**
- `CRM.md` v2.2.0: Business Rule #3 محدَّثة · **Business Rules #12 و#13 مضافتان** (Pricing Visibility · Price Guidance Panel).
- `CRM_Implementation_Guide.md` v1.2.0: **قسم Pricing Integration كامل** (Price Guidance Panel flow · PriceGuidanceRecord Schema · Error Codes · Domain Events).
- الربط مع ADR-018 (`viewSuggestedPrice` · `approvePriceException`) موثَّق.

---

## Sales / Export (Complete — Pending Approval)

**الأقسام:** 10/10 مكتملة. `Sales_Export_Test_Cases.md`: 56 حالة اختبار. `Sales_Export_Implementation_Guide.md`: مكتمل.

**🔴 أخطر اكتشاف — KI-009:** أمر التصدير (`order → invoice`) لا يُمنع من التكرار — مستخدم عادي يقدر ينشئ **فاتورة تجارية مكرّرة فعلية**.

**اكتشافات أخرى:**
- `delSalesDoc` بلا صلاحية أو فحص حالة (KI-008).
- تأجيل الإيراد EAS 48 مرصود في الكود صراحةً.
- `DAT` غير متاح للاختيار رغم وجوده في قاعدة التأجيل (SINV-07).

**RFC-SLE-001:** موثَّق في ADR-019 (Proposed). **RFC-SLE-002:** تحقيق تقني مكتمل — قرار تجاري معلَّق.

---

## Inventory & Production (Complete — Pending Approval)

**المرحلتان:** مكتملتان بالكامل.

**إجابات نهائية مؤكَّدة:**
1. `stock_reserved` بلا أي أثر فعلي على المخزون (IINV-01).
2. RFC-SLE-002: لا يوجد أي أساس تقني للشحن الجزئي — قرار تجاري معلَّق.
3. مصدر `supplierWaste()` تأكَّد بالكامل (BOM/Yield).

**فجوات جديدة:** IINV-03/04/05 — سُجِّلت كـ RFC-INV-001/002/003.

---

## Accounting (In Progress — Session 2/6)

**جلسة 1 (2026-07-21):** Framework + Catalog (27 مصدر، 11 section) + Dependencies.
**جلسة 2 (2026-07-21):** Accounting Principles & Timing — 7 محاور.

**RFC-ACC-001:** Partially Resolved — بُعد Pricing→Proforma محسوم ✅ · Inventory وAccounting مفتوحان.
**مرشّح للتحقق (جلسة 5):** مخصص مكافأة نهاية الخدمة (5430/2160).

---

## HR (Not Started)
