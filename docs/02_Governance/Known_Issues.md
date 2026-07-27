# Known Issues (مشاكل معروفة — ليست قرارات أعمال)

## Document Information
```
Document Name:  Known Issues
Version:        1.5.0
Status:         KI-001→KI-034 documented; KI-028 refuted/closed by runtime evidence; KI-010→KI-034 remain subject to the applicable Product Owner or Architecture approvals.
Classification: Reference
Owner:          Solution Architecture Team
Approved-by:    Product Owner
Approved-date:  2026-07-18
Last-Updated:   2026-07-25 (Runtime verification correction pass — `Prototype_Runtime_Verification_Report.md`)
Scope-note:     KI-001 → KI-034 موثَّقة. KI-028 مدحوض/مغلق بأدلة التشغيل (لا يُحتسب نشطاً). KI-029 وKI-032 أُعيد تصنيفهما تشغيلياً. KI-034 جديد (كسر render `pricing`). KI-010 → KI-034 تخضع لموافقة Product Owner/Architecture حيثما ينطبق.
```

> **الفرق بين هذا الملف و `docs/02_Governance/Decisions_Log.md`:** كل سطر هنا هو **خلل تقني معروف في الـ Prototype**، وليس قرار عمل مقصود.
> **قاعدة إلزامية لأي AI Agent:** هذه العناصر **لا تُنقل كسلوك مطلوب** أثناء الترحيل. لا تُبنى كـ Business Logic، ولا يُستنتج منها أي قاعدة. تُعالَج فقط في طبقة الـ Backend الجديدة حسب الحل المقترح.
> لو الكود والتوثيق (ADR) تعارضا حول أي بند هنا، **هذا الملف هو الأصح** لأنه موثّق بعد مراجعة متعمّدة، لا افتراض حر.

> **تحديث 2026-07-23 — دمج جلسة Accounting 5:**
> - **KI-008 و KI-009** احتفظا بنطاقهما ووصفهما الأصلي بالكامل، وأُضيف لكلٍّ منهما قسم **«أثر محاسبي — مضاف بمراجعة جلسة Accounting 5»** فقط.
> - **KI-010 → KI-024** (15 خللاً) أُضيفت من `Accounting_Session5_Invariants_RFC_Decisions_Reviewed.md` §8.
> - **جلسة 6.2:** كل حقول KI-010 → KI-024 اكتملت من أدلة الجلسات 3 · 4 · 4.5 · 4.6 · 5 · 6.1 — لا حقول معلَّقة متبقية.

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

- **Related:** AINV-30 · AINV-05 · **ACC-C** (ADR-022 — Proposed)
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
> `Target` يشير إلى الثابت المحاسبي (`AINV-XX`) والـADR المعالج. `ACC-A`=ADR-020 · `ACC-B`=ADR-021 · `ACC-C`=ADR-022 · `ACC-D`=ADR-023 (كلها `Proposed`، صدرت في جلسة 6.2). `ACC-E` **لم يُنشأ** — محجوب بقرار Product Owner (OQ-6.A).


