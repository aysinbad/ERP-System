# Patches — طلبات 3 و7 (مُحدَّث من project knowledge)

> كل قسم: ابحث عن النص الحالي (في "ابحث عن") واستبدله بـ "استبدل بـ".
> الاستبدال في قسم `## Cross-references` فقط لطلب 3.
> الإضافة قبل `## Cross-references` للأقسام الناقصة (طلب 7).

---

## 1. `docs/03_Business_Logic/CRM/CRM.md`

### طلب 3 — استبدل قسم Cross-references

**ابحث عن:**
```
## Cross-references
- `docs/02_Governance/ADR/ADR-001.md` (فرصة→بروفورما)، `docs/02_Governance/ADR/ADR-004.md` (ملكية العملاء)، `docs/02_Governance/ADR/ADR-015.md` (تقييد الانتقالات — Proposed)
- `docs/01_Standards/Glossary.md`: قسم 1 — مصطلحات دورة البيع (Proforma, Sales Order...)
- `docs/03_Business_Logic/CRM_Implementation_Guide.md`: State Matrix, Permissions, Notifications, Audit, Domain Events, Error Codes
- `docs/02_Governance/Known_Issues.md`: KI-007 (حذف Lead بلا صلاحية وخطر تكامل)
- `docs/00_Project/Current_State.md`: فجوة Notification Center
```

**استبدل بـ:**
```markdown
## Cross-references

### Related ADRs
- `docs/02_Governance/ADR/ADR-001.md` — فرصة→بروفورما (Accepted)
- `docs/02_Governance/ADR/ADR-004.md` — ملكية العملاء: المندوب يسجّل بنفسه (Accepted)
- `docs/02_Governance/ADR/ADR-015.md` — تقييد انتقالات Won/Lost مع تجاوز مبرَّر (Proposed)
- `docs/02_Governance/ADR/ADR-016.md` — CRM Enterprise Enhancements: حالة العميل + دمج + تقييد Lead (Proposed)
- `docs/02_Governance/ADR/ADR-019.md` — استبدال استدعاء CRM→Sales بـ Domain Event (Proposed)

### Related RFCs
- `docs/03_Business_Logic/CRM/CRM_RFC.md` — RFC-CRM-001 ✅ (ADR-015) · RFC-CRM-002 ✅ (ADR-016)

### Depends On
- `docs/01_Standards/Glossary.md` §1 — مصطلحات دورة البيع (Proforma, Sales Order…)
- `docs/02_Governance/Known_Issues.md` — KI-007 (حذف Lead بلا صلاحية وخطر تكامل)
- `docs/00_Project/Current_State.md` — فجوة Notification Center

### Referenced Modules
- `docs/03_Business_Logic/Pricing/Pricing.md` — Business Rules #7 (ملكية العميل تُغذّي حساب العمولة)
- `docs/03_Business_Logic/Sales_Export/Sales_Export.md` — ADR-019 (Domain Event)

### Implementation Guide
- `docs/03_Business_Logic/CRM/CRM_Implementation_Guide.md` — State Matrix · Permissions · Notifications · Audit · Domain Events · Error Codes

### Test Cases
- `docs/03_Business_Logic/CRM/CRM_Test_Cases.md` _(Placeholder — بانتظار اعتماد ADR-015)_
```

### طلب 7 — Header check
CRM.md ✅ يتبع الترتيب الصحيح بالكامل (Module Dependencies → Purpose → Scope → Out of Scope → Actors → Entities → Business Rules → Business Invariants → Reports → Open Questions → Future Enhancements → Cross-references). لا تعديل مطلوب في الترتيب.

---

## 2. `docs/03_Business_Logic/Pricing/Pricing.md`

### طلب 3 — استبدل قسم Cross-references

**ابحث عن:**
```
## Cross-references
- `docs/03_Business_Logic/CRM/CRM.md`: Business Rules #3 (تحويل الفرصة لبروفورما)، Business Rules #7 (اشتقاق الملكية — نفس المنطق يُعاد استخدامه في حساب العمولة)
- `docs/00_Project/Current_State.md`: ذكر "ذكاء التكلفة" و"سلّة تسعير" ضمن الميزات الناضجة
- `experiments/pricing-engine-poc.html`: نموذج تجريبي مبني على نفس صيغة `pricingCalc`
- `docs/03_Business_Logic/Pricing/Pricing_RFC.md`: RFC-PRC-001, RFC-PRC-002
```

