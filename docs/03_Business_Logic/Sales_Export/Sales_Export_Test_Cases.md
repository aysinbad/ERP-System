# Sales / Export Test Cases

## Document Information
```
Document Name:  Sales / Export Test Cases
Version:        1.0.0
Status:         Draft — In Review
Classification: Source of Truth
Owner:          Product Owner
Last-Updated:   2026-07-18
```

> **الترقيم:** `TC-SLE-XXX`. كل حالة مربوطة بقاعدة/Invariant/كود خطأ من `Sales_Export.md` أو `Sales_Export_Implementation_Guide.md`. **لا سيناريو مخترَع** — كل حالة إما (أ) تعكس سلوكاً فعلياً في الكود، أو (ب) موسومة صراحة "سلوك متوقَّع بعد الإصلاح" لو كانت تختبر حلاً لـ Known Issue لم يُنفَّذ بعد.

---

## أ. التحويلات الطبيعية (Happy Path Transitions) — 10 حالات

### TC-SLE-001 — تحويل quotation → proforma ناجح
**Given:** quotation بحالة `draft`، بدون `toDoc`
**When:** يُستدعى `sdConvert(id)`
**Then:** proforma جديدة تُنشأ بنفس البيانات، `quotation.toDoc = <proforma.id>`, `proforma.fromDoc = <quotation.id>`

### TC-SLE-002 — تحويل proforma → contract ناجح
**Given:** proforma بحالة `sent`
**When:** `sdConvert(id)`
**Then:** contract جديد يُنشأ، `proforma.toDoc` يُضبَط، السلسلة (`fromDoc`/`toDoc`) متصلة بالكامل من quotation

### TC-SLE-003 — تحويل contract → order ناجح
**Given:** contract بحالة `active`
**When:** `sdConvert(id)`
**Then:** order جديد يُنشأ بنفس الأصناف والكميات والأسعار

### TC-SLE-004 — تحويل order → invoice ناجح (أول مرة)
**Given:** order بحالة `approved`، `linkedInvoiceId` فارغ
**When:** مستخدم بصلاحية `invoices:edit` يستدعي `sdConvert(id)`
**Then:** فاتورة جديدة في `DB.invoices`، `order.status='fully_shipped'`, `order.linkedInvoiceId = <invoiceId>`

### TC-SLE-005 — السلسلة الكاملة من البداية للنهاية
**Given:** quotation جديد بـ 3 أصناف
**When:** تُنفَّذ 4 تحويلات متتالية (quotation→proforma→contract→order→invoice)
**Then:** الفاتورة النهائية تحتوي نفس الـ 3 أصناف بنفس الكميات، وسلسلة `fromDoc` كاملة تربط كل مستند بأصله حتى quotation الأصلي

### TC-SLE-006 — نسخ الأصناف بدون تعديل عند التحويل
**Given:** proforma بصنف واحد (qty=50, unitPrice=12.00)
**When:** `sdConvert` إلى contract
**Then:** الصنف في الـ contract الجديد له نفس `qty` و`unitPrice` بالحرف — لا إعادة حساب

### TC-SLE-007 — إنشاء quotation جديد بأصناف متعددة
**Given:** مستخدم يفتح `editSalesDoc(-1)`، يضيف صنفين بكميات موجبة
**When:** يحفظ
**Then:** `items.length === 2`, `audit('create', 'عرض سعر', ...)` مسجَّلة

### TC-SLE-008 — العملة تُنسَخ كما هي عبر السلسلة
**Given:** proforma بعملة EUR
**When:** تحويلها لـ contract ثم order
**Then:** كل المستندات في السلسلة تحمل نفس عملة المصدر EUR — لا تحويل عملة تلقائي في `sdConvert`

### TC-SLE-009 — مرجع الفرصة (`linkedOppId`) يبقى محفوظاً عبر التحويل
**Given:** proforma لها `linkedOppId` من فرصة CRM
**When:** تحويلها إلى contract
**Then:** **يحتاج تأكيداً بالكود** — هل `linkedOppId` يُنسَخ للمستند الجديد أم يبقى على البروفورما فقط؟ (لم يُفحَص بعد بدقة — Open Question جديد)

