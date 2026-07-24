# Known Issues (مشاكل معروفة — ليست قرارات أعمال)

## Document Information
```
Document Name:  Known Issues
Version:        1.1.0
Status:         Approved (KI-001 → KI-009) · KI-010 → KI-024 مضافة 2026-07-23 بانتظار مراجعة جلسة 6
Classification: Reference
Owner:          Solution Architecture Team
Approved-by:    Product Owner
Approved-date:  2026-07-18
Last-Updated:   2026-07-23 (Accounting Session 5 merge)
```

> **الفرق بين هذا الملف و `docs/02_Governance/Decisions_Log.md`:** كل سطر هنا هو **خلل تقني معروف في الـ Prototype**، وليس قرار عمل مقصود.
> **قاعدة إلزامية لأي AI Agent:** هذه العناصر **لا تُنقل كسلوك مطلوب** أثناء الترحيل. لا تُبنى كـ Business Logic، ولا يُستنتج منها أي قاعدة. تُعالَج فقط في طبقة الـ Backend الجديدة حسب الحل المقترح.
> لو الكود والتوثيق (ADR) تعارضا حول أي بند هنا، **هذا الملف هو الأصح** لأنه موثّق بعد مراجعة متعمّدة، لا افتراض حر.

> **تحديث 2026-07-23 — دمج جلسة Accounting 5:**
> - **KI-008 و KI-009** احتفظا بنطاقهما ووصفهما الأصلي بالكامل، وأُضيف لكلٍّ منهما قسم **«أثر محاسبي — مضاف بمراجعة جلسة Accounting 5»** فقط.
> - **KI-010 → KI-024** (15 خللاً) أُضيفت من `Accounting_Session5_Invariants_RFC_Decisions_Reviewed.md` §8.
> - الحقول التي لم يثبت تفصيلها بعد موسومة حرفياً **`To be completed in Session 6`** — **لا تُملأ بافتراض**.

---

## KI-001 — كلمات المرور نص صريح (Plaintext Passwords)
- **الموقع:** `DB.users` — الحقل `pass` مخزّن كنص صريح غير مُجزّأ.
- **الأثر:** أي قراءة للكود أو للـ localStorage تكشف كل كلمات المرور مباشرة.
- **الحل المطلوب في الـ Backend الجديد:** تجزئة (hashing) بخوارزمية معتمدة (bcrypt/argon2)، ومنع تخزين أو تسجيل كلمة المرور كنص صريح في أي طبقة (لوجز، استجابات API، إلخ).
- **لا تُنقل كما هي.**

## KI-002 — التحقق من الصلاحيات على الواجهة فقط (Client-side only)
- **الموقع:** دوال `can()` و`sectionPerm()` — التحقق كله في جافاسكريبت الواجهة، بدون أي تحقق من طرف خادم.
- **الأثر:** أي مستخدم يقدر يتخطى القيود عبر أدوات المطوّر في المتصفح (مثال: تعديل `CURRENT_USER.role` مباشرة).
- **الحل المطلوب:** كل تحقق صلاحية (`can`, `sectionPerm`, `approvalLimit`) يُعاد تنفيذه **إلزامياً على الـ API** كطبقة تفويض (authorization middleware)، والواجهة تستخدمه فقط لإخفاء العناصر (UX)، لا كحماية فعلية.
- **لا يُعتمد على منطق الواجهة كمصدر وحيد للحماية عند الترحيل.**

## KI-003 — "تشفير" النسخة الاحتياطية هو XOR وليس تشفيراً قوياً
- **الموقع:** `_xorCrypt()` في شاشة `viewBackupRestore`.
- **الحالة:** تم تصحيح النص الظاهر للمستخدم في الواجهة (v1.0.1) ليقول "تشويش بسيط (XOR)" بدل "مشفّر/AES-like" المضلِّل سابقاً. **لم يُعدَّل منطق البرمجة نفسه.**
- **الحل المطلوب في الـ Backend:** تشفير حقيقي (AES-GCM أو مكافئ) عبر مكتبة معتمدة، ومفتاح لا يُخزَّن مع البيانات نفسها.

