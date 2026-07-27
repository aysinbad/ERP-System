# Development Rules (قواعد التطوير)

## Document Information
```
Document Name:  Development Rules
Version:        0.1.0
Status:         Draft
Classification: Source of Truth
Owner:          Solution Architecture Team
Approved-by:    —
Approved-date:  —
Last-Updated:   2026-07-20
```

> **الجمهور:** مطوّرو Backend/Frontend، مراجعو Business Logic، وأي أداة AI تعمل على المستودع.
> **العلاقة مع `AGENTS.md`:** هذا الملف يحدّد **كيف نعمل** (ترتيب القراءة، مصادر الحقيقة، دورة التوثيق→الكود، المعايير التقنية). **`AGENTS.md` يحوي القواعد الصلبة للـ AI** ولا يُستبدَل به — يُقرأ معه، وليس بديلاً عنه.
> **حالة المستند:** `Draft` — للمراجعة والاعتماد من Product Owner و Solution Architecture قبل اعتباره نافذاً رسمياً.

---

## 1. ترتيب القراءة الإلزامي (Before You Code)

### 1.1 للجميع (بشر + AI)
| # | الملف | السبب |
|---|---|---|
| 1 | **`DEVELOPMENT_RULES.md`** (هذا الملف) | مسار العمل والمراجع التقنية |
| 2 | **`AGENTS.md`** | قواعد صلبة لا تُكسر (Rule Zero، ADR، Prototype، Known Issues) |
| 3 | **`docs/00_Project/Current_State.md`** | الوضع الفعلي، الفجوات، الـ Target Stack |
| 4 | **`docs/02_Governance/Decisions_Log.md`** (فهرس) ثم **`docs/02_Governance/ADR/ADR-XXX.md`** حسب النطاق | قرارات ملزمة — لا تُستنتَ من العناوين فقط |
| 5 | **`docs/01_Standards/Glossary.md`** | مصطلحات إلزامية في الكود والواجهة والتوثيق |
| 6 | **`docs/02_Governance/Known_Issues.md`** | أخطاء Prototype — **لا تُنقل** كسلوك مطلوب |
| 7 | **`docs/08_AI/AI_CONTEXT.md`** | الوحدة الجارية، آخر ADR، حالة الموديولات |
| 8 | **`docs/03_Business_Logic/{Module}/`** للموديول في النطاق | منطق الأعمال قبل أي تنفيذ |
| 9 | **`reference/prototype/README.md`** ثم **`reference/prototype/prototype_v2.html`** | فقط عند غموض لم يُحسم في (8) — Rule Zero |

### 1.2 عند كتابة كود Backend/Frontend فعلي (عندما يُنشأ)
إضافةً لما سبق:
- **`docs/06_Architecture/Architecture.md`**
- **`docs/01_Standards/API_Standards.md`** (حالة `Proposed` — لا تُعتبر عقداً ملزماً حتى اعتماد ADR/الفريق)
- **`docs/03_Business_Logic/{Module}/{Module}_Implementation_Guide.md`**
- **`docs/04_Database/`** و **`docs/05_API/`** عند توفرهما للموديول

---

## 2. هرّم مصادر الحقيقة (Sources of Truth)

عند التعارض، **لا يحسم المطوّر أو الـ AI وحده** — انظر §6.

```
┌─────────────────────────────────────────────────────────────┐
│  Accepted ADRs + Business Logic (Approved / In Review)      │
│  + Glossary                                                  │
├─────────────────────────────────────────────────────────────┤
│  Prototype (ADR-000) — عند غموض المواصفات المكتوبة          │
├─────────────────────────────────────────────────────────────┤
│  Known Issues — تصحيحات تقنية مطلوبة في البناء الجديد     │
│  (ليست "سلوكاً مقصوداً" للنسخ)                             │
├─────────────────────────────────────────────────────────────┤
│  AI_CONTEXT، README، experiments — إرشاد تشغيلي فقط         │
└─────────────────────────────────────────────────────────────┘
```

| مصدر | دور |
|---|---|
| **`docs/03_Business_Logic/`** | "ماذا يجب أن يحدث" — القاعدة الأولى للفهم المنظم |
| **`reference/prototype/prototype_v2.html`** | مرجع سلوك تاريخي (Living Spec) — **لا يُعدَّل** |
| **`docs/02_Governance/ADR/`** | أي تغيير سلوك معتمد أو مقترح |
| **`experiments/`** | تجارب UX/حساب — **غير حاكمة**، قابلة للحذف |

---

## 3. مرحلة المشروع الحالية

