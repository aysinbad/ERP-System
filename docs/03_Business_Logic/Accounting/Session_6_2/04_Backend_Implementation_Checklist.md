# Accounting — Backend Implementation Checklist (Session 6.2, Part 4)

## Document Information
```
Document Name:  Accounting Backend Implementation Checklist
Version:        1.0.0
Session:        6.2 — Final Implementation Documentation
Status:         Draft — for review (build plan)
Classification: Implementation plan
Owner:          Solution Architecture Team
Date:           2026-07-25
```

> **Complexity:** S (بسيط) · M (متوسط) · L (كبير) · XL (كبير جداً). **Priority:** P0 (حاجز — يجب أولاً) · P1 (عالٍ) · P2 (متوسط). التقديرات نسبية للتخطيط، لا التزام زمني.

---

## Phase 1 — Journal Engine · Posting Service · Immutable Posting · Audit

| # | Task | Prerequisite | Output | Related ADR | Complexity | Priority |
|---|---|---|---|---|---|---|
| 1.1 | تصميم مخطط القيود append-only (SQL Server) | Glossary §10 (COA) · `schema.sql` **(Required Phase 1 build artifact — not yet created)** | جداول `JournalEntry`/`JournalLine` غير قابلة للتعديل | ACC-A · ACC-C | M | P0 |
| 1.2 | بناء `PostingService.Post()` كنقطة دخول وحيدة | 1.1 | خدمة ترحيل ذرّية متوازنة | ACC-A | L | P0 |
| 1.3 | بناء `JournalBuilder` + تحقق التوازن (AINV-01) | 1.2 | أسطر مدين/دائن + رفض غير المتوازن | ACC-A | M | P0 |
| 1.4 | فرض الثبات (لا UPDATE/DELETE) على مستوى DB | 1.1 | triggers/permissions تمنع التعديل | ACC-C | M | P0 |
| 1.5 | آلية العكس (`ReversesEntryId`) بدل الحذف | 1.4 | قيد عكسي مؤرَّخ + مرجع الأصل | ACC-C | M | P0 |
| 1.6 | Audit log غير قابل للتلاعب (append-only، server-side) | 1.1 | تسجيل كل كتابة/حذف/عكس (KI-004) | ACC-C · KI-004 | M | P0 |
| 1.7 | إزالة «النجاح الكاذب» (لا نجاح بلا عملية مُثبَتة) | 1.5 | رسائل حالة صادقة (SP-06) | ACC-C | S | P1 |

## Phase 2 — Reconciliation Engine

| # | Task | Prerequisite | Output | Related ADR | Complexity | Priority |
|---|---|---|---|---|---|---|
| 2.1 | محرك مطابقة الأستاذ ↔ الدفاتر المساعدة (AINV-33) | Phase 1 | تقرير فروق لكل دفتر مساعد | ACC-A · ACC-B | L | P1 |
| 2.2 | قواعد قفل حسابات المراقبة (1260 · 1265 · GRNI 2016) | 2.1 | إقفال دوري + بند فرق مانع | ACC-D | M | P1 |
| 2.3 | تجميد المدخلات في نماذج القراءة مقابل الدفتر | Phase 1 · ACC-B | فصل read-model عن الدفتر الثابت | ACC-B | M | P1 |

## Phase 3 — FX Engine

| # | Task | Prerequisite | Output | Related ADR | Complexity | Priority |
|---|---|---|---|---|---|---|
| 3.1 | واجهة FX واحدة عبر الوسيط 1265 (ADR-003) | Phase 1 | محرك تقييم موحَّد (خلف علم قرار) | ADR-003 | L | P1 |
| 3.2 | تجميد `rates`+`diff` + العكس التلقائي (IAS 21) | 3.1 · ACC-B | تقييم SNAP + عكس مؤرَّخ | ACC-B | M | P1 |
| 3.3 | **[BLOCKED]** اختيار المحرك النهائي / إلغاء الخامل | **OQ-6.F (Product Owner)** | قرار توحيد FX | — | M | P1 (محجوب) |
| 3.4 | إصلاح أساس تعرُّض Engine B (سعر الفاتورة مقابل الحالي) | 3.3 | تعرُّض غير صفري صحيح | ACC-B | M | P2 |

## Phase 4 — Fixed Assets

| # | Task | Prerequisite | Output | Related ADR | Complexity | Priority |
|---|---|---|---|---|---|---|
| 4.1 | قيد اقتناء Dr 1300 / Cr (بنك **أو** ذمة 2010) | Phase 1 | اقتناء بمصدر تمويل قابل للاختيار (KI-029) | ACC-A | S | P2 |
| 4.2 | إهلاك شهري Dr حساب إهلاك مخصص / Cr 1310 | 4.1 | إهلاك بحساب مخصص بدل 5400 | ACC-A | S | P2 |
| 4.3 | ربط الأصول الافتتاحية بقيد OPENING | Phase 1 · §7 | لا إهلاك بلا أصل مقابل | ACC-A | S | P2 |

