# Sales / Export Implementation Guide

## Document Information
```
Document Name:  Sales / Export Implementation Guide
Version:        1.0.0
Status:         Draft — In Review
Classification: Source of Truth
Owner:          Solution Architecture Team
Last-Updated:   2026-07-18
```

> **العلاقة بـ `Sales_Export.md`:** نفس مبدأ `CRM/CRM_Implementation_Guide.md` و`Pricing/Pricing_Implementation_Guide.md` — هذا الملف "إزاي يتربط بالنظام تقنياً"، لا "ماذا يجب أن يحدث". كل بند هنا مُستخرَج من الكود الفعلي أو مبني مباشرة على قرار موثَّق في `Sales_Export.md` — لا افتراض جديد بلا وسم صريح.

---

## State Transition Matrix

### داخل كل نوع مستند (الحالة الفرعية) — حرة بالكامل، بلا أي قيد
`sdSetStatus(id, st)` بتضبط `doc.status` مباشرة بلا أي فحص تسلسل — **نفس النمط المعماري المتكرر للمرة الثالثة** (بعد Lead/Opportunity في CRM). القائمة المنسدلة في `salesDocForm` بتعرض كل حالات النوع مفتوحة للاختيار الحر:

| النوع | الحالات (بالترتيب المعروض، لا يعني تسلسلاً إلزامياً) |
|---|---|
| `quotation` | draft, sent, accepted, rejected, expired, cancelled |
| `proforma` | draft, sent, confirmed, converted, cancelled |
| `contract` | draft, pending_signature, active, fulfilled, cancelled |
| `order` | draft, approved, stock_reserved, partially_shipped, fully_shipped, cancelled |

### بين الأنواع (التقدّم في السلسلة) — عبر `sdConvert` فقط

| من → إلى | الحارس الموجود | ملاحظة |
|---|---|---|
| quotation → proforma | `!doc.toDoc` | يعمل بشكل صحيح |
| proforma → contract | `!doc.toDoc` | يعمل بشكل صحيح |
| contract → order | `!doc.toDoc` | يعمل بشكل صحيح |
| **order → invoice** | **لا يوجد حارس فعّال** (`toDoc` لا يُضبَط في هذا الفرع) | **SINV-02 / KI-009 — أخطر ثغرة موثَّقة في هذا الموديول** |

---

## Permissions Matrix

| العملية | الصلاحية المطلوبة فعلياً في الكود | ملاحظة |
|---|---|---|
| إنشاء/تعديل quotation, proforma, contract, order | صلاحية القسم العامة فقط | لا تمييز أدوار إضافي |
| **تحويل order → invoice** | **`can('invoices','edit')`** — الوحيدة المحمية صراحة | `sdConvert` |
| حذف أي مستند (`delSalesDoc`) | **لا يوجد فحص صلاحية إطلاقاً** | KI-008 |
| تغيير الحالة الفرعية (`sdSetStatus`) | لا يوجد فحص صلاحية إطلاقاً | — |

---

## Validation Rules

### موجودة فعلاً عند الحفظ (`editSalesDoc`)
1. **العميل إلزامي** — `if(!cust0){toast('اختر العميل','err');return false;}`
2. **بند واحد على الأقل** — `if(!items.length){toast('أضف بنداً واحداً على الأقل','err');return false;}`
3. **فلترة صامتة للكمية غير الموجبة** — `items = SD_LINES.filter(l=>l.prodCode && (l.qty||0)>0)` — بند بكمية صفر أو سالبة **لا يُرفَض بخطأ، بل يُحذَف بصمت من القائمة قبل الحفظ**. هذا سلوك مختلف عن محرك التسعير (حيث الكمية السالبة تُقبَل وتُحسَب — انظر `Pricing.md` PINV-08)؛ هنا الفلترة الصامتة تمنع الكمية السالبة تحديداً، لكن **بلا أي رسالة توضّح للمستخدم إن بنداً اتحذف**.

### 🔴 اكتشاف جديد من هذه الجلسة — تضارب قوائم شروط التسليم (Incoterms) عبر 3 شاشات مختلفة
مقارنة مباشرة بين القوائم الفعلية المتاحة للاختيار:

| الشاشة | القائمة المتاحة فعلياً |
|---|---|
| `salesDocForm` (مستند البيع) | `C&F, FOB, CIF, DDP, EXW, FCA` |
| نموذج الفاتورة (`f_terms`) | `C&F, FOB, CIF, DDP, EXW, FCA, DAP, CPT` |
| إعداد شروط العميل الافتراضية | `EXW, FCA, FOB, CFR, CIF, CPT, CIP, DAP, DPU, DDP` |
| **قاعدة تأجيل الإيراد (`isDeferredIncoterm`)** | تتحقق من: **`DAP, DPU, DDP, DAT`** |