## KI-010 — Customer claim valued at live FX rate — breaches ADR-007
- **Severity:** 🔴 Critical
- **Observed behavior:** ذمة العملة الأجنبية لا تصفو؛ رصيد متبقٍّ دائم بلا تصنيف
- **Expected behavior:** تُقيَّم المطالبة بسعر نشأة الذمة الأصلية (سعر الفاتورة)، لا بـ`fxRate()` الحيّ — التزاماً بـADR-007.
- **Accounting impact:** ذمة العملة الأجنبية لا تصفو؛ رصيد متبقٍّ دائم بلا تصنيف
- **Affected accounts:** 1100 · 5300 · 4030 · 5500
- **Evidence:** JS-08 — مطالبة عميل معتمدة تُقيَّم بـ`fxRate()` **حيّ** بدل سعر نشأة الذمة — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** أنشئ مطالبة بعملة أجنبية ⇒ غيّر سعر الصرف ⇒ أعد بناء الأستاذ ⇒ قيمة المطالبة تغيّرت.
- **Period impact:** كل فترة بها مطالبة بعملة أجنبية؛ رصيد ذمة العملة لا يصفو.
- **Temporary workaround:** إدخال سعر صرف مثبَّت على المطالبة يدوياً عند الإنشاء.
- **Target:** AINV-10 · ACC-B
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-011 — Account 1260 outside every close blocker
- **Severity:** 🔴 Critical
- **Observed behavior:** الهدر يبقى أصلاً ⇒ أصول وأرباح مُضخَّمة
- **Expected behavior:** تسوية رصيد 1260 شرط مانع فعلي في `closeReadiness().blockers` قبل السماح بإقفال الفترة.
- **Accounting impact:** الهدر يبقى أصلاً ⇒ أصول وأرباح مُضخَّمة
- **Affected accounts:** 1260 · 5030 · 2010
- **Evidence:** S4.6 DF-1 — 1260 غير مذكور في `closeReadiness().blockers`؛ الفحص مؤشر إتمام إرشادي لا مانع — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** اترك رصيداً في 1260 ⇒ افتح معالج الإقفال ⇒ زر القفل يبقى مفعّلاً (الفحص مؤشر إتمام لا مانع).
- **Period impact:** كل فترة تُقفَل برصيد هدر قائم ⇒ أصول وأرباح مُضخَّمة.
- **Temporary workaround:** الإجراء البشري في `Monthly_Close_1260_Temporary_Control.md` — مانع بديل مفروض بشرياً.
- **Target:** AINV-24 · ACC-E
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-012 — Account 5300 charged twice; 2105 never consumed
- **Severity:** 🔴 Critical
- **Observed behavior:** ازدواج تحميل على المصروف؛ التزام متراكم
- **Expected behavior:** تحميل واحد على 5300؛ و2105 يُستنفَد عند تحقّق المطالبة الفعلية.
- **Accounting impact:** ازدواج تحميل على المصروف؛ التزام متراكم
- **Affected accounts:** 5300 · 2105 · 1100
- **Evidence:** JS-08 يحمّل 5300 عند المطالبة الفعلية · JS-43 يحمّله ثانيةً كمخصص؛ 2105 بلا مسار استنفاد — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** سجّل مطالبة فعلية (JS-08 ⇒ 5300) ثم مخصصاً (JS-43 ⇒ 5300) لنفس الوعاء ⇒ 5300 مُحمَّل مرتين و2105 يتراكم.
- **Period impact:** تراكم دائم في 2105 بلا استنفاد؛ مصروف مزدوج عبر الفترات.
- **Temporary workaround:** No reliable operational workaround; remediation requires implementation change.
- **Target:** AINV-21 · ACC-E
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-013 — Claims provision base is cumulative in journal, filtered on screen
- **Severity:** 🔴 Critical
- **Observed behavior:** Displayed Basis ≠ Posted Basis؛ ويشمل proforma والمؤجَّل
- **Expected behavior:** وعاء مخصص المطالبات واحد ومتطابق بين قيد الدفتر وشاشة العرض؛ يستبعد proforma والمؤجَّل.
- **Accounting impact:** Displayed Basis ≠ Posted Basis؛ ويشمل proforma والمؤجَّل
- **Affected accounts:** 5300 · 2105
- **Evidence:** S4.6 T2 — `fInvoices()` تُنتج مجتمعين مختلفين لنفس الوعاء بين الدفتر والشاشة — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** قارن أساس `fInvoices()` في الدفتر (تراكمي) بالمعروض على الشاشة (مُفلتَر) لنفس الفترة ⇒ مجتمعان مختلفان (S4.6 T2).
- **Period impact:** كل فترة ذات مخصص مطالبات؛ Displayed Basis ≠ Posted Basis.
- **Temporary workaround:** No reliable operational workaround; remediation requires implementation change.
- **Target:** AINV-13 · AINV-22 · ACC-E
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-014 — Multiple purchase invoices per GRN not prevented
- **Severity:** 🔴 High
- **Observed behavior:** 2016 قد يصير مديناً؛ ازدواج دين المورد
- **Expected behavior:** إقفال جزئي على إذن الاستلام يمنع تعدّد فواتير الشراء على نفس GRN.
- **Accounting impact:** 2016 قد يصير مديناً؛ ازدواج دين المورد
- **Affected accounts:** 2016 · 2010 · 1200
- **Evidence:** S4.5 Target 7 — لا حارس ولا إقفال جزئي على إذن الاستلام — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** أنشئ GRN واحداً ⇒ سجّل فاتورتَي شراء تشيران إليه ⇒ 2016 قد يصير مديناً ودين المورد يتضاعف (S4.5 Target 7).
- **Period impact:** أي فترة بها استلام يسبق فاتورتين؛ GRNI مختلّ.
- **Temporary workaround:** مطابقة يدوية بين كل GRN وفاتورته الوحيدة قبل الترحيل.
- **Target:** AINV-15 · ACC-A
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-015 — `lockPeriodNow()` has no internal guard; `unlockPeriodNow()` needs no permission
- **Severity:** 🔴 High
- **Observed behavior:** القفل قابل للتجاوز والفتح بلا ضابط
- **Expected behavior:** قفل الفترة عبر خدمة مركزية بحارس داخلي؛ والفتح يتطلب صلاحية صريحة وتسجيلاً.
- **Accounting impact:** القفل قابل للتجاوز والفتح بلا ضابط
- **Affected accounts:** كل الحسابات
- **Evidence:** S4.5 Target 5 — ثمانية مسارات قفل، لا مركزي — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** استدعِ `lockPeriodNow()` بلا شرط ⇒ يقفل · و`unlockPeriodNow()` بلا صلاحية ⇒ يفتح (S4.5 Target 5).
- **Period impact:** أي فترة؛ القفل قابل للتجاوز والفتح بلا ضابط.
- **Temporary workaround:** تقييد صلاحية الوصول للدالتين تشغيلياً.
- **Target:** AINV-26 · AINV-28 · ACC-D
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-016 — Undated record bypasses the period lock
- **Severity:** 🟡 Medium
- **Observed behavior:** ترحيل في فترة مقفلة بسجل ناقص
- **Expected behavior:** السجل بلا تاريخ يُرفَض أو يُسنَد لتاريخ صريح؛ لا يتجاوز قفل الفترة.
- **Accounting impact:** ترحيل في فترة مقفلة بسجل ناقص
- **Affected accounts:** كل الحسابات
- **Evidence:** S3 §3.5.4 — مطالبة بتاريخ فارغ تمرّ من حارس القفل (OQ-3.E) — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** أنشئ مطالبة بتاريخ فارغ ⇒ اقفل الفترة ⇒ السجل يمرّ من الحارس (S3 §3.5.4 — `inPeriod` تُمرِّر بلا تاريخ).
- **Period impact:** كل فترة مقفلة معرّضة لترحيل سجل ناقص التاريخ.
- **Temporary workaround:** إلزام حقل التاريخ في كل النماذج يدوياً قبل الحفظ.
- **Target:** AINV-27 · ACC-D
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-017 — No guard against overlapping commission accrual periods
- **Severity:** 🟡 Medium
- **Observed behavior:** التزام عمولة مزدوج
- **Expected behavior:** حارس يمنع تداخل فترات استحقاق العمولة؛ ومسار استنفاد مُصمَّم للحساب 2017.
- **Accounting impact:** التزام عمولة مزدوج
- **Affected accounts:** 5440 · 2017
- **Evidence:** C-01 / OQ-5.NEW — `accrueCommission()` نموذج Snapshot سليم، لكن بلا حارس تداخل فترات — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** شغّل `accrueCommission()` مرتين على نفس المدى (`_cmFrom`/`_cmTo`) ⇒ التزام مزدوج بلا فحص تداخل.
- **Period impact:** أي فترة بها استحقاق متكرر؛ 2017 يتراكم أبدياً.
- **Temporary workaround:** No reliable operational workaround; remediation requires implementation change.
- **Target:** AINV-25 · ACC-E
- **Status:** Open
- **لا يُنقل كما هو.**
- **تحديث جلسة 6.2:** `No dedicated settlement or consumption path was found for account 2017.` النطاق يتسع إلى «لا مسار استنفاد إطلاقاً» — 2017 حساب مراقبة يتراكم أبدياً. Evidence: S6.1 §5.