| الواقع الآن | الهدف |
|---|---|
| Prototype أحادي الملف + توثيق حاكم | **Rewrite 1:1** إلى ASP.NET Core + React (انظر `Current_State.md` §4) |
| Backend/Frontend **غير مبنيين** بعد في هذا المستودع | Modular Monolith (**ADR-013**) |
| استخراج Business Logic **قسم قسم بعمق** (**ADR-012**) | Pricing = مرجع مكتمل؛ CRM = In Review؛ Sales/Export = Draft |

**قاعدة:** لا كود إنتاجي لموديول قبل أن يكون منطقه موثّقاً بحدٍّ يكفي للتنفيذ (Entities، Rules، Implementation Guide حيث ينطبق)، أو قبل حسم Open Questions الحاسمة عبر ADR/RFC.

---

## 4. دورة العمل: توثيق → قرار → تنفيذ

### 4.1 تغيير منطق الأعمال
1. حدّث **`docs/03_Business_Logic/{Module}/{Module}.md`** أولاً.
2. انعكس الأثر في **`{Module}_Implementation_Guide.md`**, **`{Module}_Test_Cases.md`**, وRFC إن وُجد.
3. إن كان تغييراً في **سلوك معتمد سابقاً**: ملف **`ADR/ADR-XXX.md` جديد** (الرقم التالي فقط)، حالة ابتدائية **`Proposed`** — **لا تعديل** ADR `Accepted` (**`AGENTS.md` قاعدة 2**).
4. سجّل سطراً في **`Decisions_Log.md`** (فهرس فقط).

### 4.2 اكتشاف خلل تقني في Prototype (ليس قرار عمل)
- أضف **`KI-XXX`** في **`Known_Issues.md`** — لا ADR ولا Business Rule مستمد من الخلل.

### 4.3 تعارض توثيق ↔ Prototype ↔ ADR
- **توقف.** سجّل في **`Known_Issues.md`** أو **Open Questions** في ملف الموديول.
- **لا تختار طرفاً من طرفك** (**`AGENTS.md` قاعدة 6**).

### 4.4 قالب مجلد Business Logic (إلزامي)
كل موديول تحت **`docs/03_Business_Logic/{Module}/`**:
- **`{Module}.md`** — يبدأ بـ **Module Dependencies** ثم Purpose, Scope, Entities, Rules, …
- **`{Module}_Implementation_Guide.md`**
- **`{Module}_RFC.md`** (عند الحاجة)
- **`{Module}_Test_Cases.md`**

المرجع الشكلي: **`docs/03_Business_Logic/CRM/`** و **`Pricing/`** (**`AGENTS.md` قاعدة 10**).

---

## 5. قواعد الترحيل من Prototype

| # | القاعدة | مرجع |
|---|---|---|
| 1 | **Port 1:1** — إعادة كتابة بنفس السلوك، لا "ترجمة" أو "تحسين" منطقي أثناء النقل | `Current_State.md` §6، `AGENTS.md` |
| 2 | **Rule Zero** — غموض المواصفات → Prototype، مع ذكر الاستنتاج صراحةً كافتراض | ADR-000 |
| 3 | **لا تعديل Prototype** "للإصلاح" | ADR-000، `reference/prototype/README.md` |
| 4 | **Known Issues لا تُنسخ** — تُعالَج في Backend/Frontend الجديد بالحل الموثّق | Known_Issues.md |
| 5 | **المحاسبة** — أسماء وترميز الحسابات (1100، 2010، 1260، 1265، …) كما في المرجع | `Current_State.md` §6 |
| 6 | **CompanyId** على كل جدول/كيان/استعلام جديد — **من اليوم الأول** | `Current_State.md` §3، `AGENTS.md` |

---

## 6. المعايير التقنية (Target Stack)

ملخّص من **`Current_State.md`** — التفاصيل المعمارية في **`Architecture.md`**:

| طبقة | التقنية |
|---|---|
| Backend | ASP.NET Core Web API، Clean Architecture، EF Core، SQL Server |
| Frontend | React + TypeScript |
| Auth | JWT + Refresh Token (يحل KI-002) |
| Validation | FluentValidation |
| Logging | Serilog (structured) |
| API docs | Swagger / OpenAPI |
| Jobs (اختياري لاحقاً) | Hangfire |
| Cache (اختياري) | Redis |

### 6.1 Modular Monolith (ADR-013)
- حدود وحدات واضحة (`crm`, `sales`, `accounting`, `inventory`, `pricing`, `hr`) — نفس تسمية **`docs/03_Business_Logic/`** و **`API_Standards.md`**.
- **ممنوع** تسرب تبعيات Domain عبر الوحدات دون عقد صريح (Events/DTOs/Application Services).

