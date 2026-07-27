# ERP-Core

نظام Vortex Books — ERP لتجارة التصدير. هذا المستودع يحوي التوثيق الحاكم والـ Prototype المرجعي، تمهيداً لبناء النسخة الإنتاجية.

## ابدأ من هنا
1. `AGENTS.md` — لو كنت أداة AI، اقرأ هذا أولاً حرفياً.
2. `docs/00_Project/Current_State.md` — الوضع الحالي بالأرقام والفجوات.
3. `docs/02_Governance/Decisions_Log.md` — فهرس كل القرارات المعتمدة (ADRs).
4. `docs/01_Standards/Glossary.md` — المصطلحات المعتمدة الإلزامية.

## هيكل المجلدات
```
docs/
├── 00_Project/          رؤية المشروع، خارطة الطريق، الوضع الحالي
├── 01_Standards/        معيار التوثيق + قاموس المصطلحات
├── 02_Governance/       القرارات (ADR)، RFCs، الأخطاء المعروفة
│   ├── ADR/                 قرارات معمارية (ADR-000 → ADR-023) — تشمل **Accepted وProposed**؛ اعتمد على حالة (Status) كل ملف ADR على حدة
│   ├── Product_Owner/       حزمة قرارات Product Owner (OQ-6.A · OQ-6.D · OQ-6.H)
│   └── Validation/          تقارير التحقق/الدمج الحوكمية
├── 03_Business_Logic/   "ماذا يجب أن يحدث" — مجلد مستقل لكل موديول
│   ├── CRM/              CRM.md + Implementation_Guide.md + Test_Cases.md + RFC.md
│   ├── Pricing/          نفس النمط
│   ├── Sales_Export/     نفس النمط
│   ├── Inventory_Production/ نفس النمط
│   └── Accounting/       نفس النمط + Session_6_2/ (مصفوفة الترحيل + توثيق Backend) · Controls/ · Sessions/Validation/
├── 04_Policies/         سياسات معتمدة (Pricing_Policy.md)
├── 05_API/              عقد الـ API (reserved / empty حالياً)
├── 06_Testing/          اختبارات (يشمل Security/Security_Test_Specification.md)
├── 07_Architecture/     المعمارية والتكاملات
├── 08_UI_UX/            (reserved / empty حالياً)
└── 09_AI/               سياق العمل الحالي للـ AI + إرشادات الـ Prompts

reference/
├── prototype/           الـ Prototype الأصلي — مرجع تاريخي مجمّد، لا يُعدَّل (اقرأ README.md بداخله أولاً)
└── Prototype_Runtime_Verification_Report.md   تقرير التحقق التشغيلي (2026-07-25)

experiments/              نماذج تجريبية منفصلة تماماً عن الحوكمة — قابلة للحذف، لا تُعتمَد كمرجع أو دليل أمني
└── pricing-engine-poc.html   تجربة UX/حسابية لمحرك السعر المقترح، مبنية على صيغة pricingCalc الفعلية
```

> **ملاحظة:** مجلد `04_Database/` أُزيل (كان فارغاً — تصحيح تدقيق AUD-03)؛ عند بدء تصميم قاعدة البيانات يُنشأ مجلد جديد تحت الترتيب المعتمَد التالي عبر تغيير حوكمي منفصل. المجلدات الموسومة `reserved / empty` قائمة بلا محتوى بعد.

### توثيق Accounting — جلسة 6.2 (مواقع رسمية)
- `docs/03_Business_Logic/Accounting/Session_6_2/01_Final_Posting_Matrix.md`
- `docs/03_Business_Logic/Accounting/Session_6_2/02_Backend_Implementation_Guide.md`
- `docs/03_Business_Logic/Accounting/Session_6_2/03_ADR_Readiness.md`
- `docs/03_Business_Logic/Accounting/Session_6_2/04_Backend_Implementation_Checklist.md`
- `docs/03_Business_Logic/Accounting/Session_6_2/05_Production_Gap_Matrix.md`
- `docs/02_Governance/Product_Owner/Accounting_Product_Owner_Decisions.md`

## الملفات الحاكمة في الجذر
- `AGENTS.md` · `DEVELOPMENT_RULES.md` (موجود) · `README.md`.

## `DEVELOPMENT_RULES.md` — موجود
`DEVELOPMENT_RULES.md` مذكور في `AGENTS.md` كأول ملف يُقرأ، وهو **موجود** في جذر المستودع. اقرأه قبل بدء أي عمل فعلي (حسب القاعدة السابعة في `AGENTS.md`).
