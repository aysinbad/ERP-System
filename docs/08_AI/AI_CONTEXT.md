# AI Context — Project Status

## Document Information
```
Version:        2.3.0 (محاذاة حوكمية شاملة: مشغِّلات الاستثناء + PriceGuidanceRecord الشرطي + priceConfidenceState + RFC-ACC-001 يشمل Commission)
Last-Updated:   2026-07-21
```

---

## Table of Contents
1. [Governance Rules](#governance-rules)
2. [Current Focus](#current-focus)
3. [ADR Status](#adr-status)
4. [Open RFCs](#open-rfcs)
5. [Module Status](#module-status)
6. [Next Milestones](#next-milestones)
7. [AI Operating Notes](#ai-operating-notes)

**Detailed references:**
- Module history & discoveries → `docs/08_AI/Module_Status.md`
- Cross-module discoveries → `docs/08_AI/Cross_Module_Discoveries.md`

---

## Governance Rules

> قاعدة مستويات الاعتماد — مستويان **مستقلان**:

| المستوى | المعنى |
|---|---|
| **Module Approved** | التوثيق مكتمل وصحيح — يُبنى عليه |
| **ADR Accepted** | القرار المعماري اعتُمد رسمياً |

مثال: `Pricing.md` → `Approved` (توثيق)، بينما ADR-017/018 → `Proposed` (قرارات). الاثنان صحيحان في آنٍ واحد.

---

## Current Focus

```
Module:   Accounting Engine
Session:  3/6 — Sales & Receivables Rules (التالية)
File:     docs/03_Business_Logic/Accounting/Accounting.md
Progress: 2/6 Sessions (33%)
```

**ما اكتمل:**
- جلسة 1: Framework + Catalog (27 مصدر قيد، 11 section)
- جلسة 2: Accounting Principles & Timing (7 محاور)

**التالي:** جلسة 3 — قواعد البيع والذمم المدينة (فاتورة بيع · مخصص مطالبات · عمولة · L/C · فرق عملة محقق).

---

## ADR Status

```
Latest Accepted:  ADR-014 — AI Development Strategy (2026-07-17)
```

**Proposed (بانتظار اعتماد Product Owner):**

| ADR | الموضوع | ملاحظة |
|---|---|---|
| ADR-015 | تقييد Won/Lost مع تجاوز مبرَّر | — |
| ADR-016 | CRM Enterprise Enhancements | — |
| ADR-017 | مصادر مكوّنات التكلفة | يحسم RFC-PRC-001 |
| **ADR-018** | **نموذج صلاحيات التسعير الكامل (5 صلاحيات)** | موسَّع 2026-07-21: `approvePriceException` بدل `approveLowMargin` · `viewSuggestedPrice` · `editCostOverride` · `editMargin` · Migration Note |
| ADR-019 | استبدال استدعاء CRM→Sales بـ Domain Event | يحسم RFC-SLE-001 |

---

## Open RFCs

| RFC | النطاق | الحالة | الملف القانوني |
|---|---|---|---|
| **RFC-ACC-001** | Snapshot vs Recalculation (Cross-Module) | 🟡 **Partially Resolved** — Pricing Guidance at Proforma Decision محسوم ✅: `unitPrice` مستقل، `PriceGuidanceRecord` metadata شرطية للتدقيق فقط. Commission وInventory وAccounting لا تزال مفتوحة | `Accounting/Accounting_RFC.md` |
| RFC-SLE-002 | الشحن الجزئي | 🟢 تحقيق مكتمل، قرار معلَّق | `Sales_Export_RFC.md` |
| RFC-PRC-003 | تجميد القيم المُشتقّة | 🟡 رُقِّي إلى RFC-ACC-001 | `Pricing_RFC.md` (cross-ref) |
| RFC-PRC-004 | Cost Override → تكلفة رسمية معتمدة | ⬜ Future Enhancement — لم يُفتَح | `Pricing_RFC.md` |

---

## Module Status

| الموديول | الحالة | تفاصيل |
|---|---|---|
| CRM | 🟡 In Review | ADR-015 · **Pricing Integration مضاف (#12, #13)** |
| Pricing | ✅ Approved | OQ-1 مغلق · ميزة السعر الاسترشادي موثَّقة · `docs/04_Policies/Pricing_Policy.md` |
| Sales / Export | 🟢 Complete — Pending Approval | 56 TC · Implementation Guide مكتمل |
| Inventory & Production | 🟢 Complete — Pending Approval | IINV-01 مؤكَّد · RFC-SLE-002 جاهز |
| **Accounting** | 🔵 **In Progress 2/6** | جلسة 3 التالية |
| HR | ⬜ Not Started | — |

→ التفاصيل الكاملة: `docs/08_AI/Module_Status.md`
→ الاكتشافات العابرة للموديولات: `docs/08_AI/Cross_Module_Discoveries.md`

---

## Next Milestones

1. **Accounting جلسة 3** — Sales & Receivables Rules
2. **Accounting جلسة 4** — Procurement, Inventory & Production Rules
3. **Accounting جلسة 5** — AINV + RFCs (نقطة التبلور) + RFC-ACC-001 الأبعاد المفتوحة
4. **Accounting جلسة 6** — Implementation Guide + Test Cases
5. **HR** — الموديول الأخير

---

## AI Operating Notes

- هيكل `docs/` تغيّر في 2026-07-18 من ملفات مفردة لمجلدات. أي مسار قديم → استخدم المسارات في `AGENTS.md`.
- القالب المرجعي للموديولات: `docs/03_Business_Logic/CRM/`.
- `AINV-XX` لا تُرقَّم قبل جلسة 5 من Accounting.
- `Accounting.md` يجيب على "ماذا يضمن النظام؟" فقط — أسماء الدوال في `Accounting_Implementation_Guide.md`.
- **`docs/04_Policies/`** مجلد جديد (2026-07-21) — وثائق سياسات إدارية بلا أسماء برمجية. أول ملف: `Pricing_Policy.md`.
- **Migration Note — `approveLowMargin`:** أي تحقق قديم من `can('pricing','approveLowMargin')` يُهاجَر لـ `can('pricing','approvePriceException')` — انظر `ADR-018.md`.
- **السعر الاسترشادي** هو مرجع داخلي غير ملزم (`Price Guidance Panel` / `Price Guidance Flow`) — لا يُنسَخ تلقائياً لسعر البروفورما. `unitPrice` الفعلي مستقل.
- **تنظيف مصطلحات (2026-07-21، جولة نهائية):** `Price Selector` → `Price Guidance Flow` · `PriceSnapshot` (كل الأشكال) أُزيلت نهائياً من Pricing_RFC/Accounting_RFC · حمولة `crm.opportunity.converted_to_proforma` مفصولة عن بيانات السعر الاسترشادي (التي تُحمَل في `pricing.price_guidance_recorded` فقط).
- **الانحراف عن السعر الاسترشادي وحده ليس استثناءً:** `approvePriceException` تُطلَب فقط عند مخالفة سياسة معتمدة (هامش/خصم/تكلفة قديمة/Override) — لا لمجرد اختلاف `unitPrice` عن السعر الاسترشادي.
- **`PriceGuidanceRecord` إلزامي فقط عند نجاح عرض الإرشاد:** إذا حُسِب السعر الاسترشادي وعُرِض بنجاح، يُسجَّل سجل واحد لكل بند. لا يُنشَأ أي سجل عند غياب الصلاحية أو عدم توفر الإرشاد (`PRC_NO_COST_DATA`).
- **إنشاء البروفورما يبقى متاحاً دائماً بسعر يدوي:** غياب أو حجب السعر الاسترشادي لا يُعيق إنشاء البروفورما أبداً — لا استجابة 422 من مسار CRM بسبب غياب الإرشاد.
- **أحداث تحويل CRM لا تحمل تفاصيل الإرشاد السعري:** `crm.opportunity.converted_to_proforma` يحمل `oppId, proformaId, custCode` فقط — بيانات السعر الاسترشادي حصراً في `pricing.price_guidance_recorded`.
- **`priceConfidenceState` هو الاسم القانوني** في كل الـ Schemas والـ Audit payloads — لا يُستخدَم `confidenceState` المختصر.