### TC-SLE-010 — الميناء وشرط التسليم يُنسخان عبر السلسلة
**Given:** quotation بـ `incoterm='FOB'`, `port='DEKHEILA'`
**When:** تحويل متتالي حتى order
**Then:** كل مستند في السلسلة يحمل نفس `incoterm`/`port` — إلا لو عُدِّلا يدوياً بعد التحويل

---

## ب. حالات الرفض/الإنهاء (Terminal States) — 8 حالات

### TC-SLE-011 — quotation مرفوض لا يُحوَّل
**Given:** quotation بحالة `rejected`
**When:** يحاول مستخدم استدعاء `sdConvert(id)`
**Then:** ⚠️ **الكود الحالي لا يمنع هذا صراحة** — `sdConvert` بتتحقق من `!doc.toDoc` بس، مش من الحالة. **متوقَّع فعلياً: التحويل ينجح رغم الرفض** — هذا سلوك موثَّق كفجوة، وليس افتراضاً؛ يحتاج قرار (انظر Open Questions في `Sales_Export.md`)

### TC-SLE-012 — quotation منتهي الصلاحية (`expired`) — نفس ملاحظة TC-SLE-011
**Given:** quotation بحالة `expired`، `validUntil` في الماضي
**When:** `sdConvert(id)`
**Then:** نفس السلوك — لا يوجد فحص فعلي لتاريخ `validUntil` ولا للحالة `expired` قبل التحويل

### TC-SLE-013 — مستند ملغي (`cancelled`) — نفس النمط
**Given:** proforma بحالة `cancelled`
**When:** `sdConvert(id)`
**Then:** نفس الفجوة — لا يوجد حارس على الحالة، فقط على `toDoc`

### TC-SLE-014 — محاولة تحويل مستند محوَّل بالفعل (السلسلة الأساسية)
**Given:** quotation له `toDoc` مملوء بالفعل
**When:** `sdConvert(id)` يُستدعى مرة تانية
**Then:** رفض فوري + رسالة "تم التحويل مسبقاً" — **يعمل بشكل صحيح** لهذا الجزء من السلسلة

### TC-SLE-015 — 🔴 محاولة تحويل order له `linkedInvoiceId` بالفعل (KI-009)
**Given:** order له `linkedInvoiceId` مملوء (فاتورة سابقة موجودة)، **لكن `toDoc` فارغ** (لأن فرع الفاتورة لا يضبطه)
**When:** `sdConvert(id)` يُستدعى مرة تانية
**Then:** ❌ **السلوك الفعلي الحالي: التحويل ينجح وينشئ فاتورة ثانية مكرّرة** — هذا الاختبار يوثّق الباگ نفسه (KI-009)، **وليس السلوك المرغوب**

### TC-SLE-016 — ✅ (بعد الإصلاح المقترح) رفض تحويل order مفوتَر بالفعل
**Given:** نفس TC-SLE-015، بعد تطبيق حل `Sales_Export_Implementation_Guide.md` (ضبط `toDoc` في فرع الفاتورة)
**When:** `sdConvert(id)` يُستدعى مرة تانية
**Then:** رفض بكود `SLE_ORDER_ALREADY_INVOICED` — **سلوك متوقَّع بعد الإصلاح، غير موجود في الكود الحالي**

### TC-SLE-017 — حذف quotation بحالة `draft` (بلا مشاكل تكامل)
**Given:** quotation `draft`، بلا `toDoc`
**When:** `delSalesDoc(id)`
**Then:** يُحذَف بنجاح — لا يوجد أثر جانبي لأن لا شيء يعتمد عليه

### TC-SLE-018 — 🔴 حذف مستند له `toDoc` مملوء (KI-008)
**Given:** proforma لها `toDoc` يشاور على contract فعلي
**When:** `delSalesDoc(proforma.id)`
**Then:** ❌ **السلوك الفعلي: الحذف ينجح بلا أي تحذير** — الـ contract يفضل بـ `fromDoc` يشاور على مستند محذوف. هذا يوثّق KI-008 نفسها، وليس سلوكاً مرغوباً

