# Accounting — Session 6.1
## Remaining Evidence Closure

## Document Information

```
Document Name:  Accounting Session 6.1 — Remaining Evidence Findings
Type:           Investigation report (no documentation was modified)
Version:        2.0.0 (منقَّحة — تسعة تصحيحات مطبَّقة قبل الاعتماد)
Status:         Draft — for review
Date:           2026-07-24
Session:        6.1 of 6
Scope:          Targets 1–6 only. No unrelated function or module investigated.
Source of truth: reference/prototype/prototype_v2.html
Inputs:         Accounting_Session4_6_Findings.md · Accounting.md v0.5.0 · Accounting_RFC.md v1.0.0 · Known_Issues.md v1.1.0
                · ADR-020 · Accounting_Session5_Invariants_RFC_Decisions_Reviewed.md v1.1.0
                · Glossary.md §10 (Chart of Accounts)
```

> **لم يُعدَّل أي ملف.** لا `Accounting.md` · لا `Accounting_RFC.md` · لا `Known_Issues.md` · لا ADR · لا `AI_CONTEXT.md` · لا `Module_Status.md` · لا كود تطبيقي · لا الـPrototype. No permanent repository file was modified. The outputs are this revised report and Correction_Log_S6_1.md.

> 🔴 **إشعار إعلاء عام (2026-07-25) — يعلو كل ما دونه في هذا التقرير:** كل تصنيف لـ **JS-C1** أو **JS-C2** كـ`Conditional` أو `Unverified candidate — likely future` في أي قسم أدناه (بما فيه الملخص التنفيذي · §6 · §6.4 · §9 · §10) **مُعلىً (superseded)** بأدلة التشغيل الحيّة. **التصنيف المعتمَد الآن: JS-C1 = `Confirmed posting source — Opening Journal` · JS-C2 = `Confirmed posting source — Fixed Asset Acquisition`.** الكتالوج المعتمَد: **62 confirmed · 0 Conditional · 1 Future · 63 total**. التفصيل في **§12.4**. النصوص التاريخية أدناه تُحفَظ **كتاريخ تحقيق فقط، لا كتصنيف حالي**.

---

# 1. Executive Summary

| # | الهدف | النتيجة | الأثر |
|---|---|---|---|
| 1 | OQ-4.G — `supplierPayments` | ✅ **`Known Issue`** | مسار الترحيل مؤكَّد · ثلاث فجوات رقابية |
| 2 | OQ-4.J — إشعارات خصم الموردين | ✅ **`Known Issue`** | الخريطة مكتملة — الحاجب مرفوع |
| 3 | الحساب 2017 | ✅ **`No dedicated settlement or consumption path found after scoped search`** | ACC-E ⇒ **`Still Blocked — evidence gap resolved, design decision pending`** |
| 4 | JS-C1 · JS-C2 | ✅ **`Unverified candidate — likely future requirement`** | إدراجهما في العدّ **مؤقّت** |
| 4b | JS-C3 | ✅ **`Specialized use case of JS-19`** — مؤكَّد بفحص الكاتب | `genRecurring()` مفحوصة مباشرةً |
| 5 | JS-19 · JS-39 · JS-41 | ✅ خرائط دقيقة | **تصحيح جوهري على JS-19** |
| 5b | **JS-38** | ✅ **مُحلّ — `DB.revaluations` عُثر عليه** | **الحاجب على Posting Matrix مرفوع** |
| 6 | ملاحظات جلسة 4.6 الست | ✅ **مسرودة ومصنَّفة** | الملف سُلِّم |

## ما تغيّر في هذه النسخة

الفحوصات الثلاثة المطلوبة في البند 8 **حوّلت نتيجتين سلبيتين إلى إيجابيتين**:

- **`DB.revaluations` عُثر عليه** ⇒ JS-38 مُحلّ بالكامل ⇒ **Posting Matrix لم يعد محجوباً**.
- **`genRecurring()` فُحصت مباشرةً** ⇒ JS-C3 ارتفع من `Inferred` إلى `Confirmed by direct code evidence`.

> **الحالة المتوقَّعة في التعليمات كانت `Posting Matrix: Blocked by JS-38`. الدليل الجديد يغيّرها** — والتعليمات تسمح بذلك صراحةً (*«unless new evidence changes it»*).

## أهم أربعة اكتشافات

**١. الحساب 2017 لا يُستنفَد.** لم يُعثَر على أي مصدر قيد يجعله مديناً. القيد اليدوي العام يبقى **قادراً تقنياً** على ذلك — **وهذا ليس مسار تسوية مُصمَّماً**.

**٢. نظاما إعادة تقييم متوازيان يعملان معاً — خطر ازدواج.**
`DB.revaluations` (JS-38) تُرحِّل على **حسابات الأطراف** 1100/2010/1020 مقابل 4030/5500 **بعكس تلقائي**؛ و`DB.fxRevaluations` (JS-34→37) تُرحِّل على **الحساب الوسيط 1265** وفق ADR-003. **كلاهما يقرأ نفس الأرصدة الأجنبية المفتوحة.**

**٣. أربعة مصادر مؤكَّدة غير مُكتلَجة** في `DB.treasuryTx` — ثلاثة منها تمسّ حقوق الملكية والنتيجة.

**٤. `JS-19` يتبع `allocType` لا `cat`** — تصحيح جوهري على جلسة 5 §2.10.

## العدد المقترح

> ## `58 ⇒ 61` — **`Proposed pending source-boundary validation`**

**+3 صافياً:** JS-C3 يخرج من Conditional (**−1**) · أربعة مصادر خزينة تُضاف (**+4**).

⚠️ **الرقم 61 غير نهائي.** يتوقف على قاعدة حدود المصدر (§10.4) وعلى حسم JS-C1/JS-C2. **لا يُعتمَد الآن.**

---

# 2. Scope and Method

## 2.1 المنهج

بحث نصّي موجَّه داخل `reference/prototype/prototype_v2.html` عبر فهرس المستودع، بحدود الأهداف الستة فقط. **لم يُفتَح أي استقصاء خارج النطاق.**

## 2.2 قيد منهجي — يجب إعلانه

الوصول للـPrototype تمّ عبر **بحث نصّي مُقطَّع (indexed snippets)**، لا بقراءة الملف كاملاً من أوله لآخره.

**الأثر:**

- **الإثبات الإيجابي قوي:** حين يُرى الكود، الاستنتاج قطعي.
- **الإثبات السلبي أضعف:** «لم يُعثَر عليه» تعني **`Not found after scoped search`**، ولا تعني «غير موجود».

⇒ لذلك يُستخدَم `Not found after scoped search` حرفياً حيثما لم يُرَ الكود، **ولا تُكتب «not implemented» في أي موضع** — التزاماً بمعيار الأدلة المفروض على هذه الجلسة.

## 2.3 الفحوصات الثلاثة الموجَّهة (البند 8)

| # | الهدف | النتيجة |
|---|---|---|
| 1 | كل ظهور لـ`DB.revaluations` | ✅ **عُثر عليه** — كيان + كاتب + حاذف + بلوك ترحيل + عكس تلقائي (§7.2) |
| 2 | الكاتب/المولّد الكامل لـ`DB.recurring` | ✅ **`genRecurring(rc, monthStr)` مفحوصة بالنص الكامل** (§6.3) |
| 3 | بلوك `DB.treasuryTx` و`TX_KINDS` | ✅ **مفحوصان بالكامل** — خمسة `kind` وخمسة فروع ترحيل (§7.6) |

**لم يُفحَص الملف كاملاً استقصائياً.** لذلك أي نتيجة سلبية متبقّية تبقى `Not found after scoped search` حصراً.

## 2.4 مصادر مُسلَّمة ومقروءة

| الملف | الحالة |
|---|---|
| `Accounting_Session4_6_Findings.md` | ✅ **مُسلَّم ومقروء** — الهدف 6 مُنجَز (§8) |
| `Accounting_Session4_Procurement_Inventory_Production_Reviewed.md` | 🟡 غير مُسلَّم · غير حاجب — الأدلة أُعيد اشتقاقها من الكود |
| `Accounting_Session4_5_Findings.md` | 🟡 غير مُسلَّم · غير حاجب — نفس السبب |
| `DEVELOPMENT_RULES.md` · `AGENTS.md` · `AI_CONTEXT.md` | ✅ مقروءة (الأول من نسخة جلسة 5؛ الأخيران من فهرس المستودع) |

---

# 3. Target 1 — Supplier Payments (OQ-4.G)

## 3.1 الكيان

**`DB.supplierPayments`** — `Confirmed by direct code evidence`.

```js
supplierPayments: [
  {id:'SPV-001', date:'2026-05-20', supplierCode:'SUP-004', expId:'EXP-001', amount:5000, method:'Cash'},
]
```

**الحقول المُشاهَدة:** `id` · `date` · `supplierCode` · `amount` · `method` · `expId` · `purchaseId` · `fxRate` · `cur` · `treasury`.

## 3.2 مسار الترحيل — JS-16

`Confirmed by direct code evidence` — بلوك `(DB.supplierPayments||[]).forEach(sp=>{…})` داخل `buildJournalCore()`.

```js
const payCur = sp.cur || s.cur || 'EGP';
const payRate = sp.fxRate || fxRate(payCur);          // سعر يوم السداد
let invRate = payRate;
if (sp.purchaseId) { const pu=…; if(pu) invRate = fxOf(pu, pu.cur||payCur); }
else if (sp.expId) { const ex=…; if(ex) invRate = ex.fxRate || fxRate(ex.cur||payCur); }
const cashEGP    = (sp.amount||0) * payRate;
const payableEGP = (sp.amount||0) * invRate;
const fxDiff     = payableEGP - cashEGP;
const lines = [{acc:'2010 الموردون', dr:payableEGP, cr:0},
               {acc:cashAcc(sp.treasury), dr:0, cr:cashEGP}];
if (Math.abs(fxDiff) > 0.01) {
  if (fxDiff > 0) lines.push({acc:'4030 أرباح فروق عملة', dr:0, cr:fxDiff});
  else            lines.push({acc:'5500 خسائر فروق عملة', dr:-fxDiff, cr:0});
}
add(sp.date, 'سداد لمورد '+sp.id+' — '+(s.name||''), lines,
    sp.purchaseId || sp.expId || sp.id, {party:s.name||'', partyType:'supplier'});
```

## 3.3 جدول التوثيق المطلوب

| البند | النتيجة | الثقة |
|---|---|---|
| **اسم الدالة (الترحيل)** | بلوك داخل `buildJournalCore()` — بلا دالة مستقلة | `Confirmed by direct code evidence` |
| **المجموعة المصدر** | `DB.supplierPayments` | `Confirmed by direct code evidence` |
| **شرط الإطلاق** | **لا شرط** — كل سجل يُرحَّل. لا فحص حالة ولا `approved` ولا حد أدنى للمبلغ | `Confirmed by direct code evidence` |
| **تاريخ الترحيل** | `sp.date` — حقل مباشر بلا fallback إلى `DB.reportDate` | `Confirmed by direct code evidence` |
| **الحسابات المدينة** | `2010 الموردون` بقيمة `payableEGP` · و`5500` عند الخسارة | `Confirmed by direct code evidence` |
| **الحسابات الدائنة** | `cashAcc(sp.treasury)` بقيمة `cashEGP` · و`4030` عند الربح | `Confirmed by direct code evidence` |
| **مصدر سعر الصرف** | **سعران:** `sp.fxRate ‖ fxRate(payCur)` للنقدية · و`fxOf(pu)` أو `ex.fxRate` للذمة | `Confirmed by direct code evidence` |
| **مُجمَّد أم مُعاد حسابه؟** | 🟡 **مُختلَط.** `sp.fxRate` مُجمَّد على السند ✅ — لكن عند غيابه يسقط إلى **`fxRate(payCur)` الحيّ**. و`invRate` يُشتقّ **حيّاً من الفاتورة وقت بناء الدفتر** | `Confirmed by direct code evidence` |
| **هل الربط بالفاتورة إلزامي؟** | ❌ **لا.** `purchaseId` و`expId` كلاهما اختياري. عند غيابهما ⇒ `invRate = payRate` ⇒ **`fxDiff = 0`** ⇒ **فرق العملة الحقيقي يختفي بلا اعتراف** | `Confirmed by direct code evidence` |
| **هل يُمنع السداد الزائد؟** | **`Not found after scoped search`** — لا حارس مُشاهَد يقارن المبلغ برصيد الفاتورة | `Not found after scoped search` |
| **هل يُمنع التطبيق المزدوج؟** | **`Not found after scoped search`** — لا `converted` ولا `applied[]` على السند؛ ولا شيء يمنع سندين على نفس `purchaseId` | `Not found after scoped search` |
| **قفل الفترة** | 🔴 **غير مُنفَّذ على هذا المسار.** لا `isLocked()` مُشاهَد في نموذج سند السداد — بخلاف `custodyTxForm` و`editCreditNote` و`accrueCommission` التي كلها تفحصه | `Not found after scoped search` |
| **هل الحذف يمحو الأثر التاريخي؟** | ✅ **نعم.** الدفتر مُشتقّ — حذف السجل يمحو قيده من كل الفترات بلا عكس | `Confirmed by multiple code paths` — نمط KI-008 |
| **التدقيق** | `Not found after scoped search` — لم يُرَ `audit()` على مسار سند السداد (بخلاف `delCreditNote` و`accrueCommission`) | `Not found after scoped search` |
| **الصلاحيات** | قسم `supplierPayments` معرَّف في كل الأدوار: admin/accountant `edit` · sales/warehouse `none` · viewer `view` | `Confirmed by direct code evidence` |