## KI-018 — NRV excludes components and discards `nrv='0'`
- **Severity:** 🟡 Medium
- **Observed behavior:** مواد خام بلا فحص هبوط؛ أقصى حالة هبوط غير مُخصَّصة
- **Expected behavior:** NRV يشمل المكوّنات ولا يُسقِط القيمة صفر؛ أقصى حالة هبوط تُخصَّص.
- **Accounting impact:** مواد خام بلا فحص هبوط؛ أقصى حالة هبوط غير مُخصَّصة
- **Affected accounts:** 5015 · 1205 · 1200
- **Evidence:** S4.5 Target 3 / C-05 — `nrvWritedown()` تستبعد `component` وتُسقِط القيمة صفر — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** صنف مكوَّن بـ`nrv='0'` ⇒ `nrvWritedown()` تستبعده وتُسقِط الصفر ⇒ لا مخصص (S4.5 Target 3 · C-05).
- **Period impact:** كل فترة بها مواد خام أو أقصى هبوط؛ مخصص NRV منقوص.
- **Temporary workaround:** حساب هبوط المكوّنات يدوياً وإدخاله كقيد تسوية.
- **Target:** AINV-23 · ACC-E
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-019 — LC margin refunded at opening rate — no FX difference
- **Severity:** 🟡 Medium
- **Observed behavior:** فرق عملة حقيقي غير معترَف به
- **Expected behavior:** رد هامش الاعتماد يعترف بفرق العملة بين سعر الفتح وسعر الرد.
- **Accounting impact:** فرق عملة حقيقي غير معترَف به
- **Affected accounts:** 1080 · 4030 · 5500
- **Evidence:** JS-33 / S3 §3.8.2 — رد الهامش بسعر الفتح (OQ-3.H) — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** افتح اعتماداً بهامش أجنبي ⇒ ردّه بسعر الفتح (JS-33) ⇒ الفرق الحقيقي غير معترَف به (S3 §3.8.2 · OQ-3.H).
- **Period impact:** كل فترة بها رد هامش اعتماد أجنبي.
- **Temporary workaround:** قيد تسوية يدوي لفرق العملة عند رد الهامش.
- **Target:** AINV-11 · ACC-B
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-020 — Sales-return cost reversal conditional on a stored field
- **Severity:** 🟡 Medium
- **Observed behavior:** إيراد معكوس بلا تكلفة معكوسة
- **Expected behavior:** عكس تكلفة مرتجع المبيعات إلزامي، لا مشروط بحقل مخزَّن اختياري.
- **Accounting impact:** إيراد معكوس بلا تكلفة معكوسة
- **Affected accounts:** 5010 · 1200 · 4090
- **Evidence:** JS-12 / S3 §3.10.1.4 — `rt.cogsEGP` قد يكون فارغاً — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** سجّل مرتجع مبيعات بـ`rt.cogsEGP` فارغ (JS-12) ⇒ الإيراد يُعكَس والتكلفة لا (S3 §3.10.1.4).
- **Period impact:** أي فترة بها مرتجع مبيعات ناقص الحقل.
- **Temporary workaround:** التأكد يدوياً من ملء `cogsEGP` قبل حفظ كل مرتجع.
- **Target:** AINV-31 · ACC-C
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-021 — Work-order entry uses 1200 on both sides
- **Severity:** 🟢 Low
- **Observed behavior:** حركة المخزون غير قابلة للفصل من الأستاذ
- **Expected behavior:** قيد أمر التشغيل يفصل حركة المخزون الداخلة عن الخارجة في الأستاذ.
- **Accounting impact:** حركة المخزون غير قابلة للفصل من الأستاذ
- **Affected accounts:** 1200 · 1260
- **Evidence:** JS-27 / S4 §4.6.2 (OQ-4.E) — `Accounting_Session5…Reviewed.md` §8.4
- **Reproduction path:** أنشئ أمر تشغيل (JS-27) ⇒ 1200 على الطرفين ⇒ حركة المخزون غير قابلة للفصل (S4 §4.6.2 · OQ-4.E).
- **Period impact:** كل فترة بها أوامر تشغيل.
- **Temporary workaround:** No reliable operational workaround; remediation requires implementation change.
- **Target:** AINV-18 · ACC-B
- **Status:** Open
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
- **Temporary workaround:** No reliable operational workaround; remediation requires implementation change.
- **Target:** AINV-08 · **ACC-A** · **ACC-C**
- **Evidence:** `Accounting_Session5_Invariants_RFC_Decisions_Reviewed.md` §8.3
- **لا يُنقل كما هو.**