---

## ج. الصلاحيات (Permissions) — 6 حالات

### TC-SLE-019 — تحويل order→invoice بلا صلاحية `invoices:edit`
**Given:** مستخدم بلا صلاحية `invoices:edit`، order بحالة `approved`
**When:** يحاول `sdConvert(orderId)`
**Then:** الفحص `can('invoices','edit')` يمنع العملية

### TC-SLE-020 — تحويل order→invoice بصلاحية `invoices:edit`
**Given:** مستخدم يملك الصلاحية
**When:** `sdConvert(orderId)`
**Then:** التحويل ينجح بلا عوائق

### TC-SLE-021 — إنشاء quotation بلا أي صلاحية خاصة
**Given:** مستخدم بصلاحية القسم العامة بس (مش `invoices:edit`)
**When:** ينشئ quotation جديد
**Then:** ينجح — لا فحص صلاحية إضافي على هذه الخطوة

### TC-SLE-022 — تحويل proforma→contract بلا أي صلاحية خاصة
**Given:** نفس المستخدم أعلاه
**When:** `sdConvert` على proforma
**Then:** ينجح بلا فحص — **فقط order→invoice محمية**، باقي السلسلة مفتوحة لأي مستخدم بصلاحية القسم

### TC-SLE-023 — حذف مستند بلا أي صلاحية خاصة (KI-008 من زاوية الصلاحيات)
**Given:** مستخدم عادي بصلاحية القسم فقط
**When:** `delSalesDoc(id)` على أي نوع مستند
**Then:** ينجح بلا أي فحص صلاحية إضافي — يوثّق غياب الحماية

### TC-SLE-024 — تغيير الحالة الفرعية (`sdSetStatus`) بلا صلاحية خاصة
**Given:** أي مستخدم بصلاحية القسم
**When:** `sdSetStatus(id, 'cancelled')`
**Then:** ينجح فوراً بلا فحص صلاحية ولا تسجيل تدقيق (يوثّق فجوة Audit Rules)

---

## د. الشحن (Shipping) — 8 حالات

### TC-SLE-025 — مستندات الشحن المطلوبة لشرط تسليم عادي (FOB)
**Given:** فاتورة بشرط `FOB`
**When:** `shipDocSummary(inv)`
**Then:** 5 مستندات مطلوبة (فاتورة تجارية، قائمة تعبئة، شهادة منشأ، شهادة صحة نباتية، بوليصة شحن) — **التأمين غير مطلوب**

### TC-SLE-026 — مستند التأمين مطلوب لشرط CIF
**Given:** فاتورة بشرط `CIF`
**When:** `shipDocRequired(inv, 'ins')`
**Then:** `true` — 6 مستندات مطلوبة بدل 5

### TC-SLE-027 — مستند التأمين مطلوب لشرط CIP
**Given:** فاتورة بشرط `CIP`
**When:** `shipDocRequired(inv, 'ins')`
**Then:** `true` — نفس منطق CIF

### TC-SLE-028 — نسبة اكتمال المستندات (`pct`) عند نقص مستند واحد
**Given:** فاتورة FOB (5 مطلوبة)، 4 معتمدة، 1 لسه `required`
**When:** `shipDocSummary(inv)`
**Then:** `req:5, done:4, pct:80`

### TC-SLE-029 — حالة الشحن الابتدائية
**Given:** فاتورة جديدة فور الإنشاء
**When:** تُقرَأ `shipStatus`
**Then:** القيمة الافتراضية "قيد التجهيز" (يحتاج تأكيداً دقيقاً من نموذج إنشاء الفاتورة)

### TC-SLE-030 — انتقال الشحن إلى "في الطريق"
**Given:** فاتورة `shipStatus='قيد التجهيز'`
**When:** يُحدَّث الحقل يدوياً إلى "في الطريق"
**Then:** يُحفَظ التغيير — **لا يوجد حدث Domain Event مرتبط حالياً** (فجوة موثَّقة في Implementation Guide)

