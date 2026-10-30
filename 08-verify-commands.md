# المرحلة 08 — التحقق التشخيصي: أوامر حقيقية على الدومينات الفعلية

> **إيه دي؟** سايت تشخيصي (cheat sheet) بيحوله الـ agent لأوامر فعلية على دومينات مشروعك: TLS 1.3 + التشفير الكمي، رفض الإصدارات القديمة، الهيدرز الأمنية، تجديد certbot. دي أول مرحلة بتفتح اتصالات شبكة حقيقية — فمحتاجة موافقتك قبل التشغيل.
> **إمتى تستخدمها؟** بعد ما تخلّص الإصلاحات (أو في أي وقت عايز تأكد بنفسك إن السيرفر بيتصرف صح).

## برومبت التشغيل

```text
You are a TLS diagnostics engineer (October 2026 standards). Build and run a VERIFICATION SUITE adapted to this project's actual domains/hosts.

STEP 1 — Extract targets: from the project (configs, env files, docs, Stage 01 inventory), list every public hostname that serves HTTPS, and note which stage's findings each test will validate. Show me the list and WAIT for my explicit confirmation before running anything.

STEP 2 — Run this suite per host (requires OpenSSL 3.5+ locally — say so if unavailable, and skip what cannot run):
1. TLS 1.3 + post-quantum hybrid negotiation:
   openssl s_client -connect HOST:443 -servername HOST -tls1_3 -groups X25519MLKEM768:X25519
   PASS if output contains: Negotiated TLS1.3 group: X25519MLKEM768
   (This proves protection against "Harvest Now, Decrypt Later" quantum attacks.)
2. Legacy rejection (MUST be rejected by the SERVER, not by your own client):
   openssl s_client -connect HOST:443 -servername HOST -tls1_1 -cipher "DEFAULT:@SECLEVEL=0"
   openssl s_client -connect HOST:443 -servername HOST -tls1   -cipher "DEFAULT:@SECLEVEL=0"
   - PASS ✅: Server sends a TLS alert (e.g., "alert protocol version" or handshake failure from the remote server).
   - FAIL ❌: Handshake succeeds (e.g. output contains "Protocol: TLSv1.1" or negotiated legacy cipher) → Server accepts deprecated protocols!
   - ⚠️ INVALID: If output says "no protocols available", your local OpenSSL 3.x client refused to send the probe without reaching the server — ensure `@SECLEVEL=0` is included.
3. Certificate dates & EKU:
   echo | openssl s_client -connect HOST:443 -servername HOST 2>/dev/null | openssl x509 -noout -dates -ext extendedKeyUsage
   PASS: notAfter is within the 2026 validity regime (≤200 days), and EKU matches the role (ServerAuth for servers; a public cert must NOT be expected to carry ClientAuth — that is Stage 05's domain).
4. Security headers:
   curl -sI https://HOST | grep -iE "strict-transport-security|alt-svc"
   (Note: on Windows PowerShell, use `curl.exe` to avoid the PowerShell alias).
   PASS: HSTS with max-age=63072000 (+ includeSubDomains) is present; Alt-Svc advertises h3 if HTTP/3 is on.
5. ECH (optional, P2 — only if ECH deployment was claimed):
   curl -v --doh-url https://cloudflare-dns.com/dns-query --ech hard https://HOST
   (--ech hard forces ECH via encrypted DNS and refuses the connection if SNI encryption fails.)
6. ACME renewal health (only for hosts managed by certbot):
   certbot certificates && certbot renew --dry-run
   PASS: dry-run succeeds; confirm the client version supports ARI.
7. Replay guard (ONLY if 0-RTT/early data is enabled anywhere): send a POST request with header `Early-Data: 1` to the endpoint → expect HTTP 425 Too Early. A 2xx response = P0 replay hole.

RULES:
- These commands open REAL network connections. Do not run ANY of them until I explicitly confirm the target list. Never test hosts outside this project's own domains.
- Present results as a table: الاختبار | الأمر | المتوقع | النتيجة الفعلية | ✅/❌
- Every failure maps back to the stage that owns fixing it: 03 (server config) / 04 (certificates) / 05 (mTLS).

OUTPUT FORMAT (write the report in Arabic; keep technical terms in English):
## تقرير المرحلة 08 — نتائج التحقق التشخيصي
### ملخص: X من Y اختبار ناجح
### جدول النتائج (كما هو محدد فوق)
### إصلاحات مرتبطة (كل فشل → المرحلة المسؤولة عن إصلاحه + أولويته)
```

## ملاحظات مهمة قبل التشغيل
- **الاتصالات حقيقية**: البرومبت مصمم يحصل على موافقتك قبل أول أمر — متعطلش الحماية دي.
- **OpenSSL 3.5+ مطلوب** لاختبار `X25519MLKEM768` (على Windows استخدم WSL أو OpenSSL حديث).
- اختبار `--ech hard` اختياري — لو المشروع مش معلن ECH أصلاً اعتبره P2.