## 3.4 الفجوات الثلاث الجديدة

| # | الفجوة | الأثر المحاسبي |
|---|---|---|
| **G-1** | **سداد غير مربوط ⇒ لا فرق عملة.** `invRate = payRate` عند غياب الربط | ذمة المورد بالعملة الأجنبية لا تصفو؛ فرق حقيقي غير معترَف به. **نفس نمط KI-019 (رد هامش الاعتماد) و KI-010 (مطالبة العميل)** |
| **G-2** | **`invRate` يُشتقّ حيّاً من الفاتورة وقت بناء الدفتر** | تعديل عملة/سعر الفاتورة لاحقاً **يغيّر قيمة قيد سداد تاريخي**. **امتداد مباشر لـKI-022** |
| **G-3** | **لا قفل فترة ولا حارس ازدواج ولا سقف مبلغ** | سداد في فترة مقفلة · سدادان على فاتورة واحدة · 2010 قد يصير مديناً |

## 3.5 إجابة OQ-4.G

> ## `Known Issue`

**التبرير:** الاستقصاء أجاب على كل بنود السؤال بدليل مباشر. المتبقّي ليس قراراً معمارياً ولا سياسة محاسبية — بل **ثلاث فجوات رقابية تقنية** تندرج تحت ثوابت قائمة بالفعل (`AINV-11` · `AINV-03` · `AINV-26`) وADRs مقرَّرة بالفعل (`ACC-B` · `ACC-D`). **لا ADR جديد مطلوب.**

**KIs مقترحة (لا تُنشأ في هذه المرحلة):** `KI-025` (G-1 + G-2) · `KI-026` (G-3).

---

# 4. Target 2 — Supplier Credit Notes (OQ-4.J)

## 4.1 الكيان والكاتب

| البند | القيمة | الثقة |
|---|---|---|
| **الكيان** | `DB.creditNotes` — **للموردين حصراً** | `Confirmed by direct code evidence` |
| **الكاتب** | `window.editCreditNote(id)` | `Confirmed by direct code evidence` |
| **مولّد المعرّف** | `cnNextId()` ⇒ `SCN-0001` | `Confirmed by direct code evidence` |
| **الحاذف** | `window.delCreditNote(id)` — حذف نهائي عبر `filter` | `Confirmed by direct code evidence` |
| **العارض** | `window.viewCreditNotes()` | `Confirmed by direct code evidence` |
| **المعاينة الحيّة** | `window.cnPrev()` — تُظهر القيد قبل الحفظ | `Confirmed by direct code evidence` |

**السجل المحفوظ:**
```js
{id, date, supplierCode, purchaseId, reason, amount, cur,
 fxRate: fxRate($('#cn_cur').value||'EGP'),   // ← مُجمَّد وقت الحفظ
 vatPct, ref, note}
```

## 4.2 قاعدة `cnPostingAccount()` — بالنص

```js
const CN_REASONS = [
  {k:'price',    ar:'فرق سعر / خصم تجاري',   acc:'1200 المخزون'},
  {k:'quality',  ar:'خصم جودة',               acc:'1260 هدر ومخالفات جودة الموردين'},
  {k:'late',     ar:'غرامة تأخير',            acc:'4020 إيرادات أخرى'},
  {k:'shortage', ar:'عجز في الكمية المستلمة', acc:'1200 المخزون'},
  {k:'other',    ar:'أخرى',                   acc:'4020 إيرادات أخرى'},
];
window.cnAccount = k => (CN_REASONS.find(x=>x.k===k) || {}).acc || '4020 إيرادات أخرى';

function cnTouchesInventory(cn){
  const a = cnAccount(cn.reason);
  if (String(a).indexOf('1200') !== 0) return false;
  return (DB.purchases||[]).some(p => p.id === cn.purchaseId);   // لازم مربوط بفاتورة
}
function cnPostingAccount(cn){
  if (cnTouchesInventory(cn)) return '1200 المخزون';
  const a = cnAccount(cn.reason);
  return (String(a).indexOf('1200') === 0) ? '4020 إيرادات أخرى' : a;
}
```

> **القاعدة بكلمات:** الحساب يُختار من `reason`. **لكن** إذا كان الحساب المختار `1200` **ولم يكن الإشعار مربوطاً بفاتورة شراء موجودة**، يُحوَّل قسراً إلى **`4020 إيرادات أخرى`** — لأن الخصم غير المنسوب لصنف يُحدث فرقاً صامتاً بين الأستاذ والكشف. **نفس القاعدة تُستخدَم في اليومية وفي `cnCostReductionEGP()` معاً، فيستحيل افتراقهما.**

## 4.3 قيد الترحيل

```js
const rate = (cn.fxRate || fxRate(cn.cur||'EGP')) || 1;
const base = amt*rate, vat = base*(cn.vatPct||0), tot = base+vat;
const acc  = cnPostingAccount(cn);
const lines = [{acc:'2010 الموردون', dr:tot, cr:0},
               {acc:acc,             dr:0,   cr:base}];
if (vat > 0.005) lines.push({acc:'1130 …مدخلات…', dr:0, cr:vat});
```
**الشرط:** `amt > 0.005` · **التاريخ:** `cn.date` (بلا fallback).

## 4.4 خريطة الترحيل الكاملة — مطلوبة صراحةً

| Credit-note category/type | Debit | Credit | VAT treatment | Inventory impact | Evidence |
|---|---|---|---|---|---|
| **`price` — فرق سعر/خصم تجاري** *(مربوط بفاتورة)* | 2010 بـ`tot` | **1200 المخزون** بـ`base` | 1130 دائن بـ`vat` (عكس مدخلات) | ✅ **نعم** — يخفّض المتوسط المرجّح عبر `cnCostReductionEGP()` | `CN_REASONS` + `cnTouchesInventory` |
| **`price`** *(غير مربوط)* | 2010 بـ`tot` | **4020 إيرادات أخرى** | نفسه | ❌ لا | `cnPostingAccount` — التحويل القسري |
| **`shortage` — عجز في الكمية** *(مربوط)* | 2010 بـ`tot` | **1200 المخزون** بـ`base` | نفسه | ✅ **نعم** | نفس المصدر |
| **`shortage`** *(غير مربوط)* | 2010 بـ`tot` | **4020 إيرادات أخرى** | نفسه | ❌ لا | نفس المصدر |
| **`quality` — خصم جودة** | 2010 بـ`tot` | **1260** بـ`base` | نفسه | ❌ لا · **يمسّ حساب مراقبة** | `CN_REASONS` |
| **`late` — غرامة تأخير** | 2010 بـ`tot` | **4020 إيرادات أخرى** | نفسه | ❌ لا | `CN_REASONS` |
| **`other` — أخرى** | 2010 بـ`tot` | **4020 إيرادات أخرى** | نفسه | ❌ لا | `CN_REASONS` |

> ⚠️ **تعارض في اسم الحساب 1260.** `CN_REASONS` تسمّيه **«هدر ومخالفات جودة الموردين»**، بينما `DB.accounts` و`Glossary.md` §10 يسمّيانه **«وسيط انحرافات وهدر المواد (تحت التسوية)»**. الكود يُطابق بالنص الكامل ⇒ **قد يُنشأ حساب ثانٍ بنفس الرقم وتسمية مختلفة في الأستاذ.** لم يُتحقَّق من سلوك `acctRemap` تجاه هذه الحالة. `Inferred — requires review`.
>
> **أثر ثانٍ:** خصم الجودة يُغذّي **1260** — وهو الحساب المحور في `KI-011` و`Monthly_Close_1260_Temporary_Control.md`. ⇒ **مصدر ثانٍ للتراكم في 1260 لم يكن معروفاً**، إضافةً لأوامر التشغيل.

## 4.5 بقية بنود التحقق

| البند | النتيجة | الثقة |
|---|---|---|
| الربط بفاتورة شراء | ✅ ممكن (`purchaseId`) · **اختياري** — لكنه **يغيّر الحساب المقابل جذرياً** | `Confirmed by direct code evidence` |
| يخفّض رصيد المورد | ✅ نعم — 2010 مدين بـ`tot` | `Confirmed by direct code evidence` |
| يؤثر على المخزون | ✅ فقط `price`/`shortage` **المربوطان** | `Confirmed by multiple code paths` |
| فرق سعر الشراء (PPV) | ❌ لا حساب PPV مستقل — الفرق يُرسمل على 1200 مباشرة | `Confirmed by direct code evidence` |
| **قفل الفترة** | ✅ **مُنفَّذ** — `if(isLocked(date)){toast('الفترة مقفولة');return false;}` | `Confirmed by direct code evidence` |
| **مصدر سعر الصرف** | ✅ **مُجمَّد على السند** وقت الحفظ · fallback حيّ عند الغياب | `Confirmed by direct code evidence` |
| **التعديل/الحذف يعيد كتابة التاريخ** | 🔴 **نعم.** `delCreditNote` حذف نهائي بلا عكس. و`editCreditNote` يستبدل السجل في مكانه ⇒ **الحذف والتعديل لا يفحصان `isLocked`** (الفحص على تاريخ **السجل الجديد** فقط) | `Confirmed by direct code evidence` |
| **منع التطبيق المزدوج** | ❌ **`Not found after scoped search`** — لا شيء يمنع إشعارين على نفس `purchaseId`؛ والمجموع قد يتجاوز قيمة الفاتورة | `Not found after scoped search` |
| **التدقيق** | ✅ **مُنفَّذ** — `audit('create'/'update'/'delete','إشعار خصم مورد',…)` | `Confirmed by direct code evidence` |
| **الصلاحيات** | `can('purchases','edit')` على الإنشاء والتعديل والحذف | `Confirmed by direct code evidence` |

## 4.6 إجابة OQ-4.J

> ## `Known Issue`

**الخريطة مكتملة ولا نقص فيها.** كل الأنواع الخمسة مُعرَّفة بالكود، وقاعدة الاختيار مُثبَتة بالنص، وسلوك VAT والمخزون قطعي.

**⇒ `Blocks Implementation Guide only` — مرفوع. Implementation Guide مفكوك الحجب من جهة OQ-4.J.**

المتبقّي فجوتان تقنيتان (`delete/edit` بلا حارس قفل · لا منع ازدواج) تحت `AINV-30` و`AINV-04` و`ACC-C`/`ACC-A` القائمة. **KI مقترح: `KI-027`.**

---

# 5. Target 3 — Account 2017 Settlement

## 5.1 الإجابات الخمس عشرة

