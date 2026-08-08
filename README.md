# خطة الانفكاك — جاهز للبناء على GitHub

الريبو ده فيه تطبيق "خطة الانفكاك" (مدير دورة الجيش والإجازة) جاهز إنه:
1. يتنشر كموقع PWA على GitHub Pages تلقائيًا مع كل push.
2. يتحوّل لتطبيق أندرويد حقيقي (APK / AAB) تلقائيًا عن طريق GitHub Actions،
   باستخدام أداة Google الرسمية [Bubblewrap](https://github.com/GoogleChromeLabs/bubblewrap)
   اللي بتغلّف أي PWA في TWA (Trusted Web Activity) — تطبيق أندرويد شبه أصلي.

---

## ⚠️ قبل ما تعمل push — خطوتين لازم تعملهم

الملفات فيها placeholders لازم تستبدلها باسم المستخدم واسم الريبو بتاعك،
وإلا التطبيق هيتبني لكنه هيشاور على رابط غلط.

### 1) استبدل `USERNAME` و `REPO_NAME` في `twa-manifest.json`

مثلاً لو اسم حسابك `ahmed` واسم الريبو `enfkak-app`:
```
"host": "ahmed.github.io"
"startUrl": "/enfkak-app/index.html"
"iconUrl": "https://ahmed.github.io/enfkak-app/icons/icon-512.png"
"maskableIconUrl": "https://ahmed.github.io/enfkak-app/icons/icon-512.png"
"webManifestUrl": "https://ahmed.github.io/enfkak-app/manifest.json"
"fullScopeUrl": "https://ahmed.github.io/enfkak-app/"
```

### 2) فعّل GitHub Pages بمصدر "GitHub Actions"

من إعدادات الريبو: **Settings → Pages → Build and deployment → Source →
GitHub Actions**. الخطوة دي لازم تتعمل مرة واحدة يدوي، مينفعش الـ workflow
يفعّلها لوحده.

بعد كده، أي push على `main` هيشغّل الـ workflows تلقائيًا.

---

## هيكل الريبو

```
.
├── .github/workflows/
│   ├── deploy-pages.yml     ← بينشر www/ على GitHub Pages (بسيط ومضمون)
│   └── build-android.yml    ← بيبني APK/AAB باستخدام Bubblewrap (يحتاج تتأكد منه أول مرة)
├── www/                      ← التطبيق نفسه (PWA)
│   ├── index.html
│   ├── manifest.json
│   ├── sw.js
│   └── icons/
├── twa-manifest.json         ← إعدادات تحويل الـ PWA لتطبيق أندرويد
└── .gitignore
```

---

## إيه اللي بيحصل في كل workflow

### `deploy-pages.yml`
- بينشر فولدر `www/` كموقع على `https://USERNAME.github.io/REPO_NAME/`.
- ده الـ workflow المضمون والبسيط — تقريبًا مش هيفشل.

### `build-android.yml`
- بيعمل build environment (Java 17 + Node + Android SDK).
- بيولّد مفتاح توقيع (keystore) مؤقت جوه الـ CI نفسه.
- بيشغّل `bubblewrap build` عشان يطلع APK و AAB.
- بيولّد `assetlinks.json` (ملف التحقق اللي بيخلي التطبيق يفتح بدون شريط
  عنوان المتصفح — "Verified TWA").
- بيرفع الملفات الناتجة (APK/AAB/assetlinks.json) كـ **Artifact** تقدر
  تنزّله من تبويب **Actions** في الريبو → تختار آخر run → **Artifacts**.

### خطوة إضافية (اختيارية بس مهمة) بعد أول build ناجح
عشان التطبيق يفتح بملء الشاشة بدون شريط عنوان (Verified TWA مش مجرد
Custom Tab)، لازم تاخد ملف `assetlinks.json` من الـ Artifact، تحطه في:
```
www/.well-known/assetlinks.json
```
وتعمل push تاني. من غير الخطوة دي، التطبيق هيشتغل عادي لكن هيظهر فيه
شريط عنوان صغير فوق زي متصفح مصغّر.

---

## ملاحظة صريحة عن الموثوقية 🙏

`bubblewrap build` بيعتمد على أدوات خارجية (Android SDK، Bubblewrap CLI)
بتتحدّث باستمرار، وأنا معنديش وصول لتشغيل الـ workflow ده فعليًا هنا
عشان أتأكد إنه هيعدي 100% من أول مرة بالظبط بنفس أسماء الملفات والـ flags.
يعني ممكن أول run تحتاج تعديل بسيط (زي اسم ملف الـ APK الناتج، أو نسخة
أداة معينة) — الـ log بتاع الـ Actions run هيوريك بالظبط فين المشكلة لو
حصلت. الـ `deploy-pages.yml` في المقابل بسيط جدًا ومفيش سبب يفشل.

لو عايز حل أبسط ومضمون 100% من غير CI معقد: بعد ما تنشر `www/` على Pages
(الـ workflow الأول)، روح على **[pwabuilder.com](https://www.pwabuilder.com)**
وحط رابط الموقع — هيطلعلك APK/AAB جاهزين من غير ما تحتاج تدير Android SDK
جوه GitHub Actions خالص.

---

## للاستخدام الإنتاجي (نشر على Google Play فعليًا)

الـ workflow الحالي بيولّد **مفتاح توقيع جديد عشوائي في كل run** — كويس
للتجربة والتنصيب اليدوي، لكن مش مناسب لو هترفع تحديثات على Google Play
(لازم نفس المفتاح كل مرة). لو وصلت للمرحلة دي:
1. اعمل keystore ثابت مرة واحدة على جهازك (`keytool -genkeypair ...`).
2. ارفعه كـ GitHub Secret (base64-encoded) بدل ما يتولّد جوه الـ CI.
3. ضيف Secrets باسم `KEYSTORE_PASSWORD` و `KEY_PASSWORD` بالقيم الحقيقية
   بتاعتك (الـ workflow بيقرأهم تلقائيًا لو موجودين، وإلا بيستخدم قيمة
   افتراضية للتجربة بس).

---

## البيانات

التطبيق بيحفظ كل بياناته (مهام، أهداف، نوم، أرشيف...) محليًا على جهاز
المستخدم (`localStorage`) — مفيش سيرفر أو حساب. فيه زرار "تصدير/استيراد
نسخة احتياطية" جوه التطبيق نفسه.
