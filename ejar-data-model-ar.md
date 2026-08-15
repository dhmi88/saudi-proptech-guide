# نموذج بيانات إيجار — مرجع تقني

مرجع مفتوح لمن يبني نظام إدارة أملاك يتعامل مع منصة إيجار السعودية: الحقول التي تتطلبها، صيغها، نموذج بيانات مقترح، ودوال تحقق جاهزة.

كل ما في هذا الملف إمّا مستند إلى مصدر رسمي، أو مُعلَّم صراحةً كمقترح هندسي.

## الأساس النظامي

| البند | المرجع |
|---|---|
| إلزامية تسجيل عقود الإيجار السكنية والتجارية | قرار مجلس الوزراء رقم ٤٠٥ وتاريخ ٢٢ رمضان ١٤٣٧هـ |
| الجهة المشرفة | الهيئة العامة للعقار (REGA) |
| المنصة | إيجار — ejar.sa |
| تسجيل الدخول | حصرياً عبر النفاذ الوطني الموحّد |
| عمولة الوساطة | ٢.٥٪ من إيجار السنة الأولى، ما لم يتفق الأطراف كتابةً على غير ذلك |
| الرقم الموحد | 199011 |

خدمة **«التكامل الرقمي بين شبكة إيجار ومنصات التسويق العقاري»** خدمة رسمية معلنة، وقد أبرمت إيجار ١٢ اتفاقية تكامل رقمي مع منصات عقارية.

**رحلة التسجيل عبر التكامل:** تسجيل بيانات العقار والوحدات في المنصة، ثم تسجيل العقد، ثم إرسال تلقائي لشبكة إيجار ليوثّقه الأطراف.

## التحقق الحكومي — لماذا يهمك تقنياً

إيجار مرتبطة إلكترونياً بجهات حكومية للتحقق:

**التكامل مكتمل:** مركز المعلومات الوطني، وزارة التجارة، وزارة العدل، وزارة الداخلية.

**التكامل جارٍ:** البريد السعودي، شركة المياه الوطنية، الشركة السعودية للكهرباء، وزارة العمل والتنمية الاجتماعية.

**الأثر الهندسي:** هويات الأطراف تُتحقق من مصدرها الرسمي. أي رقم غير مطابق يُرفض عند التسجيل، لا عند العرض. لذلك أي طبقة تحقق محلية هدفها **تقليل دورات الرفض**، لا استبدال التحقق الحكومي.

افصل بين ما تستطيع إثباته محلياً (الطول، الصيغة، منطقية التواريخ) وما لا تستطيع (أن هذا الرقم يعود لهذا الشخص).
## الحقول الإلزامية

### المؤجّر (المالك)

| الحقل | ملاحظة |
|---|---|
| الاسم الكامل | مطابق للهوية |
| رقم الهوية / الإقامة | ١٠ أرقام |
| رقم الجوال | صيغة صحيحة قابلة للتحقق |
| تاريخ الميلاد | مطلوب |
| الآيبان | SA + ٢٢ رقم، لتحويلات الإيجار |

### المستأجر

| الحقل | ملاحظة |
|---|---|
| الاسم الكامل | مطابق للهوية |
| رقم الهوية / الإقامة | ١٠ أرقام |
| رقم الجوال | صيغة صحيحة |
| تاريخ الميلاد | مطلوب |
| الجنسية | مطلوبة |

### العقار والوحدة

| الحقل | ملاحظة |
|---|---|
| رقم الصك أو رخصة البناء | مطلوب |
| العنوان / العنوان الوطني | مطلوب |
| نوع الاستخدام | سكني أو تجاري |

### العقد

| الحقل | ملاحظة |
|---|---|
| تاريخ البداية | مطلوب |
| تاريخ النهاية | يجب أن يكون بعد البداية |
| قيمة الإيجار | مطلوبة |
| دورة الدفع | شهري / كل شهرين / ربع سنوي / كل ٤ أشهر / نصف سنوي / سنوي |

## نموذج بيانات مقترح

طلب التوثيق **ليس صفة على العقد** — هو معاملة لها دورة حياة ومال ومسؤول. لذلك يستحق نموذجه المستقل:

```prisma
model EjarContractRequest {
  id String @id @default(uuid())

  // علاقة واحد لواحد مع العقد
  contractId String   @unique
  contract   Contract @relation(fields: [contractId], references: [id])

  agencyId      String
  requestedById String

  propertyUsageType     PropertyUsageType
  contractDurationYears Int

  // الرسوم — Decimal وليس Float
  ejarFee    Decimal @db.Decimal(20, 8)
  brokerFee  Decimal @db.Decimal(20, 8)
  serviceFee Decimal @default(0) @db.Decimal(20, 8)
  totalFee   Decimal @db.Decimal(20, 8)

  // ما يعود من إيجار بعد التوثيق
  ejarContractNumber String?
  ejarContractPdf    String?

  status          EjarRequestStatus @default(PENDING)
  adminNotes      String?
  rejectionReason String?

  // سجل مالي مرتبط
  walletTransactionId String?
  refundTransactionId String?

  completedAt DateTime?
  rejectedAt  DateTime?
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@index([agencyId])
  @@index([status])
  @@map("ejar_contract_requests")
}

enum EjarRequestStatus {
  PENDING
  IN_PROGRESS
  COMPLETED
  REJECTED
}

enum PropertyUsageType {
  RESIDENTIAL
  COMMERCIAL
}
```

**قرارات تصميمية في هذا النموذج:**

`contractId @unique` يمنع طلبين لعقد واحد على مستوى قاعدة البيانات لا الكود.

`Decimal(20, 8)` — المال لا يُخزَّن في `Float` أبداً. أخطاء التقريب تتراكم وتظهر في أول تسوية.

`walletTransactionId` و`refundTransactionId` يربطان الطلب بحركتيه الماليتين، فيصبح سؤال «أين ذهب هذا المبلغ؟» استعلاماً واحداً.

`rejectionReason` منفصل عن `adminNotes` — الأول يُعرض للمستخدم، والثاني داخلي.
## دوال التحقق

### مستخدمة في الإنتاج