### TC-SLE-031 — انتقال الشحن إلى "تم التسليم" يُزيل شرط تأجيل الإيراد
**Given:** فاتورة بشرط `DDP`، `shipStatus` لسه مش "تم التسليم"
**When:** `shipStatus` يتحدَّث لـ "تم التسليم"
**Then:** `isDeferredIncoterm(inv)` تتحول من `true` إلى `false` فوراً

### TC-SLE-032 — تنبيه لوحة القيادة لمستندات شحن ناقصة
**Given:** فاتورة غير مُسلَّمة، بمستندات ناقصة
**When:** `topAlerts()` تُبنى
**Then:** تنبيه يظهر يشير لنقص المستندات — تنبيه عام على مستوى النظام، مش مرتبط بمستند بيع بعينه

---

## هـ. Incoterms — 6 حالات

### TC-SLE-033 — 🔴 شرط `DAT` غير متاح للاختيار في نموذج مستند البيع (SINV-07)
**Given:** مستخدم يفتح `salesDocForm`
**When:** يفحص القائمة المنسدلة لشرط التسليم
**Then:** `DAT` **غير موجود ضمن الخيارات المتاحة** (`C&F, FOB, CIF, DDP, EXW, FCA` فقط)

### TC-SLE-034 — 🔴 شرط `DAT` غير متاح في نموذج الفاتورة كذلك
**Given:** مستخدم يفتح نموذج الفاتورة (`f_terms`)
**When:** يفحص القائمة
**Then:** `DAT` **غير موجود** (القائمة: `C&F, FOB, CIF, DDP, EXW, FCA, DAP, CPT`) — **بالتالي شرط تأجيل الإيراد الخاص بـ DAT كود ميت عملياً في كل النظام**

### TC-SLE-035 — 🔴 شرط `DPU` غير متاح في نموذج الفاتورة تحديداً
**Given:** نفس نموذج الفاتورة
**When:** فحص القائمة
**Then:** `DPU` غير موجود، رغم وجوده في إعداد شروط العميل الافتراضية — تضارب مباشر بين مصدر القيمة الافتراضية والشاشة الفعلية

### TC-SLE-036 — القيمة الافتراضية لشرط التسليم من إعداد العميل
**Given:** عميل له شرط افتراضي `DAP`
**When:** يُنشأ مستند بيع جديد لهذا العميل
**Then:** الحقل يُملأ تلقائياً بـ `DAP` عبر `syncTerms()`

### TC-SLE-037 — شرط `EXW` لا يتطلب تأمين ولا تأجيل إيراد
**Given:** فاتورة بشرط `EXW`
**When:** `shipDocRequired(inv,'ins')` و`isDeferredIncoterm(inv)`
**Then:** الاثنان `false` — أقل شروط التسليم التزاماً على البائع

### TC-SLE-038 — تغيير شرط التسليم بعد إنشاء المستند
**Given:** proforma بشرط `FOB` مُنشأة بالفعل
**When:** المستخدم يعدّل الحقل لـ `CIF` قبل التحويل لـ contract
**Then:** التعديل يُحفَظ بلا قيد، ويُطبَّق على المستند التالي في السلسلة (الـ contract الجديد يرث `CIF`)

---

## و. تأجيل الإيراد (Revenue Deferral — EAS 48) — 8 حالات

### TC-SLE-039 — تأجيل نشط لشرط DDP قبل التسليم
**Given:** فاتورة `terms='DDP'`, `shipStatus` ≠ "تم التسليم"
**When:** `isDeferredIncoterm(inv)`
**Then:** `true`

### TC-SLE-040 — تأجيل نشط لشرط DAP قبل التسليم
**Given:** فاتورة `terms='DAP'`, لم تُسلَّم بعد
**When:** `isDeferredIncoterm(inv)`
**Then:** `true`

### TC-SLE-041 — 🔴 تأجيل DPU — يعمل منطقياً لكن غير قابل للاختيار عملياً (مرتبط بـ TC-SLE-035)
**Given:** فرضاً فاتورة `terms='DPU'` (لو أُدخلت يدوياً عبر تعديل بيانات مباشر، لأنها غير متاحة من القائمة)
**When:** `isDeferredIncoterm(inv)`
**Then:** منطقياً `true` — **لكن هذا السيناريو غير قابل للوصول من أي واجهة مستخدم فعلية حالياً**