---

## KI-025 — Supplier payment without invoice link recognizes no FX difference
- **Severity:** 🔴 High
- **Affected function/entity:** `supplierPayments` block in `buildJournalCore()` · `DB.supplierPayments`
- **Observed behavior:** عند غياب `purchaseId`/`expId` أو `sp.fxRate`، يصير `invRate = payRate` ⇒ `fxDiff = 0`. فرق العملة الحقيقي على ذمة المورد بالعملة الأجنبية **لا يُعترف به**. وعند وجود الربط، `invRate` يُشتقّ حيّاً من الفاتورة وقت بناء الدفتر (امتداد KI-022).
- **Expected behavior:** تجميد سعري السداد والفاتورة على السند؛ الاعتراف بفرق العملة دائماً.
- **Accounting impact:** ذمة المورد بالعملة الأجنبية لا تصفو؛ 4030/5500 منقوصان. نفس نمط KI-019 · KI-010.
- **Violated AINV:** AINV-11 · AINV-10
- **Evidence:** `Accounting_Session6_1_Remaining_Evidence_Findings.md` §3.4 (G-1 · G-2)
- **Workaround:** إلزام ربط السداد بفاتورة يدوياً + إدخال `fxRate` صريح.
- **Target ADR:** ACC-B (ADR-021)
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-026 — Supplier payment has no duplicate guard or invoice-balance ceiling; delete is unaudited
- **Severity:** 🟡 Medium
- **Affected function/entity:** `editSuppPay` · `delSuppPay` · `DB.supplierPayments`
- **Observed behavior:** إنشاء/تعديل سند السداد يفحص `isLocked(date)` ويُسجَّل عبر `audit()`، والحذف يستدعي `canDeleteDated(sp.date)`. لكن لا يوجد حارس يمنع سندين على نفس `purchaseId`، ولا سقف يربط مبلغ السداد برصيد الفاتورة، ولا مفتاح تفرّد تجاري. `delSuppPay` لا يسجّل حدث Audit، وقد يعرض رسالة نجاح مضلِّلة حتى عندما يمنع `canDeleteDated` الحذف. كما لا يوجد تحقق صلاحية داخل إجراء الكتابة نفسه؛ الحماية تعتمد على وصول الواجهة.
- **Expected behavior:** منع التطبيق المزدوج · فرض سقف برصيد الفاتورة · منع السداد الزائد · مفتاح تفرّد مناسب · صلاحية فعلية في إجراء الكتابة · Audit صحيح للحذف بلا رسالة نجاح كاذبة.
- **Accounting impact:** سداد مزدوج أو زائد قد يجعل 2010 مديناً ويشوّه رصيد المورد، مع أثر حذف غير مكتمل التدقيق.
- **Violated AINV:** AINV-04 · AINV-30
- **Evidence:** `Accounting_Independent_Verification_Report.md` Target 2 · prototype lines 13280–13346.
- **Workaround:** مراجعة يدوية لرصيد الفاتورة ومنع تكرار السداد قبل الحفظ، ومراجعة سجل الحذف تشغيلياً.
- **Target ADR:** ACC-A (uniqueness) · ACC-C (reversal/delete) · ACC-D (centralised control)
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-027 — Credit-note delete/edit bypasses period lock; no duplicate guard
- **Severity:** 🟡 Medium
- **Affected function/entity:** `delCreditNote` · `editCreditNote` · `DB.creditNotes`
- **Observed behavior:** `delCreditNote` حذف نهائي بلا عكس. `editCreditNote` يستبدل السجل في مكانه؛ فحص `isLocked` يقع على تاريخ **السجل الجديد** فقط. لا منع لإشعارين على نفس `purchaseId`.
- **Expected behavior:** الحذف/التعديل = قيد عكسي مؤرَّخ يفحص قفل تاريخ الأصل؛ منع ازدواج التطبيق.
- **Accounting impact:** إعادة كتابة تاريخ مقفل · خصم مزدوج من ذمة المورد.
- **Violated AINV:** AINV-30 · AINV-04
- **Evidence:** `Accounting_Session6_1_Remaining_Evidence_Findings.md` §4.5
- **Workaround:** منع الحذف تشغيلياً بعد القفل.
- **Target ADR:** ACC-C (ADR-022) · ACC-D (ADR-023)
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-028 — ~~Treasury opening balance never enters the general ledger~~ · **REFUTED (runtime)** → موثَّق في القسم التاريخي
- **Severity:** ~~🔴 Critical~~ → **لا يُحتسب ضمن Critical النشطة** (مدحوض)
- **Status:** **CLOSED — Refuted by current-prototype runtime evidence** (2026-07-25)
- **Runtime finding:** الرصيد الافتتاحي للخزينة **يدخل الأستاذ فعلاً** ضمن قيد `ref:'OPENING'` عبر الحساب **3001**. مثال مُنفَّذ (OB-01): مدين 1010 = 50,000 · 1021 = 200,000 · 1022 = 252,500 (USD 5,000 @ 50.5) مقابل دائن **3001 أرصدة افتتاحية** = 2,403,839.5؛ الأستاذ والخزينة **متطابقان** وحساب 3001 حاضر.
- **Refuted statement:** «الرصيد الافتتاحي للخزينة لا يدخل الأستاذ العام / يظهر في شاشة الخزينة فقط».
- **الانتقال:** نُقل إلى القسم `## ملحق تاريخي — بنود مدحوضة بأدلة التشغيل (Superseded)` أدناه، مع الحفاظ على النص الأصلي كمرجع تحقيقي فقط. **لا يُبنى منه أي منطق، ولا يُحتسب كخلل نشط.**
- **Evidence (refuting):** `Prototype_Runtime_Verification_Report.md` §4 OB-01 (SHA-256 `8996dd68…d7c6b2`).
- **Historical provenance:** الادعاء الأصلي نشأ من `Accounting_Session6_1_Remaining_Evidence_Findings.md` §6.1 (`Not found after scoped search`) — بحث ثابت مُقيَّد لم يُنفِّذ قيد OPENING؛ التشغيل الحيّ يعلوه.

