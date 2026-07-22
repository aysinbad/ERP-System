# Accounting Engine — RFCs

## Document Information
```
Document Name:  Accounting Engine RFCs
Version:        0.5.0 (تصحيح: حالة صحيحة تشمل Commission + إزالة صياغة "يُحسَم عند توثيق Inventory" القديمة)
Status:         Active
Classification: Reference
Owner:          Solution Architecture Team
Last-Updated:   2026-07-21
```

> RFCs الخاصة بمحرك المحاسبة. النقاش التفصيلي (خيارات/تريد أوف) يُفتَح عند نضوج السؤال؛ لو استقر، يُرقَّى لـ ADR رسمي في `02_Governance/ADR/` ويُعلَّم هنا كمكتمل (لا يُحذَف — للتتبّع التاريخي).

---

## RFC-ACC-001 — Snapshot vs Recalculation (Cross-Module)

```
RFC:          RFC-ACC-001
Title:        Snapshot vs Recalculation of Derived Values
Type:         Cross-Module
Status:       Partially Resolved — Open for Commission, Inventory, and Accounting dimensions
Scope:        Accounting · Pricing · Inventory
Owner:        Solution Architecture Team
Opened:       2026-07-21
Referenced-by: Pricing.md · Inventory_Production.md
Expected-outcome: Cross-Module ADR
```

> **هذا هو المصدر القانوني الوحيد (Canonical) لهذا السؤال.** أي موديول آخر يمسّه يُشير إليه هنا عبر cross-reference، ولا يفتح نقاشاً موازياً.

### Problem (المشكلة الجذرية)
هل يُجمِّد النظام القيمة المشتقّة **لحظة الحدث** (Snapshot)، أم يعيد حسابها من **الحالة الحالية للنظام** كلما طُلبت (Recalculation)؟

### Resolution Status by Dimension (حالة الحسم لكل بُعد)

| البُعد | الموديول | الحالة | القرار |
|---|---|---|---|
| **Pricing Guidance at Proforma Decision** | Pricing → CRM | ✅ **محسوم (2026-07-21)** | `proforma.items[].unitPrice` هو السعر التجاري الفعلي — يُدخَل مستقلاً بلا اعتماد على السعر الاسترشادي. السعر الاسترشادي **metadata اختيارية للتدقيق فقط** (`PriceGuidanceRecord`). البروفورما لا تعتمد على السعر الاسترشادي ولا تُعيد حسابه. موثَّق في `Pricing.md` Business Rules §Suggested Price & CRM Integration |
| **Commission Recalculation** | Pricing (PINV-12) | 🟡 مفتوح | العمولة تُعاد حسابها من حالة النظام الحالية — Snapshot لم يُقرَّر |
| **Inventory Unit Cost** | Inventory | 🟡 مفتوح | السلوك الحالي موثَّق بالكامل: تكلفة الوحدة تُشتَق ديناميكياً وقت الطلب. القرار المعماري بين Snapshot وRecalculation لا يزال مفتوحاً |
| **Accounting Derived Values** | Accounting | 🟡 مفتوح | نصيب المصروف العام والمخصصات — يُحسَم في جلسة 5 من Accounting |

### Scope — لماذا هي عابرة للموديولات
| الموديول | مظهر المشكلة | القرينة |
|---|---|---|
| **Pricing** | السعر المقترح → **محسوم** (Pricing Guidance at Proforma Decision — metadata فقط، `unitPrice` مستقل) | `Pricing.md` — OQ-1 مغلق |
| **Pricing** | العمولة تُعاد اشتقاقها بأثر رجعي | PINV-12 (لا يزال مفتوحاً — انظر قسم "تجميد القيم المشتقة مقابل إعادة احتسابها" في `Pricing.md`) |
| **Inventory** | تكلفة الوحدة ديناميكية وقت الطلب — السلوك موثَّق بالكامل، القرار المعماري لا يزال مفتوحاً | `Inventory_Production.md` → RFC-ACC-001 (forward-reference) |
| **Accounting** | نصيب المصروف العام + المخصصات من حالة النظام الحالية | `Accounting.md` — Open Questions #2 |

### Options (تُحلَّل في جلسة 5 من Accounting للأبعاد المفتوحة)
تُترَك فارغة عمداً — الأبعاد المفتوحة تحتاج مسح الحالات المحاسبية الكامل أولاً.

### Governance Note
النطاق عابر للموديولات ⇒ المخرَج المتوقَّع **Cross-Module ADR**. نقطة التبلور: جلسة 5 من Accounting بعد اكتمال مسح جميع الحالات.

---

## مرشّحات أخرى
_(لا شيء بعد — تُضاف عند ظهورها في الجلسات 3→5.)_