هذه هي الدوال الفعلية التي نشغّلها في [أملاكي](https://amlakire.com)، مشتركة حرفياً بين الويب والموبايل:

```typescript
/**
 * Regex موحّد لرقم الهاتف.
 * يقبل: 05XXXXXXXX و 5XXXXXXXX و الصيغة الدولية E.164
 */
export const PHONE_REGEX = /^(\+[1-9]\d{6,14}|05\d{8}|5\d{8})$/;

export function isValidPhone(phone: string | null | undefined): boolean {
  if (!phone) return false;
  return PHONE_REGEX.test(phone.replace(/\s/g, ''));
}

/** Regex موحّد لرقم الهوية السعودية (10 أرقام). */
export const NATIONAL_ID_REGEX = /^\d{10}$/;

export function isValidNationalId(id: string | null | undefined): boolean {
  if (!id) return false;
  return NATIONAL_ID_REGEX.test(id);
}
```

**لماذا نقبل E.164 الدولي؟** لأن جزءاً حقيقياً من المستأجرين والملاك أرقامهم غير سعودية. حصر الصيغة في `+9665` أنظف نظرياً ويمنع بيانات صحيحة عملياً.

**لماذا الهوية عشرة أرقام دون فحص البادئة؟** قاعدة «يبدأ بـ ١ للسعودي و٢ للمقيم» صحيحة، لكن جعلها شرط رفض يعني أن ترفض محلياً رقماً قد تقبله الجهة الرسمية. طبقة الجاهزية تمسك الأخطاء الواضحة ولا تدّعي أنها بديل عن التحقق الحكومي.

### دوال مرجعية مقترحة

الدوال التالية **ليست جزءاً من نظامنا الحالي** — نعرضها كمرجع لمن يريد طبقة تحقق أعمق.

**تحقق الآيبان بـ MOD-97** (معيار ISO 13616):

```typescript
export function validateIBAN(value: string): { isValid: boolean; error?: string } {
  const cleaned = value.replace(/\s/g, '').toUpperCase();

  if (!/^SA\d{22}$/.test(cleaned)) {
    return { isValid: false, error: 'الآيبان السعودي يجب أن يكون SA + 22 رقم' };
  }

  const rearranged = cleaned.slice(4) + cleaned.slice(0, 4);
  const numeric = rearranged.replace(/[A-Z]/g, (c) => String(c.charCodeAt(0) - 55));

  const remainder = numeric
    .split('')
    .reduce((acc, digit) => (acc * 10 + Number(digit)) % 97, 0);

  return remainder === 1
    ? { isValid: true }
    : { isValid: false, error: 'الآيبان غير صحيح (فشل تحقق MOD-97)' };
}
```

**حساب مدة العقد بالسنوات** — التسعير الرسمي يعامل جزء السنة كسنة كاملة:

```typescript
export function calculateDurationYears(startDate: Date, endDate: Date): number {
  const diffDays = (endDate.getTime() - startDate.getTime()) / (1000 * 60 * 60 * 24);
  return Math.ceil(diffDays / 365);
}
```

## نمط فحص الجاهزية

التوصية: دالة واحدة تُرجع **قائمة النواقص بأسمائها**، لا قيمة منطقية.

```typescript
function validateContractData(contract: any): string[] {
  const missing: string[] = [];
  const owner = contract.unit?.property?.owner;
  const tenant = contract.tenant;
  const property = contract.unit?.property;

  if (!owner?.fullName) missing.push('اسم المالك');
  if (!owner?.nationalId) missing.push('رقم هوية المالك');
  if (!owner?.phone) missing.push('جوال المالك');
  if (!owner?.birthDate) missing.push('تاريخ ميلاد المالك');

  if (!tenant?.fullName) missing.push('اسم المستأجر');
  if (!tenant?.nationalId) missing.push('رقم هوية المستأجر');
  if (!tenant?.phone) missing.push('جوال المستأجر');
  if (!tenant?.birthDate) missing.push('تاريخ ميلاد المستأجر');
  if (!tenant?.nationality) missing.push('جنسية المستأجر');

  if (!property?.deedNumber) missing.push('رقم الصك / رخصة البناء');
  if (!property?.address) missing.push('عنوان العقار');

  if (!contract.startDate) missing.push('تاريخ بداية العقد');
  if (!contract.endDate) missing.push('تاريخ نهاية العقد');

  return missing;
}
```

**لماذا مصفوفة نصوص؟** لأن الواجهة تحتاج أن تعرض *ماذا* ينقص. «البيانات غير مكتملة» وحدها تجعل المستخدم يخمّن؛ القائمة تجعله يتصرّف.

**لماذا السلسلة الاختيارية في كل مستوى؟** عقد بلا وحدة، أو وحدة بلا عقار، أو عقار بلا مالك — كلها حالات موجودة في بيانات حقيقية. السلسلة الاختيارية تُرجع «ناقص» بدل أن ترمي TypeError.

## قواعد هندسية

1. طبقتان للتحقق: واحدة في الواجهة للاستجابة الفورية، وواحدة على الخادم كمرجع. الأولى وحدها يمكن تجاوزها بطلب HTTP مباشر.
2. بوابة التحقق تسبق أي عملية لها تكلفة. لا تخصم ثم تكتشف النقص.
3. صيغ التحقق في ملف واحد مشترك بين كل الواجهات. واجهتان بـ regex مختلفين تعني بيانات تُقبل هنا وتُرفض هناك.
4. أصلح النواقص عبر مسارات كياناتها الأصلية لا عبر endpoint خاص بإيجار. البيانات ليست بيانات إيجار — إيجار فقط تكشف نقصها.
5. افصل حالة التوثيق عن حالة العقد بنيوياً. عقد نشط داخلياً قد لا يكون له طلب توثيق أصلاً.
6. الاسترجاع جزء من انتقال الحالة، لا خطوة يدوية بعده. وسبب الرفض إلزامي.
7. لا تدّعِ تكاملاً لم تنضم إليه. «جاهزية بيانات إيجار» ليست «متكامل مع إيجار»، والفرق يلاحظه مستخدموك.

## مصادر رسمية

- منصة إيجار — ejar.sa
- الهيئة العامة للعقار — rega.gov.sa
- المنصة الوطنية — my.gov.sa
- تسجيل الدخول — eservices.ejar.sa/ar/nafath-login
- الرقم الموحد — 199011

---

*هذا المرجع مبني على خبرة تطوير [أملاكي](https://amlakire.com) — نظام إدارة أملاك سعودي. أملاكي لم تنضم بعد لبرنامج التكامل الرقمي مع شبكة إيجار؛ ما يُعرض هنا هو طبقة الجاهزية التي تسبقه.*

*المساهمات مرحّب بها عبر Issues و Pull Requests.*
