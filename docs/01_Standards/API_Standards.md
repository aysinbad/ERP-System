# API Standards (معيار الـ API)

## Document Information
```
Document Name:  API Standards
Version:        0.1.0
Status:         Proposed
Classification: Standard
Owner:          Solution Architecture Team
Approved-by:    —
Approved-date:  —
Last-Updated:   2026-07-18
```

> **حالة الملف:** `Proposed` وليس `Approved` — هذه اتفاقيات عامة (Conventions) لم يُقرَّها الفريق بعد كـ ADR رسمي. الأجزاء المبنية على قرار مُعتمَد بالفعل (زي JWT، FluentValidation) موسومة صراحة؛ الباقي مقترح افتراضي قياسي يحتاج تأكيد الفريق قبل التنفيذ الفعلي، تماشياً مع مبدأ "لا كود قبل قرار" (`AGENTS.md`).
> هذا المعيار **لا يوصف API فعلي موجود** — لا يوجد Backend مبني بعد. الهدف: يبقى جاهز فور بدء البناء، بدل ما يُرتجَل وقتها.

---

## 1. URL Convention

```
https://api.vortex.example/api/v{version}/{module}/{resource}
```

| العنصر | القاعدة | مثال |
|---|---|---|
| الموديول | جمع، lowercase، إنجليزي | `/crm/`, `/sales/`, `/inventory/` |
| المورد (Resource) | جمع دائماً، حتى لو مورد واحد | `/crm/leads`, `/crm/opportunities` |
| المعرف (ID) | جزء من المسار، لا Query String | `/crm/leads/{id}` لا `/crm/leads?id=` |
| الإجراءات غير-CRUD (Actions) | فعل بعد المعرف | `/crm/opportunities/{id}/convert-to-proforma` |

**مطابقة أسماء الموديولات لـ `docs/03_Business_Logic/`:** `crm`, `sales`, `accounting`, `inventory`, `pricing`, `hr` — نفس التسمية بالظبط لتجنّب أي التباس بين التوثيق والكود.

## 2. Versioning
- الإصدار في المسار (`/api/v1/...`) لا في الـ Header — أوضح للتتبع في اللوجز وأسهل للاختبار اليدوي.
- MAJOR جديد فقط عند تغيير كاسر (breaking change) لعقد موجود. إضافة حقل اختياري جديد لا تستدعي إصدار جديد.

## 3. Standard Error Codes (توسعة عامة لما بدأناه في CRM.md)
عقد الاستجابة الموحّد لأي خطأ:
```json
{
  "error": {
    "code": "CRM_CUSTOMER_DUPLICATE_OWNED",
    "message": "نص افتراضي قابل للترجمة",
    "details": { }
  }
}
```
- **الاصطلاح:** `{MODULE_PREFIX}_{ERROR_NAME}` بالإنجليزية، أحرف كبيرة، شرطة سفلية. مثال: `CRM_*` (مُعرَّف بالفعل في `docs/03_Business_Logic/CRM/CRM.md`)، وبالمثل `SALES_*`, `ACC_*` (Accounting), `INV_*` (Inventory) عند استخراج كل موديول.
- **لا رسائل نصية خام في منطق التحقق** — الكود هو مصدر الحقيقة للتعامل البرمجي، النص للعرض فقط (قابل للترجمة عربي/إنجليزي، اتساقاً مع ثنائية اللغة في الـ Prototype).
- **خريطة عامة لأكواد HTTP:**