## Phase 5 — Month Close

| # | Task | Prerequisite | Output | Related ADR | Complexity | Priority |
|---|---|---|---|---|---|---|
| 5.1 | خدمة إقفال واحدة (جاهزية → 3099/3020 → قفل → Audit) | Phase 1 · 2.1 | إقفال+قفل كمعاملة واحدة (KI-033) | ACC-D | L | P1 |
| 5.2 | تحويل صافي النتيجة للأرباح المحتجزة (AINV-32) | 5.1 | قيد إقفال صحيح | ACC-D | S | P1 |
| 5.3 | حارس تكرار الإقفال لنفس الفترة | 5.1 | منع إقفال مزدوج (CLOSE-01) | ACC-D | S | P1 |

## Phase 6 — Permissions

| # | Task | Prerequisite | Output | Related ADR | Complexity | Priority |
|---|---|---|---|---|---|---|
| 6.1 | Authorization middleware على الـAPI لكل كتابة | Phase 1 | إنفاذ `can()` على الخادم (KI-002) | ACC-D · KI-002 | L | P0 |
| 6.2 | سدّ التجاوزات المؤكَّدة (سداد/رواتب/تكرار/Engine A/قفل) | 6.1 | لا bypass على الخادم (SP-07 · PAY-05 · FX-01) | KI-002 | M | P0 |
| 6.3 | `PeriodLockService` مركزي يحرس كل كاتب | Phase 1 · 5.1 | قفل موحَّد (CLOSE-05) | ACC-D | L | P0 |
| 6.4 | رفض السجل بلا تاريخ (لا يتجاوز القفل) | 6.3 | KI-016 · AINV-16/27 | ACC-D | S | P1 |
| 6.5 | **[PENDING]** فحوص جاهزية مانعة فعلية | **OQ-6.G (Architecture)** | قرار المنع مقابل info | ACC-D | S | P1 (معلَّق) |

## Phase 7 — Migration

| # | Task | Prerequisite | Output | Related ADR | Complexity | Priority |
|---|---|---|---|---|---|---|
| 7.1 | ترجمة الـ35 مصدراً المُطبَّعة → Posting Rules | Part 1 §2 · Phase 1 | قواعد ترحيل منفَّذة | ACC-A/B | L | P1 |
| 7.2 | استكمال أعمدة الـ27 مصدراً `Column-review pending` | Part 1 §3 · Prototype | مصفوفة 62 كاملة (بلا اختراع) | ACC-A/B | L | P1 |
| 7.3 | قيد افتتاحي (JS-C1) كـseed ثابت مقابل 3001 | Phase 1 · §7 | رصيد افتتاحي مُرحَّل | ACC-A | M | P1 |
| 7.4 | ترحيل بيانات لتجميد القيم التاريخية (unitCost/fxRate) | Phase 1 · ACC-B | قيود تاريخية مُجمَّدة | ACC-B | XL | P1 |
| 7.5 | مفاتيح تفرّد الترحيل (فاتورة/راتب/سداد) | Phase 1 · 6.1 | منع الازدواج (KI-009/030 · SP-03) | ACC-A | M | P0 |

---

## ملاحظات حاكمة على الخطة

- **P0 حاجز:** Phase 1 كامل + 6.1/6.2/6.3 + 7.5 قبل أي وحدة أعمال (Posting Service أولاً — ACC-D/RFC).
- **بنود محجوبة (لا تمنع البدء):** 3.3 (OQ-6.F) · 6.5 (OQ-6.G) · وكل ما يخص **ACC-E (OQ-6.A)** خارج هذه الخطة كلياً.
- **لا اختراع:** 7.2 يشترط مراجعة أعمدة المصادر الـ27 من الـProttype قبل ترجمتها.

---

## Required Phase 1 build artifacts — not yet created (AUD-10)

> هذه الملفات **غير موجودة في المستودع بعد**، ولا يُدّعى وجودها. تُنشأ أثناء تنفيذ Phase 1 في مواقعها المعتمَدة أدناه:

| Artifact | الحالة | الموقع المستقبلي المقصود |
|---|---|---|
| `schema.sql` (مخطط القيود append-only) | **Required Phase 1 build artifact — not yet created** | `src/Infrastructure/Persistence/schema.sql` أو المكافئ المعتمَد في مستودع التنفيذ |
| Posting Service specification | **Required Phase 1 design artifact — not yet created** | `docs/03_Business_Logic/Accounting/Accounting_Posting_Service_Spec.md` |
| API contracts | **Required build artifact — not yet created** | تصميم عقود الـAPI تحت مجلد التنفيذ/الـAPI المعتمَد |
| Migration maps | **Required build artifact — not yet created** | تصميم الهجرة المستقبلي تحت مجلد التنفيذ/الهجرة المعتمَد |

**لا تُولَّد هذه الملفات في جلسة التوثيق هذه.**
