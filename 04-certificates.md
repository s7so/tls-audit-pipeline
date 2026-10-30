# المرحلة 04 — دورة حياة الشهادات: الإصدار والتجديد والمراقبة

> **إيه دي؟** بتدقق إزاي المشروع بيطلع ويجدد شهاداته: لو لسه في تجديد يدوي أو cron أعمى أو انتظار إيميلات — كل ده اتوفى في 2026. المعيار: ACME مؤتمت يدعم **ARI (RFC 9773)** + مراقبة انتهاء خارجية.
> **إمتى تستخدمها؟** بعد المرحلة 03 — أي مشروع فيه دومينات عامة أو شهادات.

## برومبت الفحص

```text
You are a senior TLS/SSL security auditor (October 2026 standards). Audit how this project OBTAINS, RENEWS, and MONITORS its TLS certificates. READ-ONLY.

2026 FACTS YOU MUST AUDIT AGAINST:
1. Maximum public certificate validity is 200 days (since 15 Mar 2026), enforced by browsers; the schedule continues: 100 days from Mar 2027, 47 days from Mar 2029. Any manual or annual renewal process is obsolete.
2. Let's Encrypt deleted all expiry reminder emails (Jun 2025). Waiting for email reminders is NOT a renewal strategy.
3. Let's Encrypt shut down OCSP (Aug 2025): certificates carry no OCSP URLs. Any OCSP/stapling configuration or OCSP-based monitoring is dead code.
4. ACME Renewal Information (ARI, RFC 9773) is the 2026 standard: the renewal client asks the CA "when should I renew?" every few hours; on mass-revocation events the CA forces immediate background renewal without hitting rate limits. A blind fixed-schedule cron is a smell. ARI-capable clients: certbot 3.x+, lego, recent acme.sh, Caddy.
5. Let's Encrypt offers 6-day short-lived certificates via ACME Profiles (renewed every ~3 days) — note where the project would benefit (they make revocation checks irrelevant; the cert dies before an issue can be exploited).
6. Domain Control Validation (DCV) reuse is capped at 200 days now.

FIND AND ASSESS:
- How are production certificates issued today: ACME client? manual? cloud-managed (ACM/Cloudflare)? purchased from a vendor?
- Which ACME client and version — does it support ARI?
- Renewal automation: systemd timer / cron / cert-manager in k8s / built-in (Caddy)? Has it been tested (`certbot renew --dry-run`)?
- Expiry monitoring: what alerts if renewal silently fails (external uptime monitor, Prometheus blackbox exporter, healthchecks.io...)? Absence = P1.
- Any OCSP/stapling leftovers (P1) or reliance on reminder emails (P1).
- Rate-limit awareness: wildcard vs per-domain issuance patterns, staging vs production ACME endpoints in CI.
- Certificate/key files in the repo or infra configs: paths only, and expiry if readable from context. NEVER print private keys.

RULES:
- READ-ONLY. Evidence = file:line or config/systemd/cron path. No network commands.
- If nothing in the project manages certificates (fully cloud-managed outside the repo), mark ➖ غير منطبق (N/A) with a one-line note about where management actually happens.

OUTPUT FORMAT (write the report in Arabic; keep technical terms in English):
## تقرير المرحلة 04 — دورة حياة الشهادات
### الجاهزية: ✅ | ⚠️ | ❌ | ➖ N/A
### الموجود حالياً (آلية الإصدار والتجديد والمراقبة الحالية — بالإثبات)
### الناقص أو المخالف (كل بند بأولوية P0/P1/P2)
### المطلوب عمله (الإعداد المستهدف: عميل ACME يدعم ARI + تجديد مؤتمت مُختبَر + مراقبة انتهاء خارجية مستقلة عن الـ CA)
```

## برومبت الإصلاح

```text
Based on the Stage 04 report, fix the certificate lifecycle to the October 2026 standard.

RULES OF ENGAGEMENT:
- Show the plan (files, configs, systemd timers/cron, CI steps) as diffs first; apply only after my approval.
- Target state:
  * An ARI-capable ACME client (certbot 3.x+ / lego / acme.sh / Caddy built-in) — remove any manual renewal steps.
  * Renewal runs automatically (systemd timer or equivalent) AND is verified periodically (`certbot renew --dry-run` in CI or a scheduled task).
  * No reliance on CA emails; no OCSP/stapling configuration anywhere.
  * An independent expiry monitor that alerts (external uptime/blackbox check on certificate days-remaining) — must not depend on the CA or on the same server.
  * Deploy hooks that reload the web server after renewal (e.g. `--deploy-hook "systemctl reload nginx"`).
- If the project would benefit from short-lived certs, propose 6-day certificates via ACME Profiles where appropriate (internal endpoints first).
```

## أوامر التحقق بعد الإصلاح

```bash
# قائمة الشهادات المدارة وحالتها:
certbot certificates

# اختبار التجديد دون استهلاك الـ rate limits:
certbot renew --dry-run

# تاريخ الانتهاء والـ EKU للشهادة المنشورة:
echo | openssl s_client -connect YOUR-DOMAIN:443 2>/dev/null | openssl x509 -noout -dates -ext extendedKeyUsage
```