**استبدل بـ:**
```markdown
## Cross-references

### Related ADRs
- `docs/02_Governance/ADR/ADR-017.md` — مصادر مكوّنات التكلفة: متوسط تاريخي (Proposed)
- `docs/02_Governance/ADR/ADR-018.md` — صلاحية التكلفة + حد الهامش (Proposed)

### Related RFCs
- `docs/03_Business_Logic/Pricing/Pricing_RFC.md` — RFC-PRC-001 ✅ (ADR-017) · RFC-PRC-002 ✅ (ADR-018) · RFC-PRC-003 🟡 (رُقِّي لـ RFC-ACC-001)
- `docs/03_Business_Logic/Accounting/Accounting_RFC.md → RFC-ACC-001` (Cross-Module, Canonical) — PINV-12 قرينة عليه

### Depends On
- `docs/00_Project/Current_State.md` — ذكر "ذكاء التكلفة" و"سلّة تسعير"
- `experiments/pricing-engine-poc.html` — نموذج تجريبي على نفس صيغة `pricingCalc`

### Referenced Modules
- `docs/03_Business_Logic/CRM/CRM.md` — Business Rules #3 (تحويل فرصة→بروفورما) · #7 (ملكية العميل للعمولة)
- `docs/03_Business_Logic/Inventory_Production/Inventory_Production.md` — مصدر `unitCost`/`supplierWaste`

### Implementation Guide
- `docs/03_Business_Logic/Pricing/Pricing_Implementation_Guide.md`

### Test Cases
- `docs/03_Business_Logic/Pricing/Pricing_Test_Cases.md`
```

### طلب 7 — أضف القسمين الناقصين قبل Cross-references

**أضف هذا النص مباشرةً قبل سطر `## Cross-references` الحالي:**
```markdown
## Reports

> Pricing لا تُنتج تقارير مستقلة — هي طبقة حساب تُغذّي قرارات التسعير. مخرجاتها (السعر المقترح، تفصيل التكلفة) تظهر داخل شاشة التسعير وسلّة التسعير، وتُحفَظ فقط حين تُسجَّل في مستند بيع. أي تقرير يجمع بيانات تسعير تاريخية (هامش متوسط، مقارنة سعر القائمة بسعر محرك التسعير) يُبنى في طبقة التقارير على قيم محفوظة، لا باستدعاء `pricingCalc` بأثر رجعي.

## Future Enhancements

_(لا شيء مُثبَّت — تُجمَّع لاحقاً عند ظهور متطلبات واضحة.)_

```

---

## 3. `docs/03_Business_Logic/Sales_Export/Sales_Export.md`

### طلب 3 — استبدل قسم Cross-references

**ابحث عن:**
```
## Cross-references
- `docs/02_Governance/Known_Issues.md`: KI-008 (حذف مستند بيع بلا صلاحية)، **KI-009 (فاتورة مكرّرة من نفس أمر التصدير — واجهة مؤكَّدة)**
- `docs/03_Business_Logic/CRM/CRM.md`: Module Dependencies (نفس ملاحظة الاستدعاء المباشر)
- `docs/03_Business_Logic/Pricing/Pricing.md`: Module Dependencies (فجوة "لا سعر معتمد يُستهلَك")
- `docs/03_Business_Logic/Sales_Export/Sales_Export_RFC.md`: RFC-SLE-001
```

**استبدل بـ:**
```markdown
## Cross-references

### Related ADRs
- `docs/02_Governance/ADR/ADR-001.md` — تحويل الفرصة→بروفورما (Accepted)
- `docs/02_Governance/ADR/ADR-019.md` — استبدال الاستدعاء المباشر CRM→Sales بـ Domain Event (Proposed)

### Related RFCs
- `docs/03_Business_Logic/Sales_Export/Sales_Export_RFC.md` — RFC-SLE-001 ✅ (ADR-019) · RFC-SLE-002 🟢 (قرار تجاري معلَّق)

### Depends On
- `docs/02_Governance/Known_Issues.md` — KI-008 (حذف مستند بلا صلاحية) · KI-009 (فاتورة مكرّرة — الأخطر)
- `docs/03_Business_Logic/CRM/CRM.md` — Module Dependencies (استدعاء مباشر موثَّق → ADR-019)
- `docs/03_Business_Logic/Pricing/Pricing.md` — Module Dependencies (فجوة "لا سعر معتمد يُستهلَك")

### Referenced Modules
- `docs/03_Business_Logic/Inventory_Production/Inventory_Production.md` — IINV-01 (stock_reserved بلا أثر) · IINV-06 (الشحن الجزئي)
- `docs/03_Business_Logic/Accounting/Accounting.md` — Business Rules #2 (التنفيذ المحاسبي لتأجيل إيراد EAS 48)

### Implementation Guide
- `docs/03_Business_Logic/Sales_Export/Sales_Export_Implementation_Guide.md`

### Test Cases
- `docs/03_Business_Logic/Sales_Export/Sales_Export_Test_Cases.md` — 56 حالة اختبار عبر 8 مجموعات
```

