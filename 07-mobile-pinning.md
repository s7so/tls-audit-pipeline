# المرحلة 07 — Certificate Pinning في تطبيقات الموبايل

> **إيه دي؟** بتدقق الـ pinning في تطبيقات iOS/Android/Flutter/React Native. القاعدة القاتلة في 2026: **ممنوع تثبيت الشهادة النهائية (Leaf)** — عمرها بقى 200 يوم وبيقل لـ 47 يوم، والـ ARI ممكن يجددها فجأة في أي لحظة، فالتطبيق يقع عند **كل** المستخدمين. البديل: pinning لمفاتيح الجذر SPKI بـ 3 جهات إصدار على الأقل — أو لا pinning خالص.
> **إمتى تستخدمها؟** لو المرحلة 01 طلّعت تطبيق موبايل في المشروع. لو مفيش موبايل — البرومبت نفسه هيتخطى نفسه.

## برومبت الفحص

```text
You are a senior TLS/SSL security auditor (October 2026 standards). Audit CERTIFICATE PINNING in this project's mobile apps (iOS, Android, Flutter, React Native, Xamarin).

FIRST: if this project contains NO mobile application code, output the report with ➖ غير منطبق (N/A) and one line of justification, then stop.

CRITICAL 2026 FACT:
- Public certificate lifetimes are now 200 days max (→ 100 days Mar 2027, → 47 days Mar 2029; 6-day certs exist via ACME Profiles), and ACME ARI can force instant mass renewal at any time.
- Pinning a LEAF (end-entity) certificate or its SHA-256 inside app code = the app BREAKS FOR ALL USERS the moment the server cert renews. This is a P0 violation — the pin is guaranteed to expire.

CORRECT 2026 APPROACH:
- Default: rely on the OS trust store + Certificate Transparency (Static CT). No pinning.
- If pinning is a regulatory/business requirement (e.g. banking apps): pin ROOT CA PUBLIC KEYS (SPKI hashes) of AT LEAST 3 DIFFERENT CAs (1 primary + 2 backups from other CAs), with a shipped-update path for the pin set.

SEARCH FOR:
- iOS: Info.plist NSPinnedDomains / NSAppTransportSecurity, URLSession `didReceiveChallenge` handlers doing manual pinning, TrustKit or Alamofire ServerTrustPolicy configs.
- Android: network_security_config.xml `<pin-set>`, OkHttp CertificatePinner, custom X509TrustManager implementations.
- Flutter: certificate-pinning packages, `badCertificateCallback` overrides.
- React Native: SSL pinning libraries / native modules.
- Any hardcoded SHA-256 hashes or bundled .cer/.der files, and WHICH certificate they pin (leaf? intermediate? root?).

RULES:
- READ-ONLY. Evidence = file:line.
- Classify every pin found: leaf (P0), intermediate (P1), root SPKI with ≥3 CAs (correct), root SPKI with only 1 CA (P1 — no backup on renewal/revocation).

OUTPUT FORMAT (write the report in Arabic; keep technical terms in English):
## تقرير المرحلة 07 — Certificate Pinning في الموبايل
### الجاهزية: ✅ | ⚠️ | ❌ | ➖ N/A
### الموجود حالياً (جدول: كل pin | الملف:السطر | نوع الشهادة المثبتة | الحكم)
### الناقص أو المخالف (كل بند بأولوية P0/P1/P2)
### المطلوب عمله (الهدف: إزالة leaf pinning نهائياً، أو تحويله لـ SPKI root pinning بـ 3 CAs على الأقل مع خطة تحديث)
```

## برومبت الإصلاح

```text
Based on the Stage 07 report, fix certificate pinning in the mobile app(s).

RULES OF ENGAGEMENT:
- Show the plan as diffs first; apply only after my approval.
- Target state per the October 2026 standard:
  * Remove every LEAF-certificate pin (P0) — it will brick the app on the next automatic renewal.
  * If no compliance requirement exists: delete pinning entirely and rely on the OS trust store + Certificate Transparency.
  * If pinning is mandatory (e.g. banking regulation): pin ROOT CA SPKI hashes of at least 3 different CAs (primary + 2 backups), configured in the platform's native mechanism (Android network_security_config / iOS NSPinnedDomains or TrustKit), with documentation of the update process for rotating the pin set.
- After fixing: list changed files and describe the manual test (app against a renewed cert) that proves the app survives renewal.
```