| # | السؤال | الإجابة | الثقة |
|---|---|---|---|
| 1 | **ما الذي يُنشئ الرصيد في 2017؟** | مسار واحد فقط: استحقاق عمولة المبيعات ⇒ `JS-51` | `Confirmed by multiple code paths` |
| 2 | **أي كيان يخزّن المبلغ؟** | `DB.commissionAccruals` — `{id:'CM-0001', date, user, amount, pct, basis, from, to}` | `Confirmed by direct code evidence` |
| 3 | **أي دالة تُرحِّل الاستحقاق؟** | الكاتب `window.accrueCommission()` ⇒ يُخزِّن السجل · والترحيل في بلوك `(DB.commissionAccruals||[]).forEach(ca=>…)` داخل `buildJournalCore()`:<br>`[{acc:'5440 عمولات مبيعات',dr:ca.amount},{acc:'2017 عمولات مبيعات مستحقة',cr:ca.amount}]` | `Confirmed by direct code evidence` |
| 4 | **هل يوجد مسار تسوية تلقائي أو صريح؟** | 🔴 **لا.** **`Not found after scoped search`** — لا يوجد **أي سطر في `buildJournalCore()` يجعل 2017 مديناً** | `Confirmed by multiple code paths` |
| 5 | **هل يوجد كيان دفع مرتبط بالاستحقاق؟** | 🔴 **لا.** لا `commissionPayments` ولا حقل `paid`/`settled`/`settledBy` على سجل الاستحقاق | `Not found after scoped search` |
| 6 | **هل أي دفعة تجعل 2017 مديناً؟** | 🔴 **لا.** `DB.supplierPayments` ⇒ 2010 حصراً · `DB.expenses` ⇒ 5100/5200/5400 وائتمان 2010/2015/نقدية · `DB.custodyTx` ⇒ 1120 · `DB.taxPayments` ⇒ التزامات ضريبية. **ولا واحد منها يمسّ 2017** | `Confirmed by multiple code paths` |
| 7 | **هل يمكن لقيد يدوي تسوية 2017؟** | ✅ **تقنياً نعم.** `DB.manualJournals` حرّة الحسابات: `J.push({lines:m.lines,…})` بلا أي قيد على أكواد الحسابات | `Confirmed by direct code evidence` |
| 8 | **هل التسوية اليدوية مسار مُصمَّم أم إمكانية تقنية؟** | 🔴 **إمكانية تقنية بحتة.** لا شاشة · لا زر · لا نموذج · لا وسم `kind` · لا ربط بـ`CM-XXXX`. **لا شيء في النظام يقترحها أو يوجّه إليها** | `Confirmed by multiple code paths` |
| 9 | **هل التسوية الجزئية مدعومة؟** | لا ينطبق — لا يوجد مسار تسوية أصلاً | — |
| 10 | **هل تُمنع التسوية الزائدة؟** | ❌ **لا.** القيد اليدوي لا يفحص إلا التوازن — 2017 قد يصير **مديناً** بلا إنذار | `Confirmed by direct code evidence` |
| 11 | **هل يُمنع ازدواج التسوية؟** | ❌ **لا** — لنفس السبب | `Confirmed by direct code evidence` |
| 12 | **قفل الفترة** | ✅ **مُنفَّذ على الاستحقاق:** `if(isLocked(date)){toast('الفترة مقفولة');return false;}` داخل `accrueCommission()`. ⚠️ **وغير قابل للتقييم على التسوية** لعدم وجودها | `Confirmed by direct code evidence` |
| 13 | **هل يُخزَّن أثر تدقيقي للتسوية؟** | ❌ لا تسوية ⇒ لا أثر. **للاستحقاق:** `audit('create','استحقاق عمولات','CM',_f0(tot))` ✅ | `Confirmed by direct code evidence` |
| 14 | **هل يمكن تعديل/حذف الاستحقاق بعد الترحيل؟** | **`Not found after scoped search`** — لم يُرَ `editCommissionAccrual` ولا `delCommissionAccrual`. **لا يُدّعى غيابهما** | `Not found after scoped search` |
| 15 | **هل يمكن لفترات استحقاق متداخلة أن تُضاعف الرصيد؟** | 🔴 **نعم — مؤكَّد.** `accrueCommission()` يقرأ `window._cmFrom`/`_cmTo` من فلتر الشاشة ويُنشئ السجلات **بلا أي فحص تداخل** مع `from`/`to` المخزَّنة سابقاً. تشغيله مرتين على نفس المدى ⇒ التزام مزدوج | `Confirmed by direct code evidence` — يؤكّد `KI-017` |

## 5.2 الدليل الحاسم — اعتراف الشاشة بنفسها

```js
const accrued = (DB.commissionAccruals||[]).reduce((s,a) => s + (a.amount||0), 0);
kpiCard('navy','📒', 'المُثبَت بالقيود (2017)', egp(accrued), '')
```

بطاقة رصيد 2017 في `viewCommission()` **تجمع كل الاستحقاقات منذ نشأة النظام، بلا أي طرح لأي تسوية.** لو كان في تصميم النظام مسار استنفاد، لكانت هذه البطاقة ستطرحه. **هذا اعتراف بنيوي من الواجهة نفسها بأن 2017 حساب متراكم أبدياً.**

**تأكيد ثالث** — `plReport()`:
```js
const commissionExp = (DB.commissionAccruals||[]).filter(a=>inPeriod(a.date))
                        .reduce((s,a)=>s+(a.amount||0),0);
```
قائمة الدخل تقرأ المصروف من نفس الكيان ولا تعرف شيئاً عن أي تسوية.

## 5.3 التصنيف النهائي

> ## `No dedicated settlement or consumption path found after scoped search.`

**أربعة بيانات صريحة:**

1. **لم يُعثَر على مسار تسوية مخصَّص** لـ2017 بعد بحث موجَّه.
2. **لم يُعثَر على أي مصدر قيد يجعل 2017 مديناً** في `buildJournalCore()`.
3. **القيد اليدوي العام يبقى قادراً تقنياً** على جعل 2017 مديناً — `DB.manualJournals` حرّة الحسابات بلا أي قيد على الأكواد.
4. **وهذا ليس مسار تسوية مُصمَّماً** — لا شاشة ولا زر ولا نموذج ولا وسم `kind` ولا ربط بـ`CM-XXXX`.

**الثقة:** `Confirmed by multiple code paths` — ثلاثة مسارات مستقلة (بناء الدفتر · شاشة العمولات · قائمة الدخل) تتفق جميعها.

**تحفّظ منهجي واجب:** «لم يُعثَر عليه بعد بحث موجَّه» — لا «غير موجود» بيقين مطلق. الملف لم يُفحَص استقصائياً بالكامل.

## 5.4 حالة ACC-E

> ## `Still Blocked — evidence gap resolved, design decision pending.`

| البُعد | الحالة |
|---|---|
| **فجوة الدليل** (`Blocks ACC-E only` — جلسة 5 §10) | ✅ **مُغلَقة.** سلوك 2017 صار معروفاً قطعياً: يُنشَأ من مسار واحد ولا يُستنفَد أبداً |
| **معيار فكّ الحجب الكامل** | ❌ **غير مستوفٍ.** المطلوب «`documented control workflow` بمصدر · ربط · ضبط مبلغ · منع ازدواج · أثر تدقيقي · قفل فترة» — **لا شيء من ذلك موجود** |

**ولم يُفكّ الحجب بحجة أن القيد اليدوي يستطيع جعل 2017 مديناً** — القيد اليدوي إمكانية تقنية عامة، لا مسار مُصمَّم (الإجابة 8)، والتحذير في تعليمات هذه الجلسة مُطبَّق حرفياً.

**ما تغيّر عملياً — صياغة تفسيرية لا حالة رسمية:**

يمكن وصف الوضع بأن الحجب **جزئي الانفراج** — لكن **الحالة الرسمية للجاهزية تبقى `Still Blocked`**، ولا يُستخدَم `Partially Unblocked` كحالة ADR.

`ACC-E` **لم يعد محجوباً بنقص معرفة** — صار محجوباً بـ**قرار تصميم مطلوب**. لا يمكن كتابته كتوثيق لسلوك قائم؛ يجب أن **يُنشئ** ضابط تصفية غير موجود، وهذا قرار **Product Owner** لا Solution Architect وحده.

**السؤال المفتوح الجديد المقترح (OQ-6.A):** كيف يُسوَّى الالتزام في 2017؟ كيان دفع عمولات مستقل؟ أم توسعة `supplierPayments` لتشمل الموظفين؟ أم قيد يدوي **مُوجَّه** بربط إلزامي بـ`CM-XXXX`؟

**تحديث مقترح على `KI-017`** (لا يُنفَّذ الآن): توسعة النطاق من «لا حارس تداخل» إلى «**لا مسار استنفاد إطلاقاً**» — وهو أخطر بكثير، لأنه يعني أن **2017 حساب مراقبة يتراكم أبدياً**، تماماً كـ`2105` في `KI-012`. **النمط الآن مؤكَّد في حسابَي مراقبة، لا واحد.**

---

# 6. Target 4 — Conditional Sources

> ⚠️ **SUPERSEDED (2026-07-25) — راجع §12.4.** تصنيف JS-C1/JS-C2 كـ`Unverified candidate / Conditional` في هذا القسم **مُعلىً بأدلة التشغيل**؛ كلاهما الآن **`Confirmed posting source`** (JS-C1 = Opening Journal · JS-C2 = Fixed Asset Acquisition). النص أدناه يُحفَظ **كتاريخ تحقيق فقط** ولا يُعتمَد كتصنيف حالي.

## 6.1 JS-C1 — الأرصدة الافتتاحية

| البند | النتيجة | الثقة |
|---|---|---|
| كيان أرصدة افتتاحية مستقل | **`Not found after scoped search`** | — |
| ما وُجد بدلاً منه | حقل `opening` **على كل خزينة** (`tr.opening`) تقرؤه `treasuryBalance()` · ودالة `openingDate()` تستخدمها `closingPreview()` | `Confirmed by direct code evidence` |
| هل يُرحَّل عبر `buildJournalCore()`؟ | 🔴 **لا.** `tr.opening` يدخل **حساب رصيد الخزينة فقط** — **ولم يُرَ له أي قيد في الدفتر** | `Confirmed by direct code evidence` |
| الحساب 3001 | معرَّف في `Glossary.md` §10 كـ«أرصدة افتتاحية» — **`Not found after scoped search`** كطرف في أي قيد | — |
| قفل الفترة · الثبات | لا ينطبق — لا قيد | — |

> ### 🔴 اكتشاف حرج
> `tr.opening` يُضاف إلى **رصيد شاشة الخزينة** ولا يُضاف إلى **الأستاذ العام**. ⇒ **الخزينة والأستاذ يفترقان بقيمة الرصيد الافتتاحي كاملةً.**
>
> وهذا **نفس المرض** الذي وثّق الكود إصلاحه في `cashAcc()`: *«الأستاذ يقول نقدية 2,255,936 وشاشة الخزينة تقول 666,650. عالمان منفصلان.»* — الإصلاح عالج **حساب الوجهة**، ولم يعالج **الرصيد الافتتاحي**.
>
> **يخرق `AINV-33`** (تطابق الأستاذ مع الدفاتر المساعدة) مباشرة. **KI مقترح: `KI-028` — Critical.**

### التصنيف النهائي — JS-C1

> ## `Unverified candidate — likely future requirement`

| الاحتمال | الحكم |
|---|---|
| `Confirmed posting source` | ❌ **مرفوض** — لم يُعثَر على أي قيد |
| `Use case of another source` | ❌ **مرفوض** — `tr.opening` حقل رصيد، لا مستند يُرحَّل |
| `Unverified candidate` | ✅ **نعم** — القيمة **قائمة تشغيلياً** في `treasuryBalance()`، ومسار ترحيلها لم يُعثَر عليه |
| `Future required source` | ✅ **مرجَّح** — `Accounting.md` §3 يوجب صراحةً قيداً افتتاحياً مقابل **3001** |

> ⚠️ **إدراج JS-C1 في العدّ الإجمالي يبقى مؤقّتاً (`provisional`).** هو مُدرَج اليوم كـ`Conditional` استناداً إلى كتالوج جلسة 1، **لا إلى دليل ترحيل**. إن ثبت غياب القيد نهائياً، ينتقل من `Conditional` إلى `Future` — **وهو تغيير تصنيف لا تغيير عدد**.

## 6.2 JS-C2 — شراء أصل ثابت