**الأثر المؤكَّد:** `DAT` **غير متاح للاختيار في أي شاشة بالنظام كله** — يعني الشرط الخاص بيه في قاعدة تأجيل الإيراد (`Sales_Export.md` Business Rules #2) **كود ميت عملياً (Unreachable)**. كمان `DPU` غير متاحة في نموذج الفاتورة نفسه (الشاشة الأقرب لتفعيل القاعدة)، رغم إنها متاحة في إعداد العميل الافتراضي — تضارب مباشر بين مصدر القيمة الافتراضية والشاشة اللي بتُستخدَم فيها فعلياً.

---

## Audit Rules

يُسجَّل حالياً (`audit()` داخل `editSalesDoc`):
- إنشاء/تعديل أي مستند بيع (النوع، المعرِّف، اسم العميل).

⚠️ **لا يُسجَّل حالياً:**
- حذف أي مستند (`delSalesDoc`) — يتوافق مع KI-008.
- تغيير الحالة الفرعية عبر `sdSetStatus` — لا يمر بـ `audit()` إطلاقاً، رغم إنه بيغيّر حالة مستند رسمية (زي `fulfilled` أو `cancelled`).
- أي عملية تحويل عبر `sdConvert` نفسها — إنشاء البروفورما/العقد/الأمر/الفاتورة **لا يُسجَّل كحدث تدقيق منفصل**، فقط لو تبعه `editSalesDoc` لاحقاً.

---

## Domain Events

| الحدث | يُطلَق عند | الحمولة الأساسية |
|---|---|---|
| `sales.document.created` | إنشاء مستند بيع جديد (أي نوع) | `docId, type, custCode, itemsCount` |
| `sales.document.status_changed` | أي تغيير حالة فرعية (`sdSetStatus`) | `docId, fromStatus, toStatus` — **غير مُدقَّق حالياً، يحتاج ربط بـ Audit Rules أعلاه فور التنفيذ** |
| `sales.document.converted` | نجاح `sdConvert` للسلسلة الأساسية (quotation→proforma→contract→order) | `docId, fromType, toType, newDocId` |
| `sales.invoice.created_from_order` | تحويل order→invoice | `orderId, invoiceId` — **يجب أن يكون الحدث الوحيد المسؤول عن غلق دورة الـ order، مرتبط مباشرة بإصلاح KI-009 (انظر Transaction Boundaries)** |
| `sales.shipment.delivered` *(مقترح، غير موجود كحدث فعلي — الكود يغيّر `shipStatus` مباشرة بلا حدث)* | `shipStatus` يصل لـ "تم التسليم" | `invoiceId` — **يُحفِّز مباشرة إعادة تقييم `isDeferredIncoterm` في Accounting (تكامل حرج، انظر أدناه)** |

---

## Notifications
- **الحالة الحالية:** لا توجد تنبيهات مخصَّصة لدورة مستندات البيع نفسها (لا "بروفورما بانتظار الرد"، لا "عقد قارب على الانتهاء").
- **الاستثناء الوحيد المكتشَف:** `topAlerts()` (لوحة القيادة العامة) بتفحص **مستندات الشحن الناقصة للشحنات غير المُسلَّمة** (`shipDocSummary`) — تنبيه عام على مستوى النظام، مش خاص بمستند بيع بعينه.
- فجوة "In-app فقط، لا بريد/واتساب" — نفس النمط العام الموثَّق في CRM وPricing.

---

## Standard Error Codes

| الكود | الوصف | الموقف المكافئ في Prototype | HTTP المقترح |
|---|---|---|---|
| `SLE_CUSTOMER_REQUIRED` | حفظ مستند بلا عميل | "اختر العميل" | 422 |
| `SLE_LINE_ITEM_REQUIRED` | حفظ مستند بلا أي بند صالح | "أضف بنداً واحداً على الأقل" | 422 |
| `SLE_ALREADY_CONVERTED` | محاولة تحويل مستند له `toDoc` بالفعل | "تم التحويل مسبقاً" | 409 |
| **`SLE_ORDER_ALREADY_INVOICED`** *(مقترح، يسد KI-009 مباشرة)* | محاولة تحويل `order` له `linkedInvoiceId` بالفعل | **لا يوجد مكافئ حالياً — هذا أصل المشكلة** | 409 |
| `SLE_DOC_NOT_FOUND` | مستند غير موجود | فحوصات `if(!doc)return` المتكررة | 404 |

---

## API Contracts (توضيحي، يتبع اصطلاح `docs/01_Standards/API_Standards.md`)

| العملية | Endpoint مقترح |
|---|---|
| إنشاء مستند بيع | `POST /api/v1/sales/documents` |
| تحويل مستند للخطوة التالية | `POST /api/v1/sales/documents/{id}/convert` |
| تغيير الحالة الفرعية | `PATCH /api/v1/sales/documents/{id}/status` |
| تحويل order→invoice تحديداً | نفس `convert` أعلاه — **لا endpoint منفصل**، لتفادي تكرار منطق الحماية من KI-009 في مكانين |

---

## Transaction Boundaries

> هذا القسم الحل التقني المباشر لـ **KI-009** — أهم بند في هذا الملف.

- **`sdConvert` بالكامل (أي فرع) يجب أن يكون Transaction ذرّية واحدة**: (1) إنشاء المستند/الفاتورة الجديدة، (2) ضبط `toDoc` (أو `linkedInvoiceId` + قفل صريح) على المستند المصدر — **الاثنان معاً أو ولا واحد**. الكود الحالي (client-side, لا معاملات حقيقية) بينفذهم كخطوتين منفصلتين، وده جزء من سبب المشكلة (نافذة زمنية بين الخطوتين، حتى لو الحارس نفسه كان صحيحاً).
- **فرع order→invoice تحديداً** يحتاج **قيد فريد على مستوى قاعدة البيانات** (مثال: `UNIQUE(order_id) WHERE linkedInvoiceId IS NOT NULL`، أو ببساطة معاملة تتحقق من `linkedInvoiceId IS NULL` وتضبط `toDoc` بنفس الطريقة الموحَّدة كباقي الخطوات) — **لا يكفي فحص منطقي في كود التطبيق فقط، لأن هذا بالضبط ما فشل في الـ Prototype.**
- **تحديث `sdSetStatus`** لا يحتاج معاملة معقدة (تحديث حقل واحد)، لكن يحتاج يبقى جزء من نفس الـ Audit Log Transaction (تسجيل + تحديث معاً) لسد الفجوة الموثَّقة أعلاه.

---

## Integration مع الموديولات الأخرى (تفصيل تقني، مبني على Module Dependencies في `Sales_Export.md`)

### CRM
- **الاتجاه الحالي:** CRM يستدعي كود Sales مباشرة (`crmOppToProforma`). **التوصية التقنية (مرتبطة بـ RFC-SLE-001):** عكس الاتجاه — Sales يشترك في حدث `crm.opportunity.won` (موثَّق فعلاً في `CRM_Implementation_Guide.md`) وينشئ البروفورما بنفسه، بدل ما CRM "يعرف" عن وجود دالة إنشاء بروفورما داخل Sales.

### Pricing
- **الاتجاه الحالي:** لا تكامل فعلي — `unitPrice` يُكتَب يدوياً أو يُنسَخ من `Product.price` الثابت. **لا شيء يُبنى هنا الآن** — مرتبط بـ RFC-PRC-003 (Price Snapshot) في Pricing نفسها، خارج نطاق هذا الملف حتى يُحسَم هناك أولاً (مبدأ Black Box).

### Inventory
- **الاتجاه الحالي:** غير مؤكَّد بالكامل بعد (وُسم كسؤال في `Sales_Export.md` Module Dependencies) — لا يوجد فحص توفر مخزون واضح وقت `sdConvert`. **يحتاج تحققاً مباشراً عند استخراج جلسات Inventory.**

### Accounting
- **الاتجاه الحالي:** `DB.invoices` هي نقطة التسليم الرسمية — `buildJournalCore`/`calcInvoice` بتستهلكها. **نقطة تكامل حرجة موثَّقة في `Sales_Export.md` Business Rules #2:** حقل `shipStatus` على الفاتورة هو المُحفِّز لقاعدة `isDeferredIncoterm` في القيد المحاسبي — أي تغيير على `shipStatus` (حتى لو من شاشة Sales) له أثر محاسبي مباشر في Accounting. **يُوصى بحدث `sales.shipment.delivered` صريح** (مقترح أعلاه في Domain Events) بدل الاعتماد على Accounting إنها "تلاحظ" التغيير بنفسها كل مرة تُحسَب فيها القوائم.

---

## Cross-references
- `docs/03_Business_Logic/Sales_Export/Sales_Export.md`: منطق الأعمال الأساسي (المرجع الأعلى لأي تعارض)
- `docs/02_Governance/Known_Issues.md`: KI-008, KI-009
- `docs/03_Business_Logic/Sales_Export/Sales_Export_RFC.md`: RFC-SLE-001
- `docs/03_Business_Logic/CRM/CRM_Implementation_Guide.md`: حدث `crm.opportunity.won` (نقطة التكامل المقترحة)
- `docs/01_Standards/API_Standards.md`: اصطلاح أكواد الخطأ ومسارات الـ API
