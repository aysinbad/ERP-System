# CRM Implementation Guide

## Document Information
```
Document Name:  CRM Implementation Guide
Version:        1.1.0
Status:         In Review
Classification: Source of Truth
Owner:          Solution Architecture Team
Approved-by:    —
Approved-date:  —
Last-Updated:   2026-07-18
Supersedes:     أقسام من CRM.md v1.2.0 (State Matrix, Permissions, Notifications, Audit,
                Domain Events, Error Codes) — نُقلت هنا بلا تغيير في المحتوى.
```

> **العلاقة بـ `CRM.md`:** `CRM.md` يحدد "ماذا يجب أن يحدث" (منطق الأعمال). هذا الملف يحدد "إزاي يتربط بالنظام تقنياً" — حالات الانتقال، الصلاحيات التنفيذية، الإشعارات، التدقيق، الأحداث، وأكواد الخطأ. **لا تعديل مستقل في أحدهما بمعزل عن الآخر** — أي تغيير في Business Rules هنا يجب أن ينعكس في `CRM.md` والعكس.
> **لا يُعتمَد (`Approved`) قبل حسم INV-09** (State Transition Matrix أدناه) — بانتظار اعتماد ADR-015.

---

## State Transition Matrix

### Lead — الحالة الفعلية في الكود (غير مقيَّدة)

| من \ إلى | new | contacted | qualified | unqualified | converted |
|---|---|---|---|---|---|
| **new** | — | ✅ | ✅ | ✅ | ✅ (عبر تحويل منفصل) |
| **contacted** | ✅ | — | ✅ | ✅ | ✅ |
| **qualified** | ✅ | ✅ | — | ✅ | ✅ |
| **unqualified** | ✅ | ✅ | ✅ | — | ❌ (لا مسار تحويل من هنا في الكود) |
| **converted** | ❌ | ❌ | ❌ | ❌ | — |

> القيد الوحيد الفعلي: `converted` نهائية تماماً. باقي الانتقالات حرة عبر قائمة منسدلة بلا أي تحقق من التسلسل المنطقي (مثال: يمكن الرجوع من `qualified` إلى `new`).

### Opportunity — الحالة الفعلية في الكود (غير مقيَّدة)

| من \ إلى | qualification | proposal | negotiation | won | lost |
|---|---|---|---|---|---|
| **qualification** | — | ✅ | ✅ | ✅ | ✅ |
| **proposal** | ✅ | — | ✅ | ✅ | ✅ |
| **negotiation** | ✅ | ✅ | — | ✅ | ✅ |
| **won** | ✅ | ✅ | ✅ | — | ✅ |
| **lost** | ✅ | ✅ | ✅ | ✅ | — |

> عبر السحب والإفلات (Kanban)، أي عمود قابل للإفلات من أي عمود آخر بلا استثناء — **بما في ذلك الرجوع من `won` أو `lost`**. لا يوجد أي تنبيه أو تأكيد إضافي عند هذه الانتقالات الحساسة حالياً.

**🔴 قرار مطلوب من الفريق قبل الاعتماد:** هل هذا السلوك (حرية كاملة في الانتقال) مقصود فعلاً (مرونة تشغيلية)، أم يجب تقييده في الإصدار القادم؟ **موثَّق كقرار مقترح في `docs/02_Governance/ADR/ADR-015.md` (حالته `Proposed`)** — يقيّد فقط الخروج من `won`/`lost` مع صلاحية تجاوز مبرَّرة للمدير، ويبقي التقدّم بين المراحل المفتوحة حراً.

---

## Permissions

