# المرحلة 06 — مخازن الثقة وبيئات التشغيل (Docker / CI / التطوير المحلي)

> **إيه دي؟** بتدقق إزاي كل بيئة تشغيل (Docker images، Compose، Kubernetes، CI runners، أجهزة المطورين) بتثق في الـ CAs. القاعدة: تثبيت الجذر الداخلي في البيئة (`update-ca-certificates` + `NODE_EXTRA_CA_CERTS`...) — مش قفل التحقق. وكمان: ممنوع مفاتيح خاصة متخزنة جوه الصور.
> **إمتى تستخدمها؟** لو فيه Docker أو CI أو بيئة تطوير محلية — يعني تقريباً دايماً.

## برومبت الفحص

```text
You are a senior TLS/SSL security auditor (October 2026 standards). Audit TRUST STORE MANAGEMENT across the project's runtime environments: Docker images, docker-compose, Kubernetes manifests, CI runners, and developer machines. READ-ONLY.

2026 BASELINE (audit against this):
- Private/internal CAs must be trusted by INSTALLING the root CA into each environment — never by disabling verification:
  * Debian/Ubuntu images: COPY internal-root-ca.crt /usr/local/share/ca-certificates/internal-root-ca.crt + RUN update-ca-certificates
  * Alpine images: same path convention + RUN update-ca-certificates
  * Node.js specifically: Node ignores the system store — set ENV NODE_EXTRA_CA_CERTS=/path/to/internal-root-ca.crt
  * Python (requests/httpx): ENV SSL_CERT_FILE=... and ENV REQUESTS_CA_BUNDLE=...
  * Go/Rust/curl: the system store or SSL_CERT_FILE
- Local development: mkcert (`mkcert -install` + per-domain certs) — never committed self-signed hacks, never disabled verification.
- Private keys (service keys, root CA keys) must NEVER be baked into images or committed to the repo (P0). Keys arrive via secrets/mounts/volumes at runtime.
- `NODE_TLS_REJECT_UNAUTHORIZED=0` as an ENV anywhere (Dockerfile, compose, k8s manifest, CI config) = P0.

FIND AND ASSESS:
- Every Dockerfile / compose file / k8s manifest / CI config: where CA certificates come from, and whether NODE_EXTRA_CA_CERTS / SSL_CERT_FILE / REQUESTS_CA_BUNDLE are set where the runtime needs them.
- Any key material committed to the repo (paths only — never print contents).
- The localhost HTTPS setup for development (mkcert? self-signed? plain http?).
- Mismatches: e.g. the CA is installed in the system store but the runtime is Node (which ignores it).

RULES:
- READ-ONLY. Evidence = file:line. Never print key contents.
- Nothing containerized and no CI? Mark ➖ غير منطبق (N/A) for the container part, but still audit the local dev part.

OUTPUT FORMAT (write the report in Arabic; keep technical terms in English):
## تقرير المرحلة 06 — مخازن الثقة وبيئات التشغيل
### الجاهزية: ✅ | ⚠️ | ❌ | ➖ N/A
### الموجود حالياً (لكل بيئة: إزاي بتثق في الـ CAs دلوقتي — بالإثبات)
### الناقص أو المخالف (كل بند بأولوية P0/P1/P2)
### المطلوب عمله (لكل بيئة: السطور المطلوبة بالظبط)
```

## برومبت الإصلاح

```text
Based on the Stage 06 report, fix trust store management in every environment.

RULES OF ENGAGEMENT:
- Show the plan as diffs first (Dockerfile lines, ENV blocks, compose changes, local dev setup); apply only after my approval.
- Standard pattern to apply:
  * Dockerfile (Debian/Ubuntu base):
      COPY internal-root-ca.crt /usr/local/share/ca-certificates/internal-root-ca.crt
      RUN update-ca-certificates
      ENV NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/internal-root-ca.crt
      ENV SSL_CERT_FILE=/etc/ssl/certs/ca-certificates.crt
      ENV REQUESTS_CA_BUNDLE=/etc/ssl/certs/ca-certificates.crt
    (keep only the ENV lines the image's runtimes actually need)
  * Remove any private key from images or the repo; wire keys through secrets/mounts.
  * Remove NODE_TLS_REJECT_UNAUTHORIZED=0 from every env/config if present.
  * Local dev: set up mkcert (`mkcert -install`, then `mkcert localhost 127.0.0.1 ::1 <local-domain>`) and point local services at the generated certs; add the commands to the project README (do not commit generated certs).
- After fixing: list changed files and give one command per environment to prove trust works (e.g. a fetch/curl to an internal HTTPS endpoint returning a normal status code).
```
