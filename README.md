# خطة الانفكاك — تطبيق أندرويد حقيقي (مشروع Gradle كامل)

الريبو ده مختلف عن أي محاولة سابقة: ده **مشروع Android Studio / Gradle
حقيقي بالكامل** (Kotlin + WebView)، مش تغليف لموقع مستضاف على الإنترنت.
كل ملفات التطبيق (HTML/CSS/JS + الأيقونات) متضمّنة **جوه الـ APK نفسه**
تحت `app/src/main/assets/www/` وبيتحمّلوا من `file:///android_asset/`
مباشرة — التطبيق شغال 100% أوفلاين من غير ما يحتاج أي سيرفر أو رابط
مستضاف، ومفيش صلاحية إنترنت مطلوبة أصلًا (لاحظ إن `AndroidManifest.xml`
مفيهوش `INTERNET permission`).

---

## ليه ده مختلف عن الحل اللي فات (Bubblewrap/TWA)؟

| | TWA (الحل السابق) | المشروع ده (WebView أصلي) |
|---|---|---|
| المصدر | بيفتح رابط مستضاف حقيقي على الإنترنت | كل حاجة جوه الـ APK نفسه |
| يحتاج إنترنت أول مرة | آه | لأ |
| يحتاج GitHub Pages | آه | لأ |
| أداة البناء | Bubblewrap (أداة خارجية بتتغير كتير) | Gradle نفسه (المعيار الرسمي لأي مشروع أندرويد) |
| فتحه في Android Studio | مش مصمم لده | آه، افتحه عادي زي أي مشروع أندرويد |

---

## هيكل المشروع

```
.
├── .github/workflows/build-apk.yml   ← يبني APK حقيقي تلقائيًا بـ Gradle
├── app/
│   ├── build.gradle.kts               ← إعدادات التطبيق (اسم الحزمة، الإصدار...)
│   ├── src/main/
│   │   ├── AndroidManifest.xml
│   │   ├── java/com/enfkak/app/MainActivity.kt   ← الشاشة الوحيدة (WebView)
│   │   ├── assets/www/                ← التطبيق نفسه (HTML/CSS/JS) — كل حاجة هنا محلية
│   │   └── res/                       ← الأيقونات والألوان والـ themes
├── gradle/wrapper/                    ← Gradle Wrapper (عشان تقدر تبني من غير ما تنزّل Gradle بنفسك)
├── gradlew / gradlew.bat
├── settings.gradle.kts
└── build.gradle.kts
```

---

## إزاي تبنيه

### الطريقة 1: عن طريق GitHub Actions تلقائي (زي ما طلبت)
كل `push` على `main` بيشغّل `.github/workflows/build-apk.yml` اللي:
1. بيجهّز JDK 17.
2. بيشغّل `./gradlew assembleDebug` و `./gradlew assembleRelease`.
3. بيرفع ملفات الـ APK الجاهزة كـ **Artifact**.

عشان تنزّل الـ APK: روح على تبويب **Actions** في الريبو → آخر run ناجح →
**Artifacts** → `enfkak-android-apk` → فك الضغط، هتلاقي فيه:
- `app-debug.apk` → جاهز للتنصيب المباشر على أي جهاز (Sideload) للتجربة.
- `app-release-unsigned.apk` أو مُوقّع بمفتاح الـ debug الافتراضي (حسب
  إعدادات التوقيع في `app/build.gradle.kts` — شرح تحت لو عايز توقيع حقيقي).

هذا الـ workflow بيعتمد على **Gradle نفسه بس** (مفيش أدوات خارجية زي
Bubblewrap)، وde standard رسمي معروف لأي مشروع أندرويد — احتمالية إنه
يفشل من أول push أقل بكتير من الحل اللي فات.

### الطريقة 2: على جهازك (Android Studio)
1. افتح Android Studio → **Open** → اختار فولدر الريبو.
2. سيبه يعمل Sync (هيحمّل الـ Gradle والـ SDK تلقائيًا).
3. دوس **Run ▶** — التطبيق هيتفتح على إيموليتور أو جهاز حقيقي متوصل.
4. أو من قائمة **Build → Build Bundle(s)/APK(s) → Build APK(s)** للحصول
   على ملف APK تقدر تشاركه مباشرة.

### الطريقة 3: على جهازك (Terminal، من غير Android Studio)
```bash
./gradlew assembleDebug
```
الملف الناتج هيبقى في:
```
app/build/outputs/apk/debug/app-debug.apk
```
انقله لموبايلك ونصّبه (لازم تفعّل "تثبيت من مصادر غير معروفة" أول مرة).

---

## التوقيع الحقيقي (لو هترفع على Google Play)

دلوقتي التطبيق موقّع بمفتاح الـ debug الافتراضي (كويس للتجربة والتنصيب
اليدوي بس). لو قررت تنشره على Google Play:
1. اعمل keystore حقيقي:
   ```bash
   keytool -genkeypair -v -keystore enfkak-release.keystore -alias enfkak \
     -keyalg RSA -keysize 2048 -validity 10000
   ```
2. ضيف `signingConfigs { release { ... } }` في `app/build.gradle.kts`
   يشاور على الملف ده، واستخدم GitHub Secrets عشان الباسوردات بدل ما
   تكتبها صريحة في الكود.

---

## البيانات

بيانات المستخدم (المهام، الأهداف، النوم، الأرشيف...) بتتحفظ محليًا على
الجهاز عن طريق `localStorage` جوه الـ WebView (`domStorageEnabled = true`
في `MainActivity.kt`). مفيش سيرفر ولا حساب. فيه زرار تصدير/استيراد نسخة
احتياطية (JSON) جوه التطبيق نفسه لو حبيت تنقل بياناتك لجهاز تاني.

---

## ملاحظة صريحة

المشروع ده بنيته بنفس أدوات أندرويد الرسمية (Gradle + AGP + Kotlin) اللي
بتستخدم في أي مشروع أندرويد حقيقي — أوثق بكتير من أي أداة تغليف خارجية.
لكن زي أي كود، من غير ما أقدر أشغّل الـ build فعليًا هنا (معنديش Android
SDK/إنترنت كامل جوه بيئتي)، فرصة إن نسخة معينة من مكتبة تحتاج تحديث بسيط
في أول build لسه واردة. لو حصل خطأ، الـ log بتاع الـ Actions run هيوضح
بالظبط السطر والسبب — غالبًا حل بتحديث رقم إصدار (version) في
`app/build.gradle.kts` أو `build.gradle.kts` الرئيسي.