## KI-029 — Fixed-asset acquisition always defaults to bank funding and does not support payable/credit acquisition selection
- **Severity:** 🟢 Low
- **Affected function/entity:** `DB.assets` · `editAsset` (لا يضبط `treasury`) · `buildJournalCore` (سطور 2921–2928)
- **⚠️ الاستنتاج الأصلي مدحوض (Refuted by current-prototype runtime evidence):** كانت النسخة السابقة تقول «الأصول تُهلَك بلا أي قيد اقتناء». **هذا خطأ.** التشغيل الحيّ (FA-01) أثبت أن `buildJournalCore` يشتقّ لكل أصل `!a.opening` قيد اقتناء فعلياً: **مدين 1300 / دائن البنك** (مثال: أصل جديد بتكلفة 120,000 ⇒ مدين 1300 = 120,000 / دائن 1021 = 120,000، بوصف «شراء أصل ثابت»). **لا تُوصَف كتابة الاقتناء بأنها غائبة.**
- **Observed behavior (النطاق الأضيق القائم):** الاقتناء يُموَّل **دائماً افتراضياً من البنك 1021** لأن `editAsset` لا يضبط `treasury`؛ لا توجد إمكانية اختيار مصدر تمويل **ذمة دائنة / شراء بالأجل** (2010) عند الاقتناء. شراء أصل بالأجل يُقيَّد رغم ذلك كأنه مسدَّد نقداً من البنك.
- **Expected behavior:** السماح باختيار مصدر التمويل عند الاقتناء (بنك **أو** ذمة دائنة/2010) بدل الافتراض الجامد للبنك.
- **Accounting impact:** تبسيط نمذجة فقط — الطرف الدائن قد يكون خاطئاً (بنك بدل ذمة) للمشتريات بالأجل؛ **قيد الاقتناء نفسه حاضر ومتوازن**، لا خلل ميزانية جوهري.
- **Violated AINV:** — (ليس خرق AINV-05؛ الاقتناء يسبق الإهلاك فعلاً)
- **Evidence:** `Prototype_Runtime_Verification_Report.md` §4 FA-01 · D-13 (Low).
- **Historical provenance:** الادعاء الأوسع الأصلي من `Accounting_Session6_1_Remaining_Evidence_Findings.md` §6.2 (`Not found after scoped search`) — دُحض بالتشغيل الحيّ؛ يُحفَظ كتاريخ تحقيق فقط.
- **Workaround:** ضبط مصدر التمويل يدوياً عند شراء أصل بالأجل.
- **Target ADR:** ACC-A (ADR-020) — بند تمويل الاقتناء فقط (Product Owner Decision).
- **Status:** Open (narrowed)
- **لا يُنقل كما هو.**

## KI-030 — Two payroll posting paths can double-count payroll; recurring payroll is misclassified to 5400
- **Severity:** 🔴 Critical
- **Affected function/entity:** `DB.recurring` ⇒ `genRecurring()` ⇒ `DB.expenses` (JS-19) ⟷ `postPayroll()` ⇒ `DB.manualJournals`
- **Observed behavior:** يوجد مساران فعّالان للرواتب:
  1. المسار المتكرر `DB.recurring` يولّد مصروفاً بـ`allocType:'overhead'` فيُرحَّل إلى **5400 مصاريف إدارية وعمومية** بلا فصل الرواتب أو التأمينات أو الضرائب.
  2. المسار المتخصص `postPayroll()` يرحّل فعلياً إلى **5410 · 5420 · 2120 · 2130 · 2140**.
  لا يوجد Cross-Guard يمنع تشغيل المسارين لنفس الشهر، وبالتالي يمكن إثبات الرواتب مرتين، مرة كمصروف عام ومرة كقيد رواتب متخصص.