| البند | النتيجة | الثقة |
|---|---|---|
| الكيان | `DB.assets` — موجود | `Confirmed by direct code evidence` |
| قيد **الاقتناء** | **`Not found after scoped search`** — لم يُرَ أي قيد `1300 ⇐ نقدية` | — |
| قيد **الإهلاك** | ✅ **مُرحَّل فعلاً** — `5400 / 1310`، **قسط شهري مستقل لكل شهر** (`JS-48`). تعليق الكود يوثّق إصلاحاً سابقاً: كان قيداً واحداً بالمجمّع في تاريخ التقرير | `Confirmed by direct code evidence` |
| قراءة `plReport()` | `depreciation = (DB.assets||[]).reduce(… assetDepreciation(a).accumulated …)` | `Confirmed by direct code evidence` |
| المسار البديل المحتمل | الاقتناء **قد** يدخل عبر `DB.expenses` (⇒ 5100/5200/5400 — **مصروف لا أصل**) أو عبر قيد يدوي. **لم يُرَ أيٌّ منهما** | `Inferred — requires review` |
| VAT · التاريخ · المصدر | غير قابلة للتحديد بلا قيد اقتناء | — |

### التصنيف النهائي — JS-C2

> ## `Unverified candidate — likely future requirement`

| الاحتمال | الحكم |
|---|---|
| `Confirmed posting source` | ❌ **مرفوض** — قيد الاقتناء لم يُعثَر عليه |
| `Use case of another source` | 🟡 **محتمل غير مُثبَت** — قد يدخل عبر `DB.expenses` (JS-19) أو قيد يدوي (JS-52). `Inferred — requires review` |
| `Unverified candidate` | ✅ **نعم** — الكيان `DB.assets` قائم والإهلاك يُرحَّل فعلاً (JS-48) |
| `Future required source` | ✅ **مرجَّح** — `Accounting.md` §3.3 ينص: الأصول بعد Go-Live «تدخل كمعاملات اقتناء عادية» |

> ⚠️ **إدراج JS-C2 في العدّ يبقى مؤقّتاً (`provisional`).**
>
> **الأثر المحاسبي قائم بصرف النظر عن التصنيف:** الأصل يُهلَك في الدفاتر **دون أن يكون قد دخلها**. `1310 مجمع الإهلاك` ينمو مقابل `1300` قد يكون صفراً في الأستاذ. **KI مقترح: `KI-029`.**

## 6.3 JS-C3 — الرواتب

| البند | النتيجة | الثقة |
|---|---|---|
| كيان/دالة ترحيل مخصَّصة للرواتب | **`Not found after scoped search`** | — |
| ما وُجد | **`DB.recurring`** — تعريفات مصروفات متكررة تُولَّد يدوياً عند تشغيل `runRecurring` or `runAllRecurring`:<br>`{id:'RC-002', desc:'رواتب الموظفين', amount:80000, cat:'رواتب', allocType:'overhead', day:28, active:true, lastGen:''}` | `Confirmed by direct code evidence` |
| مسار الترحيل | ✅ **التوليد يُطلَق يدوياً** عبر `runRecurring` / `runAllRecurring`. `genRecurring()` تُنشئ سجلاً في **`DB.expenses`** ⇒ يُرحَّل عبر **`JS-19`**. **لا يمرّ بـ`DB.manualJournals`.** | `Confirmed by direct code evidence` |
| هل يستخدم `DB.manualJournals` فقط؟ | ❌ **لا.** المسار المُشاهَد هو `recurring ⇒ expenses`، **لا القيد اليدوي** كما وثّقت جلسة 1 | `Confirmed by direct code evidence` (بنية البيانات) |
| هل لـ`kind`/`ref`/`PAYROLL-` سلوك خاص؟ | ❌ **لا.** `cat:'رواتب'` **وصفي بحت** ولا يدخل اختيار الحساب (انظر §7.1) | `Confirmed by direct code evidence` |
| **خريطة الحسابات الفعلية** | `allocType:'overhead'` ⇒ **`5400 مصاريف إدارية وعمومية`** · دائن: 2010 لو `supplierCode`، وإلا `cashAcc` للمدفوع و`2015` للباقي | `Confirmed by direct code evidence` |
| **الخريطة المتوقَّعة (الكتالوج)** | `5410/5420 ⇐ 2130/2120/2140` | `Accounting.md` §Payroll |
| مصدر التاريخ | `e.date` من السجل المتولّد | `Confirmed by direct code evidence` |
| قفل الفترة | 🔴 The manually triggered recurring-generation path has no observed `isLocked()` check. | `Not found after scoped search` |
| الاعتماد | لا اعتماد — `active:true` يكفي | `Confirmed by direct code evidence` |

### الفحص المباشر للكاتب (البند 8-2) — `genRecurring()` بالنص الكامل

```js
function genRecurring(rc, monthStr){
  // monthStr مثل '2026-06' — يُنشئ مصروف فعلي لهذا الشهر إن لم يكن مولّداً
  const day  = String(rc.day||1).padStart(2,'0');
  const date = monthStr+'-'+day;
  const exists = DB.expenses.some(e => e.recurringId===rc.id
                                   && (e.date||'').slice(0,7)===monthStr);
  if (exists) return null;                       // ← حارس ازدواج شهري
  const obj = {id:nextCode('exp'), date, supplierCode:'', desc:rc.desc+' ('+monthStr+')',
    cat: rc.cat||'إدارية', cur:'EGP', amount:rc.amount||0, vatPct:0, paid:0,
    allocType: rc.allocType||'overhead', allocRef:'', recurringId:rc.id};
  DB.expenses.push(obj);                          // ← يدخل المصروفات، لا القيود اليدوية
  rc.lastGen = monthStr;
  return obj;
}
```

**⇒ الاستنتاج ارتفع من `Inferred` إلى `Confirmed by direct code evidence`.** `DB.recurring` تُنتج **سجل مصروف** حصراً. لا تمرّ بـ`DB.manualJournals` إطلاقاً.

| البند | النتيجة | الثقة |
|---|---|---|
| الوجهة | `DB.expenses` ⇒ **JS-19** | `Confirmed by direct code evidence` |
| `allocType` | `rc.allocType \|\| 'overhead'` ⇒ **5400** | `Confirmed by direct code evidence` |
| **حارس الازدواج** | ✅ **موجود** — `recurringId` + شهر | `Confirmed by direct code evidence` |
| **قفل الفترة** | 🔴 **`Not found`** — The manually triggered recurring-generation path has no observed `isLocked()` check ⇒ توليد في فترة مقفلة ممكن | `Not found after scoped search` |
| **التدقيق** | 🔴 **`Not found`** — لا `audit()` | `Not found after scoped search` |
| **الإطلاق** | ⚠️ **يدوي بزر** (`runRecurring` / `runAllRecurring`) — العملية يدوية بالكامل؛ وتعليق الكود الذي يصفها بالتلقائية **غير مطابق للسلوك الفعلي** | `Confirmed by direct code evidence` |
| الصلاحية | `can('journal','edit')` | `Confirmed by direct code evidence` |

> ### 🔴 اكتشاف إضافي — نظاما «قيود متكررة» متوازيان
> يوجد كيانان: **`DB.recurring`** (`genRecurring` — شهري بيوم استحقاق) و**`DB.recurringExp`** (`generateRecurring` + `recurringDue` — بترددات `monthly`/`quarterly`/`yearly`).
>
> والأخطر: **`window.editRecurring` مُعرَّفة مرتين**، والتعريف الثاني (الخاص بـ`DB.recurringExp`) **يُظلِّل الأول**. تعليق في الكود يعترف: *«نسخة مكرّرة محذوفة — كانت كوداً ميتاً يُظلّله تعريف لاحق»*.
>
> ⇒ زر التعديل في شاشة `viewRecurring` (التي تعرض `DB.recurring`) **يفتح نموذج `DB.recurringExp`**. **نمط `KI-005` بعينه — طبقتان تاريخيتان.** **KI مقترح: `KI-031`.**
>
> و`generateRecurring()` تدفع إلى `DB.expenses` **بلا حارس ازدواج مكافئ وبلا `isLocked`**.

### التصنيف النهائي — JS-C3

> ## `Specialized use case of JS-19`

**وليس `Specialized use case of JS-52`** — الرواتب **لا تمرّ بالقيد اليدوي إطلاقاً**.

**الثقة:** `Confirmed by direct code evidence` — الكاتب مفحوص بالنص الكامل، كما طلب البند 8-2.

> **قيد على الإخراج من العدّ:** رغم أن الدليل قطعي على **مسار الترحيل**، فإن إخراج JS-C3 من الكتالوج يظل **مقترحاً غير منفَّذ** — لأنه يتوقف أيضاً على قاعدة حدود المصدر (§10.4). لم يُعدَّل أي كتالوج.

> ### 🔴 اكتشاف حرج — خمسة حسابات معطَّلة فعلياً
> `5410` (رواتب) · `5420` (تأمينات) · `2120` (ض. كسب عمل) · `2130` (تأمينات مستحقة) · `2140` (رواتب مستحقة) — **معرَّفة كلها في دليل الحسابات، ولم يُرَ لأيٍّ منها أي طرف قيد.**
>
> الرواتب كلها تُجمَّع في `5400 مصاريف إدارية وعمومية` مع الكهرباء والإيجار، **بلا فصل ولا استقطاعات ولا حصة شركة**.
>
## 6.4 أثر الهدف 4 على العدد — ⚠️ عكس المتوقَّع

التعليمات افترضت أن JS-C3 لو كان حالة استخدام ⇒ العدد **58 ⇒ 57**. **التحقق يقول إن الأثر الصافي معاكس.**

| التغيير | الأثر التراكمي |
|---|---|
| JS-C3 ⇒ حالة استخدام لـJS-19 | Conditional **3 ⇒ 2** · الإجمالي **58 ⇒ 57** |
| JS-C1 ⇒ `Unverified candidate` (يبقى في العدّ **مؤقّتاً**) | بلا تغيير عددي |
| JS-C2 ⇒ `Unverified candidate` (يبقى في العدّ **مؤقّتاً**) | بلا تغيير عددي |
| **+ أربعة مصادر مؤكَّدة** من `DB.treasuryTx` (§7.6) | Confirmed **54 ⇒ 58** · الإجمالي **57 ⇒ 61** |

> ### الاقتراح — `Proposed pending source-boundary validation`
>
> | البند | الحالي | المقترح |
> |---|---|---|
> | Confirmed | 54 | **58** |
> | Conditional | 3 | **2** *(مؤقّت — JS-C1 · JS-C2)* |
> | Future | 1 | **1** |
> | **الإجمالي** | **58** | **61** |
>
> **⇒ العدد لا يبقى 58، ولا ينزل إلى 57. المقترح 61.**
>
> ⚠️ **لا يُعتمَد.** مشروط بقاعدة حدود المصدر (§10.4) وبحسم JS-C1/JS-C2.

---

# 7. Target 5 — Detailed Account Mapping

## 7.1 JS-19 — فاتورة مصروف/خدمات

> ## 🔴 تصحيح جوهري على جلسة 5
> §2.10 سجّلت: «JS-19 — حساب المصروف يتبع `cat`». **غير صحيح.** الحساب يتبع **`allocType`** حصراً. حقل `cat` **وصفي بحت** ولا يدخل أي فرع اختيار.

**الكود:**
```js
const drAcc = e.allocType==='invoice' ? '5100 مصاريف مباشرة (شحنات)'
            : e.allocType==='product' ? '5100 مصاريف مباشرة (شحنات)'
            : e.allocType==='general' ? '5200 مصاريف تشغيلية'
            :                           '5400 مصاريف إدارية وعمومية';   // default
```

| `allocType` | حساب المدين | ملاحظة |
|---|---|---|
| `'invoice'` | **5100** مصاريف مباشرة (شحنات) | محمّل على شحنة عبر `allocRef` |
| `'product'` | **5100** مصاريف مباشرة (شحنات) | نفس الحساب |
| `'general'` | **5200** مصاريف تشغيلية | تُوزَّع |
| `'overhead'` أو أي قيمة أخرى | **5400** مصاريف إدارية وعمومية | **الافتراضي** — يبتلع كل ما لا يُطابق |

