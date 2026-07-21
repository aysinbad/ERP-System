# Accounting Engine — RFCs

## Document Information
```
Document Name:  Accounting Engine RFCs
Version:        0.1.0
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
Status:       Open
Scope:        Accounting · Pricing · Inventory
Owner:        Solution Architecture Team
Opened:       2026-07-21
Referenced-by: Pricing.md · Inventory_Production.md
Expected-outcome: Cross-Module ADR
```

> **هذا هو المصدر القانوني الوحيد (Canonical) لهذا السؤال.** أي موديول آخر يمسّه يُشير إليه هنا عبر cross-reference، ولا يفتح نقاشاً موازياً — تفادياً لثلاثة قرارات متعارضة لنفس المشكلة الجذرية.

### Problem (المشكلة الجذرية)
هل يُجمِّد النظام القيمة المشتقّة **لحظة الحدث** (Snapshot)، أم يعيد حسابها من **الحالة الحالية للنظام** كلما طُلبت (Recalculation)؟

النمط الحالي في الـ Prototype هو **Recalculation** في عدة مواضع — القيمة تُشتقّ حيّةً وقت العرض/الطلب، بلا لقطة محفوظة. الأثر: نفس السؤال التاريخي ("ليه كانت القيمة كذا يوم كذا؟") قد لا يكون له جواب قابل لإعادة البناء.

### Scope — لماذا هي عابرة للموديولات
| الموديول | مظهر المشكلة | القرينة |
|---|---|---|
| **Pricing** | السعر المقترح والعمولة يُعاد اشتقاقهما بأثر رجعي من الحالة الحالية | `Pricing.md` — PINV-12 + Price Snapshot Policy (Open Question #5) |
| **Inventory** | تكلفة الوحدة تُقيَّم ديناميكياً وقت الطلب، لا تُخزَّن كرقم ثابت | تُوثَّق عند استخراج موديول Inventory (رقم 3 في الطابور) |
| **Accounting** | نصيب المصروف العام على الشحنة + المخصصات محسوبة على حالة النظام الحالية | `Accounting.md` — Open Questions #2 |

### Options (لا تُحلَّل الآن — تُفتَح في جلسة 5)
تُترَك فارغة عمداً حتى يكتمل مسح الحالات المحاسبية في جلسة 5؛ ملء الخيارات قبل رؤية كل المظاهر يُنتج توصية مبتسرة. (مبدأ: لا قرار قبل اكتمال الاستخراج.)

### Governance Note
النطاق عابر للموديولات ⇒ المخرَج المتوقَّع **Cross-Module ADR** لا قرار محلي. لو تكرّر نمط "الـ RFC العابر للموديولات" مستقبلاً، يُنظَر في إنشاء `02_Governance/Cross_Module_RFC/` كموطن قانوني بدل حشره داخل موديول — قرار بنية توثيق مؤجَّل لك كـ architect.

---

## مرشّحات أخرى
_(لا شيء بعد — تُضاف عند ظهورها في الجلسات 2→5.)_
