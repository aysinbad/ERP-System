# Cross-Reference Update — RFC-ACC-001

```
Date:     2026-07-21
Scope:    إضافة cross-references لـ RFC-ACC-001 في 3 ملفات
Trigger:  Accounting Session 1 — إنشاء RFC-ACC-001 Canonical
```

## ملفات في هذه الحزمة

| الملف | النوع | التغيير |
|---|---|---|
| `docs/03_Business_Logic/Pricing/Pricing.md` | **استبدال كامل** | v1.1.0 → v1.2.0 |
| `docs/08_AI/AI_CONTEXT.md` | **استبدال كامل** | v1.0.0 → v1.1.0 |
| `docs/03_Business_Logic/Inventory_Production/Inventory_Production.md` | **patch فقط** | إضافة سطر واحد في Cross-references |

## التغييرات بالتفصيل

### Pricing.md (v1.2.0)
- **ترويسة الملف:** إشارة إلى أن RFC-PRC-003 رُقِّي إلى RFC-ACC-001
- **Price Snapshot Policy:** جملة القرار أصبحت "رُقِّي إلى RFC-ACC-001 (Cross-Module, Canonical)"
- **Open Question #4:** إضافة "(Cross-Module: Accounting + Inventory)" وإشارة صريحة لـ RFC-ACC-001
- **RFC References table:** RFC-PRC-003 حالته = "رُقِّي إلى RFC-ACC-001"
- **Cross-references:** سطر جديد لـ `Accounting_RFC.md → RFC-ACC-001`

### AI_CONTEXT.md (v1.1.0)
- **Verified Cross-Module Discoveries:** صف جديد لـ "Snapshot vs Recalculation" + تحديث صف Accounting
- **قسم جديد: RFCs المفتوحة** (RFC-ACC-001 · RFC-PRC-003 مرقَّى · RFC-SLE-002)
- **Pricing section:** تحديث جملة RFC-PRC-003 لتعكس الترقّي
- **الوحدة الجاري العمل عليها:** أصبحت Accounting (جلسة 1 مكتملة) بدل Inventory
- **باقي القائمة:** HR فقط

### Inventory_Production.md
- سطر واحد مضاف في Cross-references: `RFC-ACC-001 forward-reference`
- ⚠️ هذا الملف في الحزمة يحتوي على قسم Cross-references المحدَّث فقط — **لا تستبدل الملف كله**، بل استبدل قسم Cross-references فقط بما هو موجود هنا.

## git commit مقترح
```bash
git add docs/03_Business_Logic/Pricing/Pricing.md
git add docs/08_AI/AI_CONTEXT.md
git add docs/03_Business_Logic/Inventory_Production/Inventory_Production.md
git commit -m "docs: cross-reference RFC-ACC-001 (Cross-Module, Canonical) from Pricing + Inventory + AI_CONTEXT"
```