- **Expected behavior:** مسار رواتب إنتاجي واحد، أو مفتاح تفرّد مشترك على الشهر/الدورة يمنع الترحيل المزدوج، مع تعطيل أو إعادة تصنيف RC-002 بحيث لا يمر كـoverhead عام.
- **Accounting impact:** ازدواج مصروف الرواتب والالتزامات، وتشويه عرض المصروفات بين 5400 و5410/5420.
- **Violated AINV:** AINV-04 · AINV-21 · EAS 1 presentation
- **Evidence:** `Accounting_Independent_Verification_Report.md` Target 3 · prototype lines 3032–3060, 649, 12343, 12408–12435.
- **Workaround:** استخدام مسار واحد فقط لكل شهر مع مطابقة يدوية لمرجع `PAYROLL-{month}` وعدم توليد RC-002 للشهر نفسه.
- **Target ADR:** ACC-A (uniqueness) · ACC-B (classification inputs) · OQ-6.D
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-031 — Two parallel recurring engines; the older `recurringExp` editor is shadowed and unreachable
- **Severity:** 🟡 Medium
- **Affected function/entity:** `DB.recurring` (`genRecurring`) · `DB.recurringExp` (`generateRecurring`/`recurringDue`) · `window.editRecurring` (مُعرَّفة مرتين)
- **Observed behavior:** يوجد محركان متوازيان للقيود المتكررة. `window.editRecurring` مُعرَّفة مرتين، والتعريف اللاحق الخاص بـ`DB.recurring` يظلّل التعريف الأقدم الخاص بـ`DB.recurringExp`. شاشة `DB.recurring` تفتح النموذج الصحيح؛ لكن محرر `DB.recurringExp` الأقدم يصبح Dead Code غير قابل للوصول من واجهة قائمة مماثلة.
- **Expected behavior:** محرك متكرر واحد بكيان وواجهة وتسمية دوال غير متعارضة، أو أسماء مستقلة صريحة لكل محرك حتى إلغاء أحدهما.
- **Accounting impact:** خطر صيانة وارتباك مصدر البيانات، مع مسار قد يظل موجوداً في التخزين أو التوليد دون واجهة تحرير واضحة.
- **Violated AINV:** — (نمط KI-005 البنيوي)
- **Evidence:** `Accounting_Independent_Verification_Report.md` Target 3 · prototype definitions at lines 8104 and 11510.
- **Workaround:** استخدام `DB.recurring` فقط بوعي، وعدم الاعتماد على محرر `DB.recurringExp` المظلَّل.
- **Target ADR:** — (تنظيف كود في البناء الجديد)
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-032 — Two parallel FX-revaluation engines with no consolidation; Engine A bypasses 1265, Engine B is inert; double recognition is a latent code risk (not runtime-reproduced)
- **Severity:** 🟡 Medium *(خُفِّضت من Critical: الازدواج المزدوج لم يُعَد إنتاجه تشغيلياً)*
- **Affected function/entity:** المحرك A `DB.revaluations` (`postRevaluation`) ⟷ المحرك B `DB.fxRevaluations` (`postFxRevaluation`/`fxReval`/`fxExposure()`)
- **Observed behavior (مُثبَت — يبقى قائماً):**
  - **يوجد محركان** لإعادة التقييم يقرآن نفس الأرصدة الأجنبية المفتوحة. **لا يوجد قرار توحيد** بينهما.
  - **المحرك A** يُرحِّل على حسابات الأطراف (1100/1020/2010) مقابل 4030/5500 مع عكس تلقائي، **ويتجاوز 1265** (نموذج مختلف عن ADR-003). أُثبت تشغيلياً end-to-end (FX-01: `REV-001 net=27,621`).
  - **المحرك B** يستخدم نموذج **1265** (مدين 1265 / دائن 4030 عند الترحيل؛ مدين 3020 / دائن 1265 عند الإقفال).
- **Runtime corrections (مُصحَّح بأدلة التشغيل 2026-07-25):**
  - **الازدواج المزدوج (double posting) لم يُعَد إنتاجه تشغيلياً** — يُصنَّف الآن **latent code risk** (خطر كامن على مستوى الكود)، **لا سلوك تشغيلي مؤكَّد**. كلا المحركين يملكان مُصدِّر 4030/5500 غير مشروط وبلا Cross-Guard (حقيقة كود)، لكن التشغيل المتزامن لم يُنتِج قيدين.
  - **المحرك B فعلياً خامل/معطَّل (inert/broken) في التشغيل المختبَر:** `fxExposure()` يعيد **صفراً حتى بعد تحرُّك سعر 9%** (EUR 55.2→60, USD 50.5→53)، لأن «المبني بالجنيه» يُعاد حسابه بنفس السعر الحالي الذي يُقيَّم مقابله ⇒ `diff ≈ 0` لكل طرف. فيرفض الترحيل («لا يوجد فرق تقييم للترحيل») ويُنتِج **صفر تعرُّض**. لذا فسطور قيده (1265/3020) **غير قابلة للتحقق end-to-end (NOT VERIFIABLE)** — موجودة في الكود، غير قابلة للوصول عبر بيانات المحرك.
