# سجل التعديلات — تحويل متجر EGASA إلى SSG + SWR جاهز لـ Cloudflare Pages

التاريخ: 2026-11-06
الهدف: متجر سريع بدون وميض (no-flash)، محتوى حقيقي من أول بايت، منشور على Cloudflare Pages، مع الإبقاء على Firebase كخلفية حية (Real-time).

---

## ملفات أُضيفت

### 1. `src/lib/snapshot.ts` (جديد)
وحدة قراءة "لقطة النشر" (Published Snapshot) المضمّنة في HTML عند البناء.
- `getPublishedSnapshot()`: تقرأ `<script type="application/json" id="egasa-published-snapshot">` أو `window.__EGASA_SNAPSHOT__` بشكل متزامن قبل أول رسم لـ React، وتعيد `{products, settings}` مصححة الأنواع، أو `null` في وضع التطوير.
- `hasPublishedSnapshot()`: هل توجد لقطة مضمّنة.
- `getProductSlugFromPath()`: تستخرج slug المنتج من المسار النظيف `/product/<slug>` للتوجيه المباشر.
- تنظيف وتعقيم للبيانات (sanitize) لضمان توافق الأنواع.

### 2. `scripts/prerender.mjs` (جديد)
سكربت توليد ثابت (SSG) يعمل كـ postbuild بعد `vite build`.
- يقرأ `data/store_state.json` (لقطة النشر canonical).
- يحقن اللقطة في `dist/index.html` كـ `<script type="application/json" id="egasa-published-snapshot">` + `window.__EGASA_SNAPSHOT__`.
- يستبدل سبنر التحميل داخل `#root` بمحتوى HTML حقيقي للصفحة الرئيسية (شريط إعلان + بانر + بطاقات منتجات حقيقية بأسماء وأسعار وصور) — لا يظهر السبنر أبداً للمستخدم النهائي.
- يولّد صفحة مستقلة ثابتة لكل منتج: `dist/product/<slug>/index.html` بمحتوى المنتج الكامل (صورة، وصف، مميزات، مقاسات، ألوان، عرض باقة، استمارة طلب بصرية) + عنوان ووصف و canonical و og:image خاصين بالمنتج.
- يمسح مجلد `dist/product` قبل التوليد لتجنب صفحات قديمة لمنتجات حُذفت.
- يطبع ملخصاً بعدد الصفحات المولدة.

### 3. `public/_redirects` (جديد — يُنسخ إلى dist تلقائياً)
- قاعدة صريحة: `/product/:slug → /product/:slug/index.html 200` (الملفات الثابتة لها أولوية أصلاً في Cloudflare Pages).
- SPA fallback: `/* /index.html 200` (للوحة التحكم والروابط القديمة `?p=`).

### 4. `public/_headers` (جديد — يُنسخ إلى dist تلقائياً)
- `Cache-Control: public, max-age=0, must-revalidate` لملفات HTML (لتنتشر تحديثات اللقطة فوراً).
- `Cache-Control: public, max-age=31536000, immutable` لأصول `/assets/*` (المهشمة) و`/uploads/*`.
- رؤوس أمان أساسية: `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`.

---

## ملفات عُدّلت

### 5. `src/App.tsx` (معدّل)
- **تهيئة الحالة من اللقطة أولاً**: `storeSettings` و`products` يقرآن اللقطة المضمّنة قبل localStorage وقبل أي قيمة افتراضية — هذا يلغي وميض المنتج التجريبي/الفراغ.
- **لا منتج تجريبي على واجهة المتجر**: عند غياب اللقطة و localStorage فارغ (وضع تطوير)، يُعرض skeleton نظيف (شبكة بطاقات وهمية) حتى ترد بيانات Firestore، بدلاً من `INITIAL_PRODUCTS`. `INITIAL_PRODUCTS` يبقى متاحاً فقط لزر "استعادة البيانات التجريبية" في لوحة التحكم.
- **حالة `firstDataReceived`**: تُضبط عند أول رد من `subscribeProducts` (onSnapshot) لإخفاء الـ skeleton.
- **توجيه بالمسار النظيف `/product/<slug>`**: أُضيف إلى `getInitialProductFromUrl` و`isRoutePending` وتأثير توجيه URL — بالتوافق مع الصفحات الثابتة المولدة، إلى جانب `?p=` و`?product=` و`?id=` و`#product/slug` القائمين.
- **إغلاق صفحة المنتج**: عند العودة للرئيسية من مسار `/product/<slug>` يُعاد التوجيه إلى `/` بدلاً من الإبقاء على المسار.
- **SWR محفوظ**: اشتراكات `subscribeProducts`/`subscribeSettings` (onSnapshot) تبقى تحدّث الحالة في الخلفية بصمت — تعديلات المشرف تظهر للمستخدمين خلال ثانية دون إعادة نشر.
- **دالة `handleExportSnapshot`**: تنزّل `store_state.json` يحتوي products+orders+settings الحالية من Firestore، وتُمرر للوحة التحكم.

### 6. `src/components/admin/AdminDashboard.tsx` (معدّل)
- أُضيف خاصية `onExportSnapshot?: () => void` ووُصِلت إلى `ProductsTab`.

### 7. `src/components/admin/ProductsTab.tsx` (معدّل)
- أُضيف زر **"تصدير لقطة النشر"** (بلون teal مميز) بجانب "تصدير كود المنتجات"، ينفّذ `onExportSnapshot` لتنزيل `store_state.json`.

### 8. `package.json` (معدّل)
- `"build": "vite build && node scripts/prerender.mjs"` (التوليد الثابت تلقائي بعد البناء).
- أُضيف `"build:cf": "npm run build"` كاسم مستعار للنشر على Cloudflare.
- تبقى `dev` / `preview` / `lint` / `clean` / `deploy` شغالة.

---

## ملفات لم تُمس (وفقاً للقيود الصارمة)
- `firebase-applet-config.json`: مفاتيح Firebase العامة كما هي بدون أي تغيير (projectId `egasa-87fcd`).
- `firebase.json` / `.firebaserc`: تُركت كمرجع.
- `src/lib/firebase.ts` / `src/db/store.ts`: لم تُعدّل — استدعاءات Firestore الحية (`subscribeProducts`/`subscribeSettings`/`onSnapshot`) قائمة.
- مجلد `mowarid` بالكامل: لم يُمس إطلاقاً.
- `src/data/initialData.ts`: `INITIAL_PRODUCTS` باقية لزر استعادة البيانات التجريبية في لوحة التحكم فقط.

---

## نتيجة التحقق
- `npx tsc --noEmit`: يمر بدون أخطاء.
- `npm run build` (من نظيف): ينجح — يولّد `dist/index.html` + `dist/product/<slug>/index.html` لكل منتج.
- `vite preview` + curl: `/` و`/product/<slug>` يعيدان HTTP 200 ويحتويان اسم المنتج الحقيقي والسعر واللقطة المضمّنة في HTML قبل أي جافاسكربت؛ السبنر غير موجود.
- إعدادات Firebase واستدعاءات `onSnapshot`/`subscribeProducts` موجودة في الحزمة النهائية.