### TC-SLE-042 — 🔴 تأجيل DAT — نفس ملاحظة كود ميت (مرتبط بـ TC-SLE-034)
**Given:** فرضاً فاتورة `terms='DAT'`
**When:** `isDeferredIncoterm(inv)`
**Then:** منطقياً `true` — **غير قابل للوصول من أي شاشة بالنظام كله**

### TC-SLE-043 — إزالة التأجيل فور "تم التسليم"
**Given:** فاتورة `DDP` كانت مؤجَّلة (`isDeferredIncoterm=true`)
**When:** `shipStatus` يتحول لـ "تم التسليم"
**Then:** `isDeferredIncoterm(inv)` تُعيد `false` فوراً في الاستدعاء التالي

### TC-SLE-044 — لا تأجيل لشروط المصدر (FOB/EXW/CIF)
**Given:** فاتورة بأي من `FOB`, `EXW`, `CIF`
**When:** `isDeferredIncoterm(inv)`
**Then:** `false` دائماً بغض النظر عن `shipStatus` — هذه الشروط لا تُدرَج في قائمة التأجيل خالص

### TC-SLE-045 — حقل `incoterm` أو `terms` — أيهما يُقرَأ فعلياً؟
**Given:** فاتورة لها `incoterm` فارغ لكن `terms='DDP'` مملوء
**When:** `isDeferredIncoterm(inv)`
**Then:** تقرأ `inv.incoterm||inv.terms` — **`terms` يُستخدَم كـ fallback**، يعمل بشكل صحيح رغم ازدواج اسم الحقل عبر أنواع المستندات المختلفة

### TC-SLE-046 — لا إعادة تقييم تلقائية عبر Domain Event
**Given:** فاتورة تتحول لـ "تم التسليم"
**When:** لا يوجد حدث `sales.shipment.delivered` مُطلَق فعلياً (مقترح فقط في Implementation Guide)
**Then:** أي تقرير محاسبي (Accounting) يحتاج **يعيد استدعاء `isDeferredIncoterm` بنفسه من جديد** كل مرة، بدل ما يُخطَر تلقائياً — فجوة تكامل موثَّقة

---

## ز. حالات Known Issues (اختبارات موثِّقة للثغرات نفسها) — 6 حالات

### TC-SLE-047 — KI-008: حذف مستند بلا تسجيل تدقيق
**Given:** أي مستند بيع
**When:** `delSalesDoc(id)`
**Then:** لا يوجد أي استدعاء لـ `audit()` — الحذف يمر بصمت تماماً في سجل التدقيق

### TC-SLE-048 — KI-009: زر "إنشاء فاتورة" يفضل ظاهراً بعد أول استخدام
**Given:** order له `linkedInvoiceId` مملوء بالفعل
**When:** تُعاد قراءة `canConv` في `viewSalesDocs`
**Then:** `canConv === true` (لأن `!doc.toDoc` لسه `true`) — الزر ظاهر وقابل للنقر رغم وجود فاتورة سابقة

### TC-SLE-049 — SINV-06: تعيين حالة `partially_shipped` يدوياً بلا أثر فعلي
**Given:** order بحالة `stock_reserved`
**When:** `sdSetStatus(id, 'partially_shipped')`
**Then:** الحالة تتغيّر بصرياً بس — لا يوجد أي حساب لكمية مشحونة فعلياً، ولا ربط بفواتير جزئية متعددة

### TC-SLE-050 — SINV-03: حذف مستند له `linkedInvoiceId` (بلا `toDoc`)
**Given:** order له `linkedInvoiceId` مملوء
**When:** `delSalesDoc(orderId)`
**Then:** الحذف ينجح بلا مانع — الفاتورة تفضل موجودة في `DB.invoices` لكن `fromSalesDoc` بتاعها يشاور على مستند محذوف

