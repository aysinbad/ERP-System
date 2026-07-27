# Accounting — Production Gap Matrix (Session 6.2, Part 5)

## Document Information
```
Document Name:  Accounting Production Gap Matrix
Version:        1.0.0
Session:        6.2 — Final Implementation Documentation
Status:         Draft — for review
Classification: Migration-input
Owner:          Solution Architecture Team
Date:           2026-07-25
```

> **Breaking change (Yes)** = يغيّر سلوكاً أو مخرجاً محاسبياً/واجهة يعتمد عليها مستخدم أو بيانات قائمة (يحتاج ترحيل بيانات أو إعادة تدريب). **(No)** = تحسين داخلي بلا كسر متعاقَد.

| # | المجال | Prototype behaviour | Target backend behaviour | Related ADR | Migration required | Breaking change |
|---|---|---|---|---|---|---|
| G-01 | نموذج الدفتر | الدفتر **مُشتقّ** وقت العرض (`buildJournalCore`) | دفتر **مُخزَّن ثابت** (append-only) يُرحَّل عند الحدث | ACC-A | Yes (ترحيل القيود التاريخية) | **Yes** |
| G-02 | تجميد القيم | مدخلات حيّة (`unitCost`/`fxRate`) وقت البناء | كل قيمة **مُجمَّدة** على السطر (ACC-B) | ACC-B | Yes (تجميد التاريخي) | **Yes** |
| G-03 | الحذف | حذف مباشر يمحو من كل الفترات | **عكس مؤرَّخ** لا محو (ACC-C) | ACC-C | Yes (تحويل مسارات الحذف) | **Yes** |
| G-04 | التعديل | استبدال في مكانه يعيد كتابة التاريخ | **عكس + إعادة إثبات** | ACC-C | Yes | **Yes** |
| G-05 | قفل الفترة | 5 كتّاب `DB.lockDate` غير منسّقين | **`PeriodLockService` مركزي واحد** (ACC-D) | ACC-D | Yes (إعادة توجيه الكتّاب) | **Yes** |
| G-06 | الصلاحيات | تحقق على الواجهة فقط؛ تجاوزات handler | **إنفاذ إلزامي على الـAPI** (KI-002) | ACC-D | Yes (authorization middleware) | **Yes** (لأدوار كانت تتجاوز) |
| G-07 | سداد المورد — ازدواج/سقف | لا حارس ازدواج ولا سقف (SP-03/04) | مفتاح تفرّد + سقف برصيد الفاتورة | ACC-A · ACC-D | Yes (قواعد تحقق) | **Yes** |
| G-08 | سداد غير مربوط — FX | `fxDiff=0` قسري (SP-02 · KI-025) | الاعتراف بفرق العملة دائماً | ACC-B | Yes | **Yes** |
| G-09 | الرواتب — مساران | Path A (5400) + Path B (5410) بلا Cross-Guard (PAY-04) | مسار رواتب **واحد** + مفتاح تفرّد شهري | ACC-A · OQ-6.D | Yes | **Yes** |
| G-10 | المحرك المتكرر | محركان؛ `recurringExp` Dead Code؛ توليد بلا قفل | محرك **واحد** + توليد يمرّ بالقفل | ACC-D | Yes | No (تنظيف داخلي) |
| G-11 | إعادة تقييم FX | محركان؛ Engine A يتجاوز 1265؛ Engine B خامل | محرك FX **واحد** عبر 1265 (ADR-003) | ADR-003 · **OQ-6.F** | Yes (بعد قرار OQ-6.F) | **Yes** (عند التوحيد) |
| G-12 | الأرصدة الافتتاحية | قيد OPENING حيّ مقابل 3001 (OB-01/02) | قيد OPENING كـ**seed ثابت** | ACC-A | Yes (seed) | No |
| G-13 | اقتناء الأصل | Dr 1300 / Cr **بنك دائماً** (KI-029) | اختيار تمويل بنك **أو** ذمة 2010 | ACC-A | No (إضافة خيار) | No |
| G-14 | إهلاك الأصل | Dr **5400** / Cr 1310 | Dr **حساب إهلاك مخصص** / Cr 1310 | ACC-A | No | No (تحسين عرض) |
| G-15 | الإقفال | `doClosing` (كامل) مقابل `doMonthEndClose` (قفل فقط) | **خدمة إقفال واحدة** (جاهزية+قيد+قفل+Audit) | ACC-D | Yes | **Yes** |
| G-16 | السجل بلا تاريخ | يتجاوز القفل (KI-016) | **يُرفَض** أو يُسنَد لتاريخ صريح | ACC-D | Yes | **Yes** |
| G-17 | Audit log | قابل للتلاعب؛ هاش djb2 على العميل | **append-only server-side** غير قابل للتلاعب | KI-004 | Yes | No |
| G-18 | كلمات المرور | نص صريح (KI-001) | تجزئة bcrypt/argon2 | KI-001 | Yes (re-hash) | No |
| G-19 | النسخ الاحتياطي | XOR مُضلِّل (KI-003) | تشفير AES-GCM حقيقي | KI-003 | Yes | No |
| G-20 | VAT العهدة | لا معالجة VAT إطلاقاً | VAT كسطر صريح حيث ينطبق | ACC-B | Yes | **Yes** |
| G-21 | مرتجع المبيعات — عكس التكلفة | مشروط بـ`rt.cogsEGP` المخزَّن (KI-020) | عكس التكلفة **إلزامي** | ACC-C | Yes | **Yes** |
| G-22 | حساب 2017 (عمولة) | لا مسار استنفاد/تسوية (KI-017) | **[BLOCKED — ACC-E · OQ-6.A]** | ACC-E (محجوب) | Pending PO | Pending |
| G-23 | 27 مصدراً غير مُراجَعة الأعمدة | مُشتقّة في الـProttype | Posting Rules بعد مراجعة الأعمدة | ACC-A/B | Yes (بلا اختراع) | TBD (بعد المراجعة) |

---

## ملخّص

- **Breaking changes (Yes):** G-01 · G-02 · G-03 · G-04 · G-05 · G-06 · G-07 · G-08 · G-09 · G-11 · G-15 · G-16 · G-20 · G-21 = **14** فجوة كاسرة تتطلب ترحيل بيانات و/أو إعادة تدريب.
- **Non-breaking:** G-10 · G-12 · G-13 · G-14 · G-17 · G-18 · G-19 = **7** تحسينات داخلية.
- **Blocked/TBD:** G-22 (ACC-E · OQ-6.A) · G-23 (بعد مراجعة الأعمدة) · G-11 يعتمد على OQ-6.F.
