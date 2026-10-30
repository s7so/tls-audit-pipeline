# المرحلة 05 — mTLS وهوية الميكروسيرفس الداخلية

> **إيه دي؟** بتدقق أي TLS بين الخدمات داخلياً (service-to-service). فخ 2026 القاتل: شهادات Let's Encrypt **بقت ممنوعة كـ client certificates** (اتشال منها EKU بتاع Client Auth في 2026) — أي mTLS بتعتمد عليها هيقع فوراً. البديل: SPIFFE/SPIRE أو Service Mesh أو `step-ca` داخلي.
> **إمتى تستخدمها؟** لو المرحلة 01 طلّعت نداءات داخلية بشهادات عميل (client certificates).

## برومبت الفحص

```text
You are a senior TLS/SSL security auditor (October 2026 standards). Audit ALL service-to-service / internal TLS (mTLS) in this project: microservice calls, internal APIs, databases requiring client certs, message queues with TLS, Kubernetes service communication. READ-ONLY.

CRITICAL 2026 FACT:
- Public CA certificates (Let's Encrypt et al.) can NO LONGER be used as CLIENT certificates: Let's Encrypt removed the "TLS Client Authentication" EKU from its certificates in Feb & Jul 2026. Any mTLS setup where a service presents a public-CA cert as its client identity WILL FAIL the handshake. This is a P0 finding.

CORRECT 2026 APPROACHES (audit against these):
1. Kubernetes / cloud-native: SPIFFE/SPIRE or a service mesh (Linkerd, Istio) issuing short-lived workload identities — auto-rotated every few hours, held in memory, zero manual work.
2. VMs / bare-metal: an internal certificate authority like step-ca (Smallstep) issuing internal certs with BOTH ServerAuth and ClientAuth EKUs, with automated issuance and rotation.
3. Trust distribution: every consuming service trusts ONLY the internal root CA for internal calls (via NODE_EXTRA_CA_CERTS / SSL_CERT_FILE / the system store) — never by disabling verification.

FIND AND ASSESS:
- Every place a client certificate is configured: the cert/key paths, and which CA signed them (public vs internal).
- Whether ANY client certificate is a public-CA certificate → P0 violation (it will fail / has no ClientAuth EKU).
- Whether internal certs auto-rotate, or are long-lived files copied by hand (manual rotation = P1).
- Whether internal calls verify the peer (rejectUnauthorized: true equivalent) or whether verification was weakened to make mTLS "work".
- Where internal root CA trust is provisioned (Dockerfiles, env vars, k8s secrets/ConfigMaps, system stores).

RULES:
- READ-ONLY. Evidence = file:line. Never print private keys.
- If the project has no service-to-service TLS at all, mark ➖ غير منطبق (N/A) with one line.

OUTPUT FORMAT (write the report in Arabic; keep technical terms in English):
## تقرير المرحلة 05 — mTLS وهوية الميكروسيرفس
### الجاهزية: ✅ | ⚠️ | ❌ | ➖ N/A
### الموجود حالياً (جدول: الاتصال من → إلى | نوع الشهادة | مين صادرها | التجديد إزاي بيحصل | الحكم) — بالإثبات
### الناقص أو المخالف (كل بند بأولوية P0/P1/P2)
### المطلوب عمله (التصميم المستهدف: SPIFFE/SPIRE أو mesh أو step-ca + توزيع الثقة للجذر الداخلي + تجديد تلقائي قصير العمر)
```

## برومبت الإصلاح

```text
Based on the Stage 05 report, fix internal mTLS to the October 2026 standard.

RULES OF ENGAGEMENT:
- Show the plan (new components, configs, code changes) as diffs first; apply only after my approval.
- Target state:
  * Kubernetes/cloud-native: adopt SPIFFE/SPIRE or a service mesh (Linkerd/Istio) for automatic short-lived workload identities.
  * VMs/bare-metal: deploy step-ca as the internal CA; issue mTLS certs with ServerAuth + ClientAuth EKUs; automate issuance and rotation (no hand-copied long-lived files).
  * Replace ANY public-CA client certificate with an internal one (P0 fix — the public one cannot do Client Auth anymore).
  * Every consuming service trusts only the internal root CA (NODE_EXTRA_CA_CERTS / SSL_CERT_FILE / system store), with verification fully on.
  * Node.js mTLS client pattern to use where relevant:
      new https.Agent({ cert, key, ca: internalRootCA, minVersion: 'TLSv1.3', rejectUnauthorized: true })
- After fixing: list changed files/configs, and give the exact command to test one internal call end-to-end.
```
