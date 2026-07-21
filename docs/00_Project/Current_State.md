# 00 — Current State (الوضع الحالي)

## Document Information
```
Document Name:  Current State
Version:        1.0.0
Status:         Approved
Classification: Reference
Owner:          Product Owner
Approved-by:    Product Owner
Approved-date:  2026-07-17
Last-Updated:   2026-07-17
```

> نقطة البداية الحقيقية للمشروع. اقرأ هذا الملف أولاً قبل أي شيء آخر.
> الهدف: أن يفهم أي مطور جديد أن `Prototype` يعمل كـ Living Specification كامل بمنطق أعمال حقيقي — وليس مشروعاً من الصفر.

---

## 1. ما هو Vortex الآن؟

Prototype أحادي الملف (`reference/prototype/prototype_v2.html`)، مُتخصص في نظام ERP لتجارة التصدير (Export Trading).

| المقياس | القيمة |
|---|---|
| إجمالي الأسطر | ~24,150 |
| عدد الشاشات (Views) | ~160 |
| عدد الدوال (Functions) | ~1,277 |
| عدد الجداول/المجموعات (Data Collections) | ~105 |
| اللغات | ثنائي اللغة — عربي (RTL) + إنجليزي كامل |
| العملات المدعومة | EGP, USD, EUR, GBP, SAR, AED (قابل للتوسيع) |
| التخزين الحالي | كائن `DB` في الذاكرة + `localStorage` |

> **ملاحظة حاسمة:** هذا Prototype يحتوي على منطق أعمال حقيقي ناضج (محرك قيود محاسبية، تسعير، كشوف حساب متعددة العملات)، لكنه **ليس مُنتَج production**. عند الترحيل لـ backend، المطلوب ترحيل هذا المنطق لا إعادة كتابته.

---

## 2. ما الذي يعمل فعلاً (Working Features)

### ناضج — المحاسبة (Accounting Engine)
- محرك قيود مزدوجة حقيقي — يشتق كل القيود من المستندات (`buildJournalCore`).
- حارس توازن يرفض أي قيد غير متوازن (`window._jrnBad`).
- دليل حسابات كامل (~40+ حساب) بترميز معياري.
- كشوف حساب موردين وعملاء متعددة العملات — كل عملة قسم منفصل بلا تحويل للجنيه.
- إعادة تقييم فروق العملة عبر حساب وسيط (1265) + الفروق المحققة (4030/5500).
- تسويات الانحرافات والهدر (حساب وسيط 1260).
- قائمة الدخل، الميزانية، ميزان المراجعة، تدفقات نقدية.

### ناضج — دورة التصدير (Export Cycle)
- سلسلة مستندات بيع كاملة: عرض سعر → بروفورما → عقد → أمر تصدير → فاتورة تجارية (`salesDocs`).
- Incoterms، BL، أوزان، تكاليف شحنة موزّعة.
- فواتير بعملات مختلفة مع أسعار صرف مثبّتة تاريخياً.

### متكامل — CRM
- Leads → Opportunities → Pipeline (kanban) → تحويل لبروفورما.
- ملكية العملاء (كل مندوب يرى عملاءه فقط) + منع التسجيل المزدوج + إعادة توزيع للمدير.
- أنشطة وتفاعلات، مهام CRM.

### متقدم — المخزون والإنتاج
- مخزون، دفعات (Lots)، حركات مخزون، إذون استلام (GRN).
- أوامر تشغيل بوضعين: BOM (تصنيع) و Yield (فرزة خام) — حساب هدر مختلف لكل وضع.
- تقارير الإنتاج والرصيد النشط، تقارير نسب الهدر لكل مورد.
- الرصيد المتاح الفعلي من إذون الاستلام حسب المورد.

### متقدم — التسعير (Pricing)
- حاسبة تسعير تصديري: تكلفة + هدر متوقع + مصاريف عبر مباشرة + هامش (Overhead absorption).
- ذكاء التكلفة: متوسط مرحّح + آخر شراء + آخر بيع + تكلفة القيمة (<90 يوم).
- سلّة تسعير متعددة المنتجات مع تعديل يدوي لكل بند.

### قسم مستقل — الموارد البشرية
- لوحة HR + موظفون + رواتب + حضور + إجازات + تأمين.
- تنبيهات انتهاء العقود والإقامات.

### أخرى
- Audit log بـ before/after diff + IP (جزئي).
- صلاحيات role-based (`can`, `perm`, `approvalLimit`).
- مستحقات موردين + أعمار ديون (aging).
- لوحة أوامر، مركز تقارير، بحث شامل (Ctrl+K).
- تيمات + تكبير نص + تباين عالٍ + dark mode.

---

## 3. ما هو غير موحود (Gaps) للـ Production

| الفجوة | الأولوية | ملاحظة |
|---|---|---|
| Multi-tenant (CompanyId) | حرجة | يُسحب من الـ schema من مستوى الـ schema — إضافته لازم من اليوم الأول |
| REST API + Backend | حرجة | كل المنطق حالياً على مستوى المتصفح |
| JWT Authentication | حرجة | Session بسيط فقط حالياً |
| Audit immutable | مهمة | موجود لكن قابل للحذف حالياً |
| Notification Center (Email/WhatsApp) | مهمة | In-app فقط موجود حالياً |
| Customer/Supplier Portals | خطة v1.5+ | غير موجود |
| Shipping Platform integration | خطة v1.5+ | غير موجود |
| Permissions per-action (Print/Export/Approve) | مهمة | section-level فقط موجود حالياً |

---

## 4. المعمارية المستهدفة (Target Stack)

```
Backend:     ASP.NET Core Web API + Clean Architecture + EF Core + SQL Server
Frontend:    React + TypeScript
Auth:        JWT + Refresh Token
Logging:     Serilog (structured)
Validation:  FluentValidation
Docs:        Swagger / OpenAPI
Cache:       Redis (optional)
Jobs:        Hangfire (background)
Storage:     Local / Azure Blob / S3
```

---

## 5. المرافقة — الملفات المولَّدة (Generated Artifacts)

| الملف | المحتوى | يذهب إلى |
|---|---|---|
| `schema.sql` | SQL Server، 25+ جدول، ~576 سطر، `CompanyId` على الكل، `AuditLog` immutable | `04_Database/` |
| `migration_map.md` | كل دالة من الـ prototype → C# service method | `03_Business_Logic/` (مرجعية) |
| `api_contract.json` | OpenAPI endpoints مشتقّة من المنطق | `05_API/` |

---

## 6. المبادئ الحاكمة على الترحيل

1. **الـ Prototype هو المرجع** — أي غموض في المواصفات، ارجع للكود.
2. **`CompanyId` مؤجَّله مكلف جداً (retrofitting)** — لا تُؤجَّل، من اليوم الأول.
3. **احترم المنطق المحاسبي** — هو مُختبَر ويوازن. لا تُعيد تفسيره.
4. **الترحيل يكون Rewrite 1:1 — لا Translation.** الكود منظّم وينتظم.
5. **حافظ على أسماء الحسابات وترميزها** — 1100 عملاء، 2010 موردون، 1260 وسيط انحرافات، 1265 فروق عملة، إلخ.
