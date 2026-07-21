# Verified Cross-Module Discoveries

## Document Information
```
Document Name:  Cross-Module Discoveries
Version:        1.0.0
Status:         Active
Classification: Reference
Owner:          Solution Architecture Team
Last-Updated:   2026-07-21
```

> اكتشافات مؤكَّدة من الكود أثّرت على أكثر من موديول. كل بند يشاور على مصدره الكامل.
> للحالة الحالية السريعة: انظر `AI_CONTEXT.md`.

---

| الاكتشاف | الموديولات المتأثرة | الحالة |
|---|---|---|
| **`supplierWaste()`** — مصدرها أوامر التشغيل (BOM/Yield)، تُستهلَك في التسعير | Pricing ← Inventory | ✅ مؤكَّد (`Inventory_Production.md` Entities #3) |
| **Partial Shipment** — لا أساس تقني (لا حجز، لا تتبع كمية جزئية) | Sales/Export ← Inventory | 🟢 تحقيق مكتمل، قرار تجاري معلَّق (RFC-SLE-002) |
| **Stock Reservation** — `stock_reserved` بلا أثر فعلي على المخزون | Sales/Export ← Inventory | ✅ مؤكَّد نهائياً (IINV-01) |
| **لا سعر معتمَد يُستهلَك** — `crmOppToProforma` يستخدم `Product.price` الثابت | CRM ← Pricing | ✅ مؤكَّد (Module Dependencies للاثنين) |
| **استدعاء مباشر بدل Domain Event** — CRM يستدعي كود Sales مباشرة | CRM → Sales/Export | 🟡 موثَّق في ADR-019 (Proposed) |
| **تأجيل الإيراد (EAS 48)** — `DAT` غير قابل للاختيار من أي شاشة | Sales/Export → Accounting | 🟡 الشرط موثَّق، التنفيذ المحاسبي قيد التوثيق (Accounting جلسة 3) |
| **Snapshot vs Recalculation** — القيم المشتقّة تُعاد حسابها بدل تجميدها | Pricing · Inventory · Accounting | 🟡 RFC-ACC-001 (Cross-Module, Canonical) — `Accounting_RFC.md` |
