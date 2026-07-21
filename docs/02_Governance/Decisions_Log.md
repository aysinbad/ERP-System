# Decisions Log — Index (فهرس القرارات)

## Document Information
```
Document Name:  Decisions Log (Index)
Version:        2.0.0
Status:         Approved
Classification: Source of Truth
Owner:          Product Owner
Approved-by:    Product Owner
Approved-date:  2026-07-18
Last-Updated:   2026-07-18
```

> **تغيير هيكلي (MAJOR):** هذا الملف كان يحتوي كل تفاصيل الـ ADRs (v1.1.0). بدءاً من هذا الإصدار، هو **فهرس فقط** — كل ADR أصبح ملفاً مستقلاً في `ADR/ADR-XXX.md`. القالب والقواعد (الترقيم الأبدي، حالات الاعتماد) في `01_Standards/Document_Standard.md`.
> **لأي AI Agent:** لا تفترض تفاصيل أي قرار من عنوانه هنا — افتح الملف الفعلي في `ADR/` قبل أي استنتاج.

---

## الفهرس

| # | العنوان | الحالة | التاريخ | الملف |
|---|---|---|---|---|
| ADR-000 | Prototype is the Source of Truth (الأصل المرجعي) | Accepted | 2026-07-17 | `ADR/ADR-000.md` |
| ADR-001 | تحويل الفرصة الناجحة إلى بروفورما (لا فاتورة مباشرة) | Accepted | — | `ADR/ADR-001.md` |
| ADR-002 | كشوف الحسابات بالعملة الأصلية بلا تحويل للجنيه | Accepted | — | `ADR/ADR-002.md` |
| ADR-003 | فروق العملة: حساب وسيط + تسوية دورية | Accepted | — | `ADR/ADR-003.md` |
| ADR-004 | ملكية العملاء في CRM (Option A: المندوب يسجّل بنفسه) | Accepted | — | `ADR/ADR-004.md` |
| ADR-005 | أوامر التشغيل بوضعين (BOM vs Yield) | Accepted | — | `ADR/ADR-005.md` |
| ADR-006 | العلامة العشرية (منزلتان) على كل المبالغ المالية | Accepted | — | `ADR/ADR-006.md` |
| ADR-007 | سعر الصرف مثبّت تاريخياً عند التحصيل/السداد | Accepted | — | `ADR/ADR-007.md` |
| ADR-008 | منع التحويل/السحب عند عدم كفاية الرصيد | Accepted | — | `ADR/ADR-008.md` |
| ADR-009 | دمج شغل المطورين المتوازي كموديولات معزولة | Accepted | — | `ADR/ADR-009.md` |
| ADR-010 | طباعة بلا أزرار داخلية وبلا auto-print | Accepted | — | `ADR/ADR-010.md` |
| ADR-011 | الشريط الجانبي داكن دائماً | Accepted | — | `ADR/ADR-011.md` |
| ADR-012 | استراتيجية الترحيل: قسم قسم بعمق | Accepted | — | `ADR/ADR-012.md` |
| ADR-013 | Architecture Style — Modular Monolith | Accepted | 2026-07-17 | `ADR/ADR-013.md` |
| ADR-014 | AI Development Strategy | Accepted | 2026-07-17 | `ADR/ADR-014.md` |
| **ADR-015** | **تقييد التراجع عن الحالات النهائية في مسار الفرصة البيعية (Won/Lost) مع صلاحية تجاوز مبرَّرة للمدير** | **Proposed** | **2026-07-18** | `ADR/ADR-015.md` |
| **ADR-016** | **إضافات مقترحة لرفع نضج CRM لمستوى الأنظمة المؤسسية (حالة العميل، دمج العملاء، تقييد جزئي لدورة الـ Lead)** | **Proposed** | **2026-07-18** | `ADR/ADR-016.md` |
| **ADR-017** | **مصادر مكوّنات التكلفة التصديرية: متوسط تاريخي بدل إدخال حر (يحسم RFC-PRC-001)** | **Proposed** | **2026-07-18** | `ADR/ADR-017.md` |
| **ADR-018** | **صلاحية رؤية التكلفة وحد أدنى للهامش مع تجاوز مبرَّر (يحسم RFC-PRC-002)** | **Proposed** | **2026-07-18** | `ADR/ADR-018.md` |
| **ADR-019** | **استبدال الاستدعاء المباشر CRM→Sales/Export بحدث نطاقي (يحسم RFC-SLE-001)** | **Proposed** | **2026-07-18** | `ADR/ADR-019.md` |

> ⚠️ **ملاحظة تحويل:** ADR-001 إلى ADR-012 كُتبت قبل اعتماد `Document_Standard.md` الحالي، ولذلك لا تحمل كل الحقول الشكلية (Owner, Approved-by...) في ترويستها. المحتوى الفعلي (Context/Decision/Consequences) كامل وصحيح، والتاريخ الدقيق لبعضها غير مسجَّل ("—" أعلاه). لا تُعاد كتابتها لإضافة الحقول الشكلية إلا إذا استلزم تعديل جوهري لاحق (MAJOR/MINOR) نفس الفرصة.

## قرارات معلّقة (Pending Decisions)
- **Multi-branch consolidation**: كيف تتجمّع حسابات فروع الشركة الواحدة؟ (خطة v2.0)
- **AI pricing**: ما مصدر بيانات التدريب؟ (خطة v2.0)
- **تكامل NAFEZA/الجمارك**: أي APIs متاحة رسمياً؟ (بحث مطلوب)

## قاعدة الإضافة
أي قرار جديد: (1) يُنشأ كملف `ADR/ADR-XXX.md` بالرقم التالي مباشرة (لا يُعاد استخدام رقم محذوف)، (2) يُضاف سطر واحد هنا في الفهرس، (3) الحالة الابتدائية دائماً `Proposed` حتى اعتماد الـ Product Owner.
