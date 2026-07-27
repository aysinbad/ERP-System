# Accounting — ADR Readiness (Session 6.2, Part 3)

## Document Information
```
Document Name:  Accounting ADR Readiness Assessment
Version:        1.0.0
Session:        6.2 — Final Implementation Documentation
Status:         Draft — for review
Classification: Governance-input (does NOT modify any ADR)
Owner:          Solution Architecture Team
Date:           2026-07-25
Scope:          ADR-021 (ACC-B) · ADR-022 (ACC-C) · ADR-023 (ACC-D) ONLY
```

> **حدود صارمة:** لا يُنشأ ADR-024. **ACC-E يبقى محجوباً** بقرار Product Owner (OQ-6.A). لا تُعدَّل حالة أي ADR هنا — هذا تقييم جاهزية فقط. القرارات المعتمدة (ADR-021/022/023 = `Proposed`) لا تُغيَّر.

---

## ADR-021 — ACC-B · Immutable Valuation Inputs (تجميد المدخلات)

- **Current status:** `Proposed` (2026-07-24) — بلا تغيير.
- **Dependencies:** ADR-020 (ACC-A) — الأساس التبعي (الدفتر سجل ثابت). يعتمد عليه ACC-C (لأن العكس يحتاج قيماً مُجمَّدة).
- **Runtime validation:**
  - **مؤكَّد داعم:** JS-16 unlinked يفقد فرق العملة تشغيلياً (SP-02 — `fxDiff=0`) ⇒ يثبت الحاجة للتجميد (KI-025).
  - **مؤكَّد داعم:** JS-38 (Engine A) يُجمِّد `rates`+`diff` بالفعل (SNAP ✅ · FX-01) ⇒ النمط المطلوب قائم ويُعمَّم.
  - **غير مُتحقَّق تشغيلياً:** تجميد `unitCost` في JS-08 وسلسلة الـ14 مصدراً (`Not runtime-verified` — code-confirmed فقط عبر KI-022).
- **Remaining blockers:** لا مانع قرار. البند التنفيذي الوحيد: تحديد **نقطة التجميد** لكل من الـ22 مصدر RECALC/MIXED (عمل تصميم، لا قرار Product Owner).
- **Ready for implementation:** **Yes** — جاهز للتنفيذ في Phase 1/2 مع قاعدة التجميد في Journal Builder.

---

## ADR-022 — ACC-C · Reversal Instead of Delete (العكس بدل الحذف)

- **Current status:** `Proposed` (2026-07-24) — بلا تغيير.
- **Dependencies:** ADR-020 (ACC-A) + ADR-021 (ACC-B) — العكس يحتاج قيداً ثابتاً بقيم مُجمَّدة.
- **Runtime validation:**
  - **مؤكَّد داعم:** حذف السداد في فترة مقفلة يُنتج «نجاح كاذب» ويُبقي السجل (SP-06) ⇒ يثبت خطورة المحو المباشر.
  - **مؤكَّد داعم:** العكس التلقائي لإعادة التقييم (JS-38b · FX-01) سلوك سليم قائم يُبنى عليه نمط العكس.
  - **غير مُتحقَّق تشغيلياً:** `delCreditNote`/`delSalesDoc`/`delRevaluation` (`Not runtime-verified` — code-confirmed عبر KI-008/020/024/027).
- **Remaining blockers:** لا مانع قرار. يحتاج **واجهة «عكس» واضحة** (تصميم UX) وترحيل سلوك المستخدم من «تعديل» إلى «عكس».
- **Ready for implementation:** **Yes** — جاهز؛ يُبنى في Phase 1 (immutable) ويُفعَّل مساره في كل وحدة عند هجرتها.

---

## ADR-023 — ACC-D · Centralized Period Lock (قفل مركزي)

- **Current status:** `Proposed` (2026-07-24) — بلا تغيير. **أساس تبعي لـ ACC-E (محجوب — OQ-6.A).**
- **Dependencies:** ADR-020 (ACC-A).
- **Runtime validation:**
  - **مؤكَّد داعم قوي:** خمسة كتّاب `DB.lockDate` غير منسّقين ⇒ حالة قفل متضاربة (CLOSE-05) · فتح فترة بلا صلاحية (CLOSE-03) · `doMonthEndClose` يقفل بلا قيد إقفال (CLOSE-02) · توليد التكرار في فترة مقفلة بلا حارس (REC-04).
  - **مؤكَّد داعم:** السداد يفحص القفل على الإنشاء لكن الحذف يعطي نجاحاً كاذباً (SP-05/06).
- **Remaining blockers:**
  - **قرار معلَّق (لا يمنع البدء):** **OQ-6.G** — هل تصير فحوص الجاهزية مانعة فعلية؟ ADR-023 يحدّد الاتجاه لا يحسمه.
  - **تبعية لاحقة:** ACC-E مبني فوق ACC-D لكنه محجوب بـOQ-6.A — **لا يؤثر على جاهزية ACC-D نفسه**.
- **Ready for implementation:** **Yes** — جاهز؛ `PeriodLockService` يُبنى في Phase 5/6. بند فحوص الجاهزية المانعة يُنفَّذ خلف علم قرار حتى حسم OQ-6.G.

---

## ملخّص الجاهزية

| ADR | Alias | Status | Ready for implementation | Blocker يمنع البدء؟ |
|---|---|---|---|---|
| ADR-021 | ACC-B | Proposed | **Yes** | لا |
| ADR-022 | ACC-C | Proposed | **Yes** | لا |
| ADR-023 | ACC-D | Proposed | **Yes** | لا (OQ-6.G يوجّه بنداً واحداً فقط) |

**خارج النطاق (لا يُقيَّم هنا):** ACC-A (ADR-020) مُعتمَد كأساس · **ACC-E محجوب (OQ-6.A)** · **OQ-6.F (توحيد FX) مفتوح** · OQ-6.D (مصدر الراتب الواحد) · OQ-6.G (فحوص مانعة). **لا ADR-024.**