- **Expected behavior:** نظام إعادة تقييم **واحد** متسق (نموذج ترحيل/عكس واحد)، مع إصلاح أساس تعرُّض المحرك B (مقارنة السعر وقت الفاتورة بالسعر الحالي) أو إلغاؤه.
- **Accounting impact:** خطر كامن لازدواج فرق العملة غير المحقق **إن أُصلح المحرك B وشُغِّل مع A**؛ حالياً لا ازدواج مُنتَج تشغيلياً. اختلاف نموذج A عن 1265 يبقى مسألة معمارية.
- **Violated AINV:** AINV-10 (نمط) · تعارض نموذجَي 1265
- **Evidence:** `Prototype_Runtime_Verification_Report.md` §4 FX-01/FX-02/FX-03 · §5 · `Accounting_Session6_1_Remaining_Evidence_Findings.md` §7.2 (تاريخي).
- **Workaround:** استخدام المحرك A فقط تشغيلياً حتى صدور قرار التوحيد؛ عدم الاعتماد على المحرك B (خامل).
- **Target ADR:** يحتاج قرار **OQ-6.F (Product Owner) — يبقى 🔴 Open** — **ليس قراراً يُحسَم في KI**
- **Status:** Open (reclassified — latent risk)
- **لا يُنقل كما هو.**

## KI-033 — Multiple uncentralized closing/period-lock paths; `doMonthEndClose()` locks without the closing entry
- **Severity:** 🟡 Medium
- **Affected function/entity:** `doClosing()` · `doMonthEndClose()` · `lockPeriodNow()` · settings/fiscal-year lock paths · `DB.lockDate` · `DB.closings`
- **Observed behavior:** `doClosing()` ينشئ قيد الإقفال 3099⇄3020 داخل `DB.manualJournals`، يضبط `DB.lockDate`، ويمتلك Permission وAudit وحارس تكرار. لكنه واحد من عدة مسارات غير مركزية تكتب `DB.lockDate`. المسار الشقيق `doMonthEndClose()` يضبط القفل فقط بلا إنشاء قيد 3099/3020 وبلا نفس مستوى الـPermission/Audit. قيد الإقفال نفسه لم يكن مُكتلَجاً قبل اقتراح JS-59.
- **Expected behavior:** خدمة إقفال وقفل مركزية واحدة تنفذ الجاهزية والصلاحيات والـAudit، وتضمن أن الإقفال المحاسبي والقفل يحدثان كعملية واحدة متسقة.
- **Accounting impact:** إمكانية وجود فترة مقفلة بلا قيد إقفال مقابل، واختلاف النتائج حسب مسار الواجهة المستخدم.
- **Violated AINV:** AINV-26 · AINV-32
- **Evidence:** `Accounting_Independent_Verification_Report.md` Target 5 · prototype lines 8281–8285, 11272–11296, 14212–14213.
- **Workaround:** استخدام `doClosing()` فقط تشغيلياً، ومنع مسارات القفل المنفردة حتى توحيدها.
- **Target ADR:** ACC-D (ADR-023)
- **Status:** Open
- **لا يُنقل كما هو.**

## KI-034 — Pricing screen fails to render because `esc` is undefined
- **Severity:** 🔴 High
- **Affected function/entity:** شاشة `pricing` (render) · `const esc` معرَّف على المستوى الأعلى داخل **7** كتل `<script>` منفصلة (21983 · 22611 · 22806 · 23167 · 23569 · 23808 …)
- **Observed behavior (runtime):** فتح شاشة `pricing` يرمي **`ReferenceError: esc is not defined`** لحظة الـrender، فتفشل الشاشة في الظهور كلياً. السبب: `esc` مُعرَّف كـ`const` علوي مكرَّر عبر كتل سكربت متعددة، فلا يُحَل المعرِّف (`undefined`) في سياق الـrender.
- **Impact:** واجهة التسعير **غير متاحة** بالكامل (شاشة مكسورة — لا محتوى).
- **Destination:** **Production Frontend** (وأيضاً قابلة لـ**Verification Patch**).
- **Verification-patch eligibility:** ✅ **نعم** — التصحيح مجرد توحيد تعريف `esc` واحد قابل للوصول (dedupe الـ7 تعريفات العلوية) حتى يُحَل المعرِّف. **أثر منطق الأعمال = NONE** (حَلّ معرِّف بحت؛ لا تغيير حساب).
- **Evidence:** `Prototype_Runtime_Verification_Report.md` §3 (BROKEN) · §8 D-01 · §9 (Verification-patch candidate).
- **Related:** (خلل تشغيلي جديد — لم يكن في التدقيق الثابت)
- **Target ADR:** — (إصلاح Frontend/Verification Patch، لا قرار أعمال)
- **Status:** Open
- **لا يُنقل كما هو.**

---

## ملاحظة تقاطعية — تجاوزات صلاحية مؤكَّدة تشغيلياً (Handler-level bypass) — تُعزِّز KI-002

