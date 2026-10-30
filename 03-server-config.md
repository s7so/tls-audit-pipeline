# المرحلة 03 — إعدادات السيرفر والـ Gateway

> **إيه دي؟** بتدقق أي نقطة TLS termination في المشروع (Nginx / Caddy / Traefik / HAProxy / Ingress...) مقابل baseline أكتوبر 2026: TLS 1.3، التشفير ما بعد الكم `X25519MLKEM768`، قفل 0-RTT أو تأمينه، وإزالة `ssl_stapling` الميت.
> **إمتى تستخدمها؟** لو المرحلة 01 طلّعت سيرفرات أو reverse proxies في المشروع.

## برومبت الفحص

```text
You are a senior TLS/SSL security auditor (October 2026 standards). Audit every server-side TLS termination point in this project: Nginx, Caddy, Traefik, HAProxy, Envoy, Apache, IIS, Kubernetes Ingress/Gateway, cloud load balancer configs, and any TLS-terminating middleware. READ-ONLY: report only.

2026 REFERENCE BASELINE (audit against THIS, point by point):
1. Protocols: TLS 1.3 primary. SSL v1–v3 and TLS 1.0/1.1 must be ABSENT. TLS 1.2 only if legacy clients truly require it, and then restricted to ECDHE + AEAD ciphers only (AES-GCM / CHACHA20-POLY1305) with server cipher preference off.
2. Post-quantum hybrid key exchange: the server should negotiate `X25519MLKEM768` (protection against "Harvest Now, Decrypt Later" quantum attacks). Requires OpenSSL 3.5+ under Nginx. Missing support = P1 (not P0).
3. 0-RTT / early data: must be OFF unless the backend is proven replay-safe. If enabled: the proxy MUST pass the `Early-Data` header to the backend AND the backend MUST reject non-idempotent requests (POST/PUT/DELETE) carrying `Early-Data: 1` with HTTP 425 Too Early. 0-RTT enabled without the 425 guard = P0 (replay attacks).
4. Forward secrecy: session tickets OFF (static tickets break PFS); a session cache is fine.
5. OCSP: `ssl_stapling` / OCSP must be ABSENT for Let's Encrypt certificates — LE shut down OCSP in Aug 2025, so stapling now only produces log errors. Revocation in 2026 = CRLite / CRLs.
6. HSTS: `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload` with `always` (persistent) header. Plus `Alt-Svc: h3=":443"` if HTTP/3 is enabled.
7. HTTP/3 (QUIC): enabled where the stack supports it.
8. ECH (Encrypted Client Hello): optional — note as P2 whether the stack supports it (Nginx 1.29.4+ / OpenSSL 4.0, Caddy natively).
9. Certificate paths referenced by the server must exist, and must not be near expiry (deep cert lifecycle is Stage 04's job — only flag what is visible in the config).
10. Caddy/Traefik: verify they were NOT manually downgraded. Caddy's defaults (TLS 1.3, ML-KEM, ACME + ARI auto-renewal) are already correct for 2026 — flag any override that weakens them.

For every server config found: list each setting that matches or violates the baseline, with file:line evidence.

RULES:
- READ-ONLY. CITE EVIDENCE. No network commands (live testing is Stage 08).
- If there are no server-side TLS configs in the repo, mark ➖ غير منطبق (N/A) and add one line on what likely terminates TLS outside the repo (e.g. a cloud load balancer).

OUTPUT FORMAT (write the report in Arabic; keep technical terms in English):
## تقرير المرحلة 03 — إعدادات السيرفر والـ Gateway
### الجاهزية: ✅ | ⚠️ | ❌ | ➖ N/A
### الموجود حالياً (جدول: النقطة | الملف:السطر | الإعداد الحالي | الحكم مقابل الـ baseline)
### الناقص أو المخالف (كل بند بأولوية P0/P1/P2)
### المطلوب عمله (بالترتيب، مع الإعداد المستهدف النهائي لكل سطر)
```

## برومبت الإصلاح

```text
Based on the Stage 03 report, fix all server TLS configurations to the October 2026 baseline.

RULES OF ENGAGEMENT:
- Show the complete corrected config file(s) as diffs first; apply only after my approval.
- Canonical Nginx 2026 template (adapt domains, paths, and ports to this project):
  * listen 443 ssl + listen [::]:443 ssl + (optional) listen 443 quic reuseport, http2 on
  * ssl_protocols TLSv1.3;  (add TLSv1.2 ONLY if legacy support is truly required)
  * ssl_ecdh_curve X25519MLKEM768:X25519:secp256r1;
  * if TLS 1.2 kept: ssl_ciphers limited to ECDHE + AES-GCM / CHACHA20-POLY1305 set, ssl_prefer_server_ciphers off;
  * ssl_session_timeout 1d; ssl_session_cache shared:TLS:10m; ssl_session_tickets off;
  * ssl_early_data off;  (do not enable unless the backend rejects POST/PUT/DELETE carrying `Early-Data: 1` with HTTP 425 — and then also patch that backend and pass `proxy_set_header Early-Data $ssl_early_data;`)
  * add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
  * add_header Alt-Svc 'h3=":443"; ma=86400' always;
  * NO ssl_stapling directive at all with Let's Encrypt certificates.
- For Caddy: plain site block with `tls { protocols tls1.3 }` + the HSTS header — Caddy handles X25519MLKEM768, ACME ARI, and renewal automatically.
- For each change, give the exact validation command after reload (e.g. `nginx -t && systemctl reload nginx`).
```

## أوامر التحقق بعد الإصلاح

```bash
# TLS 1.3 + التشفير ما بعد الكم (يحتاج OpenSSL 3.5+ محلياً):
openssl s_client -connect YOUR-DOMAIN:443 -tls1_3 -groups X25519MLKEM768:X25519
# المطلوب تشوفه في المخرجات: Negotiated TLS1.3 group: X25519MLKEM768

# الإصدارات القديمة لازم ترفض — من ناحية السيرفر مش من ناحيتك:
openssl s_client -connect YOUR-DOMAIN:443 -servername YOUR-DOMAIN -tls1_1 -cipher "DEFAULT:@SECLEVEL=0"
openssl s_client -connect YOUR-DOMAIN:443 -servername YOUR-DOMAIN -tls1   -cipher "DEFAULT:@SECLEVEL=0"
# ✅ صح: alert من السيرفر (مثلاً "alert protocol version").
# ❌ فشل: ظهر "Protocol: TLSv1.1" أو "Cipher is ECDHE-..." = السيرفر قبل الإصدار القديم.
# ⚠️ لو ظهر "no protocols available" = العميل هو اللي رفض (OpenSSL 3.x) — النتيجة باطلة، مش نجاح.

# الهيدرز الأمنية (على Windows PowerShell استخدم curl.exe — كلمة curl لوحدها alias لـ Invoke-WebRequest):
curl -sI https://YOUR-DOMAIN | grep -iE "strict-transport|alt-svc"
```
