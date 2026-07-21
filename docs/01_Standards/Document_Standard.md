<!--
═══════════════════════════════════════════════════════════════
  DOCUMENT HEADER STANDARD — قالب الترويسة المعتمد
  يُطبَّق على كل ملفات الـ Blueprint (00 → 09)
═══════════════════════════════════════════════════════════════
-->

# Documentation Standard (معيار التوثيق)

## Document Information
```
Document Name:  <اسم الوثيقة>
Version:        <MAJOR.MINOR.PATCH>
Status:         Draft | In Review | Approved | Deprecated
Classification: Source of Truth | Reference | Working Draft
Owner:          <حسب نوع الوثيقة — انظر أدناه>
Approved-by:    <الدور المعتمِد>
Approved-date:  <YYYY-MM-DD>
Last-Updated:   <YYYY-MM-DD>
```

---

## قواعد الملكية (Ownership)
| نوع الوثيقة | الأمثلة | Owner |
|---|---|---|
| **Business Documents** | Vision, Business Logic, Workflows, Decisions Log | **Product Owner** |
| **Technical Documents** | Architecture, Database, APIs, Coding Standards, UI Standards | **Solution Architecture Team** |

---

## قواعد الترقيم (Versioning — Semantic)
| التغيير | مثال | القاعدة |
|---|---|---|
| قرار/سلوك **تغيّر أو أُلغي** | 1.4.2 → **2.0.0** | MAJOR — تغيير كاسر للتوافق |
| قرار/قسم **جديد أُضيف** | 1.4.2 → **1.5.0** | MINOR — إضافة غير كاسرة |
| **تصحيح لغوي / توضيح** | 1.4.2 → **1.4.3** | PATCH — لا يغيّر المعنى |

---

## قواعد حالة الوثيقة (Document Status)
- **Draft** — قيد الكتابة، غير معتمد.
- **In Review** — جاهز للمراجعة، لم يُعتمد بعد.
- **Approved** — معتمد ونافذ.
- **Deprecated** — لم يعد ساريًا (يُشار للبديل).

---

## قواعد حالة القرار (ADR Status — دورة الحياة)
- **Proposed** — مقترح، لم يُعتمد.
- **Accepted** — معتمد ونافذ.
- **Superseded** — أُلغي بقرار أحدث (يُذكر: `Superseded-by: ADR-XXX`).
- **Deprecated** — لم يعد ساريًا دون بديل مباشر.

> **قاعدة الترقيم الأبدي:** رقم الـ ADR لا يُعاد استخدامه أبدًا، حتى لو حُذف القرار. الأرقام تتزايد للأمام فقط.

---

## قالب الـ ADR المعتمد (ADR Template)
```
## ADR-XXX — <العنوان>
- Status:        Proposed | Accepted | Superseded | Deprecated
- Date:          YYYY-MM-DD
- Owner:         <الدور>
- Supersedes:    <ADR-XXX أو —>
- Superseded-by: <ADR-XXX أو —>

### Context (السياق)
<لماذا نحتاج هذا القرار؟>

### Decision (القرار)
<ما القرار المتخذ؟>

### Alternatives Rejected (البدائل المرفوضة)
<ما الذي رُفض ولماذا؟>

### Consequences (النتائج)
**الإيجابية:** <المكاسب>
**السلبية / الثمن:** <ما الذي نتنازل عنه — كل قرار له ثمن>
```