- **تعريف "المدير":** `role==='owner'` أو أي دور له `all:true` في مصفوفة الأدوار — تعريف ديناميكي وليس اسم دور ثابت.
- **فلترة كاملة** (Leads, Pipeline, Tasks, Activities, لوحة CRM) للمندوب غير المدير: يرى فقط `owner===نفسه` (أو `by===نفسه` للأنشطة).
- **استثناء شاشة العملاء:** المندوب يرى كل الأسماء (لمنع التكرار)، لكن التفاصيل مقفلة لغير المملوك له.
- **إعادة التوزيع:** حصرية للمدير، وتُحدِّث `owner` على العميل مباشرة (مصدر الحقيقة الأساسي بعد أي إسناد يدوي).
- ⚠️ **حذف Lead (`delLead`): بلا أي تحقق صلاحية في الكود حالياً** — أي مستخدم شايف الـ Lead (حسب فلترة الملكية) يقدر يحذفه نهائياً، بما في ذلك Lead بحالة `converted`. لا يوجد تمييز "المدير فقط" هنا كما هو الحال في `crmReassign`. **فجوة موثَّقة — انظر `Known_Issues.md` KI-007.**
- ⚠️ **كل ما سبق تحقق على الواجهة (Client-side) فقط حالياً** — انظر `docs/02_Governance/Known_Issues.md` KI-002. لازم يُفرض على مستوى الـ API عند البناء، لا يُعتمد على إخفاء العناصر كحماية فعلية.

## Notifications

- **الحالة الحالية:** تنبيهات داخل التطبيق فقط (In-app) — لا بريد إلكتروني ولا واتساب.
- الأمثلة الموجودة: مهمة متأخرة/قريبة الاستحقاق، عميل محتمل "واعد" بدأ يبرد (>14 يوم بلا نشاط)، فرصة تجاوزت موعد الإغلاق المتوقع، عميل بمخاطر فقد مرتفعة.
- **فجوة موثّقة مسبقاً:** "Notification Center (Email/WhatsApp)" مذكورة كأولوية "مهمة" في `docs/00_Project/Current_State.md` — تحتاج قرار تصميم منفصل، خارج نطاق هذا الملف.
- ربط عملي بقسم **Domain Events** أدناه: أي قناة إشعار جديدة (بريد/واتساب/n8n) تُبنى فوق نفس الأحداث الموثّقة هناك، لا منطق منفصل.

## Audit Rules

يُسجَّل حالياً (عبر `audit()`):
- إنشاء/تعديل عميل (قبل/بعد التعديل كاملاً)
- تحويل Lead إلى عميل (مع اسم الـ Lead الأصلي)
- إعادة توزيع ملكية عميل (المالك القديم/الجديد)
- كل نشاط CRM مسجَّل (`crmLog`) يُدرَج في سجل التدقيق العام

⚠️ **لا يُسجَّل حالياً:** عمليات حذف الـ Lead (`delLead`) — لا يوجد أي أثر تدقيق لمن حذف، متى، أو هل كان الـ Lead محوَّلاً. هذا يفاقم خطر KI-007 (لا صلاحية) وخطر التكامل الموثَّق في `CRM.md` Business Rules #11 — لو الحذف حصل، مفيش حتى دليل لاحق يوضّح إيه اللي حصل.

⚠️ **لا يُسجَّل حالياً:** تغييرات مراحل الفرصة (`moveOppStage`)، ولا حذف الأنشطة/المهام بتفاصيل كافية للمراجعة. **مرتبط مباشرة بـ ADR-015** — لو اعتُمد تقييد الخروج من `won`/`lost`، تسجيل التجاوز في الـ Audit يصبح إلزامياً كجزء من نفس القرار، لا اختيارياً.
سجل التدقيق نفسه قابل للتلاعب حالياً كملف Prototype (`docs/02_Governance/Known_Issues.md` KI-004) — لا يُعتمَد عليه كدليل رقابي قبل بناء نسخة الـ Backend غير القابلة للتعديل.

---

## Domain Events (لاستخدام الـ API وn8n والإشعارات)

> كل حدث هنا هو **حقيقة تجارية وقعت فعلاً**، لا مجرد استدعاء دالة. الـ Backend الجديد، وأي أتمتة عبر n8n، وأي قناة إشعار مستقبلية، تُبنى فوق هذه الأحداث — لا تُعاد استنتاجها من تغييرات الحقول مباشرة.