**معالجة VAT** — `vatDeductibleExp(e)`:
- قابلة للخصم و`vat>0.01` ⇒ `drAcc` بـ`baseEGP` **+ `1130` مدين بـ`vatEGP`**
- غير قابلة ⇒ `drAcc` بالإجمالي `t` (**الضريبة تُرسمل على المصروف**)

**ضريبة الخصم والإضافة:** `whtEGP = baseEGP*(e.whtPct||0)` ⇒ `2110` دائن.

**منطق الطرف الدائن:**
```js
const netPayable = t - whtEGP;
if (e.supplierCode) {
  lines.push({acc:'2010 الموردون', dr:0, cr:netPayable});      // الالتزام كله للمورد
} else {
  const paidEGP = Math.max(0, Math.min((e.paid||0)*rate, netPayable));
  const owedEGP = netPayable - paidEGP;
  if (paidEGP>0.001) lines.push({acc:cashAcc(e.treasury), dr:0, cr:paidEGP});
  if (owedEGP>0.001) lines.push({acc:'2015 مصروفات مستحقة (غير مدفوعة)', dr:0, cr:owedEGP});
}
```
> **قاعدة حاكمة:** وجود `supplierCode` **يُلغي حقل `paid` تماماً** — الالتزام كله على 2010 ويُسدَّد عبر `supplierPayments`.

**هل الخريطة قابلة للتهيئة؟** ❌ **لا — مُصلَّبة (hard-coded)** في تعبير شرطي داخل بناء الدفتر. لا جدول تهيئة ولا إعداد.

**مصدر السعر:** `expenseRate(e)` · **التاريخ:** `e.date` · **القفل:** ✅ مُنفَّذ في نموذج فاتورة الخدمات.

## 7.2 JS-38 — إعادة تقييم أرصدة (مُرحَّلة) — ✅ **مُحلّ**

> ### تصحيح على النسخة 1.0.0
> النسخة السابقة قالت `Not found after scoped search`. **الفحص الموجَّه المطلوب في البند 8-1 عثر على الكيان بالكامل.** ولم يُقَل في أي موضع إن `DB.revaluations` غير موجود.

### الكيان والكاتب

| البند | القيمة | الثقة |
|---|---|---|
| **الكيان** | `DB.revaluations` — **مستقل تماماً عن `DB.fxRevaluations`** | `Confirmed by direct code evidence` |
| **الكاتب** | `window.postRevaluation()` | `Confirmed by direct code evidence` |
| **الحاذف** | `window.delRevaluation(i)` — `DB.revaluations.splice(i,1)` بصلاحية `can('journal','edit')` | `Confirmed by direct code evidence` |
| **العارض** | `viewRevaluation()` | `Confirmed by direct code evidence` |
| **الحاسبة** | `revaluation(closeRates)` — تقرأ أسعار إقفال قابلة للتعديل من `window._revalRates` | `Confirmed by direct code evidence` |

**السجل المحفوظ:**
```js
{id: nextCode('rev'), date: DB.reportDate,            // ← تاريخ التقرير قسراً
 rates: {...closeRates}, gain, loss, net,
 reverse: _reverse,                                    // افتراضياً true
 reverseDate: _reverse ? firstOfNextMonth(DB.reportDate) : '',
 items: [{type, code, name, cur, diff}]}               // type ∈ customer|supplier|(bank)
```

### الأنواع المدعومة وخريطة الحسابات — مطلوبة صراحةً

```js
const partyAcc = it.type==='customer' ? '1100 العملاء'
               : it.type==='supplier' ? '2010 الموردون'
               :                        '1020 البنك';
if (it.diff>0) lines.push({acc:partyAcc, dr:it.diff, cr:0});
else           lines.push({acc:partyAcc, dr:0, cr:-it.diff});
…
if (Math.abs(rv.net)>0.01){
  if (rv.net>0) lines.push({acc:'4030 أرباح فروق عملة', dr:0, cr:rv.net});
  else          lines.push({acc:'5500 خسائر فروق عملة', dr:-rv.net, cr:0});
}
add(rv.date, 'إعادة تقييم أرصدة أجنبية (غير محقق) '+rv.id, lines, rv.id);
```

| `it.type` | `diff > 0` | `diff < 0` | مصدر النوع |
|---|---|---|---|
| **`customer`** | **1100 العملاء** مدين | **1100 العملاء** دائن | `rev.items[].type` من `revaluation()` |
| **`supplier`** | **2010 الموردون** مدين | **2010 الموردون** دائن | نفسه |
| **`bank`** (وأي قيمة أخرى) | **1020 البنك** مدين | **1020 البنك** دائن | نفسه — **الافتراضي** |
| **صافي القيد** | `net>0` ⇒ **4030** دائن | `net<0` ⇒ **5500** مدين | `rv.net` |

### 🔴 العكس التلقائي — قيد ثانٍ لم يكن موثَّقاً

```js
if (lines.length && rv.reverse!==false && (rv.reverseDate||rv.date)){
  const revDate = rv.reverseDate || firstOfNextMonth(rv.date);
  const rlines  = lines.map(l => ({acc:l.acc, dr:l.cr||0, cr:l.dr||0}));
  add(revDate, 'عكس إعادة تقييم أرصدة أجنبية (بداية الفترة الجديدة) '+rv.id, rlines, rv.id);
}
```

**⇒ `JS-38` تُنتج قيدين لا قيداً واحداً:** قيد التقييم بتاريخ `rv.date` · وقيد **عكس كامل** بتاريخ `firstOfNextMonth`. تعليق الكود يوثّق الغرض: منع ازدواج فرق العملة في الأرباح والخسائر — **EAS 13 / IAS 21**.

**مقترح:** فصل العكس إلى **`JS-38b`** — قيد مستقل بتاريخ مستقل وحسابات معكوسة (§10.4).

### بقية بنود التحقق

| البند | النتيجة | الثقة |
|---|---|---|
| **مصدر التاريخ** | 🔴 **`DB.reportDate` قسراً** عند الكتابة — لا حقل تاريخ للمستخدم | `Confirmed by direct code evidence` |
| **Snap/Recalc** | ✅ **`SNAP`** — `rates` و`items[].diff` كلها **مُجمَّدة على السجل** | `Confirmed by direct code evidence` |
| **قفل الفترة** | 🔴 **`Not found`** — لا `isLocked()` في `postRevaluation()` ولا في `delRevaluation()`. **بخلاف `postFxRevaluation()` الذي يفحصه صراحةً** | `Not found after scoped search` |
| **سلوك الحذف** | 🔴 يمحو **القيدين معاً** (التقييم وعكسه) من كل الفترات بلا أثر — `securityLog` عند الترحيل فقط، **ولا `audit()` عند الحذف** | `Confirmed by direct code evidence` |
| **الصلاحية** | `can('journal','edit')` على الحذف · `securityLog('ترحيل إعادة تقييم')` على الترحيل | `Confirmed by direct code evidence` |

### 🔴🔴 الاكتشاف الأخطر — نظاما إعادة تقييم متوازيان

| | `DB.revaluations` (JS-38) | `DB.fxRevaluations` (JS-34→37) |
|---|---|---|
| **الكاتب** | `postRevaluation()` | `postFxRevaluation()` |
| **الحسابات** | **حسابات الأطراف** 1100 · 2010 · 1020 | **الحساب الوسيط 1265** |
| **مقابل** | 4030 / 5500 مباشرةً | 4030 / 5500 ثم إقفال على 3020 |
| **العكس** | ✅ تلقائي أول الشهر التالي | ❌ لا — إقفال يدوي بـ`closeFxRevaluation()` |
| **قفل الفترة** | 🔴 غير مفحوص | ✅ مفحوص |
| **التدقيق** | `securityLog` فقط | `audit()` كامل |
| **المرجع** | — | **ADR-003** |

**كلاهما يقرأ نفس الأرصدة الأجنبية المفتوحة** (`revaluation()` مقابل `fxExposure()`/`fxRevalNet()`).

> ⇒ **تشغيل الاثنين في نفس الفترة يُثبت فرق العملة غير المحقق مرتين** — مرة على حسابات الأطراف ومرة على 1265. **ولا شيء في النظام يمنع ذلك أو ينبّه إليه.**
>
> والأخطر: **`DB.revaluations` تخالف ADR-003 مباشرةً** — الذي يوجب المرور بالحساب الوسيط 1265.
>
> **KI مقترح: `KI-032` — 🔴 Critical.** وهو **سؤال قرار معماري** لا مجرد خلل: أي النظامين يبقى؟

### أثر حسم JS-38

| البند | قبل | بعد |
|---|---|---|
| **Posting Matrix** | 🔴 محجوب | ✅ **مفكوك الحجب** |
| **OQ-6.E** | مفتوح | ✅ **مُغلَق `Closed by Evidence`** — ويُستبدَل بـ**OQ-6.F**: أي نظامَي إعادة التقييم يبقى؟ |

## 7.3 JS-39 — تحويل بين الخزائن

**الكود:**
```js
} else if (tx.kind==='transfer') {
  const toTr=treas(tx.toTreasury); const toAcc=(toTr.acc||'1020');
  const srcCur=tr.cur||'EGP', dstCur=toTr.cur||'EGP';
  const toAmt = txToAmount(tx);              // بعملة الحساب المستلم
  const toEgp = toAmt * fxRate(dstCur);      // الطرف المدين
  const srcEgp = (tx.amount||0)*fxRate(tr.cur||'EGP');   // الطرف الدائن
  const lines=[{acc:toLbl,dr:toEgp,cr:0},{acc:accLbl,dr:0,cr:srcEgp}];
  const diff = toEgp - srcEgp;
  if (diff>1e-9)  lines.push({acc:'4030 أرباح فروق عملة',dr:0,cr:diff});
  else if(diff<-1e-9) lines.push({acc:'5500 خسائر فروق عملة',dr:-diff,cr:0});
  add(tx.date,'تحويل بين حسابات '+tx.id+xcur,lines,tx.id);
}
```

| البند | النتيجة |
|---|---|
| **خزينة المصدر** | `treas(tx.treasury).acc` ⇒ حساب الأستاذ · **دائن** بـ`srcEgp` |
| **خزينة الوجهة** | `treas(tx.toTreasury).acc` ⇒ **مدين** بـ`toEgp` |
| **المبلغ المستلم** | `txToAmount(tx)` — يفضّل `tx.toAmount` المخزَّن؛ وإلا يحوّل بأسعار الصرف (توافق خلفي) |
| **منطق فرق العملة المحقَّق** | `diff = toEgp − srcEgp` · موجب ⇒ **4030** · سالب ⇒ **5500** · **السماحية `1e-9`** — أدقّ بكثير من `0.005` المستخدمة في بقية المسارات |
| **معالجة العملتين** | كل طرف بـ`fxRate` عملة خزينته |
| 🔴 **Snapshot** | **`fxRate()` حيّ لكلا الطرفين** — لا سعر مُجمَّد على الحركة. تغيّر جدول الأسعار **يُغيّر قيمة تحويل تاريخي وقيمة فرق العملة معه** |

> **هذا خرق لـ`AINV-10` لم يكن مُصنَّفاً.** الكتالوج صنّف JS-39 `SNAP` — **والدليل يقول `RECALC`.** انظر §10.2.

## 7.4 JS-41 — صرف من عهدة

**الكود:**
```js
const expAcc = ct.invId ? '5100 مصاريف مباشرة (شحنات)'
             : (ct.cat==='تشغيلية' ? '5200 مصاريف تشغيلية'
             :  ct.cat==='مباشرة'  ? '5100 مصاريف مباشرة (شحنات)'
             :                       '5400 مصاريف إدارية وعمومية');
```

**قاعدة الاختيار بثلاث درجات:**

| # | الشرط | الحساب |
|---|---|---|
| 1 | `ct.invId` مملوء (صرف على شحنة) | **5100** — **يتقدّم على كل شيء** |
| 2 | `ct.cat === 'تشغيلية'` | **5200** |
| 3 | `ct.cat === 'مباشرة'` | **5100** |
| 4 | غير ذلك (`'إدارية'` أو فارغ) | **5400** — الافتراضي |

**فرع رابع مستقل — شراء صنف:**
```js
if (ct.prodCode && ct.qty>0) {   // 1200 المخزون / 1120 عهد — لا مصروف إطلاقاً
```

