# المرحلة 01 — الاكتشاف: خريطة كل نقاط TLS في المشروع

> **إيه دي؟** أول خطوة قبل أي فحص. بتطلع خريطة كاملة لكل الأماكن اللي فيها TLS/SSL في المشروع (سيرفرات، عملاء HTTP، شهادات، Docker، CI/CD، موبايل...)، وبتحدد أنهي مراحل لاحقة (02–08) منطبقة على المشروع وأيه مش منطبقة.
> **إمتى تستخدمها؟** أول ما تفتح الـ pipeline على مشروع جديد.

## برومبت الفحص

```text
You are a senior TLS/SSL security auditor working against October 2026 standards. Your job in THIS step is DISCOVERY ONLY: build a complete inventory of every TLS/SSL touchpoint in this project. You are NOT judging quality yet and NOT fixing anything.

IMPORTANT 2026 CONTEXT (facts, not opinions):
- "SSL" no longer exists in practice: SSL v1–v3 and TLS 1.0/1.1 are banned everywhere; what actually runs is TLS 1.3 (the standard) or TLS 1.2 (legacy only).
- Post-quantum hybrid key exchange (X25519MLKEM768) and Encrypted Client Hello (ECH) are the new server baselines.
- Let's Encrypt shut down OCSP (Aug 2025) and deleted all reminder emails (Jun 2025); renewal must be ACME+ARI automated.
- Maximum public certificate validity is 200 days since 15 Mar 2026 (heading to 100 then 47 days).

FIND AND MAP ALL OF THE FOLLOWING (search the whole repo thoroughly):
1. Web servers / reverse proxies / gateways: Nginx, Caddy, Traefik, HAProxy, Envoy, Apache, IIS, Kubernetes Ingress/Gateway, cloud load balancers (look for: ssl, tls, listen 443, ssl_certificate, gateway/ingress resources).
2. Outbound HTTPS clients in application code: Node.js (fetch, axios, got, https.Agent), Python (requests, httpx, aiohttp, urllib3), Go (net/http, tls.Config), Java (HttpClient, SSLContext), PHP (cURL), Ruby, .NET, and shell scripts calling curl/wget over https.
3. Certificate & key material in the repo: *.pem, *.crt, *.cer, *.key, *.p12, *.pfx, *.jks, keystores, CA bundles. NOTE PATHS ONLY — never print the content of private keys or secrets.
4. TLS-specific code configuration: minVersion/maxVersion, ciphers, SSL context creation, agent options, client certificates, custom trust stores.
5. Containers & images: Dockerfiles/Compose installing CA certificates, NODE_EXTRA_CA_CERTS, SSL_CERT_FILE, REQUESTS_CA_BUNDLE, certificates copied into images.
6. CI/CD and automation: pipelines touching TLS, ACME clients (certbot, lego, acme.sh, Caddy automatic TLS), renewal cronjobs/scripts.
7. Mobile apps (if any): iOS App Transport Security settings, Android network_security_config, Flutter/RN TLS plugins, certificate pinning code.
8. Local development: mkcert, self-signed certificates, localhost HTTPS setups.
9. mTLS / service-to-service: any place a client certificate is configured for internal calls.
10. User-configurable TLS features (critical when the project is itself a TOOL — monitoring, proxies, HTTP/API clients, notification senders): every UI/config option that changes TLS behavior for end users ("ignore TLS errors", custom CA, client certificate fields...). For each: where its DEFAULT is defined (file:line), and whether a SAFE ALTERNATIVE exists next to it (a field to supply a custom CA / private root). Stage 02 needs this table to classify severity correctly.

RULES:
- READ-ONLY: do not modify, create, or delete anything. Do not run network commands.
- CITE EVIDENCE: every item must reference a file path (and line number where applicable).
- SECRETS: never print private keys, tokens, or passwords; reference their path only.
- If an area has nothing at all, write "لا يوجد" for it.

OUTPUT FORMAT (write the report in Arabic; keep technical terms, paths, and code identifiers in English):
## تقرير المرحلة 01 — خريطة نقاط TLS
### الجاهزية: ✅ | ⚠️ | ❌  (المعيار: هل نقاط TLS في المشروع معروفة وموثّقة، ولا مبنية على مصادفات؟)
### خريطة نقاط TLS (جدول: المجال | الملف/المكان | ملاحظة سريعة)
### مناطق الخطر الأولى (مرتبة حسب الأهمية — لكل واحدة ليه دي أخطر من غيرها)
### المراحل المطبقة: من 02 إلى 08 — لكل مرحلة: منطبقة ✅ / غير منطبقة ➖ + السبب في سطر
```

## ملاحظة بعد التقرير
خد قائمة "المراحل المطبقة" دي وامشي عليها بالترتيب. المراحل غير المنطبقة اتخطاها نهائياً — ده بيوفّر وقت ومخلي تقرير 09 النهائي دقيق.