> **الجذر المشترك = KI-002** (التحقق على الواجهة فقط، بلا تفويض على الخادم). البنود التالية **أدلة تشغيلية** على أن بعض مُعالِجات الكتابة **لا تفحص `can()` إطلاقاً** (تجاوز على مستوى الـHandler، لا مجرد إخفاء زر). **لا تُنشأ KIs مكرَّرة** — تُربَط بـKI-002 وبالـKI المختص:
>
> | إجراء الكتابة | تجاوز مؤكَّد (runtime) | التمييز | KI مرتبط |
> |---|---|---|---|
> | `editSuppPay` (إنشاء/تعديل/حذف سند سداد) | Viewer أنشأ سنداً عبر زر ظاهر + handler بلا `can()` | UI ظاهر **و** handler غير محمي | **KI-002** · KI-026 |
> | `postPayroll` (ترحيل رواتب) | Accountant (`hr:edit=false`) رحّل `PAYROLL-2026-06` | زر ظاهر **و** handler غير محمي | **KI-002** · KI-030 |
> | `editRecurring` (إنشاء/تعديل متكرر) | Viewer: الزر مخفي لكن الـhandler ينفَّذ عند الاستدعاء المباشر | إخفاء UI فقط؛ handler غير محمي | **KI-002** · KI-031 |
> | `postRevaluation` (المحرك A لإعادة التقييم) | Viewer رحّل عبر استدعاء مباشر؛ **الـhandler بلا `can()`** | إخفاء UI فقط؛ handler غير محمي | **KI-002** · KI-032 |
> | `lockPeriodNow`/`unlockPeriodNow` (قفل/فتح فترة) | غير مُقيَّدة بصلاحية؛ أي دور يملك الوصول ينفّذها | handler غير محمي | **KI-002** · KI-033 |
>
> **مقابلها — مُعالِجات محميّة صحيحة (`can()` أعلى الـhandler):** `postFxRevaluation` · `closeFxRevaluation` · `doClosing` · `editAsset` (`accounts:edit`).
>
> **ثلاث طبقات إنفاذ يجب فصلها في الترحيل:** (1) **إخفاء UI** (UX فقط) · (2) **إنفاذ على مستوى الـHandler** (ناقص في الأعلى) · (3) **الإنفاذ الإلزامي على الـBackend الجديد** (المصدر الوحيد للحماية — KI-002). **الواجهة والـhandler الحاليان لا يُعتمَد عليهما كحماية.**
>
> **Evidence:** `Prototype_Runtime_Verification_Report.md` §6 (Permission runtime matrix) · §4 SP-07 · PAY-05 · FX-01/§5.

---

## ملحق تاريخي — بنود مدحوضة بأدلة التشغيل (Superseded)

> هذه البنود **مغلقة/مدحوضة** بأدلة تشغيل `prototype_v2.html` الحيّة. **لا تُحتسب كأخطاء نشطة، ولا يُبنى منها منطق.** تُحفَظ للـprovenance فقط.

### KI-028 (Superseded) — النص الأصلي (تحقيقي فقط)
> **الادعاء الأصلي (مدحوض):** «Treasury opening balance never enters the general ledger — الرصيد الافتتاحي للخزينة يُضاف إلى شاشة الخزينة فقط ولا قيد له في الأستاذ مقابل 3001.»
> **مصدر الادعاء:** `Accounting_Session6_1_Remaining_Evidence_Findings.md` §6.1 (`Not found after scoped search`).
> **الدحض:** `Prototype_Runtime_Verification_Report.md` §4 OB-01 — قيد `OPENING` يدين الخزينة ويقيّد **3001**؛ الأستاذ والخزينة متطابقان. **مغلق 2026-07-25.**

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
| KI-025 | 🔴 High | ACC-B |
| KI-026 | 🟡 Medium | ACC-A · ACC-C · ACC-D |
| KI-027 | 🟡 Medium | ACC-C · ACC-D |
| ~~KI-028~~ | **Refuted/Closed** (runtime) | — (تاريخي) |
| KI-029 | 🟢 Low *(narrowed)* | ACC-A (funding only) |
| KI-030 | 🔴 Critical | ACC-B · OQ-6.D |
| KI-031 | 🟡 Medium | — |
| KI-032 | 🟡 Medium *(reclassified)* | OQ-6.F 🔴 Open |
| KI-033 | 🟡 Medium | ACC-D |
| KI-034 | 🔴 High | — (Frontend / Verification Patch) |

**التوزيع النشط (KI-010→034، باستثناء KI-028 المدحوض):** Critical **9** · High **4** · Medium **9** · Low **2** = **24 نشطة**
> **تغييرات المراجعة التشغيلية (2026-07-25):**
> - **KI-028:** Critical → **مدحوض/مغلق** (لا يُحتسب نشطاً).
> - **KI-029:** Critical → **Low** (ضُيِّق إلى «تمويل الاقتناء يفترض البنك دائماً»؛ قيد الاقتناء نفسه مؤكَّد تشغيلياً).
> - **KI-032:** Critical → **Medium** (الازدواج latent لا مؤكَّد؛ المحرك B خامل).
> - **KI-034:** جديد 🔴 High (كسر render شاشة `pricing`).
> - صافي Critical النشطة: 11 → **9** (KI-028 خرج، KI-029 وKI-032 خُفِّضتا).

---

## قاعدة الإضافة
أي خلل تقني جديد يُكتشف أثناء استخراج الـ Business Logic يُضاف هنا بنفس القالب (KI-XXX)، ولا يُدمَج داخل `docs/02_Governance/Decisions_Log.md` أو ملفات `ADR/` — لأن هذا الملف للأخطاء، وتلك للقرارات المقصودة.
