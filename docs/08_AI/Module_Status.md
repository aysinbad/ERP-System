# Module Status — Detailed History

## Document Information
```
Document Name:  Module Status (Detailed)
Version:        1.0.0
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
| **CRM** | 🟡 In Review | لا يُعتمد قبل حسم ADR-015 (وADR-016 اختيارياً) |
| **Pricing** | ✅ Approved | RFC-PRC-003 رُقِّي إلى RFC-ACC-001 (Cross-Module) |
| **Sales / Export** | 🟢 Complete — Pending Approval | المحتوى كامل، لم يُعتمد رسمياً بعد |
| **Inventory & Production** | 🟢 Complete — Pending Approval | المرحلتان مكتملتان، لم يُعتمد رسمياً بعد |
| **Accounting** | 🔵 In Progress — Session 2/6 | جلستان مكتملتان (2026-07-21) |
| **HR** | ⬜ Not Started | آخر موديول في الطابور |

---

## Pricing (Approved — 2026-07-18)

**الجلسات:** 6/6 مكتملة. الحالة: `Approved`.

**اكتشافات جوهرية:**
- محرك التسعير (`pricingCalc`) منفصل تماماً عن مسار CRM→بروفورما — `crmOppToProforma` يستخدم `Product.price` الثابت.
- باگ مؤكَّد في حساب تكلفة BOM متداخل (PINV-07) — كامن في الكود، غير مُفعَّل بالبيانات الحالية.
- ADR-017/018: `Proposed` — لم يُعتمدا رسمياً (انظر قاعدة مستويات الاعتماد في `AI_CONTEXT.md`).
- RFC-PRC-003 رُقِّي إلى RFC-ACC-001 (Cross-Module, Canonical).

---

## CRM (In Review)

الحالة: `In Review`. لا يُعتمد قبل حسم ADR-015 (وADR-016 اختيارياً).

---

## Sales / Export (Complete — Pending Approval)

**الأقسام:** 10/10 مكتملة. `Sales_Export_Test_Cases.md`: 56 حالة اختبار. `Sales_Export_Implementation_Guide.md`: مكتمل.

**🔴 أخطر اكتشاف — KI-009:** أمر التصدير (`order → invoice`) لا يُمنع من التكرار — لا في الكود ولا في الواجهة. مستخدم عادي يقدر ينشئ **فاتورة تجارية مكرّرة فعلية**.

**اكتشافات أخرى:**
- `delSalesDoc` بلا صلاحية أو فحص حالة (KI-008).
- تأجيل الإيراد EAS 48 مرصود في الكود صراحةً.
- `DAT` غير متاح للاختيار في أي شاشة رغم وجوده في قاعدة التأجيل (SINV-07).

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
**جلسة 2 (2026-07-21):** Accounting Principles & Timing — 7 محاور (Recognition · Historical FX · Opening · Period · Manual vs Auto · General Principles · Source of Truth).

**مرشّح للتحقق (جلسة 5):** مخصص مكافأة نهاية الخدمة (5430/2160).
**RFC-ACC-001:** Canonical Cross-Module RFC — Snapshot vs Recalculation.

---

## HR (Not Started)
