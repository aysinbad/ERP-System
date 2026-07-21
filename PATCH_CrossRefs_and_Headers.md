# Patches — طلبات 3 و7 (ملفات خارج قرص Claude)

> هذه تعديلات يدوية على ملفات في ريبوك. طبّق كل قسم على الملف المذكور.
> التغييرات: إعادة ترتيب فقط — لا تعديل في محتوى قواعد الأعمال.

---

## طلب 3 — توحيد Cross-references

**الترتيب المعتمد لكل ملف:**
```
### Related ADRs
### Related RFCs
### Depends On
### Referenced Modules
### Implementation Guide
### Test Cases
```

---

### `docs/03_Business_Logic/Pricing/Pricing.md`

**استبدل قسم `## Cross-references` الحالي بـ:**

```markdown
## Cross-references

### Related ADRs
- `docs/02_Governance/ADR/ADR-017.md` — مصادر مكوّنات التكلفة (يحسم RFC-PRC-001)
- `docs/02_Governance/ADR/ADR-018.md` — صلاحية التكلفة + حد الهامش (يحسم RFC-PRC-002)

### Related RFCs
- `docs/03_Business_Logic/Accounting/Accounting_RFC.md → RFC-ACC-001` (Cross-Module, Canonical) — PINV-12 قرينة عليه
- `docs/03_Business_Logic/Pricing/Pricing_RFC.md` — RFC-PRC-001 ✅ · RFC-PRC-002 ✅ · RFC-PRC-003 (رُقِّي لـ RFC-ACC-001)

### Depends On
- `docs/03_Business_Logic/CRM/CRM.md` — Business Rules #7 (اشتقاق الملكية للعمولة)
- `docs/00_Project/Current_State.md` — ذكر "ذكاء التكلفة" و"سلّة تسعير"
- `experiments/pricing-engine-poc.html` — نموذج تجريبي على نفس صيغة `pricingCalc`

### Referenced Modules
- `docs/03_Business_Logic/CRM/CRM.md` — Business Rules #3 (تحويل الفرصة لبروفورما)

### Implementation Guide
- `docs/03_Business_Logic/Pricing/Pricing_Implementation_Guide.md`

### Test Cases
- `docs/03_Business_Logic/Pricing/Pricing_Test_Cases.md`
```

---

### `docs/03_Business_Logic/Inventory_Production/Inventory_Production.md`

**استبدل قسم `## Cross-references` الحالي بـ:**

```markdown
## Cross-references

### Related ADRs
- `docs/02_Governance/ADR/ADR-005.md` — أوامر التشغيل بوضعين (BOM vs Yield)

### Related RFCs
- `docs/03_Business_Logic/Accounting/Accounting_RFC.md → RFC-ACC-001` (Cross-Module, Canonical) — تكلفة الوحدة الديناميكية قرينة عليه
- `docs/03_Business_Logic/Sales_Export/Sales_Export_RFC.md → RFC-SLE-002` — الشحن الجزئي (IINV-06 يوفّر الحقيقة التقنية)

### Depends On
- `docs/03_Business_Logic/Pricing/Pricing.md` — `unitCost`/`supplierWaste` مصدرهما موثَّق هنا

### Referenced Modules
- `docs/03_Business_Logic/Sales_Export/Sales_Export.md` — SINV-06 (الشحن الجزئي)

### Implementation Guide
- `docs/03_Business_Logic/Inventory_Production/Inventory_Production_Implementation_Guide.md` _(Placeholder)_

### Test Cases
- `docs/03_Business_Logic/Inventory_Production/Inventory_Production_Test_Cases.md` _(Placeholder)_
```

---

### `docs/03_Business_Logic/Sales_Export/Sales_Export.md`

**استبدل قسم `## Cross-references` الحالي بـ:**

```markdown
## Cross-references

### Related ADRs
- `docs/02_Governance/ADR/ADR-019.md` — استبدال الاستدعاء المباشر CRM→Sales بـ Domain Event (يحسم RFC-SLE-001)

### Related RFCs
- `docs/03_Business_Logic/Sales_Export/Sales_Export_RFC.md` — RFC-SLE-001 ✅ محسوم · RFC-SLE-002 🟢 قرار معلَّق

### Depends On
- `docs/03_Business_Logic/CRM/CRM.md` — Module Dependencies (استدعاء مباشر موثَّق)
- `docs/03_Business_Logic/Pricing/Pricing.md` — فجوة "لا سعر معتمَد يُستهلَك"
- `docs/02_Governance/Known_Issues.md` — KI-008 (حذف مستند بلا صلاحية) · KI-009 (فاتورة مكرّرة)

### Referenced Modules
- `docs/03_Business_Logic/Inventory_Production/Inventory_Production.md` — IINV-01 (stock_reserved) · IINV-06 (شحن جزئي)

### Implementation Guide
- `docs/03_Business_Logic/Sales_Export/Sales_Export_Implementation_Guide.md`

### Test Cases
- `docs/03_Business_Logic/Sales_Export/Sales_Export_Test_Cases.md`
```

---

### `docs/03_Business_Logic/CRM/CRM.md`

**استبدل قسم `## Cross-references` الحالي بـ:**

```markdown
## Cross-references

### Related ADRs
- `docs/02_Governance/ADR/ADR-004.md` — ملكية العملاء في CRM
- `docs/02_Governance/ADR/ADR-015.md` — تقييد Won/Lost (Proposed)
- `docs/02_Governance/ADR/ADR-016.md` — CRM Enterprise Enhancements (Proposed)
- `docs/02_Governance/ADR/ADR-019.md` — استبدال الاستدعاء المباشر بـ Domain Event (Proposed)

### Related RFCs
- `docs/03_Business_Logic/CRM/CRM_RFC.md` (إن وُجد)

### Depends On
- `docs/02_Governance/Known_Issues.md` — KI-007 (delLead بلا صلاحية)

### Referenced Modules
- `docs/03_Business_Logic/Pricing/Pricing.md` — Business Rules #7 (العمولة تعتمد على ملكية CRM)
- `docs/03_Business_Logic/Sales_Export/Sales_Export.md` — ADR-019 (Domain Event)

### Implementation Guide
- `docs/03_Business_Logic/CRM/CRM_Implementation_Guide.md`

### Test Cases
- `docs/03_Business_Logic/CRM/CRM_Test_Cases.md` _(إن وُجد)_
```

---

## طلب 7 — توحيد Header Structure

**الترتيب المعتمد لكل ملف Business Logic:**
```
1. Document Information (+ Session Map إن وُجد + Progress إن وُجد)
2. Module Dependencies
3. Purpose
4. Scope
5. Out of Scope
6. Actors
7. Entities
8. Business Rules
9. Business Invariants
10. Reports
11. Open Questions
12. Future Enhancements
13. Cross-references
```

**ملاحظات per-file:**
- `CRM.md` و`Pricing.md`: تحقق من وجود `Future Enhancements` — أضفه كـ `_(لا شيء بعد.)_` إن غاب.
- `Sales_Export.md`: تحقق من ترتيب `Reports` قبل `Open Questions`.
- `Inventory_Production.md`: نفس التحقق.
- الحقل `Progress: X/Y Sessions (Z%)` يُضاف فقط للموديولات ذات خطة جلسات (Accounting حالياً).