### 6.2 Multi-tenant
- **`CompanyId`** جزء من كل Entity وكل Query — لا استثناء "مؤقت".
- لا endpoint بلا سياق شركة (Tenant) بعد تفعيل المصادقة.

### 6.3 الأمان والصلاحيات (إلزامي في البناء الجديد)
- تجزئة كلمات المرور — لا plaintext (KI-001).
- **Authorization على API** لكل عملية حساسة — الواجهة للـ UX فقط (KI-002).
- Audit **append-only** على الخادم (KI-004).

### 6.4 API (حتى اعتماد المعيار رسمياً)
- اتبع **`API_Standards.md`**: مسارات `/api/v1/{module}/{resource}`, أخطاء `{MODULE}_{ERROR_NAME}`, pagination، 422 للتحقق.
- أكواد الأخطاء لكل موديول تُعرَّف في Business Logic / Implementation Guide (مثال CRM).

### 6.5 اللغة والمصطلحات
- **Glossary فقط** — لا مرادفات (مثال: **فاتورة مبدئية (Proforma)**).
- النظام **ثنائي اللغة** (عربي RTL + إنجليزي) — رسائل API قابلة للترجمة؛ الرموز والأكواد بالإنجليزية.

### 6.6 المبالغ المالية
- **منزلتان عشريتان** على كل المبالغ (**ADR-006**).
- أسعار الصرف **مثبتة تاريخياً** عند التحصيل/السداد (**ADR-007**).

---

## 7. مجلدات المستودع

| المسار | الاستخدام |
|---|---|
| **`docs/`** | توثيق حاكم — يخضع **`Document_Standard.md`** |
| **`reference/prototype/`** | مرجع مجمّد — قراءة فقط |
| **`experiments/`** | POC منفصلة — **لا** تُعتمد كمصدر أعمال أو أمان |

---

## 8. التطوير بالـ AI (ADR-014)

- الـ AI **لا يخترع Business Logic** — Prototype + Business Spec + ADRs المعتمدة فقط.
- **`AGENTS.md`** ملزم لأي Agent.
- تحديث **`AI_CONTEXT.md`** دورياً (أسبوعياً/عند تغيّر الوحدة أو ADR) — مسؤولية الفريق، ليس افتراضاً على الـ AI.

---

## 9. متى تتوقف وتسأل

توقف **قبل** كتابة كود أو ADR إذا:
- الموديول في النطاق **غير موجود** أو **Draft** بفجوات حاسمة غير مغطاة في Open Questions/RFC.
- **`DEVELOPMENT_RULES.md`** أو **`API_Standards.md`** أو عقد DB/API للموديول **ناقص** والمهمة تتطلب تنفيذاً.
- سلوك Prototype ي contradict توثيقاً **Approved** دون KI مسجّل.
- التغيير المطلوب يمس **workflow معتمد** بدون ADR `Proposed` جاهز.

---

## 10. فهرس سريع للمراجع

| الموضوع | الملف |
|---|---|
| تعليمات AI الصلبة | `AGENTS.md` |
| الوضع والفجوات | `docs/00_Project/Current_State.md` |
| قرارات | `docs/02_Governance/Decisions_Log.md` → `ADR/` |
| أخطاء Prototype | `docs/02_Governance/Known_Issues.md` |
| مصطلحات | `docs/01_Standards/Glossary.md` |
| شكل الوثائق | `docs/01_Standards/Document_Standard.md` |
| API | `docs/01_Standards/API_Standards.md` |
| معمارية | `docs/06_Architecture/Architecture.md` |
| سياق أسبوعي | `docs/08_AI/AI_CONTEXT.md` |
| Prototype | `reference/prototype/README.md` |

---

## Open Questions (للاعتماد النهائي لهذا المستند)

1. **حالة الاعتماد:** هل يُرفَع إلى `Approved` مع `AGENTS.md` دفعة واحدة، أم `In Review` من Solution Architecture ثم PO؟
2. **معايير الكود (C#/TS):** هل يُنشأ لاحقاً `docs/01_Standards/Coding_Standards.md` منفصل، أم تُضاف أقسام FRONTEND/BACKEND هنا عند بدء المستودع البرمجي؟
3. **Git workflow:** branches، PR، تعريف "Done" للموديول — هل تُثبت في هذا الملف أم `CONTRIBUTING.md`؟
4. **اختبارات:** حد أدنى إلزامي (Unit للقواعد المحاسبية، Integration للـ API) — يُحسم عند أول موديول Backend.

---

*بعد اعتمادك: حدّث `Status` / `Version` / `Approved-by`، ويمكن تحديث `README.md` (قسم "ملف مفقود") ليشير إلى هذا الملف.*
