# Pricing — RFCs

## Document Information
```
Document Name:  Pricing RFCs
Version:        1.2.0 (توحيد مصطلحات PriceGuidanceRecord + RFC-PRC-004 Cross-Module)
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
- **محسوم:** `unitPrice` الفعلي على كل بند في البروفورما يُدخَل مستقلاً — لا يعتمد على السعر الاسترشادي ولا يُنسَخ منه تلقائياً. السعر الاسترشادي يُخزَّن اختيارياً كـ `PriceGuidanceRecord` (metadata للتدقيق فقط، على مستوى كل بند).
- **لا يزال مفتوحاً في RFC-ACC-001:** نمط Recalculation في حساب العمولة (`commissionRows`) وتكلفة الوحدة في Inventory.

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
لا يُفتَح النقاش التفصيلي قبل:
1. اكتمال توثيق Inventory & Production.
2. حسم RFC-ACC-001 في جلسة 5 من Accounting.

---

## ملاحظة: RFC-ACC-001 وبُعد Pricing→Proforma Snapshot

> للسجل فقط — المصدر القانوني الكامل في `Accounting_RFC.md`.

بُعد "Pricing→Proforma Snapshot" من RFC-ACC-001 **محسوم** بقرار ميزة السعر الاسترشادي:
- السعر المختار قبل التحويل يُحفَظ Snapshot على البروفورما.
- البروفورما لا تعيد الحساب من `pricingCalc`.
- Open Question #1 في `Pricing.md` أُغلق بناءً على هذا القرار.

RFC-ACC-001 **يبقى مفتوحاً** لبُعدَي Inventory وAccounting.