| البند | النتيجة |
|---|---|
| **حقل الفئة** | `ct.cat` — ثلاث قيم: `إدارية` · `تشغيلية` · `مباشرة` (من نموذج `custodyTxForm`) |
| **قابلة للتهيئة؟** | ❌ **لا — مُصلَّبة** |
| **VAT** | ❌ **لا معالجة ضريبية إطلاقاً** — لا `vatPct` على حركة العهدة ولا سطر 1130. ⚠️ **ض.ق.م مدخلات على مصروفات العهدة تُفقَد بالكامل** |
| **القفل** | ✅ مُنفَّذ — `if(isLocked($('#ct_date').value)){…return false;}` |
| **حارس الرصيد** | ⚠️ **تحذير لا منع** — `if(amt>bal+0.01){toast(…,'warn');}` بلا `return false` ⇒ رصيد العهدة قد يصير سالباً |
| **الصلاحية** | `can('journal','edit')` على إذن الصرف المتعدد |

## 7.5 الجدول الموحَّد المطلوب

| JS ID | Condition | Debit | Credit | FX | Date | Lock | Evidence |
|---|---|---|---|---|---|---|---|
| **JS-19** | `allocType='invoice'\|'product'` | **5100** (+1130 لو قابلة للخصم) | `supplierCode` ⇒ **2010** · وإلا `cashAcc`+**2015** · (+**2110** لو WHT) | `expenseRate(e)` | `e.date` | ✅ | `buildJournalCore` — بلوك المصروفات |
| **JS-19** | `allocType='general'` | **5200** | نفسه | نفسه | نفسه | ✅ | نفسه |
| **JS-19** | `allocType='overhead'` / افتراضي | **5400** | نفسه | نفسه | نفسه | ✅ | نفسه |
| **JS-38** | `type='customer'` | **1100** | (+**4030** لو `net>0`) | 🔴 `DB.reportDate` قسراً · SNAP | `DB.reportDate` | 🔴 غير مفحوص | `postRevaluation` |
| **JS-38** | `type='supplier'` | **2010** | (+**5500** لو `net<0`) | نفسه | نفسه | نفسه | نفسه |
| **JS-38** | `type='bank'`/افتراضي | **1020** | نفسه | نفسه | نفسه | نفسه | نفسه |
| **JS-38b** | العكس التلقائي | حسابات معكوسة | — | نفسه | `firstOfNextMonth` | نفسه | نفسه — بلوك العكس |
| **JS-39** | `tx.kind='transfer'` | خزينة الوجهة بـ`toEgp` · (+**5500** لو `diff<0`) | خزينة المصدر بـ`srcEgp` · (+**4030** لو `diff>0`) | 🔴 `fxRate()` **حيّ لكل طرف** | `tx.date` | ⚠️ غير مُتحقَّق | بلوك `treasuryTx` |
| **JS-41** | `kind='expense'` · `invId` مملوء | **5100** | **1120** | — | `ct.date` | ✅ | بلوك `custodyTx` |
| **JS-41** | `kind='expense'` · `cat='تشغيلية'` | **5200** | **1120** | — | نفسه | ✅ | نفسه |
| **JS-41** | `kind='expense'` · `cat='مباشرة'` | **5100** | **1120** | — | نفسه | ✅ | نفسه |
| **JS-41** | `kind='expense'` · افتراضي | **5400** | **1120** | — | نفسه | ✅ | نفسه |
| **JS-41b** | `kind='expense'` · `prodCode && qty>0` | **1200** | **1120** | — | نفسه | ✅ | نفسه — **فرع رابع غير مُكتلَج** |

## 7.6 أربعة مصادر مؤكَّدة غير مُكتلَجة

**الفحص الكامل المطلوب في البند 8-3:** بلوك `(DB.treasuryTx||[]).forEach(tx=>…)` و`TX_KINDS` مفحوصان بالنص الكامل. **خمسة `kind`** بخمسة فروع ترحيل. `transfer` هو JS-39؛ والأربعة الباقية غير مُكتلَجة:

```js
const TX_KINDS = {
  deposit:  {sign:+1}, drawing: {sign:-1}, transfer:{sign:0},
  bankfee:  {sign:-1}, interest:{sign:+1},
};
```

| المقترح | `tx.kind` | مدين | دائن | التاريخ |
|---|---|---|---|---|
| **JS-55** | `'deposit'` — إيداع رأس مال | حساب الخزينة | **3010 رأس المال** | `tx.date` |
| **JS-56** | `'drawing'` — مسحوبات المالك | **3030 مسحوبات المالك** | حساب الخزينة | `tx.date` |
| **JS-57** | `'bankfee'` — عمولة بنكية | **5400** مصاريف إدارية | حساب الخزينة | `tx.date` |
| **JS-58** | `'interest'` — فائدة بنكية | حساب الخزينة | **4020 إيرادات أخرى** | `tx.date` |

جميعها `Confirmed by direct code evidence` — مقروءة من `TX_KINDS` ومن بلوك الترحيل معاً. **ثلاثة منها تمسّ حقوق الملكية والنتيجة مباشرة.**

---

# 8. Target 6 — Deferred Observations from Session 4.6

✅ **`Accounting_Session4_6_Findings.md` سُلِّم ومقروء.** قسم `Deferred Observation — Not Investigated` يحوي **ستة بنود بالضبط**، مسرودة أدناه حرفياً كما وردت، ثم مصنَّفة.

| # | الملاحظة (حرفياً من المصدر) | التصنيف | المبرِّر |
|---|---|---|---|
| **DO-1** | «`closeReadiness()` تحتوي أربعة بنود موسومة `info:true` بينها بندان يحملان `ok` محسوباً — الوسم يُلغي أثرهما كمانع؛ لم يُفحَص سبب الوسم.» | **Converted to OQ** ⇒ **OQ-6.G** | سؤال تصميم: لماذا فحوص حقيقية موسومة `info`؟ يمسّ `ACC-D` (القفل المركزي) |
| **DO-2** | «`subledgerRecon()` و`invReconData()` أدوات مطابقة قائمة (1100 · 2010 · مخزون) لم تُفحَص.» | **Still deferred** | خارج نطاق الأهداف الستة لجلسة 6.1؛ أدوات مطابقة تدعم ACC-A مستقبلاً |
| **DO-3** | «`doMonthEndClose` تعتمد `allOk` من مجموعة فحوص مستقلة لم تُفحَص.» | **Duplicate of KI-015** | مسار قفل إضافي بلا حارس مركزي — نفس نطاق `KI-015` · `AINV-26` |
| **DO-4** | «`doClosing()` تُنشئ قيد إقفال (3099/3020) وتضبط `DB.lockDate` — مسار قفل ثالث لم يُفحَص محاسبياً.» | **Converted to KI** ⇒ **KI-033** + **مصدر قيد جديد JS-59** | قيد إقفال 3099⇒3020 **غير مُكتلَج**؛ ومسار قفل ثالث خارج المركزية ⇒ `AINV-26` · `ACC-A` · `ACC-D` |
| **DO-5** | «`inPeriod()` تُمرِّر السجل بلا تاريخ في **كل** الدوال المُرشَّحة، لا `fInvoices` وحدها.» | **Duplicate of KI-016** | نفس خلل «السجل بلا تاريخ يتجاوز القفل» المعمَّم — `KI-016` · `AINV-27` |
| **DO-6** | «`suggestedClaimsRate()` تشتق نسبة من تاريخ المطالبات — أساسها لم يُفحَص.» | **Converted to OQ** ⇒ **OQ-6.H** | أساس اشتقاق نسبة المخصص غير معروف؛ يمسّ `KI-013` (وعاء المخصص) |

**التوزيع:** Converted to OQ **2** · Converted to KI **1** · Duplicate **2** · Still deferred **1** = **6**.

> **لم يُوسَّع النطاق.** الملاحظات الست فقط، بلا إضافة. `DO-4` كشف **مصدر قيد جديد (قيد الإقفال 3099⇒3020)** لم يكن في الكتالوج — يُسجَّل كأثر، ويؤثّر على العدّ (§10.4).

---

# 9. Open Question Resolution Matrix

| OQ | Primary Classification | الدليل | القرار المطلوب | Reviewer Recommendation — Not a Decision | Owner | Target artifact |
|---|---|---|---|---|---|---|
| **OQ-4.G** | **`Known Issue`** | S6.1 §3 — بلوك `supplierPayments` في `buildJournalCore` | حرّاس: ربط · قفل · ازدواج · سقف | إلزام الربط لتفعيل فرق العملة ⇒ AINV-11 | Dev Lead | `Known_Issues.md` (KI-025 · KI-026) |
| **OQ-4.J** | **`Known Issue`** | S6.1 §4 — `CN_REASONS` + `cnPostingAccount()` | حارس ازدواج + فحص قفل على الحذف/التعديل | نفس نمط KI-008 ⇒ ACC-C | Dev Lead | `Known_Issues.md` (KI-027) |
| **OQ-6.A** *(جديد)* | **`Accounting Policy Decision`** | S6.1 §5 — لا مسار استنفاد لـ2017 | **كيف يُسوَّى الالتزام في 2017؟** | كيان دفع عمولات مستقل بربط إلزامي بـ`CM-XXXX` | **Product Owner** | **ACC-E** |
| **OQ-6.B** *(جديد)* | **`Known Issue`** | S6.1 §6.1 — `tr.opening` خارج الأستاذ | قيد افتتاحي مقابل 3001 | إلزامي ⇒ AINV-33 | Dev Lead | `Known_Issues.md` (KI-028) |
| **OQ-6.C** *(جديد)* | **`Known Issue`** | S6.1 §6.2 — إهلاك بلا اقتناء | قيد اقتناء الأصل | إلزامي | Dev Lead | `Known_Issues.md` (KI-029) |
| **OQ-6.D** *(جديد)* | **`Accounting Policy Decision`** | S6.1 §6.3 — الرواتب في 5400 | فصل 5410/5420 والاستقطاعات | إلزامي لعرض EAS 1 | **Product Owner** | `Accounting.md` + KI-030 |
| **OQ-6.E** *(جديد)* | ✅ **`Closed by Evidence`** | S6.1 §7.2 — `DB.revaluations` عُثر عليه بالكامل | — | — | — | مُغلَق |
| **OQ-6.F** *(جديد)* | **`Architecture Decision`** | S6.1 §7.2 — نظاما إعادة تقييم متوازيان (revaluations ⟷ fxRevaluations) | **أيّهما يبقى؟** revaluations يخالف ADR-003 | إبقاء `fxRevaluations` (متوافق ADR-003) وإلغاء `revaluations` | **Product Owner** | ADR جديد + KI-032 |
| **OQ-6.G** *(جديد)* | **`Architecture Decision`** | S4.6 DO-1 — فحوص `closeReadiness` موسومة `info` | لماذا لا تمنع؟ | ⇒ ACC-D | Solution Architect | ACC-D |
| **OQ-6.H** *(جديد)* | **`Accounting Policy Decision`** | S4.6 DO-6 — أساس `suggestedClaimsRate()` | كيف تُشتقّ نسبة المخصص؟ | يمسّ KI-013 | Product Owner | Accounting.md |

---

# 10. Impact on JS Catalog

## 10.1 تغييرات مقترحة على العدد

| # | التغيير | الأثر التراكمي |
|---|---|---|
| 1 | **JS-C3 ⇒ حالة استخدام لـJS-19** | Conditional 3 ⇒ **2** ⇒ الإجمالي 57 |
| 2 | **+JS-55 → JS-58** من `DB.treasuryTx` | Confirmed 54 ⇒ **58** ⇒ الإجمالي **61** |
| 3 | JS-C1 · JS-C2 يبقيان `Unverified candidate` في العدّ مؤقّتاً | بلا تغيير عددي |
| 4 | 🟡 **+JS-38b** (عكس التقييم) — بقاعدة §10.4 | **قد يرفع 61 ⇒ 62** — للمراجعة |
| 5 | 🟡 **+JS-59** (قيد إقفال 3099⇒3020 من DO-4) | **قد يرفع العدّ أكثر** — للمراجعة |

