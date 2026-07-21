# Sales / Export — RFC

## Document Information
```
Document Name:  Sales / Export RFC
Version:        0.1.0
Status:         Active
Classification: Working Draft
Owner:          Solution Architecture Team
Last-Updated:   2026-07-18
```

---

## RFC-SLE-001 — استبدال الاستدعاء المباشر لـ CRM→Sales بـ Domain Event
**مصدره:** `Sales_Export.md` Module Dependencies — ملاحظة معمارية عن `crmOppToProforma`.

**✅ محسوم — انظر `docs/02_Governance/ADR/ADR-019.md` (حالته `Proposed`، بانتظار اعتماد Product Owner).** القرار: CRM ينشر `crm.opportunity.won`، وSales يشترك فيه بدل الاستدعاء المباشر.

---

## RFC-SLE-002 — دعم الشحن الجزئي (Partial Shipment) — الحقيقة التقنية اكتملت، القرار لسه مفتوح
**مصدره:** `Sales_Export.md` SINV-06 + Open Questions #2.

**✅ تحديث من `Inventory_Production.md` (IINV-06):** الفحص التقني اكتمل بالكامل الآن — **لا يوجد أي أساس تقني قائم لدعم الشحن الجزئي** في أي مكان بالنظام (لا حجز مخزون، لا تتبع كمية منتَجة/مشحونة جزئياً). هذا **ليس رأياً، دي حقيقة مؤكَّدة من فحص `DB.stockMoves` و`DB.workOrders` بالكامل**.

**⚠️ لكن القرار نفسه لسه مفتوح — الحقيقة التقنية لا تحسم القرار التجاري:**
- **خيار أ:** نبني آلية شحن جزئي من الصفر (تتبع `shippedQty`، حجز مخزون فعلي) — تكلفة تطوير معتبرة.
- **خيار ب:** نلغي حالة `partially_shipped` من الواجهة نهائياً لتفادي اللبس — الأبسط، لكن يفقد مرونة تجارية لو احتاجها العمل مستقبلاً.

**الحالة:** 🟢 **Technical investigation completed — Awaiting business decision.** التحقيق التقني اكتمل بالكامل (IINV-06)؛ المتبقي قرار تجاري بحت بين خيارين (بناء آلية جديدة / إلغاء الحالة) — **لن يُحسَم افتراضياً بدون طلب صريح**.

---



## قاعدة الإضافة
أي سؤال من `Sales_Export.md` يحتاج نقاشاً أعمق يُنقَل هنا. لو استقر، يترقّى لـ ADR في `02_Governance/ADR/`.