| الحدث (Event) | يُطلَق عند | الحمولة الأساسية (Payload) | المستهلكون المحتملون |
|---|---|---|---|
| `crm.lead.created` | إنشاء Lead جديد | `leadId, owner, source, estValue, cur` | تقارير، لوحة التحكم |
| `crm.lead.stage_changed` | تغيّر `stage` لأي Lead | `leadId, fromStage, toStage, owner` | تقارير، (لاحقاً) تنبيه عند دخول `unqualified` |
| `crm.lead.converted` | نجاح `convertLeadToCustomer` | `leadId, customerCode, owner` | إشعار المدير، مزامنة CRM↔Customers |
| `crm.opportunity.created` | إنشاء فرصة (مباشرة أو من Lead) | `oppId, custCode, owner, value, cur, stage` | تقارير المسار |
| `crm.opportunity.stage_changed` | أي تغيّر مرحلة (بما فيها عبر Kanban) | `oppId, fromStage, toStage, owner` | Audit (إلزامي عند اعتماد ADR-015)، n8n لأتمتة تنبيهات |
| `crm.opportunity.won` | الوصول لمرحلة `won` تحديداً | `oppId, custCode, value, cur, owner` | إنشاء المهمة التلقائية، إشعار الإدارة، محرك التوقّع |
| `crm.opportunity.lost` | الوصول لمرحلة `lost` | `oppId, custCode, value, cur, lostReason, owner` | تقارير معدّل الفوز |
| `crm.opportunity.converted_to_proforma` | نجاح `crmOppToProforma` وربط `proformaId` | `oppId, proformaId, custCode` | وحدة المبيعات (تفعيل السلسلة التالية) |
| `crm.customer.duplicate_blocked` | رفض حفظ بسبب INV-01 | `attemptedName, country, existingCode, existingOwner, blockedBy` | مراقبة (KPI: محاولات تضارب)، لا يُخزَّن كعميل |
| `crm.customer.ownership_reassigned` | نجاح `crmReassign` | `customerCode, fromOwner, toOwner, reassignedBy` | Audit، إشعار المندوب الجديد/القديم |
| `crm.activity.logged` | أي `crmLog()` ناجح | `activityId, refType, refId, type, by` | تحديث `lastActivity`، تقارير النشاط |
| `crm.task.created` | إنشاء مهمة (يدوي أو تلقائي عند `won`) | `taskId, refType, refId, due, priority, owner` | إشعارات المواعيد |
| `crm.task.completed` | إتمام مهمة | `taskId, owner` | تقارير الإنتاجية |
| `crm.lead.deleted` | نجاح `delLead` | `leadId, deletedBy, wasConverted` | Audit (إلزامي — انظر KI-007)، تنبيه لو `wasConverted=true` (خطر تكامل) |
| `crm.customer.health_band_changed` *(مقترح — غير موجود كحدث فعلي في الكود حالياً، محسوب فقط عند الطلب)* | عبور نطاق الصحة (مثال: `good → at-risk`) | `customerCode, fromBand, toBand, score` | **يحتاج Job مجدوَل عند البناء (الكود حالياً يحسبها فقط عند فتح الشاشة، لا يوجد رصد تغيّر عبر الزمن)** |
| `crm.customer.churn_risk_escalated` *(مقترح، نفس ملاحظة أعلاه)* | عبور `churnRisk.level` إلى `high` | `customerCode, risk, level` | إشعار فوري للمندوب/المدير |

> ⚠️ آخر بندين (`health_band_changed`, `churn_risk_escalated`) **غير موجودين كأحداث فعلية في الـ Prototype** — الحساب حالياً "عند الطلب" فقط (وقت فتح الشاشة)، مش مرصود كتغيّر عبر الزمن. أضفناهم هنا كمقترح ضروري لأي نظام إشعارات حقيقي (n8n مثلاً)، لكن لازم يُعتمدوا صراحة كقرار جديد، مش يُفهموا كسلوك موجود بالفعل.

---

## Standard Error Codes (بدل نصوص الرسائل)