## KI-004 — سجل التدقيق (Audit Log) قابل للحذف/التلاعب
- **الموقع:** `DB.auditLog` و`DB.securityLog` — مخزّنان في نفس بنية `DB` القابلة للتعديل من نفس الجلسة.
- **ملاحظة:** فيه محاولة تسلسل hash (`_hash`, `auditChainOk`) لكنها هاش غير تشفيري (djb2-style)، ومحسوبة ومُتحقَّق منها بالكامل على العميل — لا تصلح كدليل رقابي موثوق.
- **الحل المطلوب:** Audit log immutable على مستوى قاعدة البيانات (append-only table أو ما يعادلها)، محسوب ومُتحقَّق منه على الخادم فقط. **موثّق بالفعل كأولوية "مهمة" في `00_Current_State.pdf`.**

## KI-005 — طبقتان تاريخيتان مختلفتان من التطوير داخل نفس الملف
- **السياق:** حسب ADR-009، جزء من الميزات (مستحقات، أعمار ديون، مركز تقارير، Ctrl+K) أُضيف كموديولات IIFE معزولة من فرع تطوير متوازٍ، وليس بنفس أسلوب أو انضباط باقي الكود.
- **الأثر على الاستخراج:** عند توثيق `migration_map.md`، لازم يُذكر مصدر كل دالة (الفرع الأساسي أم المدمج) لأن أسلوب معالجة الأخطاء وتسمية المتغيرات يختلف بين الطبقتين، وقد يُضلِّل من يفترض اتساقاً كاملاً.

---

## KI-006 — لوحة الفرص (Kanban) لا تدعم السحب باللمس على الموبايل
- **الموقع:** شاشة Pipeline في CRM — `crmDragStart`, `crmDrop`, `crmAllowDrop` (HTML5 Drag & Drop API القياسي، بلا معالجة لأحداث اللمس `touchstart`/`touchmove`).
- **الأثر:** مندوب يفتح شاشة الفرص من موبايل مش هيقدر يسحب كارت الفرصة بين الأعمدة بالإصبع — الميزة بصرياً موجودة لكن غير قابلة للاستخدام باللمس.
- **بديل موجود بالفعل (يُستخدم كحل مؤقت):** أزرار خطوات المرحلة (`crm-step`) جوه شاشة تفاصيل الفرصة (`onclick="moveOppStage(...)"`) — تعمل بالضغط العادي على أي جهاز، لكنها تتطلب فتح تفاصيل الفرصة أولاً بدل التغيير المباشر من اللوحة.
- **الحل المطلوب عند بناء الـ Frontend الجديد (React):** استخدام مكتبة سحب وإفلات تدعم اللمس فعلياً (مثال: `dnd-kit` أو `react-beautiful-dnd` بنسخة تدعم Touch Backend)، أو تصميم بديل مخصص للموبايل (قائمة منسدلة لتغيير المرحلة بدل السحب على الشاشات الصغيرة).
- **لا يُنقل كما هو.** هذا قيد تقني في الـ Prototype، ونمط الحل (سحب فعلي أو بديل) قرار Technical Design لاحق، وليس تغييراً في منطق الأعمال نفسه (لا يمسّ `CRM.md`).

