# Pricing — RFCs

## Document Information
```
Document Name:  Pricing RFCs
Version:        1.5.0 (تصحيح: RFC-ACC-001 بثلاثة أبعاد مفتوحة في RFC-PRC-003 والملاحظة الختامية)
Status:         Active
Classification: Reference
Owner:          Solution Architecture Team
Last-Updated:   2026-07-21
```

> RFCs تُستكشَف هنا عمقاً (خيارات/مقارنة/تريد أوف) قبل ترقيتها لـ ADR رسمي. لو استقر RFC، يُعلَّم كمكتمل هنا (لا يُحذَف — للتتبع التاريخي).

---

## RFC-PRC-001 — إعادة تصميم مصادر مكوّنات التكلفة
**الحالة:** ✅ مكتمل — انظر `ADR-017.md`

---

## RFC-PRC-002 — تصميم مسار موافقة مخصَّص للتسعير
**الحالة:** ✅ مكتمل — انظر `ADR-018.md`

---

## RFC-PRC-003 — سياسة تجميد القيم المُشتقّة (السعر + العمولة)
**الحالة:** 🟡 رُقِّي إلى RFC-ACC-001 (Cross-Module, Canonical) في `Accounting_RFC.md`.

**ملاحظة (2026-07-21):** بُعد واحد من هذا RFC تم حسمه ضمن ميزة السعر الاسترشادي السري:
- **محسوم — Pricing Guidance at Proforma Decision:** `unitPrice` الفعلي على كل بند في البروفورما يُدخَل مستقلاً — لا يعتمد على السعر الاسترشادي ولا يُنسَخ منه تلقائياً. السعر الاسترشادي يُخزَّن **شرطياً** كـ `PriceGuidanceRecord` (metadata للتدقيق فقط، على مستوى كل بند، فقط عند عرضه بنجاح) — وليس السعر التجاري المختار.
- **لا يزال مفتوحاً في RFC-ACC-001:**
  - **Commission Recalculation** — نمط إعادة الاشتقاق في حساب العمولة (`commissionRows`، PINV-12).
  - **Inventory Unit Cost** — تكلفة الوحدة المُشتَقّة ديناميكياً وقت الطلب.
  - **Accounting Derived Values** — نصيب المصروف العام والمخصصات المحسوبة من الحالة الحالية.

---

## RFC-PRC-004 — تحويل Cost Override إلى تكلفة رسمية معتمدة

```
RFC:          RFC-PRC-004
Title:        Cost Override → Official Product Cost
Type:         Cross-Module
Scope:        Pricing · Inventory
Status:       Future Enhancement — لم يُفتَح للنقاش بعد
Opened:       2026-07-21
```

### السياق
قرار ميزة السعر الاسترشادي السري (2026-07-21) يحدد Cost Override كـ **تعديل مؤقت لعملية تسعير واحدة فقط** — لا يغيّر التكلفة الرسمية في المخزون أو المحاسبة.

**السؤال المفتوح:** هل يُراد في المستقبل السماح بترقية Override معتمد ليصبح تكلفة رسمية للمنتج؟

### لماذا هو Future Enhancement وليس قراراً الآن
- يمسّ موديولين: Pricing (مصدر القرار) + Inventory (المكلَّف بالتكلفة الرسمية).
- يحتاج نقاش أعمق حول: هل "التكلفة الرسمية" تُحدَّث بقرار مالي أم بحدث شراء؟
- الحل الجزئي الحالي (Override مؤقت) يغطي 90%+ من حالات الاستخدام الفعلي.
- ربط محتمل بـ RFC-ACC-001 (Snapshot Policy) — لا يُحسَم قبل جلسة 5 من Accounting.

### الحالة
`RFC-PRC-004` يمكن فتحه للنقاش بعد:
1. تأكيد قواعد ملكية التكلفة (Cost Ownership) ذات الصلة من توثيق Inventory المكتمل.
2. حسم تبعات Snapshot vs Recalculation في RFC-ACC-001 بالقدر الكافي.
3. تحديد Product Owner صراحةً أولوية ترقية Cost Override المؤقت لتكلفة رسمية معتمدة.

---

## ملاحظة: RFC-ACC-001 وبُعد Pricing Guidance at Proforma Decision

> للسجل فقط — المصدر القانوني الكامل في `Accounting_RFC.md`.

بُعد "Pricing Guidance at Proforma Decision" من RFC-ACC-001 **محسوم** بقرار ميزة السعر الاسترشادي:
- `proforma.items[].unitPrice` هو السعر التجاري الفعلي — يُدخَل مستقلاً بلا اعتماد على السعر الاسترشادي.
- السعر الاسترشادي metadata اختيارية للتدقيق فقط (`PriceGuidanceRecord`، على مستوى كل بند) — **تُنشَأ فقط عند عرض السعر الاسترشادي فعلياً**؛ لا سجل عند غياب الصلاحية أو عدم توفر السعر.
- البروفورما لا تعتمد على السعر الاسترشادي ولا تُعيد حسابه.
- Open Question #1 في `Pricing.md` أُغلق بناءً على هذا القرار.

RFC-ACC-001 **يبقى مفتوحاً** لأبعاد Commission Recalculation وInventory Unit Cost وAccounting Derived Values.
