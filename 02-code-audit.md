# المرحلة 02 — تدقيق الكود: أنماط تعطيل التحقق القاتلة

> **إيه دي؟** أهم مرحلة أمنياً في الـ pipeline كله. بتدور على أي كود بيقفل التحقق من الشهادات (`rejectUnauthorized: false` في Node، `verify=False` في Python، `InsecureSkipVerify` في Go...) — دي ثغرة بتسمح لأي حد على الشبكة يقرأ ويعدّل الترافيك "المشفّر". وبتقفل كمان على الحلول الصح: `mkcert` محلياً و`NODE_EXTRA_CA_CERTS` في الإنتاج.
> **إمتى تستخدمها؟** مباشرة بعد المرحلة 01 (دايماً منطبقة — أي مشروع فيه كود).

## برومبت الفحص

```text
You are a senior TLS/SSL security auditor (October 2026 standards). Audit this project's source code, scripts, and configuration for DANGEROUS TLS VERIFICATION BYPASS patterns. This is a READ-ONLY audit: report only, fix nothing.

2026 CONTEXT (facts):
- Disabling certificate verification destroys TLS entirely: any network attacker can read and modify "encrypted" traffic (MITM). There is NO acceptable environment for this in committed code — not dev, not CI, not "just for testing".
- The correct alternatives are: mkcert for local development; NODE_EXTRA_CA_CERTS (Node), SSL_CERT_FILE / REQUESTS_CA_BUNDLE (Python/Go) or installing the internal root CA inside the image (Docker) for private CAs. Never bypass verification.

SEARCH FOR ALL OF THESE PATTERNS (adapt to every language actually present in the project — also search .env files, Dockerfiles, CI configs, test setup files, and scripts):
- Node.js / JavaScript / TypeScript: `rejectUnauthorized: false` (https.Agent, fetch dispatcher, axios httpsAgent), `process.env.NODE_TLS_REJECT_UNAUTHORIZED = '0'` anywhere including jest/vitest setup files, `tls.connect` with rejectUnauthorized false, custom agents with insecure options.
- Python: `verify=False` (requests, httpx), `ssl._create_unverified_context()`, `ssl.CERT_NONE`, `check_hostname = False`, aiohttp with `ssl=False`.
- Go: `InsecureSkipVerify: true` in tls.Config.
- Java/Kotlin: a TrustManager accepting all certificates, custom SSLContext with empty trust, X509TrustManager with empty `checkServerTrusted`.
- .NET: `ServerCertificateCustomValidationCallback` returning true, `DangerousAcceptAnyServerCertificateValidator`, `ServicePointManager.ServerCertificateValidationCallback` overrides.
- PHP: `CURLOPT_SSL_VERIFYPEER => false`, `CURLOPT_SSL_VERIFYHOST => 0`, stream context `verify_peer => false`.
- Ruby: `verify_mode: OpenSSL::SSL::VERIFY_NONE`.
- Rust: `danger_accept_invalid_certs(true)`.
- Shell / CI / Docker: `curl -k` / `--insecure`, `wget --no-check-certificate`, `GIT_SSL_NO_VERIFY`, `npm config set strict-ssl false`, `pip install --trusted-host`, any exported env var disabling verification.
- PARTIAL WEAKENING (verification still on, but TLS is degraded — the 2026 baseline bans SSL and TLS 1.0/1.1): `minVersion: 'TLSv1'` / `'TLSv1.1'`, `secureProtocol: 'TLSv1_method'` (or any legacy method), `secureOptions` containing `SSL_OP_LEGACY_SERVER_CONNECT` or `SSL_OP_ALLOW_UNSAFE_LEGACY_RENEGOTIATION`, cipher strings with `@SECLEVEL=0` / `@SECLEVEL=1` / `NULL` / `RC4` / `3DES`, `checkServerIdentity: () => undefined` (hostname check disabled), Go `MinVersion: tls.VersionTLS10/11`, Python `ssl.PROTOCOL_TLSv1` / `minimum_version = TLSVersion.TLSv1`. Default severity P2; P1 if applied unconditionally to all outbound traffic.
- Also flag as P1: hardcoded certificate paths that will break in another environment, and expired or self-signed certificates committed to the repo (paths only — do not print contents).

RULES:
- READ-ONLY. CITE EVIDENCE (file:line) for every finding and quote the offending line.
- Never print private keys or secrets.
- Severity — classify every bypass by WHO turns it on. P0 must mean "exploitable as shipped"; inflating severity is a false alarm:
  * P0: verification disabled UNCONDITIONALLY (hardcoded) or ON BY DEFAULT on a path carrying real traffic, credentials, or secrets — including `NODE_TLS_REJECT_UNAUTHORIZED=0` / `verify=False` set in committed code, env files, Dockerfiles, or CI.
  * P1: hardcoded bypass confined to loopback/localhost or carrying no credentials (say why it is confined, with evidence); OR an end-user "ignore TLS errors" option that is OFF by default but has NO safe alternative next to it (no way to supply a custom CA / private root) — this pushes users into disabling verification.
  * P2: an end-user opt-in option that is OFF by default AND a safe alternative exists next to it (custom CA field, documented NODE_EXTRA_CA_CERTS). Recommend a visible warning in the UI + a log line when it is enabled.
  * NOT a finding: unit tests that only assert an option object is passed through, with no real network connection. Tests that actually connect with verification off = P1 (replace with mkcert).
  * To claim "off by default" you MUST cite the file:line where the default is defined. Unproven default = treat as ON.
  * Never write that a bypass is acceptable: every one appears in the report with its class and evidence. Group identical opt-in options into one row (list all file:line) instead of one row each.
  * Downgrade anything to "dead code" only if it is provably unreachable, and say exactly why.
- If the project has no code at all (configs only), say so explicitly in the report.

OUTPUT FORMAT (write the report in Arabic; keep technical terms, paths, and code in English):
## تقرير المرحلة 02 — أنماط تعطيل التحقق
### الجاهزية: ✅ جاهز | ⚠️ جاهز جزئياً | ❌ غير جاهز
### الموجود حالياً (ممارسات التحقق السليمة اللي شفتها — مع الإثبات)
### الناقص أو المخالف (جدول: النمط | الملف:السطر | السطر نفسه | الخطورة P0/P1/P2 | الخطر الفعلي في سطر)
### المطلوب عمله (مرتبة بالأولوية، مع البديل الصحيح لكل حالة: mkcert / NODE_EXTRA_CA_CERTS / SSL_CERT_FILE / تثبيت CA داخلي في الصورة)
```