> **الإجمالي المقترح: 58 ⇒ 61** *(حدّ أدنى)* — `Proposed pending source-boundary validation`.
>
> البندان 4 و5 قد يرفعانه. **الرقم النهائي يُحسَم في جلسة 6.2 بعد مراجعة حدود المصدر. لم يُعدَّل أي كتالوج.**

## 10.2 تغييرات مقترحة على تصنيف Snap/Recalc

| JS | التصنيف الحالي | المقترح | الدليل |
|---|---|---|---|
| **JS-39** | `SNAP` | 🔴 **`RECALC`** | `fxRate()` حيّ لكلا الطرفين — لا سعر مُجمَّد على `treasuryTx` |
| **JS-16** | `SNAP` | 🟡 **`MIXED`** | `sp.fxRate` مُجمَّد ✅ لكن `invRate` يُشتقّ حيّاً من الفاتورة |

> **أثر مركّب:** RECALC+MIXED ترتفع من **20** إلى **22**؛ و`MIXED` لم تعد مستخدَمة في JS-01 وحده. **يتطلب تحديث §2.10 و§6.2 و§6.6 من مستند جلسة 5 وADR-020 §Context.**

## 10.3 فروع غير مُكتلَجة داخل مصادر قائمة

| المصدر | الفرع | الحالة |
|---|---|---|
| JS-41 | `prodCode && qty>0` ⇒ **1200 / 1120** (شراء صنف من العهدة) | فرع مستقل غير مذكور |
| JS-19 | `allocType='product'` ⇒ 5100 | قيمة رابعة غير مذكورة |
| JS-38 | العكس التلقائي ⇒ JS-38b | قيد ثانٍ بتاريخ مستقل |

---

## 10.4 قاعدة حدود المصدر — تطبيق متّسق (البند 6)

> **هذه القاعدة تحسم هل الرقم 61 صحيح.** لا يُعتمَد قبل تطبيقها.

**السؤال:** هل `DB.treasuryTx` **مصدر واحد بخمسة فروع**، أم **خمسة مصادر**؟

**القاعدة المستخرَجة من الكتالوج القائم** (لا مخترَعة): جلسة 5 عدّت المصادر بـ**الحدث الاقتصادي المتمايز الذي يُنتج تركيبة حسابات مختلفة**، لا بالكيان المخزِّن ولا بالدالة. الدليل من قرارات سابقة مثبَتة في الكتالوج:

| الحالة المرجعية | الكيان الواحد | كم مصدراً عدّته جلسة 5؟ | القاعدة المستخلَصة |
|---|---|---|---|
| **فاتورة البيع + COGS** | `DB.invoices` | **مصدر واحد (JS-01)** — الإيراد والتكلفة **قيد واحد لحدث واحد** (الشحن) | حدث اقتصادي واحد ⇒ مصدر واحد، ولو تعدّدت السطور |
| **فاتورة الشراء مربوطة/غير مربوطة بـGRN** | `DB.purchases` | **مصدر واحد (JS-08)** — نفس الحدث (ورود الفاتورة)، والفرع شرطي داخلي | تفرّع شرطي داخل نفس الحدث ⇒ مصدر واحد |
| **أمر التشغيل منتِج/غير منتِج** | `DB.workOrders` | **مصدر واحد** — `woProduces()` فرع داخل نفس الحدث | نفس القاعدة |
| **إعادة تقييم ربح/خسارة/إقفال** | `DB.fxRevaluations` | **ثلاثة مصادر (JS-34/35 · JS-36/37)** — التقييم حدث، والإقفال **حدث اقتصادي منفصل بتاريخ منفصل** | حدثان اقتصاديان مختلفان ⇒ مصدران |
| **حركات العهدة** | `DB.custodyTx` | **ثلاثة مصادر (issue · expense · settle)** — كل `kind` **حدث اقتصادي متمايز** بتركيبة حسابات مختلفة | `kind` متمايز ⇒ مصدر مستقل |

**التطبيق على `DB.treasuryTx`:**

الحالة المرجعية الحاسمة هي **`custodyTx`**: كيان واحد، وقد عدّت جلسة 5 كل `kind` فيه **مصدراً مستقلاً** لأن كلاً منها حدث اقتصادي متمايز. `treasuryTx` **مطابقة بنيوياً** — خمسة `kind` بأحداث اقتصادية متمايزة (إيداع رأس مال ≠ مسحوبات ≠ تحويل ≠ عمولة ≠ فائدة) وتركيبات حسابات مختلفة تماماً.

> ## ⇒ الحكم: **خمسة مصادر، لا مصدر واحد.**
>
> بالقياس على `custodyTx` المثبَت في الكتالوج. عدّها مصدراً واحداً **يناقض** كيفية عدّ `custodyTx`.

**لكن هذا يكشف تناقضاً يجب رفعه للمراجعة:**

| الملاحظة | الأثر على العدّ |
|---|---|
| `transfer` (JS-39) **مُكتلَج بالفعل** كمصدر مستقل | ✅ متّسق مع القاعدة |
| الأربعة الباقية (deposit/drawing/bankfee/interest) **غير مُكتلَجة** | ⇒ **+4 مصادر** (JS-55→58) — متّسق |
| **العكس التلقائي لـJS-38** حدث بتاريخ منفصل | بالقاعدة نفسها (كإقفال FX) ⇒ **+1** (JS-38b) — **لم يُحتسَب في الـ61** |

> ### ⚠️ أثر على الرقم المقترح
>
> تطبيق القاعدة **باتّساق تام** قد يرفع الرقم فوق 61 (بإضافة JS-38b، وربما مراجعة فروع custody/treasury أخرى). **لذلك:**
>
> **الرقم 61 يبقى `Proposed pending source-boundary validation` — والمراجعة النهائية لحدود المصدر تُجرى في جلسة 6.2 قبل أي تعديل دائم على الكتالوج.**

---

# 11. Impact on AINV / KI / ADR

## 11.1 الثوابت — لا ثابت جديد

| AINV | الأثر |
|---|---|
| `AINV-10` · `AINV-11` | خرقان جديدان مؤكَّدان: JS-39 · JS-16 غير المربوط |
| `AINV-33` | **خرق جديد خطير** — `tr.opening` خارج الأستاذ (§6.1) |
| `AINV-21` · `AINV-24` | **يتقوّيان** — 2017 حساب مراقبة متراكم أبدياً كـ2105 |
| `AINV-26` | خرق إضافي — `supplierPayments` بلا قفل |
| `AINV-04` | خرق إضافي — لا منع ازدواج في `supplierPayments` ولا `creditNotes` |

**⇒ العدد يبقى 34. لا `AINV-35`.** كل ما وُجد يندرج تحت ثوابت قائمة.

## 11.2 مشاكل معروفة مقترحة (لا تُنشأ الآن)

| المقترح | العنوان | Severity | Target |
|---|---|---|---|
| **KI-025** | Supplier payment without invoice link recognizes no FX difference | 🔴 High | AINV-11 · ACC-B |
| **KI-026** | Supplier payment path has no period lock, no duplicate guard, no ceiling | 🔴 High | AINV-26 · AINV-04 · ACC-D |
| **KI-027** | Credit-note delete/edit bypasses period lock; no duplicate guard | 🟡 Medium | AINV-30 · ACC-C |
| **KI-028** | Treasury opening balance never enters the general ledger | 🔴 **Critical** | AINV-33 · ACC-A |
| **KI-029** | Fixed assets are depreciated without any acquisition entry | 🔴 **Critical** | AINV-05 · ACC-A |
| **KI-030** | Payroll posts to 5400; accounts 5410/5420/2120/2130/2140 never used | 🔴 **Critical** | EAS 1 · ACC-B |
| **KI-031** | Two parallel recurring engines (`DB.recurring`/`DB.recurringExp`); `editRecurring` shadowed | 🟡 Medium | نمط KI-005 |
| **KI-032** | Two parallel FX-revaluation engines; `DB.revaluations` bypasses control 1265 (violates ADR-003) | 🔴 **Critical** | ADR-003 · AINV-10 · OQ-6.F |
| **KI-033** | `doClosing()` is a third uncentralized period-lock path; closing entry 3099⇒3020 uncatalogued | 🟡 Medium | AINV-26 · ACC-D |

**تحديث مقترح على `KI-017`:** توسعة النطاق من «لا حارس تداخل» إلى «**لا مسار استنفاد إطلاقاً لـ2017**».

## 11.3 ADRs

| ADR | الأثر |
|---|---|
| **ADR-020 (ACC-A)** | ✅ **يتقوّى** — KI-028 وKI-029 قرينتان جديدتان على أن الدفتر المُشتقّ يفقد أحداثاً كاملة. **لا تغيير على الحالة `Proposed`.** |
| **ACC-B** | نطاق أوسع: JS-39 وJS-16 يُضافان لقائمة القيم التي تحتاج تجميداً |
| **ACC-C** | نطاق أوسع: `delCreditNote` و`delSupplierPayment` |
| **ACC-D** | نطاق أوسع: مسار `supplierPayments` بلا قفل |
| **ACC-E** | 🔴 **`Still Blocked`** — فجوة الدليل مُغلَقة · قرار التصميم مفتوح (OQ-6.A) |

---

# 12. Session 6.2 Readiness

## الجاهزية المنفصلة — ستة بنود (البند 9)

| البند | الحالة | المبرِّر |
|---|---|---|
| **Test Cases** | ✅ **Ready** | الأدلة كافية لكتابة Given/When/Then لكل AINV-01→34 |
| **ACC-B** (Snapshot & Valuation Inputs) | ✅ **Ready — with catalog-count correction pending** | الخرائط مكتملة؛ لكن إعادة تصنيف JS-16 ⇒ MIXED · JS-39 ⇒ RECALC · JS-38 ⇒ SNAP معلَّقة على مراجعة العدّ (§10) |
| **ACC-C** (Reversal Instead of Delete) | ✅ **Ready** | أدلة الحذف المدمِّر مكتملة عبر supplierPayments · creditNotes · revaluations |
| **ACC-D** (Centralized Period Lock) | ✅ **Ready** | مسارات القفل المتعددة موثَّقة — أُضيف DO-4 (`doClosing`) وDO-1 كأدلة داعمة |
| **Posting Matrix** | ✅ **Ready — JS-38 مفكوك الحجب** | **تغيّر عن المتوقَّع:** `DB.revaluations` عُثر عليه (§7.2). كل مصادر الترحيل معروفة الآن |
| **ACC-E** (Control Account Settlement) | 🔴 **Blocked — OQ-6.A Product Owner decision** | لا مسار استنفاد لـ2017؛ يحتاج قرار تصميم لا توثيق سلوك قائم |

> ### مقارنة بالحالة المتوقَّعة في التعليمات
>
> | البند | المتوقَّع | الفعلي | السبب |
> |---|---|---|---|
> | Test Cases | Ready | ✅ Ready | — |
> | ACC-B | Ready w/ count correction | ✅ مطابق | — |
> | ACC-C | Ready | ✅ Ready | — |
> | ACC-D | Ready | ✅ Ready | — |
> | **Posting Matrix** | **Blocked by JS-38** | ✅ **Ready** | **دليل جديد:** `DB.revaluations` عُثر عليه — التعليمات تسمح بالتغيير «unless new evidence changes it» |
> | ACC-E | Blocked by OQ-6.A | 🔴 مطابق | — |

## بند واحد متبقٍّ خارج الجاهزية الست

| البند | الحالة |
|---|---|
| **حدود المصدر / العدّ النهائي** | 🟡 **مراجعة مطلوبة في 6.2** — الرقم 61 حدّ أدنى مقترح (§10.4) |
| **OQ-6.F** (أي نظامَي إعادة التقييم يبقى) | 🔴 قرار Product Owner — لا يحجب Test Cases ولا ACC-B/C/D |

---

# 13. Validation

