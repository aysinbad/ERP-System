# Sales / Export Business Logic

## Document Information
```
Document Name:  Sales / Export Business Logic
Version:        0.3.0 (SINV-07 مضافة من اكتشافات دليل التنفيذ)
Status:         Draft — In Progress
Classification: Source of Truth
Owner:          Product Owner
Approved-by:    —
Approved-date:  —
Last-Updated:   2026-07-18
```

> **المصدر:** مراجعة مباشرة لكود `reference/prototype/prototype_v2.html` — `SD_TYPES`, `sdConvert`, `sdNextId`, `sdDoc`, `DB.salesDocs`, `DB.invoices`. لا افتراض.
> **مبدأ Black Box (مثبَّت قبل البدء):** لا تُعاد مناقشة منطق CRM أو Pricing هنا. أي شيء يخص تلك الوحدتين يُذكَر فقط كـ**عقد استهلاك فعلي موثَّق بالكود**، لا كطموح.

---

## Module Dependencies

**Depends On (Business Modules):**

| الموديول | العقد **الفعلي الحالي** | هل هو الطموح المرغوب؟ |
|---|---|---|
| **CRM** | `crmOppToProforma` تستدعي مباشرة كود إنشاء بروفورما في هذه الوحدة (استدعاء دالة عبر حدود الموديولات، لا حدث نظيف) | ❌ لا — موثَّق كملاحظة معمارية في `CRM.md` Module Dependencies، يحتاج مراجعة لاحقاً (هل يُستبدَل بـ Domain Event؟) |
| **Pricing** | مستندات البيع (`salesDocs`) تُنشأ ببند `unitPrice` يُكتَب يدوياً وقت إنشاء المستند، **أو** يُنسَخ من `Product.price` (سعر قائمة ثابت) عند التحويل من فرصة CRM — **وليس من `pricingCalc()` أو أي "سعر معتمَد" رسمي** | ❌ لا — `Pricing.md` نفسها توثّق هذه الفجوة (Open Question #1)؛ RFC-PRC-003 (Price Snapshot) لسه مفتوح، فمفيش "سعر معتمد" رسمي يُستهلَك أصلاً |
| **Inventory & Production** | عند تحويل مستند لفاتورة (`sdConvert`)، الأصناف (`items[]`) تُنسَخ بالكمية والسعر بس — **لا يوجد فحص فعلي لتوفر المخزون** وقت التحويل في الكود المفحوص حتى الآن (يحتاج تأكيد إضافي عند الغطس في Inventory) | 🟡 يحتاج تحققاً في جلسة Inventory لاحقاً |

**Depends On (Shared Utilities):**
- لا يوجد اعتماد مباشر مؤكَّد حتى الآن على `fxRate()` داخل سلسلة `sdConvert` نفسها (الأصناف تُنسَخ بنفس عملة المستند الأصلي بلا تحويل صريح في هذه الدالة).

**Produces (يُستهلَك من موديولات أخرى):**
- **`DB.invoices`** (الفاتورة التجارية) — نقطة الالتقاء مع وحدة Accounting بالكامل (`buildJournalCore`, `calcInvoice`).
- **`linkedOppId`** على كل مستند — يُبقي أثراً عكسياً لـ CRM (لكن CRM لا "يستهلك" هذا الحقل حالياً بشكل نشط، مجرد مرجع).

---

## Purpose (مبدئي)
إدارة دورة حياة مستند البيع الكاملة من عرض السعر وحتى الفاتورة التجارية، عبر سلسلة مستندات متتابعة (عرض سعر → بروفورما → عقد → أمر تصدير → فاتورة)، مع حفظ أثر كل تحويل ومصدره.

## Scope (مبدئي)

**داخل النطاق:**
- سلسلة مستندات البيع الأربعة (`quotation`, `proforma`, `contract`, `order`) وتحويلاتها المتتابعة
- إنشاء الفاتورة التجارية (`DB.invoices`) كنقطة نهاية للسلسلة
- حالات كل نوع مستند ومساراتها

## Out of Scope (مبدئي — يُوسَّع في جلسة لاحقة)
- منطق حساب القيود المحاسبية من الفاتورة — وحدة Accounting
- تفاصيل تتبع الشحنة بعد إنشاء الفاتورة (BL, tracking) — تُغطَّى في جلسة Workflow القادمة، قد تستحق قسماً فرعياً
- استخراج منطق الـ CRM أو Pricing نفسه — **Black Box بحسب مبدأ هذا الملف**

---

## Actors (مبدئي)

| الفاعل | الوصف |
|---|---|
| **أي مستخدم بصلاحية `invoices:edit`** | الوحيد المسموح له بتحويل "أمر تصدير" لفاتورة فعلية (فحص صلاحية موجود فعلاً في `sdConvert`) |
| **أي مستخدم بصلاحية القسم** | يقدر ينشئ/يحوّل باقي أنواع المستندات (quotation→proforma→contract→order) بلا فحص صلاحية إضافي مكتشَف حتى الآن |

---

## Entities

### 1. Sales Document (`salesDocs`) — 4 أنواع في سلسلة خطية واحدة

| النوع | الاسم | الحالات الممكنة | التالي في السلسلة |
|---|---|---|---|
| `quotation` | عرض سعر | draft, sent, accepted, rejected, expired, cancelled | → `proforma` |
| `proforma` | فاتورة مبدئية (بروفورما) | draft, sent, confirmed, converted, cancelled | → `contract` |
| `contract` | عقد بيع | draft, pending_signature, active, fulfilled, cancelled | → `order` |
| `order` | أمر تصدير | draft, approved, stock_reserved, partially_shipped, fully_shipped, cancelled | → `invoice` (كيان مختلف تماماً، `DB.invoices`) |

**الحقول الحاكمة المشتركة:** `id` (بادئة حسب النوع: `QUO-`, `PRO-`, `CTR-`, `ORD-`)، `custCode`, `cur`, `incoterm`, `port`, `items[]` (`{prodCode, qty, unitPrice}`)، `linkedOppId` (مرجع لفرصة CRM إن وُجد)، `fromDoc`/`toDoc` (سلسلة الروابط بين المستندات)، `linkedInvoiceId` (يُملأ فقط عند وصول السلسلة لمرحلة `order→invoice`).

### 2. Commercial Invoice (`DB.invoices`) — كيان منفصل، ليس امتداداً لنفس بنية `salesDocs`
عند التحويل الأخير (`order → invoice`)، تُنشأ فاتورة بحقول مختلفة تماماً عن باقي السلسلة: `bl`, `shipDate`, `shipStatus`, `freight`, `discPct`, `netWeight`, `shipCosts[]`, `fromSalesDoc` (مرجع عكسي وحيد للمستند الأصلي). **لا يوجد `toDoc` بعد الفاتورة — هذه نهاية السلسلة.**

⚠️ **ملاحظة بنيوية:** الفاتورة **لا ترث** حقول `incoterm`/`port` مباشرة من مستند الـ `order` في كود `sdConvert` نفسه (يحتاج تأكيد إضافي هل تُنسَخ من مكان آخر أم تُفقَد).

---

## State Machine (المرحلة الأولى — التسلسل الأساسي، بدون Business Rules التفصيلية بعد)

```
quotation ──sdConvert──► proforma ──sdConvert──► contract ──sdConvert──► order ──sdConvert──► DB.invoices
   │                        │                        │                       │
   ▼(بديل)                  ▼(بديل)                  ▼(بديل)                 ▼(بديل)
rejected/expired/       cancelled                cancelled               cancelled
cancelled (نهائية)      (نهائية)                 (نهائية)                (نهائية)
```

⚠️ **حارس تحويل واحد فقط موجود فعلياً في الكود** (`sdConvert`): `if(doc.toDoc){...'تم التحويل مسبقاً'...return;}` — يمنع تحويل نفس المستند مرتين. **لا يوجد أي فحص آخر** على حالة المستند نفسها قبل التحويل (مثال: تحويل `quotation` بحالة `rejected` لـ `proforma` — هل ده ممنوع فعلاً؟ لم يُتحقَّق بعد بدقة، يحتاج فحصاً إضافياً في جلسة Business Rules القادمة).

⚠️ **اكتشاف مطابق تماماً لحالة `delLead` في CRM (KI-007):** `delSalesDoc` **بلا أي فحص صلاحية أو حالة** — حتى مستند له `toDoc` مملوء (يعني اتحوّل بالفعل لمرحلة تالية) يُحذَف بلا مانع، وده هيكسر سلسلة `fromDoc`/`toDoc` بصمت (المستند التالي هيفضل يشاور على `fromDoc` مش موجود). هذا نفس نمط الخطأ بالظبط، مش حالة جديدة — يستحق تسجيله في `Known_Issues.md` بنفس رقم النمط.

---

## Workflow (تدفق العملية الفعلي الكامل)

```
عرض سعر (quotation) ── sdConvert ──► بروفورما (proforma) ── sdConvert ──► عقد بيع (contract) ── sdConvert ──► أمر تصدير (order)
                                                                                                         │
                                                                                              sdConvert (يتطلب صلاحية invoices:edit)
                                                                                                         │
                                                                                                         ▼
                                                                                          فاتورة تجارية (DB.invoices) — كل أصناف الـ order تُنسَخ دفعة واحدة
                                                                                                         │
                                                                                                         ▼
                                                                              فحص مستندات الشحن المطلوبة (SHIP_DOCS) — 6 مستندات:
                                                                              فاتورة تجارية، قائمة تعبئة، شهادة منشأ، شهادة صحة نباتية، بوليصة شحن
                                                                              + التأمين (شرطي — انظر Business Rules #1)
                                                                                                         │
                                                                                                         ▼
                                                                              تتبع الشحنة (shipStatus): قيد التجهيز → في الطريق → تم التسليم
                                                                                                         │
                                                                                                         ▼
                                                                              ⚠️ لحظة "تم التسليم" حاسمة محاسبياً لبعض شروط التسليم (انظر Business Rules #2)
```

⚠️ **ملاحظة على "الشحن الجزئي" (`partially_shipped`):** الحالة موجودة في قائمة حالات `order` وقابلة للتعيين يدوياً (`sdSetStatus`)، **لكن لا يوجد أي آلية فعلية في `sdConvert` لتقسيم أمر تصدير واحد لعدة فواتير جزئية** — التحويل الحالي بينسخ **كل الأصناف دفعة واحدة** في فاتورة واحدة، ويضبط الحالة مباشرة على `fully_shipped`. الحالة `partially_shipped` تبدو **تسمية متاحة بلا تنفيذ فعلي خلفها** (Vestigial Status).

---

## Business Rules

### 1. مستند التأمين شرطي حسب شرط التسليم (Incoterm)
`shipDocRequired(inv, 'ins')` — مستند "تأمين" مطلوب **فقط** لو شرط التسليم `CIF` أو `CIP`. باقي الـ 5 مستندات (فاتورة تجارية، قائمة تعبئة، شهادة منشأ، شهادة صحة نباتية، بوليصة شحن) **مطلوبة دائماً بلا شرط**.

### 2. تأجيل إثبات الإيراد لشروط تسليم معيّنة — مبني على معيار محاسبي مصري صريح (EAS 48)
اكتشاف مهم: الكود نفسه فيه تعليق صريح **"معايير المحاسبة المصرية (EAS)"** ودالة `isDeferredIncoterm`:
> شرط تسليم (Incoterm) يؤجّل إثبات الإيراد حتى الوصول (تنتقل المخاطر والمنافع عند الوجهة) — EAS 48

**القاعدة:** لو شرط التسليم `DAP`, `DPU`, `DDP`, أو `DAT` (شروط "تسليم عند الوجهة")، **والشحنة لسه معندهاش `shipStatus==='تم التسليم'`** → الإيراد **لا يُعتبَر محقَّقاً بعد محاسبياً**، حتى لو الفاتورة نفسها صدرت. بمجرد وصول `shipStatus` لـ "تم التسليم"، الشرط ده يزول والإيراد يُعتبَر محقَّقاً.

⚠️ **هذا يخص Sales/Export بحكم أن `incoterm`/`shipStatus` بياناته الأساسية موجودة هنا، لكن التنفيذ الفعلي لتأجيل القيد المحاسبي نفسه من مسؤولية وحدة Accounting** — هذا الملف يوثّق **الشرط المُحفِّز (Trigger Condition)** فقط، لا آلية القيد.

### 3. تحويل "أمر تصدير → فاتورة" هو الوحيد المحمي بصلاحية صريحة
`sdConvert` بتفحص `can('invoices','edit')` **فقط** عند هذه الخطوة تحديداً (`order→invoice`). باقي التحويلات في السلسلة (quotation→proforma→contract→order) **بلا أي فحص صلاحية إضافي** غير صلاحية القسم العامة.

---

## Business Invariants

| # | القاعدة الثابتة / الفجوة | المصدر/الدليل |
|---|---|---|
| SINV-01 | مستند لا يُحوَّل مرتين عبر السلسلة الأساسية (quotation→proforma→contract→order) — الحارس `if(doc.toDoc)` يمنع ذلك | `sdConvert` |
| **SINV-02 (باگ مؤكَّد، مؤكَّد أيضاً على مستوى الواجهة)** | **خطوة `order → invoice` تحديداً غير محمية من التكرار — لا في الكود ولا في الواجهة.** فرع الفاتورة في `sdConvert` يضبط `doc.linkedInvoiceId` لكن **لا يضبط `doc.toDoc`**. شرط ظهور زر "إنشاء فاتورة" نفسه (`canConv = t.next && !doc.toDoc && !['cancelled','rejected','expired'].includes(doc.status)`) **بيفضل `true` بعد إنشاء أول فاتورة** — يعني **الزر نفسه يفضل ظاهر وقابل للنقر**، ومستخدم يقدر يضغطه تاني وينشئ فاتورة ثانية مكرّرة من نفس أمر التصدير، **من غير أي حاجة لتلاعب بالكود أو API مباشر** | `sdConvert` (فرع `t.next==='invoice'`) + شرط `canConv` في `viewSalesDocs`/`openSalesDocDetail` |
| **SINV-03 (فجوة، مطابقة لـ KI-007/KI-008)** | حذف أي مستند بيع (`delSalesDoc`) بلا فحص صلاحية أو حالة — حتى مستند له `toDoc` أو `linkedInvoiceId` مملوء يُحذَف بلا مانع | `delSalesDoc` — انظر `Known_Issues.md` KI-008 |
| SINV-04 | مستند تأمين الشحن مطلوب حصراً لشروط `CIF`/`CIP` — لا يُطلَب لأي شرط تسليم آخر | `shipDocRequired()` |
| SINV-05 | تأجيل الإيراد ينطبق فقط على 4 شروط تسليم محدَّدة (`DAP/DPU/DDP/DAT`) ويزول تلقائياً بمجرد `shipStatus==='تم التسليم'` | `isDeferredIncoterm()` |
| **SINV-06 (فجوة)** | حالة `partially_shipped` تسمية بلا آلية تنفيذ فعلية — لا يوجد تتبع لكمية مشحونة جزئياً عبر فواتير متعددة لنفس الـ `order` | لا توجد دالة `shippedQty` أو ما يعادلها |
| **SINV-07 (اكتشاف من دليل التنفيذ)** | **قوائم شروط التسليم (Incoterms) المتاحة للاختيار غير متسقة عبر 3 شاشات مختلفة — وشرط `DAT` غير متاح للاختيار في أي شاشة بالنظام كله**، رغم كونه أحد الشروط الأربعة في قاعدة تأجيل الإيراد (Business Rules #2) — الشرط الخاص به **كود ميت عملياً (Unreachable)** | تفصيل كامل في `Sales_Export_Implementation_Guide.md` قسم Validation Rules |

---

## Out of Scope (موسَّع)

| البند | أين يُغطَّى فعلياً |
|---|---|
| تنفيذ القيد المحاسبي الفعلي لتأجيل الإيراد (SINV-05/Business Rules #2) | وحدة Accounting — هنا نوثّق الشرط المُحفِّز فقط |
| منطق CRM (تأهيل العملاء المحتملين، الفرص) | Black Box — `docs/03_Business_Logic/CRM/` |
| منطق التسعير وحساب السعر المقترح | Black Box — `docs/03_Business_Logic/Pricing/` |
| تفاصيل حساب تكاليف الشحن الموزَّعة على الفاتورة (`shipCosts[]`) بالتفصيل الكامل | يحتاج جلسة فرعية مخصَّصة إن استلزم — مذكور هنا كحقل فقط |
| منطق المخزون وتوفر الكمية وقت تأكيد أمر التصدير | وحدة Inventory & Production (يحتاج تحققاً مباشراً هناك) |

---

## Open Questions
1. ✅ **محسوم ومسجَّل فوراً كـ `KI-009`** (بدل الانتظار) — خطورته (تكرار مستند مالي فعلي) استدعت التوثيق الفوري بدل تأجيله لقرار لاحق.
2. هل يُراد تفعيل "الشحن الجزئي" (SINV-06) كميزة حقيقية مستقبلاً (تقسيم أمر تصدير لعدة فواتير)، أم تُحذَف الحالة من قائمة الحالات المتاحة لتفادي اللبس؟

---

## Cross-references
- `docs/02_Governance/Known_Issues.md`: KI-008 (حذف مستند بيع بلا صلاحية)، **KI-009 (فاتورة مكرّرة من نفس أمر التصدير — واجهة مؤكَّدة)**
- `docs/03_Business_Logic/CRM/CRM.md`: Module Dependencies (نفس ملاحظة الاستدعاء المباشر)
- `docs/03_Business_Logic/Pricing/Pricing.md`: Module Dependencies (فجوة "لا سعر معتمد يُستهلَك")
- `docs/03_Business_Logic/Sales_Export/Sales_Export_RFC.md`: RFC-SLE-001