## برومبت الإصلاح

```text
Based on the Stage 02 audit report, fix every dangerous TLS verification bypass in this project.

RULES OF ENGAGEMENT:
- First, show me a plan: for each finding (file:line), the exact replacement code/config.
- Apply fixes ONLY after I approve the plan.
- Follow the October 2026 standard replacements:
  * Local development (localhost / local domains): generate trusted certs with mkcert (`mkcert -install` then `mkcert localhost 127.0.0.1 ::1 <local-domain>`) and use them — never disable verification.
  * Node.js connecting to services behind an internal/private CA: keep `rejectUnauthorized: true` (the default) and set `NODE_EXTRA_CA_CERTS=/path/to/internal-root-ca.crt` via environment, not code.
  * Python (requests/httpx) against a private CA: set `REQUESTS_CA_BUNDLE` / `SSL_CERT_FILE` env vars, or pass `verify="/path/to/internal-root-ca.crt"` — never `verify=False`.
  * Docker images: `COPY internal-root-ca.crt /usr/local/share/ca-certificates/internal-root-ca.crt` + `RUN update-ca-certificates` + `ENV NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/internal-root-ca.crt`.
  * If a bypass existed for a corporate/proxy MITM box: install that proxy's root CA into the trust store instead.
  * mTLS clients: an https.Agent with cert + key + ca, `minVersion: 'TLSv1.3'`, `rejectUnauthorized: true`.
  * End-user "ignore TLS errors" options (P1/P2 class): add a "custom CA certificate" field next to the option, passed as `ca` to the agent with `rejectUnauthorized: true`, so users can trust a self-signed device/private root instead of disabling verification. Keep the ignore option OFF by default, show a warning in the UI, and log a line whenever it is used.
  * Partial weakening: remove legacy `minVersion` / `secureProtocol` / `SSL_OP_LEGACY_*` / low `@SECLEVEL`; if a legacy endpoint truly needs it, make it an explicit per-target opt-in, never a global default.
- After fixing: list every changed file with a one-line summary, and give me the exact command to verify each fix.
```

## أوامر التحقق بعد الإصلاح

```bash
# التأكد إن مفيش متغير بيوقف التحقق في البيئة:
env | grep -i "NODE_TLS_REJECT_UNAUTHORIZED"   # المفروض يرجع حاجة

# اختبار إن الاتصال بيتم بالتحقق الكامل:
node -e "fetch('https://YOUR-INTERNAL-HOST').then(r => console.log(r.status))"
```