| الفحص | النتيجة |
|---|---|
| الأهداف الستة عولجت | ✅ **6/6 completed · no blocked target** — الهدف 6 اكتمل بعد تسليم `Accounting_Session4_6_Findings.md` |
| **لم يُعدَّل أي ملف دائم** | ✅ **مؤكَّد** |
| لم يُعَد فتح أي استنتاج مُغلَق | ✅ — C-01→C-08 لم تُمسّ |
| لم يُستقصَ خارج النطاق | ✅ |
| كل استنتاج له دالة وشرط ومجموعة وأكواد حسابات | ✅ عدا ما وُسم `Not found` |
| **«not implemented» لم تُستخدَم مطلقاً** | ✅ — `Not found after scoped search` حصراً |
| توصية المراجع لم تُقدَّم كقرار | ✅ — عمود منفصل موسوم |
| ACC-E لم يُفكّ حجبه بحجة القيد اليدوي | ✅ — الإجابة 8 صريحة |
| ADR-020 لم تتغيّر حالته | ✅ `Proposed` |
| العدّ المقترح لم يُطبَّق | ✅ مقترح فقط |

---

# الملخص النهائي

| Item | Final result | Documentation impact | Blocks |
|---|---|---|---|
| **OQ-4.G** | `Known Issue` | KI-025 · KI-026 مقترحان · JS-16 ⇒ `MIXED` | ❌ لا شيء |
| **OQ-4.J** | `Known Issue` | KI-027 مقترح · خريطة CN كاملة | ❌ **مرفوع** |
| **الحساب 2017** | `No dedicated settlement or consumption path found after scoped search` | OQ-6.A جديد · تحديث KI-017 | 🔴 **ACC-E فقط** |
| **JS-C1** | `Unverified candidate — likely future requirement` | KI-028 مقترح · العدّ مؤقّت | ❌ لا شيء |
| **JS-C2** | `Unverified candidate — likely future requirement` | KI-029 مقترح · العدّ مؤقّت | ❌ لا شيء |
| **JS-C3** | `Specialized use case of JS-19` (الكاتب مفحوص) | Conditional 3⇒2 · KI-030/031 مقترحان | ❌ لا شيء |
| **JS-19** | ✅ محلول | 🔴 **تصحيح: `allocType` لا `cat`** | ❌ لا شيء |
| **JS-38** | ✅ **مُحلّ — `DB.revaluations` عُثر عليه** | 🔴 نظاما تقييم متوازيان · KI-032 · JS-38b | ❌ **Posting Matrix مرفوع** |
| **JS-39** | ✅ محلول | 🔴 إعادة تصنيف `SNAP` ⇒ `RECALC` | ❌ لا شيء |
| **JS-41** | ✅ محلول | قاعدة ثلاثية + فرع رابع · لا VAT | ❌ لا شيء |
| **DO-1 (4.6)** | Converted to OQ ⇒ OQ-6.G | ACC-D | ❌ لا شيء |
| **DO-2 (4.6)** | Still deferred | — | ❌ لا شيء |
| **DO-3 (4.6)** | Duplicate of KI-015 | — | ❌ لا شيء |
| **DO-4 (4.6)** | Converted to KI ⇒ KI-033 + JS-59 | العدّ · ACC-D | ❌ لا شيء |
| **DO-5 (4.6)** | Duplicate of KI-016 | — | ❌ لا شيء |
| **DO-6 (4.6)** | Converted to OQ ⇒ OQ-6.H | Accounting.md | ❌ لا شيء |

## البيانات الأربعة المطلوبة

| السؤال | الإجابة |
|---|---|
| **هل ACC-E ما زال محجوباً؟** | 🔴 **`Still Blocked — evidence gap resolved, design decision pending`.** فجوة الدليل مُغلَقة (`No dedicated settlement path found`، `Confirmed by multiple code paths`). معيار فكّ الحجب — `documented control workflow` — غير مستوفٍ. **لا يُستخدَم `Partially Unblocked` كحالة رسمية.** القرار لـ Product Owner (OQ-6.A). |
| **هل يبقى عدد المصادر 58؟** | ❌ **لا.** المقترح **61** كحدّ أدنى — `Proposed pending source-boundary validation` (§10.4). قد يرتفع بـ JS-38b و JS-59. **لم يُطبَّق.** |
| **هل Implementation Guide مفكوك الحجب؟** | ✅ **نعم — بالكامل الآن.** OQ-4.J مرفوع · **و JS-38 مُحلّ** ⇒ **Posting Matrix لم يعد محجوباً**. البند الوحيد المتبقّي (`ACC-E`) قرار سياسة، لا فجوة خريطة. |
| **هل يجوز بدء جلسة 6.2؟** | ✅ **نعم.** Test Cases · ACC-B (بتصحيح العدّ) · ACC-C · ACC-D · Posting Matrix: **كلها Ready**. الوحيد غير الجاهز: **ACC-E** (قرار Product Owner) — ولا يحجب البقية. |

---

**لم يُمسّ الـPrototype. لم يُنشَأ ولا يُعدَّل أي ADR.** No permanent repository file was modified. The outputs are this revised report and Correction_Log_S6_1.md.

---

# Post-Session Independent Verification Corrections

> **Status:** Superseding evidence note for the delivered 24,149-line `prototype_v2.html`. The original Session 6.1 text remains as investigation history; the statements below override the listed absence claims for current-prototype use.
>
> **Source:** `Accounting_Independent_Verification_Report.md` — clean-room Phase A prototype-only verification, followed by Phase B documentation comparison.

| Earlier Session 6.1 statement | Current verified evidence | Governance effect |
|---|---|---|
| Supplier-payment create/update had no observed `isLocked()` | `editSuppPay` checks `isLocked(date)` | KI-026 narrowed; AINV-26 still fails globally through other writers |
| Supplier-payment path had no observed `audit()` | create/update call `audit()`; delete remains unaudited | KI-026 narrowed to dedup/ceiling/delete controls |
| No dedicated payroll posting path; payroll accounts unused | `postPayroll()` is button-wired and posts 5410/5420/2120/2130/2140 | KI-030 rewritten around two paths and double-count risk |
| `DB.recurring` edit opens `DB.recurringExp` form | later `DB.recurring` editor wins; the older `recurringExp` editor is dead code | KI-031 direction corrected |
| `doClosing()` treated as an unguarded/third path | it has permission, audit, and repeat guard; it is one of several paths | KI-033 rewritten; `doMonthEndClose()` lock-only path highlighted |

**Unchanged findings:** KI-025 FX behaviour · two FX-revaluation engines and double-recognition mechanism · account 2017 evidence · ACC-E blocker · source counts resolved later in Session 6.2.

**Final synchronized verdict:** `PASS WITH CORRECTIONS + EXPLICIT MATRIX LIMITATION`.


---

# 12. Post-Runtime Verification Corrections (2026-07-25)

> **لا يُعاد كتابة التقرير التاريخي.** أقسام §3.4 · §6.1 · §6.2 · §7.2 تبقى كما هي **كتاريخ تحقيق**. هذا القسم يسجّل فقط أن ثلاثة استنتاجات سابقة **عُلِيَت (superseded)** بأدلة تشغيل حيّة.
>
> **Source of runtime evidence:** `Prototype_Runtime_Verification_Report.md` — تشغيل headless (jsdom) على نسخة runtime مطابقة للبايت (SHA-256 `8996dd68…d7c6b2`)؛ لم يُعدَّل أي كود أو توثيق.

## 12.1 العبارات المُعلاة (Superseded statements)

| البيان السابق (تاريخي) | القسم الأصلي | الدليل التشغيلي الحاليّ | الأثر الحوكمي |
|---|---|---|---|
| «الأرصدة الافتتاحية غائبة عن الأستاذ» — الرصيد الافتتاحي للخزينة لا قيد له مقابل 3001 | §6.1 (JS-C1) | قيد `OPENING` يدين الخزينة ويقيّد **3001** = 2,403,839.5؛ الأستاذ والخزينة متطابقان (OB-01)؛ المخزون الافتتاحي مدين 1200 (OB-02) | **KI-028 مدحوض/مغلق**؛ AINV-33 يبقى مفتوحاً عموماً لبقية الدفاتر المساعدة |
| «قيد اقتناء الأصل الثابت غائب» — `Not found after scoped search` | §6.2 (JS-C2) | `buildJournalCore` (2921–2928) يشتقّ لكل أصل `!opening` قيد **مدين 1300 / دائن البنك** (FA-01) | **KI-029 ضُيِّق إلى Low**: «تمويل الاقتناء يفترض البنك دائماً بلا اختيار ذمة دائنة»؛ **لا يُوصَف الاقتناء بأنه غائب** |
| «ازدواج الاعتراف بفرق العملة سلوك تشغيلي حيّ» — الفرق يُثبَت مرتين فعلياً | §7.2 (JS-38) | المحرك B خامل (`fxExposure()`=صفر حتى بعد تحرُّك 9%)؛ الازدواج المتزامن **لم يُعَد إنتاجه end-to-end** (FX-02/FX-03) | **KI-032 أُعيد تصنيفه إلى Medium**: ازدواج **latent code risk** لا سلوك مؤكَّد؛ المحرك B خامل؛ **OQ-6.F يبقى 🔴 Open** |

## 12.2 حدّ منهجي صريح (Explicit matrix limitation)

- سبب الفارق: §6.1/§6.2 اعتمدتا **بحثاً ثابتاً مُقيَّداً** (`scoped search`) لم يُنفِّذ قيد `OPENING` ولا اشتقاق `buildJournalCore` للأصول. التشغيل الحيّ يعلو البحث الثابت للسيناريوهات المُنفَّذة.
- مطابقة الأستاذ ↔ الدفاتر المساعدة أُثبتت **فقط لبيانات seed/runtime المختبَرة** (خزينة + مخزون افتتاحي)؛ ليست إثباتاً شاملاً لثابت AINV-33 عبر كل الدفاتر المساعدة.
- سطور قيد المحرك B (1265/3020) **NOT VERIFIABLE** end-to-end لأن حساب التعرُّض يعيد صفراً؛ المنطق موجود في الكود، غير قابل للوصول عبر بيانات المحرك.

## 12.3 ما يبقى دون تغيير

- **KI-025** (سلوك FX لسند بلا ربط) · **ACC-E** (محجوب) · **OQ-6.A** (تسوية 2017) · **OQ-6.F** (توحيد محركَي FX — 🔴 Open) — **بلا تغيير**.
- **إعادة تصنيف مصادر الكتالوج (JS-C1/JS-C2):** انتقلا من `Conditional / Unverified candidate` إلى **`Confirmed posting source`** بأدلة التشغيل. الكتالوج المنقَّح المعتمَد: **62 confirmed · 0 Conditional · 1 Future · 63 total** (`Accounting_Implementation_Guide.md` §0). **الإجمالي 63 ثابت.**

## 12.4 إعادة تصنيف مصادر الكتالوج — Confirmed (supersedes §6.1/§6.2/§10 labels)

> **العبارات التاريخية `Unverified candidate — likely future requirement` لـ JS-C1/JS-C2 في §6.1 · §6.2 · §10 مُعلاة (superseded).** تُحفَظ كتاريخ تحقيق فقط. **التصنيف المعتمَد الآن:**

| المصدر | التصنيف السابق (تاريخي) | التصنيف المعتمَد الآن | القاعدة/الدليل |
|---|---|---|---|
| **JS-C1** | `Conditional / Unverified candidate` | **`Confirmed posting source — Opening Journal`** | قيد `OPENING` حيّ ⇒ 3001 (OB-01). **الخزينة والمخزون الافتتاحي فرعان/سطران من نفس الحدث الاقتصادي `OPENING`** ⇒ مصدر واحد؛ **لا JS-C1b مُحتسَب** (لم يُثبَت أنهما حدثان مستقلان بموجب قاعدة حدود المصدر §10.4) |
| **JS-C2** | `Conditional / Unverified candidate` | **`Confirmed posting source — Fixed Asset Acquisition`** | قيد اقتناء حيّ Dr 1300 / Cr bank للأصول `!opening` (FA-01) |

**أثر العدّ:** Confirmed 60 → **62** · Conditional 2 → **0** · Future **1** · **Total 63 ثابت**. **لم يُضاعَف أي مصدر** (فرع المخزون الافتتاحي سطر داخل JS-C1، لا مصدر منفصل).

**حدّ التغطية:** الاعتراف بالترحيل مؤكَّد تشغيلياً؛ لم تُستكمَل الأعمدة الـ18 للمصفوفة لكل المصادر ⇒ **Partial posting-matrix coverage**.

**Final synchronized verdict (post-runtime):** `PASS WITH RUNTIME CORRECTIONS + EXPLICIT MATRIX LIMITATION`.
