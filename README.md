# نظام مكتب معراج للعقار

نظام إدارة عقارات وحسابات لـ **مكتب معراج للعقار** (ميسان، العراق)، بواجهتين على نفس قاعدة البيانات:

- **لوحة الإدارة** (`#admin`): العقارات، العملاء، الشركاء، الحسابات، العقود، إحصائيات الزوار، الإعدادات.
- **واجهة الزبون** (`#store`): عرض العقارات المنشورة بأسعارها، سلايدر العقارات المميزة، من نحن / تواصل / موقعنا.

> النسخة الحالية صفحة واحدة (`index.html`) بدون أي build أو مكتبات خارجية: HTML + CSS + JavaScript عادي.

---

## التشغيل محلياً

```bash
# أي سيرفر ملفات ثابتة يكفي
npx serve .
# أو
python3 -m http.server 8080
```

افتح `http://localhost:8080/#admin` للإدارة و `#store` لواجهة الزبون.

بدون قاعدة بيانات يشتغل النظام بـ **وضع تجريبي**: البيانات بالذاكرة فقط وتنمسح عند تحديث الصفحة.

---

## قاعدة البيانات (مهم للمطوّر)

الكود كُتب أصلاً كـ Claude Artifact، ويحصل على قاعدة البيانات من:

```js
const db = await window.claude.use('db')   // null خارج بيئة Claude
```

الـ API مشابه لـ Firestore، والكود يستخدم منه فقط:

| الاستخدام | الدالة |
|---|---|
| اشتراك مباشر بمجموعة | `db.collection(name).onSnapshot(cb)` |
| كتابة / حذف وثيقة | `db.collection(name).doc(id).set(data)` / `.delete()` |
| الإعدادات | `db.doc('config/settings').onSnapshot(cb)` / `.set(data)` |
| استعلام الإحصائيات | `db.collection('visits').where('date','>=',since).get()` |

**الخطوة الأولى للاستضافة الخاصة:** كتابة adapter صغير يوفّر نفس الدوال فوق Supabase (أو Firebase)، وتمريره بدل `window.claude.use('db')` بدالة `init()`. باقي الكود ما يحتاج تغيير.

### المجموعات (Collections)

| المجموعة | الوصف | أهم الحقول |
|---|---|---|
| `properties` | العقارات | `code` (MR-001…), `title`, `type`, `purpose` (بيع/إيجار), `status` (متاح/محجوز/مباع/مؤجر), `price`, `size`, `gov`, `area`, `address`, `mapUrl`, `lat`, `lng`, `ownerName`, `ownerPhone`, `own` (broker/invest), `published`, `featured`, `snoozeUntil`, `sale{price,date,buyer,kind}`, `createdAt` |
| `photos` | صور العقارات (base64) | `propertyId`, `data`, `order` |
| `clients` | العملاء | `name`, `phone`, `email`, `budget`, `interests` |
| `partners` | الشركاء | `name`, `phone`, `capital` |
| `tx` | الحركات المالية | `kind`, `amount`, `date`, `category`, `propertyId`, `partnerId` |
| `contracts` | العقود المحفوظة | `type` (sale/rent), `no`, `date`, حقول الطرفين (`sName/bName` أو `lName/tName`…), المبالغ |
| `visits` | أحداث إحصائيات الزوار | `type` (visit/view/contact), `vid`, `sid`, `ts`, `date`, `pid`, `src`, `dev`, `os` |
| `config/settings` | الإعدادات (وثيقة واحدة) | `officeName`, `phone`, `whatsapp`, روابط السوشل, `mapsUrl`, `officeLL`, `featMode`, `featSpeed`, `metaPixel`, `gtagId`, ترويسة العقود |

### أنواع الحركات المالية (`tx.kind`)

| kind | المعنى |
|---|---|
| `inc_office` | إيراد عام للمكتب |
| `exp_office` | مصروف عام للمكتب |
| `exp_property` | مصروف خاص على عقار (يُطرح من ربحه وقت البيع) |
| `inc_partner` | إيراد خاص لشريك |
| `exp_partner` | مصروف خاص على شريك |
| `capital_in` | إضافة رأس مال (شريك) |
| `withdraw` | سحب (شريك) |

العملة: الدينار العراقي فقط. حصة الشريك من الربح تتحدد وقت البيع: نسبة % أو مبلغ ثابت.

---

## خريطة الكود داخل `index.html`

- **CSS**: الثيم (متغيرات `--gold`, `--char`…)، واجهة الزبون، السلايدر (10 حركات)، الطباعة A4 للعقود.
- **JS** (بالترتيب تقريباً):
  - الثوابت والحالة: `KINDS`, `DEF_SET`, `COLS`, `S`, `UI`
  - الطبقة الحسابية: أرصدة الشركاء، ربح المكتب
  - أكواد MR: `mrCode`, `nextMR`, `migrateCodes`
  - لوحة الإدارة: `dash`, `props`, `propForm`, `clients`, `partners`, `finance`, `settings`
  - تذكيرات العقارات الراكدة (+30 يوم): `staleList`, `staleCard`
  - العقود: `ctEditor`, `saleSheet`, `rentSheet`, `printView`, التفقيط `words()`
  - واجهة الزبون: `storeView`, `storeDetail`, `featStage` + `applyFs` (السلايدر), `storeInfo`, `sideCta`
  - الخرائط: `parseLL`, `mapEmbed`, `propMapField`
  - الإحصائيات والبكسل: `track`, `loadPixels`, `stats`
  - `render()` و `init()`

---

## الخطوات القادمة المقترحة

1. Adapter لقاعدة بيانات Supabase + نقل البيانات الحالية.
2. تسجيل دخول للإدارة، وحساب لكل شريك يشوف حصته فقط.
3. نقل الصور من base64 إلى Storage.
4. استضافة على دومين خاص (Vercel / Netlify / Cloudflare Pages) حتى تشتغل الإحصائيات والبكسل على الزبائن الحقيقيين.
5. تقسيم `index.html` لملفات/مكوّنات عند الحاجة.

> ⚠️ لا ترفع بيانات الزبائن أو المالكين أو أي مفاتيح API على GitHub. البيانات بقاعدة البيانات، مو بالكود.
