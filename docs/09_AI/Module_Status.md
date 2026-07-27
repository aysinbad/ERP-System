# Module Status — Detailed History

## Document Information
```
Document Name:  Module Status (Detailed)
Version:        1.6.0 (تحديث حالة Accounting إلى جلسة 6.2 — تصحيح تدقيق AUD-02)
Status:         Active
Classification: Reference
Owner:          Solution Architecture Team
Last-Updated:   2026-07-25
```

> **الغرض:** سجلّ تفصيلي لحالة كل موديول وأبرز اكتشافاته. يُقرأ عند الحاجة للسياق التاريخي.
> للحالة الحالية السريعة: انظر `AI_CONTEXT.md`.

---

## Documentation Status Dashboard

| الموديول | حالة التوثيق | ملاحظة |
|---|---|---|
| **CRM** | 🟡 In Review | لا يُعتمد قبل حسم ADR-015 · **Pricing Integration مضاف (#12, #13)** |
| **Pricing** | ✅ Approved | OQ-1 مغلق · ميزة السعر الاسترشادي · `docs/04_Policies/Pricing_Policy.md` |
| **Sales / Export** | 🟢 Complete — Pending Approval | المحتوى كامل، لم يُعتمد رسمياً بعد |
| **Inventory & Production** | 🟢 Complete — Pending Approval | المرحلتان مكتملتان، لم يُعتمد رسمياً بعد |
| **Accounting** | 🟢 Session 6.2 Documentation Complete — Pending Approval | جلسات 1→6.2 مكتملة · تحقق تشغيلي + تصحيحات مطبَّقة · مصفوفة ترحيل نهائية + توثيق Backend + حزمة قرارات PO جاهزة للمراجعة (2026-07-25) |
| **HR** | ⬜ Not Started | آخر موديول في الطابور |

---

## أحدث الإصدارات (Version Tracking)

> مرجع سريع للتحقق من تطابق نسخ الملفات — يُحدَّث مع كل جولة تصحيح.

| الملف | الإصدار |
|---|---|
| `Pricing.md` | v1.7.0 |
| `Pricing_Implementation_Guide.md` | v1.4.0 |
| `Pricing_RFC.md` | v1.5.0 |
| `CRM.md` | v2.3.0 |
| `CRM_Implementation_Guide.md` | v1.6.0 |
| `Accounting.md` | v0.5.1 |
| `Accounting_RFC.md` | v1.2.1 |
| `Accounting_Implementation_Guide.md` | v0.9.0 |
| `Accounting_Test_Cases.md` | v0.8.0 |
| `Known_Issues.md` | v1.5.0 |
| `Accounting_Product_Owner_Decisions.md` | v1.0.0 |
| `ADR-018.md` | Rev. 3 |
| `Pricing_Policy.md` | v1.2.0 |
| `AI_CONTEXT.md` | v2.3.0 |
| `Module_Status.md` | v1.6.0 (هذا الملف) |

---

## Pricing (Approved)

**الجلسات:** 6/6 مكتملة. الحالة: `Approved`.

**تحديث 2026-07-21 — ميزة السعر الاسترشادي السري:**
- `Pricing.md` v1.7.0: Entities #8 (SuggestedPrice) · #9 (PriceConfidenceState) · #10 (CostOverride) · Business Rules §Suggested Price & CRM Integration (6 قواعد، قاعدة 5 شرطية).
- **OQ-1 مغلق** — بالصياغة الصحيحة: CRM يعرض السعر الاسترشادي كمرجع اختياري مستقل في **Price Guidance Panel**، البروفورما حرة في استخدام `unitPrice` مختلف.
- `Pricing_Implementation_Guide.md` v1.4.0: Permissions Matrix (5 صلاحيات) · Cost Override Audit Entity · Staleness Display Logic · PriceGuidanceRecord Schema (line-level، شرطي).
- `Pricing_RFC.md` v1.5.0: RFC-PRC-004 (Future Enhancement، شروط الفتح مُعدَّلة) · RFC-ACC-001 بُعد "Pricing Guidance at Proforma Decision" محسوم.
- `docs/04_Policies/Pricing_Policy.md` v1.2.0: سياسة إدارية بلا أرقام ثابتة ولا أسماء برمجية.
- **ADR-018 (Rev. 3):** 5 صلاحيات · `approvePriceException` بدل `approveLowMargin` (4 فئات استثناء بالضبط) · Migration Note.

**اكتشافات جوهرية سابقة (لا تزال صالحة):**
- باگ مؤكَّد في BOM متداخل (PINV-07) — غير مُفعَّل بالبيانات الحالية.
- ADR-017/018: `Proposed` — لم يُعتمدا رسمياً (قاعدة مستويات الاعتماد).
- RFC-PRC-003 رُقِّي إلى RFC-ACC-001.

**مفاهيم جوهرية للحفظ:**
- السعر الاسترشادي = مرجع غير ملزم (`Price Guidance Panel` / `Price Guidance Flow`) — لا يُنسَخ لسعر البروفورما.
- `approvePriceException` تُطبَّق على مخالفة السياسة (هامش، خصم، قِدَم، Override) — لا على الانحراف عن السعر الاسترشادي.
- `suggestedPriceAtDecision` = metadata للتدقيق فقط، على مستوى كل بند (`PriceGuidanceRecord`) — لا يُعرَض للعميل ولا يؤثر على حسابات.
- **حداثة التكلفة:** ≤90 يوماً مكتملاً = محدَّث، >90 = قديم (اليوم 90 محدَّث، اليوم 91 قديم).
- **`PriceGuidanceRecord` شرطي، ليس إلزامياً لكل بروفورما:** يُنشَأ سجل واحد لكل بند فقط إذا عُرِض السعر الاسترشادي بنجاح. غياب الصلاحية أو عدم توفر البيانات → لا سجل، ولا عائق لإنشاء البروفورما بسعر يدوي.
- **`approvePriceException` على 4 فئات بالضبط:** هامش منخفض / خصم مفرط / تكلفة قديمة / Cost Override — الانحراف عن السعر الاسترشادي وحده ليس فئة استثناء.

**تنظيف مصطلحات (جولة أخيرة، 2026-07-21):**
- `Price Selector` → `Price Guidance Flow` في `CRM_Implementation_Guide.md`.
- إزالة كل بقايا `PriceSnapshot`/"السعر المختار يُحفَظ Snapshot" من `Pricing_RFC.md` و`Accounting_RFC.md`.
- حمولة `crm.opportunity.converted_to_proforma` مفصولة عن `suggestedPriceAtDecision` — الأخيرة تُسجَّل حصراً عبر `pricing.price_guidance_recorded`.

---

## CRM (In Review)

الحالة: `In Review`. لا يُعتمد قبل حسم ADR-015 (وADR-016 اختيارياً).

**تحديث 2026-07-21:**
- `CRM.md` v2.3.0: Business Rule #3 محدَّثة · **Business Rules #12 و#13 مضافتان** (Pricing Visibility · Price Guidance Panel، بقاعدة إنشاء شرطية).
- `CRM_Implementation_Guide.md` v1.6.0: **قسم Pricing Integration كامل** (Price Guidance Flow بمنطق شرطي · PriceGuidanceRecord Schema · Error Codes · Domain Events).
- الربط مع ADR-018 (`viewSuggestedPrice` · `approvePriceException`) موثَّق.

---

## Sales / Export (Complete — Pending Approval)

**الأقسام:** 10/10 مكتملة. `Sales_Export_Test_Cases.md`: 56 حالة اختبار. `Sales_Export_Implementation_Guide.md`: مكتمل.

**🔴 أخطر اكتشاف — KI-009:** أمر التصدير (`order → invoice`) لا يُمنع من التكرار — مستخدم عادي يقدر ينشئ **فاتورة تجارية مكرّرة فعلية**.

**اكتشافات أخرى:**
- `delSalesDoc` بلا صلاحية أو فحص حالة (KI-008).
- تأجيل الإيراد EAS 48 مرصود في الكود صراحةً.
- `DAT` غير متاح للاختيار رغم وجوده في قاعدة التأجيل (SINV-07).

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

## Accounting (Session 6.2 Documentation Complete — Pending Approval)

**جلسة 1 (2026-07-21):** Framework + Catalog (27 مصدر، 11 section) + Dependencies.
**جلسة 2 (2026-07-21):** Accounting Principles & Timing — 7 محاور.
**جلسات 3 → 5:** استخراج/مراجعة المصادر · AINV-01→34 · الكتالوج النهائي.
**جلسة 6.1:** إغلاق الأدلة المتبقّية (Targets 1–6).
**Runtime Verification (2026-07-25):** تحقق تشغيلي على `prototype_v2.html` (headless) — مكتمل؛ لم يُعدَّل الـProttype.
**تصحيحات ما بعد التشغيل (2026-07-25):** مطبَّقة على التوثيق (لا الكود).
**جلسة 6.2 (2026-07-25):** توثيق التنفيذ النهائي — مكتمل:
- **Final Posting Matrix** — `docs/03_Business_Logic/Accounting/Session_6_2/01_Final_Posting_Matrix.md`
- **Backend Implementation Guide** — `…/Session_6_2/02_Backend_Implementation_Guide.md`
- **ADR Readiness** — `…/Session_6_2/03_ADR_Readiness.md`
- **Backend Implementation Checklist** — `…/Session_6_2/04_Backend_Implementation_Checklist.md`
- **Production Gap Matrix** — `…/Session_6_2/05_Production_Gap_Matrix.md`
- **Product Owner decision package** — `docs/02_Governance/Product_Owner/Accounting_Product_Owner_Decisions.md` — **جاهزة للمراجعة**.

**الحالة الحاكمة (Baseline 2026-07-25):**
- `Accounting_RFC.md` = **v1.2.1**.
- **ADR-020 · ADR-021 · ADR-022 · ADR-023** موجودة — تبقى **Proposed** ما لم تُشِر ملفاتها لغير ذلك.
- **Known Issues:** المدى يشمل **KI-025 → KI-034**؛ **KI-028 مدحوض/مغلق (runtime)**؛ KI-029 **Low**؛ KI-032 **Medium**؛ KI-034 جديد (High — كسر render `pricing`).
- **Posting Source Catalog:** **62 confirmed · 0 conditional · 1 future · 63 total**.
- **Test Cases:** **PASS 12 · PARTIAL 10 · FAIL 40 · TOTAL 62**.
- **ACC-E:** غير مُنشأ — محجوب بقرار Product Owner **OQ-6.A**.
- **OQ-6.F:** يبقى **Open** (Architecture).
- **RECALC baseline reconciliation (AUD-08):** 22/60 ≈ 36.7% (تحليل جلسة 6.2) ⟷ 22/62 ≈ 35.5% (المقام المُحدَّث) — التسوية في `Accounting_RFC.md` §69؛ ADR-021 لم يُعدَّل.

**الخطوة التالية (تصميم التنفيذ):** **مواصفة Phase 1 — Posting Service** (`Accounting_Posting_Service_Spec.md` — لم تُنشأ بعد).

> ⚠️ **تنفيذ الـBackend الإنتاجي لم يبدأ بعد.** ما اكتمل هو **التوثيق** فقط.

**RFC-ACC-001 — Snapshot vs Recalculation (Cross-Module):**

```
Resolved:
- Pricing Guidance at Proforma Decision

Open:
- Commission Recalculation
- Inventory Unit Cost
- Accounting Derived Values
```

---

## HR (Not Started)