### TC-SLE-051 — Regression: تحويل عادي (quotation→proforma) لا يتأثر بإصلاح KI-009 المقترح
**Given:** بعد تطبيق إصلاح KI-009 (ضبط `toDoc` في فرع الفاتورة)
**When:** `sdConvert` على quotation عادي (مش order)
**Then:** السلوك يبقى كما هو تماماً (`toDoc` يُضبَط زي ما كان) — الإصلاح لا يكسر باقي السلسلة

### TC-SLE-052 — Regression: إصلاح KI-009 لا يمنع إنشاء فاتورة يدوية منفصلة (غير مرتبطة بـ order)
**Given:** بعد تطبيق الإصلاح
**When:** مستخدم ينشئ فاتورة جديدة مباشرة من وحدة Invoices (بلا مرور بـ `sdConvert`)
**Then:** ينجح بلا قيد — الإصلاح خاص بمسار `sdConvert` فقط، لا يمس إنشاء الفواتير المستقلة

---

## ح. Regression Tests إضافية (تكامل بين الأقسام) — 4 حالات

### TC-SLE-053 — تسلسل كامل مع شرط CIF (تأمين + عدم تأجيل)
**Given:** quotation بشرط `CIF` يُحوَّل بالكامل حتى فاتورة
**When:** `shipDocSummary` و`isDeferredIncoterm` يُستدعيان معاً
**Then:** التأمين مطلوب (`req` تشمله)، لكن **لا تأجيل إيراد** (`CIF` ليست ضمن قائمة DAP/DPU/DDP/DAT)

### TC-SLE-054 — تسلسل كامل مع شرط DDP (تأجيل + بلا تأمين)
**Given:** نفس التسلسل بشرط `DDP`
**When:** نفس الفحصين
**Then:** **لا تأمين مطلوب**، لكن **تأجيل إيراد نشط** حتى "تم التسليم" — الشرطان مستقلان تماماً عن بعض، لا تداخل بينهما في الكود

### TC-SLE-055 — حذف مستند وسط بالسلسلة يكسر التتبع للمستندات اللاحقة
**Given:** سلسلة كاملة quotation→proforma→contract→order، تُحذَف الـ proforma في المنتصف
**When:** يُفتَح `openSalesDocDetail(contract.id)` ويُحسَب `back` (المسار للخلف عبر `fromDoc`)
**Then:** حلقة `while(cur&&cur.fromDoc){const p=sdDoc(cur.fromDoc);if(!p)break;...}` **تتوقف بصمت** عند أول مستند مفقود — لا رسالة خطأ، السلسلة المعروضة تبدو أقصر من الحقيقة بلا تنبيه

### TC-SLE-056 — Regression شامل: التقاء الوحدات الثلاث في مستند واحد
**Given:** proforma جاءت من CRM (`linkedOppId` موجود)، بسعر مكتوب يدوياً (لا علاقة بـ Pricing)، بشرط `CIF`
**When:** تُحوَّل بالكامل حتى فاتورة
**Then:** الفاتورة النهائية تحمل أثر CRM (`linkedOppId` إن انتقل)، سعراً غير مرتبط بمحرك التسعير (يوثّق فجوة Pricing↔Sales)، ومتطلبات شحن صحيحة حسب `CIF` — يفحص أن الثغرات الموثَّقة في الموديولات الثلاثة لا تتعارض مع بعضها بشكل إضافي غير متوقَّع

---

## ملخص التغطية

| المجموعة | عدد الحالات |
|---|---|
| أ. التحويلات الطبيعية | 10 |
| ب. حالات الرفض/الإنهاء | 8 |
| ج. الصلاحيات | 6 |
| د. الشحن | 8 |
| هـ. Incoterms | 6 |
| و. تأجيل الإيراد | 8 |
| ز. Known Issues | 6 |
| ح. Regression إضافية | 4 |
| **الإجمالي** | **56** |

---

## Cross-references
- `docs/03_Business_Logic/Sales_Export/Sales_Export.md`: كل القواعد والـ Invariants المرجعية
- `docs/03_Business_Logic/Sales_Export/Sales_Export_Implementation_Guide.md`: أكواد الخطأ، Transaction Boundaries
- `docs/02_Governance/Known_Issues.md`: KI-008, KI-009
