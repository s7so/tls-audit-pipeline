# التقرير النهائي — جاهزية TLS/SSL في مشروع Uptime Kuma (أكتوبر 2026)

> نتيجة تشغيل TLS Audit Pipeline كامل على [louislam/uptime-kuma](https://github.com/louislam/uptime-kuma) — commit `73a50f9` (30 أكتوبر 2026)
> المرحلة 08 نُفذت بموافقة صريحة على 3 دومينات: `uptime.kuma.pet` · `demo.kuma.pet` · `weblate.kuma.pet`

## 1) جدول الجاهزية

| المجال | الحالة | P0 | P1 | P2 | أهم ملاحظة |
|---|---|---|---|---|---|
| 01 — الاكتشاف والتوثيق | ✅ | 0 | 0 | 0 | 14 نقطة TLS: سيرفر HTTPS اختياري، +10 عملاء (monitors/notifications)، Docker، healthcheck بـ Go |
| 02 — أنماط الكود | ⚠️ | 0 | 0 | 4 | **9 مواضع `rejectUnauthorized: false` — كلها خيارات يفعّلها المستخدم ومقفولة افتراضياً** (مش ثغرات) |
| 03 — إعدادات السيرفر | ⚠️ | 0 | 1 | 1 | سيرفر HTTPS المدمج بلا HSTS؛ الإعدادات = Node defaults |
| 04 — دورة حياة الشهادات | ➖ N/A | — | — | 2 | المشروع مش بيصدر شهادات (المشغّل بيوفرها)؛ ملاحظات على ميزة مراقبة الانتهاء نفسها |
| 05 — mTLS الداخلي | ➖ N/A | — | — | 1 | مفيش service-to-service؛ client certs ميزة للمستخدم في HTTP/WebSocket monitors |
| 06 — مخازن الثقة | ⚠️ | 0 | 1 | 1 | **جذور LE Gen Y بتتثبت في system store — لكن Node بيتجاهله** |
| 07 — الموبايل (Pinning) | ➖ N/A | — | — | — | لا يوجد كود موبايل |
| 08 — التحقق التشخيصي | ❌ | 0 | 1 | 0 | **الـ 3 دومينات بيقبلوا TLS 1.0 و 1.1 في الإنتاج** (Cloudflare) |

## 2) الحكم النهائي: 🟡 جاهز بشروط

مفيش أي P0: التحقق من الشهادات مفعّل افتراضياً في كل عميل، وكل تعطيل للتحقق قرار صريح من المستخدم لهدف معيّن. في 3 بنود P1 لازم يتصلحوا: TLS القديم على الإنتاج، وعدم تطابق مخزن الثقة مع Node، وغياب HSTS.

## نتائج المرحلة 08 — التحقق الحي (30 أكتوبر 2026)

الأدوات: OpenSSL 3.5.4 (Git for Windows) ✅ · `curl.exe` 8.13 / Schannel · certbot غير مثبت (مُسقط) · ECH مُسقط (غير مُعلن)

| الاختبار | uptime.kuma.pet | demo.kuma.pet | weblate.kuma.pet |
|---|---|---|---|
| TLS 1.3 + `X25519MLKEM768` | ✅ `Negotiated TLS1.3 group: X25519MLKEM768` | ✅ | ✅ |
| رفض TLS 1.1 (يجب أن يفشل) | ❌ **قُبل** — `ECDHE-ECDSA-AES128-SHA` | ❌ قُبل | ❌ قُبل |
| رفض TLS 1.0 (يجب أن يفشل) | ❌ **قُبل** | ❌ قُبل | ❌ قُبل |
| الصلاحية + EKU | ✅ 90 يوم (GTS WE1) — ServerAuth فقط | ✅ | ✅ |
| HSTS | ❌ غير موجود | ❌ غير موجود | ⚠️ `max-age=31536000` (أقل من 63072000) |
| Alt-Svc h3 | ✅ | ✅ | ✅ |

> ⚠️ **ملاحظة منهجية مهمة:** أمر `openssl s_client -tls1_1` العادي **بيفشل من ناحية العميل** على OpenSSL 3.x (`no protocols available` — بسبب SECLEVEL الافتراضي) قبل ما يوصل للسيرفر — ولو اتقرا كـ "مرفوض" هيدّي ✅ مزيف. النتيجة دي اتطلعت بـ `-cipher "DEFAULT:@SECLEVEL=0"` والسطر `Protocol: TLSv1.1` بيثبت إن السيرفر هو اللي قبل.

`uptime.kuma.pet` بالذات هو الـ endpoint اللي **كل** instance بتكلمه للتحقق من التحديثات (`server/check-version.js:11`).

## 3) خطة الإصلاح بالأولوية

### P1
1. **TLS 1.0/1.1 مفتوح على الـ 3 دومينات** (infra — Cloudflare): Dashboard → SSL/TLS → Edge Certificates → **Minimum TLS Version = 1.2** (أو 1.3). + تفعيل HSTS بـ `max-age=63072000; includeSubDomains`.
2. **عدم تطابق مخزن الثقة مع Node** — `docker/debian-base.dockerfile:58-60`
   - الصورة بتنزّل جذور ISRG Gen Y (`root-ye`, `root-yr`) وتشغّل `update-ca-certificates` — بس الـ HTTP monitors بتشتغل بـ Node، وNode بيستخدم مخزنه المدمج ومش بيقرا system store. (اتأكد محلياً: Node 24.12 فيه `ISRG Root X1/X2` بس — لازم يتأكد جوه `node:26-trixie-slim`.)
   - الأثر: موقع بسلسلة Gen Y هيظهر DOWN بخطأ شهادة ← المستخدم يفعّل "Ignore TLS" ← تعطيل حقيقي للتحقق سببه الصورة.
   - الإصلاح: `ENV NODE_USE_SYSTEM_CA=1` (Node ≥ 22.15 / 23.8) — أو `ENV NODE_EXTRA_CA_CERTS=/etc/ssl/certs/ca-certificates.crt`.
3. **سيرفر HTTPS المدمج بلا HSTS** — `server/uptime-kuma-server.ts:81-90`: لما `isSSL`، يتبعت `Strict-Transport-Security: max-age=63072000; includeSubDomains`.

### P2
4. `server/model/monitor.js:588` — `secureOptions: SSL_OP_LEGACY_SERVER_CONNECT` **على كل** HTTPS monitor (Node قفلها افتراضياً من v17): تبقى opt-in مع خيار Ignore TLS أو خيار مستقل.
5. `extra/healthcheck.go:26-28` — `InsecureSkipVerify: true` hardcoded. الهدف الافتراضي `127.0.0.1` ومفيش credentials بتتبعت ← خطر منخفض. الأنظف: التحقق ضد `SSL_CERT` نفسه كـ RootCAs.
6. تحذير ظاهر في الـ UI + سطر log لما أي خيار Ignore TLS يتفعل (`monitor.js:587`، `indigo.js:47`، `teltonika.js:51`، `smtp.js:34`، `redis.js:27`، `steam.js:58`، `websocket-upgrade.js:110`).
7. `test/manual-test-radius-tls/certs/redis.key` — مفتاح اختبار ملتزم؛ يتولد وقت التشغيل (`mkcert`).
8. ميزة مراقبة الانتهاء (`server/util-server.js:786`): `!certInfo.daysRemaining` بيرجع بدري لما الباقي **0 يوم** ← مفيش تنبيه في آخر يوم. + الافتراضي `[7, 14, 21]` هيطلع تنبيهات مع **كل** دورة لشهادات LE الـ 6 أيام.
9. `extra/healthcheck.go` و `uptime-kuma-server.ts`: الشهادة بتتقرا مرة واحدة عند التشغيل (`readFileSync`) ← التجديد محتاج restart؛ مع 47 يوم دي عملية متكررة.
10. Python/Apprise في الصورة بيستخدم `certifi` ومش system store — لو اتضاف CA داخلي: `ENV REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt`.

## 4) القواعد الذهبية الخمس مقابل المشروع

| # | القاعدة | الحالة | الإثبات |
|---|---|---|---|
| 1 | TLS 1.3 baseline + مفيش 0-RTT | ⚠️ | الكود: Node defaults (TLS 1.2+) بلا early data. الإنتاج: TLS 1.0/1.1 مقبولين (مرحلة 08) |
| 2 | X25519MLKEM768 | ✅ | اتأكد حياً على الـ 3 دومينات |
| 3 | ACME + ARI، لا إيميلات، لا OCSP | ➖ | المشروع مش بيصدر شهادات؛ لا OCSP في الكود (grep فاضي) |
| 4 | لا stapling، لا public CA كـ client cert | ✅ | client certs بيوفرها المستخدم (`monitor.js:668-676`) |
| 5 | لا تعطيل للتحقق | ✅ | كل التعطيلات opt-in ومقفولة افتراضياً (`monitor.js:327-329`)؛ الاستثناء الوحيد hardcoded هو healthcheck محلي (P2) |

## 5) إعادة الفحص بعد الإصلاح
- المرحلة 08 على الـ 3 دومينات (`-tls1_1 -cipher "DEFAULT:@SECLEVEL=0"` لازم يرجع alert).
- المرحلة 06 جوه الصورة: `docker run --rm louislam/uptime-kuma:2 node -p "require('tls').getCACertificates?.('system')?.length"`.
- المرحلة 03 على instance بـ `SSL_KEY/SSL_CERT` للتأكد من HSTS.

---
*مولّد بواسطة TLS Audit Pipeline (أكتوبر 2026) — كل البنود بإثبات file:line.*