| الحالة | HTTP | مثال |
|---|---|---|
| قاعدة عمل مُنتهَكة (تعارض حالة) | 409 Conflict | `CRM_CUSTOMER_DUPLICATE_OWNED` |
| صلاحية غير كافية | 403 Forbidden | `CRM_REASSIGN_MANAGER_ONLY` |
| مصادقة غير موجودة/منتهية | 401 Unauthorized | — |
| بيانات إدخال غير صالحة | 422 Unprocessable Entity | `CRM_LEAD_NAME_REQUIRED` |
| مورد غير موجود | 404 Not Found | `CRM_ENTITY_NOT_FOUND` |
| تحذير غير مانع (العملية تمت مع ملاحظة) | 200 OK + `warnings[]` في الاستجابة | `CRM_CUSTOMER_DUPLICATE_UNOWNED` |
| خطأ خادم غير متوقَّع | 500 Internal Server Error | `SYS_UNEXPECTED_ERROR` |

## 4. Pagination
**مقترح:** Offset-based كافتراضي (أبسط للفرق المألوفة بـ CRUD تقليدي)، مع إمكانية التحول لـ Cursor-based لاحقاً للجداول كبيرة الحجم (كشوف حسابات، سجل تدقيق) إذا ظهرت مشاكل أداء.
```
GET /api/v1/crm/leads?page=1&pageSize=25
```
```json
{
  "data": [ /* ... */ ],
  "pagination": { "page": 1, "pageSize": 25, "totalItems": 340, "totalPages": 14 }
}
```
- الحد الأقصى لـ `pageSize`: 100 (يمنع سحب جداول كاملة بلا قصد).
- الافتراضي: 25.

## 5. Filtering
```
GET /api/v1/crm/leads?stage=qualified&owner=user_123&source=exhibition
```
- كل حقل فلترة = اسم الحقل نفسه من الـ Entity (مطابقة `docs/03_Business_Logic/{module}.md` قسم Entities).
- فلاتر النطاق (Range) بلاحقة: `?estValueMin=100000&estValueMax=500000`.
- فلاتر التاريخ بصيغة ISO 8601: `?createdFrom=2026-01-01&createdTo=2026-06-30`.

## 6. Sorting
```
GET /api/v1/crm/leads?sortBy=score&sortDir=desc
```
- `sortBy` = اسم حقل واحد فقط في v1 (لا ترتيب متعدد المستويات مبدئياً — تبسيط متعمَّد).
- `sortDir`: `asc` | `desc` (افتراضي: `asc`).

## 7. Validation
- **مُعتمَد بالفعل في Target Stack** (`Current_State.md`): FluentValidation على مستوى الـ Application layer، قبل وصول الطلب للـ Domain.
- استجابة التحقق الفاشل تتبع نفس عقد الخطأ (بند 3) مع `details` تحوي مصفوفة أخطاء لكل حقل:
```json
{
  "error": {
    "code": "VALIDATION_FAILED",
    "message": "...",
    "details": { "fields": [ { "field": "name", "code": "CRM_LEAD_NAME_REQUIRED" } ] }
  }
}
```

## 8. Authentication
- **مُعتمَد بالفعل في Target Stack:** JWT Bearer Token + Refresh Token (`Authorization: Bearer {token}`).
- **يحل محل** الـ Session البسيط الحالي في الـ Prototype (موثَّق كفجوة حرجة في `Known_Issues.md` KI-002 و`Current_State.md` قسم 3).
- كل endpoint محمي بصلاحية على مستوى الـ API (middleware) — **وليس إخفاء عناصر واجهة فقط**، تفعيلاً لما وثّقناه في قسم Permissions بكل ملف Business Logic.

---

## Open Questions (تحتاج قرار الفريق قبل الاعتماد كـ ADR)
1. هل Offset pagination كافٍ لكل الشاشات، أم بعض الجداول (سجل التدقيق، كشوف الحسابات) تحتاج Cursor-based من البداية لأسباب أداء؟
2. هل `sortBy` بحقل واحد فقط كافٍ لـ v1، أم فيه شاشات (زي لوحة CRM) محتاجة ترتيب متعدد من اليوم الأول؟
3. رمز `SYS_UNEXPECTED_ERROR` مقترح عام — هل نحتاج تصنيف أدق لأخطاء الخادم (DB timeout مثلاً) من الإصدار الأول؟
