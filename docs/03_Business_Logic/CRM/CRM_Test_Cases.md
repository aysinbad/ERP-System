# CRM Test Cases

## Document Information
```
Document Name:  CRM Test Cases
Version:        0.0.0
Status:         Placeholder
Classification: Source of Truth
Owner:          —
Approved-by:    —
Approved-date:  —
Last-Updated:   2026-07-18
```

> **حالة الملف:** Placeholder فارغ عمداً. حالات الاختبار تُبنى بعد استقرار `CRM.md` و`CRM_Implementation_Guide.md` (تحديداً بعد حسم INV-09 / ADR-015)، وبعد وجود كود Backend فعلي تُختبر عليه — كتابتها الآن ستكون تخميناً بلا قيمة (نفس مبدأ "لا اختبار قبل كود، ولا كود قبل قرار").

## الشكل المتوقَّع لاحقاً
كل حالة اختبار ستُربط صراحة بقاعدة عمل من `CRM.md` (رقم Business Rule أو Invariant)، بنفس فكرة الـ Traceability Matrix المذكورة في خطة الحوكمة (`ERP_System.pdf`):

| Test ID | القاعدة المرتبطة (Rule/Invariant) | السيناريو | النتيجة المتوقعة |
|---|---|---|---|
| *(فارغ حالياً)* | | | |

## لا تُملأ هذه الجداول قبل:
1. اعتماد ADR-015 (يحدد شكل State Transition Matrix النهائي).
2. وجود Backend فعلي (API endpoints) تُنفَّذ عليه الاختبارات.