> النصوص الحالية في الكود (زي "محجوز لـ..." أو "تم تحويله مسبقاً") مكتوبة بالعربي/الإنجليزي داخل منطق العرض مباشرة، بلا كود موحّد. الجدول ده يقترح الأكواد المعتمدة للـ API القادم (بنفس اصطلاح `docs/01_Standards/API_Standards.md`)، مع إبقاء النص الحالي كـ "الرسالة الافتراضية" فقط.

| الكود | الوصف | الموقف المكافئ في Prototype | HTTP المقترح |
|---|---|---|---|
| `CRM_CUSTOMER_DUPLICATE_OWNED` | عميل بنفس الاسم+الدولة محجوز لمندوب آخر | `editCustomer` → "محجوز لـ..." | 409 Conflict |
| `CRM_CUSTOMER_DUPLICATE_UNOWNED` | تحذير تكرار غير حاسم (بلا مالك/نفس المالك) | `editCustomer` → "يوجد عميل بنفس الاسم" | 200 OK + warning flag (ليس خطأ) |
| `CRM_LEAD_ALREADY_CONVERTED` | محاولة تحويل Lead محوّل مسبقاً | `convertLeadToCustomer` → "تم تحويله مسبقاً" | 409 Conflict |
| `CRM_CUSTOMER_NOT_OWNED` | مندوب يحاول تعديل عميل غير مملوك له | `editCustomer` (حارس `customerSaveGuard`) | 403 Forbidden |
| `CRM_REASSIGN_MANAGER_ONLY` | محاولة إعادة توزيع من غير مدير | `crmReassign` → "للمدير فقط" | 403 Forbidden |
| `CRM_LEAD_NAME_REQUIRED` | حفظ Lead بدون اسم | `editLead` → "أدخل الاسم" | 422 Unprocessable |
| `CRM_CUSTOMER_NAME_REQUIRED` | حفظ عميل بدون اسم | `editCustomer` → "أدخل اسم العميل" | 422 Unprocessable |
| `CRM_ENTITY_NOT_FOUND` | Lead/Opportunity غير موجود عند الفتح/التحديث | فحوصات `if(!o)return` / `if(!l)return` المتكررة | 404 Not Found |
| `CRM_PROFORMA_PRODUCT_UNMATCHED` | تحويل فرصة لبروفورما بدون تطابق كود منتج | `crmOppToProforma` → تنبيه "اختر المنتج يدوياً" | 200 OK + warning flag (لا يمنع الحفظ) |
| `CRM_LEAD_DELETE_CONVERTED_BLOCKED` *(مقترح، مشروط بقرار — انظر Open Questions #5 في `CRM.md`)* | محاولة حذف Lead بحالة `converted` | لا يوجد مكافئ حالياً (الحذف مسموح بلا قيد — KI-007) | 409 Conflict |
| `CRM_OPPORTUNITY_STAGE_TRANSITION_BLOCKED` *(مقترح، مشروط باعتماد ADR-015)* | انتقال مرحلة غير مسموح به بعد تفعيل التقييد | لا يوجد مكافئ حالياً (كل الانتقالات مسموحة) | 409 Conflict |

---

## Open Questions (خاصة بالدمج التقني)

1. هل نطاق التدقيق (Audit Rules أعلاه) يحتاج توسيع ليشمل `crm.opportunity.stage_changed` بشكل عام، أم فقط عند تجاوز المدير بعد ADR-015؟
2. هل نعتمد حدثي `health_band_changed` و`churn_risk_escalated` المقترحين (يحتاجان Job مجدول جديد لا يوجد له مكافئ في الـ Prototype)؟

---

## Cross-references
- `docs/03_Business_Logic/CRM.md`: منطق الأعمال الأساسي (المرجع الأعلى لأي تعارض)
- `docs/02_Governance/ADR/ADR-015.md`: قرار تقييد الانتقالات (Proposed)
- `docs/01_Standards/API_Standards.md`: اصطلاح أكواد الخطأ العام ومسارات الـ API
- `docs/02_Governance/Known_Issues.md`: KI-002 (صلاحيات واجهة فقط)، KI-004 (سجل تدقيق قابل للتلاعب)
