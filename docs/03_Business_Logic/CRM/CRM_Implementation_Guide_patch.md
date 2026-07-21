# CRM_Implementation_Guide — Pricing Integration Patch
# أضف هذا القسم كاملاً في CRM_Implementation_Guide.md
# المكان: بعد قسم Permissions وقبل قسم Notifications

---

```markdown
## Pricing Integration

> المرجع الكامل لما يرى / ما لا يرى: `CRM.md` Business Rules #12 و#13. هنا التفاصيل التقنية فقط.

### عرض السعر الاسترشادي في CRM

**فحص الصلاحية:**
```
can('pricing','viewSuggestedPrice') → عرض السعر + PriceConfidenceState
can('pricing','viewCost') → يتضمن viewSuggestedPrice ضمنياً
لا صلاحية → لا عرض لأي معلومات تسعير
```

**حقول العرض (للمستخدم بصلاحية viewSuggestedPrice):**
```typescript
interface SuggestedPriceDisplay {
  suggestedPrice: number;           // السعر بعملة المنتج
  currency: string;
  confidenceState: 'fresh' | 'stale' | 'no_purchase';
  staleBadgeText: string;           // نص الـ badge للمستخدم
  // confidenceDetails: HIDDEN — للإدارة فقط (viewCost)
}
```

**نصوص الـ Badge (للمستخدم):**
- `fresh` → "سعر مرجعي محدَّث ✓"
- `stale` → "يحتاج مراجعة الإدارة ⚠️" + tooltip: *"السعر الاسترشادي مبني جزئياً على بيانات تجاوزت 3 أشهر تقريباً."*
- `no_purchase` → "لا يوجد سعر مرجعي حديث — برجاء طلب اعتماد تسعير."

**للإدارة (viewCost) — إضافي:**
- قائمة البنود القديمة: اسم البند، تاريخ آخر تحديث، المصدر، القيمة.
- إشارة إذا كان السعر يعكس Cost Override.

### Price Selector قبل تحويل الفرصة لبروفورما

**تدفق الشاشة:**
```
المستخدم يضغط "تحويل لبروفورما" على فرصة ناجحة
        │
        ▼
هل المستخدم يملك pricing.viewSuggestedPrice؟
        │
   لا ──► تحويل مباشر بـ Product.price (السلوك القديم — fallback)
        │
   نعم  ▼
جلب السعر الاسترشادي + PriceConfidenceState لكل منتج في الفرصة
        │
        ▼
عرض Price Selector للمستخدم:
  - السعر الاسترشادي (مع badge الموثوقية)
  - حقل سعر يدوي (اختياري)
        │
        ▼
هل البيانات قديمة (stale) أو يوجد Override أو سعر يدوي خارج السياسة؟
        │
   نعم ──► فحص pricing.approvePriceException
           └─ لا صلاحية → رفض + PRC_PRICE_EXCEPTION_REQUIRED
           └─ يملك صلاحية → طلب سبب إجباري → متابعة
        │
   لا   ▼
حفظ PriceSnapshot على البروفورما → تنفيذ التحويل
```

**حقول PriceSnapshot المحفوظة على البروفورما:**
```typescript
proforma.priceSnapshot = {
  suggestedPrice: number,
  confidenceState: string,
  costOverridePresent: boolean,
  selectedPrice: number,      // ما اختاره المستخدم
  selectedBy: userId,
  selectedAt: DateTime,
  exceptionApproved: boolean,
  exceptionReason?: string,
}
```

### Error Codes الخاصة بـ Pricing Integration في CRM

| الكود | السياق في CRM | HTTP |
|---|---|---|
| `PRC_PRICE_EXCEPTION_REQUIRED` | تحويل فرصة بسعر يستوجب اعتماد استثناء | 403 |
| `PRC_NO_COST_DATA` | لا يوجد سعر استرشادي للمنتج | 422 |
| `CRM_PROFORMA_PRICE_REQUIRED` | المستخدم لم يختر سعراً قبل التحويل | 422 |
```
