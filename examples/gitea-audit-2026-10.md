# التقرير النهائي — جاهزية TLS/SSL في مشروع Gitea (أكتوبر 2026)

> نتيجة تشغيل TLS Audit Pipeline كامل على [go-gitea/gitea](https://github.com/go-gitea/gitea) (clone: `D:\test\demo\gitea`)
> المرحلة 08 (اختبارات الشبكة الحية) لم تُنفذ — تحتاج موافقة صريحة على قائمة الدومينات.

## 1) جدول الجاهزية

| المجال | الحالة | P0 | P1 | P2 | أهم ملاحظة |
|---|---|---|---|---|---|
| 01 — الاكتشاف والتوثيق | ✅ | 0 | 0 | 0 | خريطة كاملة لـ 11 نقطة TLS |
| 02 — أنماط الكود | ⚠️ | 0 | 1 | 2 | bypass واحد hardcoded (محصور بالـ loopback) |
| 03 — إعدادات السيرفر | ⚠️ | 0 | 1 | 2 | **مفيش HSTS نهائياً** |
| 04 — دورة حياة الشهادات | ✅ | 0 | 0 | 2 | certmagic v0.25.6 = ARI كامل ✅ |
| 05 — mTLS الداخلي | ➖ N/A | — | — | 1 | مفيش mTLS داخلي أصلاً (unix socket/HTTP محلي) |
| 06 — مخازن الثقة | ✅ | 0 | 0 | 2 | `ca-certificates` مثبت في الصورتين |
| 07 — الموبايل (Pinning) | ➖ N/A | — | — | — | لا يوجد كود موبايل في المشروع |
| 08 — التحقق التشخيصي | ⚠️ | 0 | 1 | 0 | اكتشاف حي: `dl.gitea.com` يقبل TLS قديم (≤1.1) + تأكيد غياب HSTS في الإنتاج |

## 2) الحكم النهائي: 🟡 جاهز بشروط

### نتائج المرحلة 08 — التحقق التشخيصي الحي (نُفذت 30 أكتوبر 2026)

الأدوات المحلية: OpenSSL 3.5.4 ✅ (يدعم اختبار ML-KEM)، curl 8.17/Schannel (اختبار ECH مُسقط — غير مدعوم في هذه الحزمة)، certbot غير مثبت (مُسقط).

| الاختبار | try.gitea.io | gitea.com | dl.gitea.com |
|---|---|---|---|
| DNS | ❌ لا يستجيب | ✅ | ✅ |
| TLS 1.3 + `X25519MLKEM768` | ⏭ تُسقط | ✅ `Negotiated TLS1.3 group: X25519MLKEM768` | ✅ `Negotiated TLS1.3 group: X25519MLKEM768` |
| رفض TLS 1.1 (يجب أن يفشل) | ⏭ تُسقط | ✅ مرفوض — alert 70 من السيرفر نفسه | ❌ **المصافحة نجحت** بـ `ECDHE-RSA-AES128-SHA` (قبل TLS 1.2) — **P1 حي على الـ infra** |
| صلاحية الشهادة + EKU | ⏭ تُسقط | ✅ ~197 يوماً (ضمن حد 200) + ServerAuth فقط | ✅ 90 يوماً (Google Trust Services WR1) |
| HSTS / Alt-Svc | ⏭ تُسقط | ❌ **غير موجود** (`Server: Caddy`) — تأكيد حي لنتيجة المرحلة 03 | — |

**الاستنتاج التشخيصي:** المرحلة 08 أمسكت بما لا يراه فحص الريبو (إعدادات الـ infra خارج الكود)، وأكدت نتيجة الريبو ضد الإنتاج (غياب HSTS رغم تشغيل Caddy — لأن HSTS ليس تلقائياً في Caddy).

السبب: مفيش أي P0 (مفيش ثغرة قابلة للاستغلال مباشرة، والتحقق مفعّل افتراضياً في كل العملاء الخارجيين) — لكن فيه P1 اتنين لازم يتصلحوا قبل ما التقييم يبقى 🟢: الـ bypass الـ hardcoded في الـ internal API، وغياب HSTS.

## 3) خطة الإصلاح بالأولوية

### P1 (قبل الإصدار القادم)
1. **`modules/private/internal.go:77` — `InsecureSkipVerify: true` hardcoded**
   - السياق: الاتصال من git hook لـ internal API لما يكون loopback/unix — فيه guard (`internalAPIConnectionIsLocal`, سطر 58–72) و`INTERNAL_TOKEN` حماية، و`ServerName: domain` مضبوط.
   - الإصلاح: بما إن `ServerName` مضبوط على الدومين العام، التحقق الكامل يشتغل مع شهادة ACME — استبدال بسطر `return &tls.Config{}` (نفس سلوك non-local) أو التحقق ضد `TrustedRoots` من certmagic. لو الخوف من self-signed للمستخدمين: التحقق المشروط بوجود شهادة صالحة مع fallback موثّق.
2. **إضافة HSTS** — لا يوجد أي `Strict-Transport-Security` في الكود كله (grep فاضي).
   - الإصلاح: خيار `[server] HSTS_MAX_AGE` (0 = off) يبعت الهيدر لما `PROTOCOL=https`: `Strict-Transport-Security: max-age=63072000; includeSubDomains` — أو على الأقل توثيق إلزامي لقاعدة proxy مع HSTS.

### P2 (تحسينات)
3. تقييد `SSL_MIN_VERSION` (`cmd/web_https.go:17-22`): القبول الصريح لـ `tlsv1.0/1.1` في الـ map — إزالتهما أو رفع تحذير Fatal.
4. تحذير/سجل تدقيق عند تفعيل أي `SKIP_TLS_VERIFY` (webhook/migrations/incoming email/LDAP/MinIO) — كلها off افتراضياً ✅.
5. `tests/integration/private-testing.key` — مفتاح اختبار ملتزم بالريبو؛ استبداله بمفتاح يتولد وقت التشغيل.
6. FIXME في `cmd/web_acme.go:63-66` — مسار تخزين شهادات ACME على `AppWorkPath` بدل `AppDataPath`.
7. توثيق `mkcert` للتطوير المحلي بجانب أمر `gitea cert` المدمج (`cmd/cert.go`).
8. مراقبة انتهاء شهادات مستقلة خارجية (قرار تشغيلي لمن يشغل self-host).

## 4) القواعد الذهبية الخمس مقابل المشروع

| # | القاعدة | الحالة | الإثبات |
|---|---|---|---|
| 1 | TLS 1.3 baseline + مفيش 0-RTT بلا replay guard | ✅ | Go defaults (Go 1.27, `go.mod:3`) — MinVersion الافتراضي TLS 1.2 (`cmd/web_https.go:79`)؛ Go stdlib لا يدعم 0-RTT أصلاً فلا خطر replay |
| 2 | X25519MLKEM768 negotiation | ✅ | Go 1.24+ يفعّل ML-KEM افتراضياً؛ الـ curve map بيعرض `x25519mlkem768/mlkem1024` صراحة (`cmd/web_https.go:39-42`) |
| 3 | ACME + ARI، لا إيميلات، لا OCSP | ✅ | certmagic v0.25.6 (`go.mod`) دعم ARI كامل (RFC 9773) + `AcmeProfile` للشهادات قصيرة العمر (`cmd/web_acme.go:80`)؛ لا OCSP ولا stapling في الكود |
| 4 | لا stapling مع LE، لا public CA كـ client cert | ✅ | لا يوجد stapling (Go server)؛ شهادة العميل الوحيدة اختيارية للـ SMTP (`modules/setting/mailer.go:44-46`) ومصدرها يحدده المشغّل |
| 5 | لا تعطيل للتحقق / mkcert محلياً / SPKI pinning | ⚠️ | كل العملاء verification ON افتراضياً؛ استثناء واحد P1 محصور بالـ loopback (`modules/private/internal.go:77`)؛ مفيش pinning أصلاً (مفيش موبايل) |

## 5) إيه اللي يتعاد فحصه بعد الإصلاح؟
- المرحلة 02 على `modules/private/internal.go` بعد الإصلاح.
- المرحلة 03 للتحقق من هيدر HSTS.
- المرحلة 08 (اختبارات الشبكة الحية: `openssl s_client -groups X25519MLKEM768:X25519`, `curl -sI` للهيدرز) على instance حقيقية بعد النشر.

---
*مولّد بواسطة TLS Audit Pipeline (أكتوبر 2026) — كل البنود بإثبات file:line. المرجع: قواعد 2026 من الدليل الشامل لبروتوكول TLS.*