## KI-007 — حذف العميل المحتمل (Lead) بلا تحقق صلاحية، وخطر تكامل مع Lead محوَّل
- **الموقع:** `delLead` في CRM — حذف نهائي حقيقي (`DB.leads=DB.leads.filter(...)`)، بلا أي فحص صلاحية (`crmIsManager()` غير مُستدعاة هنا، خلافاً لـ `crmReassign`).
- **الأثر الأول:** أي مستخدم (مش المدير بس) يقدر يحذف أي Lead يقدر يشوفه نهائياً — تأكيد بصري بس (نافذة "متأكد؟")، لا صلاحية فعلية.
- **الأثر الثاني (أخطر):** لا يوجد فحص لحالة الـ Lead — حتى Lead بحالة `converted` (محوَّل فعلياً لعميل حقيقي) قابل للحذف. حذفه يكسر سلسلة اشتقاق مالك العميل (`CRM.md` Business Rules #7) بصمت، لأن أحد مصادر الاشتقاق يعتمد على وجود الـ Lead الأصلي المرتبط بـ `convertedCustCode`.
- **لا يُسجَّل في سجل التدقيق حالياً** — لا أثر لمن حذف أو متى (`CRM_Implementation_Guide.md` قسم Audit Rules).
- **الحل المطلوب عند بناء الـ Backend:**
  1. فرض صلاحية (مدير فقط، أو مالك الـ Lead نفسه) على مستوى الـ API — لا الواجهة فقط.
  2. منع حذف أي Lead بحالة `converted` (أو تحويله لحذف منطقي soft delete بدل الحذف النهائي لكل الحالات — قرار مفتوح في `CRM.md` Open Questions #5).
  3. تسجيل كل عملية حذف في الـ Audit Log إلزامياً.
- **لا يُنقل كما هو.** غياب الصلاحية والتدقيق هنا خلل تقني، لا قرار عمل مقصود.

## KI-008 — حذف مستند البيع (`salesDocs`) بلا تحقق صلاحية، وخطر تكامل مع مستند محوَّل بالفعل
- **الموقع:** `delSalesDoc` في وحدة Sales/Export — حذف نهائي حقيقي، بلا أي فحص صلاحية أو حالة.
- **نفس نمط الخطأ بالضبط الموثَّق في KI-007 (`delLead` في CRM)** — مش حالة جديدة، بل تكرار لنفس الثغرة المعمارية في موديول مختلف.
- **الأثر:** أي مستخدم يقدر يحذف أي مستند بيع نهائياً، **حتى لو كان له `toDoc` مملوء** (يعني اتحوّل بالفعل لمرحلة تالية في السلسلة: عرض سعر → بروفورما → عقد → أمر تصدير) — الحذف يكسر سلسلة `fromDoc`/`toDoc` بصمت، والمستند التالي في السلسلة يفضل يشاور على مصدر غير موجود.
- **لا يُسجَّل في سجل التدقيق حالياً.**
- **الحل المطلوب عند بناء الـ Backend:** نفس حل KI-007 بالضبط — فرض صلاحية على مستوى الـ API، منع حذف أي مستند له `toDoc` أو `linkedInvoiceId` مملوء (أو تحويله لحذف منطقي)، وتسجيل كل عملية حذف في الـ Audit Log.
- **لا يُنقل كما هو.**

### أثر محاسبي — مضاف بمراجعة جلسة Accounting 5
> `delSalesDoc` بلا فحص صلاحية تسمح بحذف مستند مصدر لفاتورة مُرحَّلة. ولأن الدفتر **مُشتقّ**، فحذف المستند **يمحو قيوده من كل الفترات** بلا قيد عكسي — بما فيها فترات مقفلة، لأن `canDeleteDated` غير مطبَّقة على هذا المسار.
> ⇒ خرق مباشر لضمانة «المُرحَّل لا يُعدَّل ولا يُحذَف» المعلَنة في `Glossary.md` §12.

- **Related:** AINV-30 · AINV-05 · **ACC-C** (ADR مرشَّح — جلسة 6)
- **Evidence:** `Accounting_Session5_Invariants_RFC_Decisions_Reviewed.md` §8.5

## KI-009 — إمكانية إنشاء فاتورة تجارية مكرّرة من نفس أمر التصدير (ثغرة واجهة مؤكَّدة، لا افتراض كود فقط)
- **الموقع:** `sdConvert` (فرع `t.next==='invoice'`) + شرط `canConv` في `viewSalesDocs`/`openSalesDocDetail`.
- **السبب الجذري:** خطوة `order → invoice` هي الوحيدة في سلسلة مستندات البيع اللي **لا تضبط `doc.toDoc`** بعد التحويل (تضبط `linkedInvoiceId` بس). كل باقي الخطوات (quotation→proforma→contract→order) بتضبط `toDoc` فتُمنع من التكرار تلقائياً.
- **الأثر المؤكَّد (مش نظري):** شرط ظهور زر "إنشاء فاتورة" (`canConv = t.next && !doc.toDoc && !['cancelled','rejected','expired'].includes(doc.status)`) **يفضل `true` بعد إنشاء أول فاتورة بالفعل** — الزر نفسه يفضل ظاهر وقابل للنقر في الواجهة، ومستخدم عادي (بلا أي تلاعب بالكود) يقدر يضغطه تاني وينشئ **فاتورة تجارية ثانية مكرّرة بالكامل** من نفس أمر التصدير، بنفس الأصناف والكميات.
- **الخطورة:** أعلى من KI-007/KI-008 (حذف بيانات) — هنا **إنشاء مستند مالي مكرّر فعلياً** قابل إنه يدخل قيود محاسبية مزدوجة لو اتصرف على أساسه.
- **الحل المطلوب عند بناء الـ Backend:** ضبط `doc.toDoc` (أو حقل مكافئ صريح، مثل `converted:true`) في فرع `order→invoice` بالضبط زي باقي الخطوات، لمنع التكرار من جذره — لا يكفي إخفاء الزر في الواجهة فقط (نفس مبدأ "لا اعتماد على الواجهة كحماية وحيدة" المتكرر في كل الوحدات).
- **لا يُنقل كما هو.**

### أثر محاسبي — مضاف بمراجعة جلسة Accounting 5
> لأن `sdConvert` لا تضبط `doc.toDoc` في فرع `order → invoice`، يمكن إنشاء فاتورتين تجاريتين من نفس أمر التصدير من الواجهة بلا أي تلاعب. و`buildJournalCore()` تعالج **كل سجل في `DB.invoices` باستقلال تام** — لا تفحص `fromSalesDoc` ولا تبحث عن تكرار.
> ⇒ **قيدان كاملان: إيراد مزدوج (4010) · ذمة مزدوجة (1100) · التزام ضريبي مزدوج (2100) · وخصم مزدوج من المخزون (1200 عبر 5010).** كلاهما متوازن فيمرّ من حارس `add()` بلا إنذار.
> **الحد الوحيد ضد ازدواج القيد يقع خارج موديول المحاسبة كلياً.**

- **Related:** AINV-04 · **ACC-A** (`ADR-020` — `Proposed`)
- **Evidence:** `Accounting_Session5_Invariants_RFC_Decisions_Reviewed.md` §8.5

---

> ## KI-010 → KI-024 — أخلال محاسبية (جلسة Accounting 5 · 2026-07-23)
>
> المصدر الموثِّق لكل ما يلي: `Accounting_Session5_Invariants_RFC_Decisions_Reviewed.md` v1.1.0 §8.
> `Target` يشير إلى الثابت المحاسبي (`AINV-XX`) والـADR المعالج. `ACC-A` = `ADR-020` (`Proposed`); `ACC-B/C/D/E` مرشّحون لجلسة 6، و`ACC-E` **محجوب** حتى حسم استنفاد الحساب 2017.


## KI-010 — Customer claim valued at live FX rate — breaches ADR-007
- **Severity:** 🔴 Critical
- **Observed behavior:** ذمة العملة الأجنبية لا تصفو؛ رصيد متبقٍّ دائم بلا تصنيف
- **Expected behavior:** `To be completed in Session 6`
- **Accounting impact:** ذمة العملة الأجنبية لا تصفو؛ رصيد متبقٍّ دائم بلا تصنيف
- **Affected accounts:** 1100 · 5300 · 4030 · 5500
- **Evidence:** JS-08 — مطالبة عميل معتمدة تُقيَّم بـ`fxRate()` **حيّ** بدل سعر نشأة الذمة — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** `To be completed in Session 6`
- **Period impact:** `To be completed in Session 6`
- **Temporary workaround:** `To be completed in Session 6`
- **Target:** AINV-10 · ACC-B
- **لا يُنقل كما هو.**

## KI-011 — Account 1260 outside every close blocker
- **Severity:** 🔴 Critical
- **Observed behavior:** الهدر يبقى أصلاً ⇒ أصول وأرباح مُضخَّمة
- **Expected behavior:** `To be completed in Session 6`
- **Accounting impact:** الهدر يبقى أصلاً ⇒ أصول وأرباح مُضخَّمة
- **Affected accounts:** 1260 · 5030 · 2010
- **Evidence:** S4.6 DF-1 — 1260 غير مذكور في `closeReadiness().blockers`؛ الفحص مؤشر إتمام إرشادي لا مانع — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** `To be completed in Session 6`
- **Period impact:** `To be completed in Session 6`
- **Temporary workaround:** `To be completed in Session 6`
- **Target:** AINV-24 · ACC-E
- **لا يُنقل كما هو.**

## KI-012 — Account 5300 charged twice; 2105 never consumed
- **Severity:** 🔴 Critical
- **Observed behavior:** ازدواج تحميل على المصروف؛ التزام متراكم
- **Expected behavior:** `To be completed in Session 6`
- **Accounting impact:** ازدواج تحميل على المصروف؛ التزام متراكم
- **Affected accounts:** 5300 · 2105 · 1100
- **Evidence:** JS-08 يحمّل 5300 عند المطالبة الفعلية · JS-43 يحمّله ثانيةً كمخصص؛ 2105 بلا مسار استنفاد — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** `To be completed in Session 6`
- **Period impact:** `To be completed in Session 6`
- **Temporary workaround:** `To be completed in Session 6`
- **Target:** AINV-21 · ACC-E
- **لا يُنقل كما هو.**

## KI-013 — Claims provision base is cumulative in journal, filtered on screen
- **Severity:** 🔴 Critical
- **Observed behavior:** Displayed Basis ≠ Posted Basis؛ ويشمل proforma والمؤجَّل
- **Expected behavior:** `To be completed in Session 6`
- **Accounting impact:** Displayed Basis ≠ Posted Basis؛ ويشمل proforma والمؤجَّل
- **Affected accounts:** 5300 · 2105
- **Evidence:** S4.6 T2 — `fInvoices()` تُنتج مجتمعين مختلفين لنفس الوعاء بين الدفتر والشاشة — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** `To be completed in Session 6`
- **Period impact:** `To be completed in Session 6`
- **Temporary workaround:** `To be completed in Session 6`
- **Target:** AINV-13 · AINV-22 · ACC-E
- **لا يُنقل كما هو.**

## KI-014 — Multiple purchase invoices per GRN not prevented
- **Severity:** 🔴 High
- **Observed behavior:** 2016 قد يصير مديناً؛ ازدواج دين المورد
- **Expected behavior:** `To be completed in Session 6`
- **Accounting impact:** 2016 قد يصير مديناً؛ ازدواج دين المورد
- **Affected accounts:** 2016 · 2010 · 1200
- **Evidence:** S4.5 Target 7 — لا حارس ولا إقفال جزئي على إذن الاستلام — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** `To be completed in Session 6`
- **Period impact:** `To be completed in Session 6`
- **Temporary workaround:** `To be completed in Session 6`
- **Target:** AINV-15 · ACC-A
- **لا يُنقل كما هو.**

## KI-015 — `lockPeriodNow()` has no internal guard; `unlockPeriodNow()` needs no permission
- **Severity:** 🔴 High
- **Observed behavior:** القفل قابل للتجاوز والفتح بلا ضابط
- **Expected behavior:** `To be completed in Session 6`
- **Accounting impact:** القفل قابل للتجاوز والفتح بلا ضابط
- **Affected accounts:** كل الحسابات
- **Evidence:** S4.5 Target 5 — ثمانية مسارات قفل، لا مركزي — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** `To be completed in Session 6`
- **Period impact:** `To be completed in Session 6`
- **Temporary workaround:** `To be completed in Session 6`
- **Target:** AINV-26 · AINV-28 · ACC-D
- **لا يُنقل كما هو.**

## KI-016 — Undated record bypasses the period lock
- **Severity:** 🟡 Medium
- **Observed behavior:** ترحيل في فترة مقفلة بسجل ناقص
- **Expected behavior:** `To be completed in Session 6`
- **Accounting impact:** ترحيل في فترة مقفلة بسجل ناقص
- **Affected accounts:** كل الحسابات
- **Evidence:** S3 §3.5.4 — مطالبة بتاريخ فارغ تمرّ من حارس القفل (OQ-3.E) — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** `To be completed in Session 6`
- **Period impact:** `To be completed in Session 6`
- **Temporary workaround:** `To be completed in Session 6`
- **Target:** AINV-27 · ACC-D
- **لا يُنقل كما هو.**

## KI-017 — No guard against overlapping commission accrual periods
- **Severity:** 🟡 Medium
- **Observed behavior:** التزام عمولة مزدوج
- **Expected behavior:** `To be completed in Session 6`
- **Accounting impact:** التزام عمولة مزدوج
- **Affected accounts:** 5440 · 2017
- **Evidence:** C-01 / OQ-5.NEW — `accrueCommission()` نموذج Snapshot سليم، لكن بلا حارس تداخل فترات — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** `To be completed in Session 6`
- **Period impact:** `To be completed in Session 6`
- **Temporary workaround:** `To be completed in Session 6`
- **Target:** AINV-25 · ACC-E
- **لا يُنقل كما هو.**

## KI-018 — NRV excludes components and discards `nrv='0'`
- **Severity:** 🟡 Medium
- **Observed behavior:** مواد خام بلا فحص هبوط؛ أقصى حالة هبوط غير مُخصَّصة
- **Expected behavior:** `To be completed in Session 6`
- **Accounting impact:** مواد خام بلا فحص هبوط؛ أقصى حالة هبوط غير مُخصَّصة
- **Affected accounts:** 5015 · 1205 · 1200
- **Evidence:** S4.5 Target 3 / C-05 — `nrvWritedown()` تستبعد `component` وتُسقِط القيمة صفر — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** `To be completed in Session 6`
- **Period impact:** `To be completed in Session 6`
- **Temporary workaround:** `To be completed in Session 6`
- **Target:** AINV-23 · ACC-E
- **لا يُنقل كما هو.**

## KI-019 — LC margin refunded at opening rate — no FX difference
- **Severity:** 🟡 Medium
- **Observed behavior:** فرق عملة حقيقي غير معترَف به
- **Expected behavior:** `To be completed in Session 6`
- **Accounting impact:** فرق عملة حقيقي غير معترَف به
- **Affected accounts:** 1080 · 4030 · 5500
- **Evidence:** JS-33 / S3 §3.8.2 — رد الهامش بسعر الفتح (OQ-3.H) — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** `To be completed in Session 6`
- **Period impact:** `To be completed in Session 6`
- **Temporary workaround:** `To be completed in Session 6`
- **Target:** AINV-11 · ACC-B
- **لا يُنقل كما هو.**

## KI-020 — Sales-return cost reversal conditional on a stored field
- **Severity:** 🟡 Medium
- **Observed behavior:** إيراد معكوس بلا تكلفة معكوسة
- **Expected behavior:** `To be completed in Session 6`
- **Accounting impact:** إيراد معكوس بلا تكلفة معكوسة
- **Affected accounts:** 5010 · 1200 · 4090
- **Evidence:** JS-12 / S3 §3.10.1.4 — `rt.cogsEGP` قد يكون فارغاً — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** `To be completed in Session 6`
- **Period impact:** `To be completed in Session 6`
- **Temporary workaround:** `To be completed in Session 6`
- **Target:** AINV-31 · ACC-C
- **لا يُنقل كما هو.**

## KI-021 — Work-order entry uses 1200 on both sides
- **Severity:** 🟢 Low
- **Observed behavior:** حركة المخزون غير قابلة للفصل من الأستاذ
- **Expected behavior:** `To be completed in Session 6`
- **Accounting impact:** حركة المخزون غير قابلة للفصل من الأستاذ
- **Affected accounts:** 1200 · 1260
- **Evidence:** JS-27 / S4 §4.6.2 (OQ-4.E) — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** `To be completed in Session 6`
- **Period impact:** `To be completed in Session 6`
- **Temporary workaround:** `To be completed in Session 6`
- **Target:** AINV-18 · ACC-B
- **لا يُنقل كما هو.**

## KI-022 — Live `unitCost()` recalculates historical journal values
- **Severity:** 🔴 Critical
- **Root cause:** `unitCost()` / `actualUnitCost()` مستدعاة **داخل** `buildJournalCore()`
- **Observed behavior:** قيمة قيد تخريد أو هدر أو تكلفة مبيعات مؤرَّخ سابقاً تتغيّر عند إدخال فاتورة شراء جديدة تُغيّر المتوسط المرجّح
- **Expected behavior:** تكلفة الوحدة تُقرأ مرة واحدة عند الترحيل وتُخزَّن مع سطر القيد؛ لا يتغيّر قيد مُرحَّل بحدث لاحق
- **Accounting impact:** إدخال فاتورة شراء جديدة قد يغيّر قيم قيود تاريخية **دون قيد تعديل أو عكس**؛ القيود تبقى متوازنة فتمرّ من `add()` بلا إنذار
- **Affected accounts:** 1200 · 5010 · 5020 · 5011 · 1190 · 1260 · 1205 · 5015
- **Affected sources — **مباشرة (12):** JS-02 · JS-03 · JS-05 · JS-06 · JS-20 · JS-21 · JS-22 · JS-23 · JS-26 · JS-27 · JS-28 · JS-53
- **Affected sources — غير مباشرة (2):** JS-46 (عبر `plReport`) · JS-47 (عبر `nrvWritedown`)
- **سلسلة التبعية الكاملة:** **14** مصدراً
- **Reproduction path:** سجّل رصيد 5020 لشهر سابق ⇒ أدخل فاتورة شراء بسعر مختلف للصنف نفسه ⇒ أعد فتح تقرير الشهر السابق ⇒ الرصيد تغيّر
- **Period impact:** **كل الفترات السابقة** — بلا حد زمني
- **Temporary workaround:** طباعة وأرشفة ميزان المراجعة PDF عند كل إقفال شهري كمرجع ثابت خارج النظام
- **Target:** AINV-03 · AINV-02 · **ACC-B**
- **Evidence:** `Accounting_Session5_Invariants_RFC_Decisions_Reviewed.md` §8.1
- **لا يُنقل كما هو.**

## KI-023 — Journal sources use mutable `DB.reportDate`
- **Severity:** 🔴 Critical
- **Root cause:** تواريخ قيود مشتقّة من متغيّر عرض لا من حدث اقتصادي
- **Observed behavior:** تغيير تاريخ التقرير في الواجهة يُحرّك تواريخ قيود مُرحَّلة
- **Expected behavior:** تاريخ القيد من حدث اقتصادي مؤرَّخ ومثبَّت، لا من متغيّر عرض
- **Accounting impact:** قيود تنتقل بين الفترات بلا حركة؛ ويتعذّر إغلاق فترة فعلياً
- **Affected accounts:** 5300 · 2105 · 5015 · 1205 · 5400 · 1110 · 5430 · 2160 · 5900 · 2150 · 1340 · 5910
- **Affected sources — **استخدام قسري:** JS-43 · JS-26 · JS-44 · JS-45 · JS-46 · JS-47
- **Affected sources — fallback متغيّر:** JS-22 · JS-27 · JS-28 · JS-29 · JS-30 · JS-31 · JS-32 · JS-33 (و`recognitionDate()` كخيار ثالث)
- **Reproduction path:** غيّر `DB.reportDate` من نهاية يونيو إلى نهاية يوليو ⇒ قيد المخصص ينتقل بالكامل
- **Period impact:** الفترة الجارية وكل فترة يُعاد عرضها
- **Temporary workaround:** تثبيت `DB.reportDate` على نهاية الفترة المحاسبية أثناء الإقفال، وعدم تغييره قبل الأرشفة
- **Target:** AINV-34 · **ACC-A** · **ACC-B**
- **Evidence:** `Accounting_Session5_Invariants_RFC_Decisions_Reviewed.md` §8.2
- **لا يُنقل كما هو.**

## KI-024 — Deferred journal entries disappear without reversal
- **Severity:** 🔴 Critical
- **Root cause:** قيد التأجيل (EAS 48) يُشتقّ لا يُخزَّن ⇒ يزول بتغيّر حالة الشحنة
- **Observed behavior:** عند تغيير حالة الشحنة إلى «تم التسليم»، يختفي قيد 1190 التاريخي ويظهر قيد البيع الكامل بدلاً منه — **بلا حركة عكسية مؤرَّخة**
- **Expected behavior:** قيد تأجيل مُخزَّن + قيد عكس مؤرَّخ بتاريخ التسليم
- **Accounting impact:** رصيد 1190 يتغيّر بلا حركة تفسّره؛ لا أثر تدقيقي للتأجيل ولا لزواله؛ والإيراد يُثبَّت **بتاريخ الشحن الأصلي** رجعياً
- **Affected accounts:** 1190 · 1200 · 1100 · 4010 · 2100 · 5010
- **Affected sources — JS-05 · JS-06 · JS-07 ⇒ JS-01 · JS-02
- **Reproduction path:** أنشئ فاتورة بشرط DAP غير مُسلَّمة ⇒ سجّل رصيد 1190 ⇒ غيّر الحالة إلى «تم التسليم» ⇒ 1190 يعود صفراً بلا قيد
- **Period impact:** فترة الشحن الأصلية — وقد تكون مقفلة
- **Temporary workaround:** `To be completed in Session 6` — لا حل تشغيلي واضح دون تعديل الكود
- **Target:** AINV-08 · **ACC-A** · **ACC-C**
- **Evidence:** `Accounting_Session5_Invariants_RFC_Decisions_Reviewed.md` §8.3
- **لا يُنقل كما هو.**

---

## فهرس سريع — KI-010 → KI-024

| ID | Severity | Target ADR |
|---|---|---|
| KI-010 | 🔴 Critical | ACC-B |
| KI-011 | 🔴 Critical | ACC-E 🔴 |
| KI-012 | 🔴 Critical | ACC-E 🔴 |
| KI-013 | 🔴 Critical | ACC-E 🔴 |
| KI-014 | 🔴 High | ACC-A (`ADR-020`) |
| KI-015 | 🔴 High | ACC-D |
| KI-016 | 🟡 Medium | ACC-D |
| KI-017 | 🟡 Medium | ACC-E 🔴 |
| KI-018 | 🟡 Medium | ACC-E 🔴 |
| KI-019 | 🟡 Medium | ACC-B |
| KI-020 | 🟡 Medium | ACC-C |
| KI-021 | 🟢 Low | ACC-B |
| KI-022 | 🔴 Critical | ACC-B |
| KI-023 | 🔴 Critical | ACC-A (`ADR-020`) · ACC-B |
| KI-024 | 🔴 Critical | ACC-A (`ADR-020`) · ACC-C |

**التوزيع:** Critical **7** · High **2** · Medium **5** · Low **1** = **15**

---

## قاعدة الإضافة
أي خلل تقني جديد يُكتشف أثناء استخراج الـ Business Logic يُضاف هنا بنفس القالب (KI-XXX)، ولا يُدمَج داخل `docs/02_Governance/Decisions_Log.md` أو ملفات `ADR/` — لأن هذا الملف للأخطاء، وتلك للقرارات المقصودة.