### طلب 7 — أضف القسمين الناقصين قبل Cross-references

**أضف هذا النص مباشرةً قبل سطر `## Cross-references` الحالي:**
```markdown
## Reports

- **لوحة مستندات البيع:** قائمة بجميع المستندات مع حالتها وسلسلة التحويل (quotation→proforma→contract→order→invoice).
- **تتبع الشحن:** حالة الشحن و Incoterm وتاريخ التسليم المتوقع لكل فاتورة.
- **لوحة الإيراد المؤجَّل:** الفواتير ذات شروط التسليم عند الوجهة (DAP/DPU/DDP) التي لم تُسلَّم بعد.
- **ربحية الشحنة:** إيراد الفاتورة مقابل تكاليف الشحن المباشرة لكل شحنة.

## Future Enhancements

_(لا شيء مُثبَّت — تُجمَّع لاحقاً عند ظهور متطلبات واضحة.)_

```

---

## 4. `docs/03_Business_Logic/Inventory_Production/Inventory_Production.md`

### طلب 3 — استبدل قسم Cross-references

**ابحث عن (النص الأحدث بعد إضافة RFC-ACC-001):**
```
## Cross-references
- `docs/03_Business_Logic/Pricing/Pricing.md`: `unitCost`/`supplierWaste` — مصدرهما الفعلي موثَّق هنا الآن بالتفصيل الكامل
- `docs/03_Business_Logic/Sales_Export/Sales_Export.md`: SINV-06 (الشحن الجزئي) — IINV-06 هنا يوفّر الإجابة التقنية الكاملة
- `docs/03_Business_Logic/Sales_Export/Sales_Export_RFC.md`: RFC-SLE-002 — جاهزة للنقاش بمعلومات كاملة الآن
- `docs/02_Governance/ADR/ADR-005.md`: القرار الأصلي (BOM vs Yield) — هذا الملف يوثّق التفصيل الكامل الذي أشار له فقط
- `docs/03_Business_Logic/Accounting/Accounting_RFC.md → RFC-ACC-001` (Cross-Module, Canonical): تكلفة الوحدة المُقيَّمة ديناميكياً وقت الطلب هي أحد مظاهر سؤال Snapshot vs Recalculation — لا يُفتَح نقاش موازٍ هنا، الإشارة لـ RFC-ACC-001 فقط
```

**استبدل بـ:**
```markdown
## Cross-references

### Related ADRs
- `docs/02_Governance/ADR/ADR-005.md` — أوامر التشغيل بوضعين (BOM vs Yield) — هذا الملف يوثّق التفصيل الكامل

### Related RFCs
- `docs/03_Business_Logic/Accounting/Accounting_RFC.md → RFC-ACC-001` (Cross-Module, Canonical) — تكلفة الوحدة الديناميكية قرينة على Snapshot vs Recalculation
- `docs/03_Business_Logic/Sales_Export/Sales_Export_RFC.md → RFC-SLE-002` — IINV-06 يوفّر الحقيقة التقنية الكاملة للنقاش

### Depends On
- _(لا اعتماد على موديولات أخرى — Inventory هو مصدر التكلفة لـ Pricing، وليس العكس)_

### Referenced Modules
- `docs/03_Business_Logic/Pricing/Pricing.md` — `unitCost`/`supplierWaste` مصدرهما موثَّق هنا
- `docs/03_Business_Logic/Sales_Export/Sales_Export.md` — SINV-06 (الشحن الجزئي) — IINV-06 يجيب عليه

### Implementation Guide
- `docs/03_Business_Logic/Inventory_Production/Inventory_Production_Implementation_Guide.md` _(Placeholder)_

### Test Cases
- `docs/03_Business_Logic/Inventory_Production/Inventory_Production_Test_Cases.md` _(Placeholder)_
```

### طلب 7 — أضف القسمين الناقصين قبل Cross-references

**أضف هذا النص مباشرةً قبل سطر `## Cross-references` الحالي:**
```markdown
## Reports

- **تقرير المخزون الحالي:** رصيد كل صنف بالكمية والتكلفة والقيمة الإجمالية.
- **بطاقة صنف:** حركة الوارد والصادر لصنف محدد مع مصدر كل حركة.
- **تقرير الجرد:** مقارنة المخزون الدفتري بالفعلي (محضر الجرد).
- **تقرير الهدر والتالف:** ملخص التخريد خلال الفترة مع مصدر الحدث (شركة/مورد).
- **لوحة أوامر التشغيل:** الانحرافات الفعلية عن BOM المعتمد لكل أمر.

## Future Enhancements

_(لا شيء مُثبَّت — تُجمَّع لاحقاً عند ظهور متطلبات واضحة.)_

```
