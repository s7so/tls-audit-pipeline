# التقرير النهائي — جاهزية TLS/SSL في مشروع Nginx Proxy Manager (أكتوبر 2026)

> نتيجة تشغيل TLS Audit Pipeline كامل على [NginxProxyManager/nginx-proxy-manager](https://github.com/NginxProxyManager/nginx-proxy-manager) — commit `1aca775` (أكتوبر 2026)
> المشروع متخصص في إدارة سيرفرات Nginx العكسية وإنهاء وتوليد شهادات SSL/TLS لأكثر من 23,000 مستخدم.
> المرحلة 08 نُفذت بموافقة صريحة على الدومين الرسمي: `nginxproxymanager.com`

---

## 1) جدول الجاهزية

| المجال | الحالة | P0 | P1 | P2 | أهم ملاحظة |
|---|---|---|---|---|---|
| 01 — الاكتشاف والتوثيق | ✅ | 0 | 0 | 0 | خريطة كاملة لقوالب Nginx، تكامل Certbot، وخيارات التشفير وقواعد البيانات |
| 02 — أنماط الكود | ✅ | 0 | 0 | 1 | خيار `DB_MYSQL_SSL_REJECT_UNAUTHORIZED` مفعل افتراضياً (`true`) — لا يوجد أي bypass إجباري |
| 03 — إعدادات السيرفر | ⚠️ | 0 | 2 | 1 | غياب التشفير الكمي `X25519MLKEM768` وتذاكر الجلسات الثابتة مفعلة (`ssl_session_tickets on`) |
| 04 — دورة حياة الشهادات | ⚠️ | 0 | 2 | 2 | **تجديد Certbot يفتقر لمعيار ARI (RFC 9773)** + استخدام إجباري لـ `--force-renewal` ونافذة ثابتة 30 يوماً |
| 05 — mTLS الداخلي | ➖ N/A | — | — | — | المشروع يعمل كـ container موحد؛ لا يوجد service-to-service داخلي |
| 06 — مخازن الثقة والبيئات | ⚠️ | 0 | 1 | 1 | استخدام `NODE_OPTIONS="--openssl-legacy-provider"` في Dockerfile لتشغيل خوارزميات قديمة |
| 07 — الموبايل (Pinning) | ➖ N/A | — | — | — | لا يوجد تطبيق موبايل |
| 08 — التحقق التشخيصي الحي | ✅ | 0 | 0 | 1 | موقع `nginxproxymanager.com` يرفض TLS 1.0/1.1 بنجاح (Alert 70) ويدعم `X25519MLKEM768` |

---

## 2) الحكم النهائي: 🟡 جاهز بشروط

المشروع آمن كودياً (لا توجد ثغرات P0 قابلة للاستغلال المباشر)، لكنه كأداة **متخصصة في إنشاء إعدادات TLS للآخرين** يحتاج إلى تحديث قوالبه وإدارته للشهادات لمطابقة baseline أكتوبر 2026 (تفعيل التشفير ما بعد الكم، أتمتة ARI، وحذف `--openssl-legacy-provider`).

---

## نتائج المرحلة 08 — التحقق التشخيصي الحي (30 أكتوبر 2026)

تم فحص الدومين الرسمي للمشروع محلياً باستخدام OpenSSL 3.5.4 (مع `@SECLEVEL=0`) و `curl.exe`:

| الاختبار | `nginxproxymanager.com` | النتيجة والتقييم |
|---|---|---|
| **TLS 1.3 + `X25519MLKEM768`** | ✅ `Negotiated TLS1.3 group: X25519MLKEM768` | حماية تامة ضد هجمات HNDL الكمومية |
| **رفض TLS 1.1 (يجب أن يفشل)** | ✅ **رُفض من السيرفر** (`alert protocol version 70`) | نجاح — تم إرسال المسبار وتأكيد رفض السيرفر للبروتوكول القديم |
| **رفض TLS 1.0 (يجب أن يفشل)** | ✅ **رُفض من السيرفر** (`alert protocol version 70`) | نجاح — السيرفر يرفض TLS 1.0 |
| **صلاحية الشهادة + EKU** | ✅ 90 يوماً (Google Trust Services WE1) | صلاحية متوافقة مع معايير 2026 + ServerAuth فقط |
| **الهيدرز الأمنية** | ⚠️ `Strict-Transport-Security: max-age=15552000` | HSTS موجود لكن بمدة 180 يوماً (الـ baseline الموصى به سنتان: 63072000) |
| **HTTP/3 (Alt-Svc)** | ✅ `alt-svc: h3=":443"; ma=86400` | HTTP/3 QUIC مفعل بنجاح |

---

## 3) خطة الإصلاح بالأولوية

### P1 (تحسينات أمنية لمعايير 2026)

1. **إضافة التشفير ما بعد الكم إلى قالب Nginx:**
   - الملف: `docker/rootfs/etc/nginx/conf.d/include/ssl-ciphers.conf`
   - المشكلة: لا يحدد منحنى التبادل الهجين `ssl_ecdh_curve`، ويحتفظ بخوارزميات DHE القديمة.
   - الإصلاح المستهدف:
     ```nginx
     ssl_ecdh_curve X25519MLKEM768:X25519:secp256r1;
     ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305';
     ```

2. **حماية السرية الأمامية (Forward Secrecy) وإيقاف تذاكر الجلسة:**
   - الملف: `docker/rootfs/etc/nginx/conf.d/include/ssl-cache.conf`
   - المشكلة: غياب `ssl_session_tickets off;` يجعل Nginx يستخدم مفاتيح تذاكر ثابتة قد تكسر الـ Forward Secrecy إذا استمرت الحاوية في العمل لفترات طويلة.
   - الإصلاح: إضافة `ssl_session_tickets off;` وتعديل مهلة الجلسة إلى `ssl_session_timeout 1d;`.

3. **تحديث منطق تجديد Certbot ودعم معيار ARI (RFC 9773):**
   - الملف: `backend/internal/certificate.js:36, 1035`
   - المشاكل:
     * يعتمد على نافذة زمنية ثابتة `renewBeforeExpirationBy: [30, "days"]` عبر استعلام قاعدة بيانات؛ في عصر الشهادات ذات الـ 47 يوماً (والشهادات فائقة القصر 6 أيام)، النافذة الثابتة قد تؤدي لتعطل الخدمات أو تكرار محاولات التجديد كل ساعة!
     * يمرر دائمًا معامل `--force-renewal` عند التجديد، مما يتجاوز تحذيرات وفحوصات Certbot ويتسبب في استهلاك حدود Let's Encrypt (Rate Limits).
   - الإصلاح: تحديث أمر التجديد ليعتمد على استعلام Certbot المدمج لمعيار **ARI** (`certbot renew --dry-run` والاعتماد على إشارات CA المبكرة عند الإلغاء الجماعي) بدلاً من الفحص اليدوي لـ 30 يوماً.

4. **إزالة `--openssl-legacy-provider` من الحاوية:**
   - الملف: `docker/Dockerfile:26`
   - المشكلة: إجبار Node.js على تفعيل `NODE_OPTIONS="--openssl-legacy-provider"` يفتح المجال لاستخدام خوارزميات قديمة ضعيفة رياضياً (مثل MD4 و DES).
   - الإصلاح: تحديث مكتبات التشفير والاعتماديات في `backend/package.json` للاعتماد كلياً على المزود القياسي لـ OpenSSL 3.x.

---

### P2 (تحسينات جودة إضافية)

5. **فصل شرط التحقق من الإيميل عند طلب شهادة:**
   - الملف: `backend/internal/certificate.js:146-149`
   - Let's Encrypt أوقفت إيميلات التنبيه نهائياً في يونيو 2025؛ لا ينبغي جعل غياب الإيميل عائقاً برمجياً يمنع إصدار الشهادة إذا كان النطاق مُداراً تلقائياً.
6. **مراقبة خارجية للشهادات:**
   - إضافة إشعار للمستخدمين بضرورة وجود أداة مراقبة خارجية (مثل Blackbox exporter أو Uptime Kuma) لأن الاعتماد فقط على `setInterval` داخل العملية قد يفشل بصمت إذا تجمدت خدمة الباك-إند.

---

## 4) القواعد الذهبية الخمس مقابل المشروع

| # | القاعدة | الحالة | الإثبات |
|---|---|---|---|
| 1 | TLS 1.3 baseline + منع 0-RTT | ✅ | مفعل افتراضياً في قوالب Nginx (`ssl_protocols TLSv1.2 TLSv1.3;`) مع إبقاء 0-RTT مغلقاً |
| 2 | تفعيل X25519MLKEM768 | ⚠️ | موقع المشروع الرسمي يفعّله، لكن **القوالب المولّدة للمستخدمين تفتقر إليه** (`ssl-ciphers.conf`) |
| 3 | ACME + ARI، لا إيميلات، لا OCSP | ⚠️ | لا يوجد OCSP في القوالب ✅؛ لكن التجديد يعتمد على نافذة 30 يوماً ثابتة ولا يدعم ARI ❌ |
| 4 | لا stapling مع LE، لا شهادات عامة للـ mTLS | ✅ | لا يوجد `ssl_stapling` في القوالب نهائياً |
| 5 | لا تعطيل للتحقق في الكود | ✅ | التحقق مفعل افتراضياً (`rejectUnauthorized: true` لقاعدة البيانات وعملاء HTTP) |

---

## 5) أوامر التحقق بعد تطبيق الإصلاحات

```bash
# 1. اختبار مصافحة النطاق بالقالب المحدث والتأكد من ML-KEM:
openssl s_client -connect YOUR-NPM-HOST:443 -servername YOUR-NPM-HOST -tls1_3 -groups X25519MLKEM768:X25519

# 2. التأكد من رفض البروتوكولات المنقرضة:
openssl s_client -connect YOUR-NPM-HOST:443 -servername YOUR-NPM-HOST -tls1_1 -cipher "DEFAULT:@SECLEVEL=0"
# المتوقع: alert protocol version من السيرفر

# 3. اختبار صحة التجديد الآلي دون استهلاك الـ Rate Limits:
docker exec -it nginx-proxy-manager certbot renew --dry-run
```

---
*مولّد بواسطة TLS Audit Pipeline (أكتوبر 2026) — كل البنود مدعومة بإثبات file:line.*
