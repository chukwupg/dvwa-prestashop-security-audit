# PrestaShop Attack Simulation Report

**Target**: PrestaShop lab instance (AWS EC2 + RDS), version 8.2.1

**Date**: 20/09/2026

> Redaction note: Given the cloud live deployment environment of the lab, RDS endpoint, admin account email, admin password, public EC2 IP, and personal source IP are redacted below (`<redacted>`). Non-identifying architecture and configuration details are left as-is. Raw packet captures and unredacted command output are retained locally only and are not committed to this repository.

---

## 1. Objective

Demonstrate practical, exploitable impact from the gaps identified in the completed security checklist [`security-checklist.md`](/prestashop-security/security-checklist.md), specifically targeting findings from checklist items 1 (outdated version), 3 (no MFA), 8 (no TLS), and 9 (no IP restriction/WAF on admin). Three distinct attack vectors were attempted against the live lab instance.

## 2. Environment / Scope

- **Target**: PrestaShop 8.2.1, EC2 (Ubuntu, Apache/PHP) + RDS (MySQL), previously deployed lab instance. 
- **Access level assumed**: Unauthenticated (Vectors 1 & 3) and authenticated-attacker-perspective for demonstrating impact (Vector 2 uses a known valid account to confirm absence of rate-limiting, not to gain unauthorized access)
- **Scope boundary**: Testing confined to this self-owned lab instance; no external systems targeted

## 3. Methodology

Each vector was selected directly from the checklist finding, then tested against the live, unpatched instance:
1. Reviewed checklist findings for items with genuine exploitability (as opposed to hygiene-only findings).
2. For each selected vector, attempted a proof-of-concept exploitation, iterating on payloads/technique where initial attempts were blocked or inconclusive.
3. Captured evidence (screenshots, HTTP responses, database queries, packet captures) for each attempt, including failed/partial attempts, since these still demonstrate real underlying weaknesses.
4. Mapped each finding back to its root cause and a specific, actionable mitigation.

---

## 4. Vector 1: Stored XSS via Contact Form Email Field (CVE-2026-44212)

### 4.1 Vulnerability Background
CVE-2026-44212 (CVSS 9.3, Critical) affects PrestaShop versions prior to 8.2.6 and 9.1.1. An unauthenticated attacker can submit the public Contact Us form with a maliciously crafted "email address" exploiting a permissive server-side validator. The payload is stored and rendered without escaping in the back-office Customer Service thread view, executing in an authenticated employee's browser session.

### 4.2 Root Cause
PrestaShop's email validator correctly implements RFC 5321's "quoted-string" local-part rule — a legitimate email syntax allowing the portion before `@` to be wrapped in double quotes and contain otherwise-restricted characters (spaces, symbols, angle brackets) as long as they're enclosed by the quote pair. This is valid by specification, so the validator's acceptance is not itself a bug. The actual flaw is that this data is later rendered in the admin UI without consistent HTML escaping.

### 4.3 Methodology
1. Attempted to submit `<script>` and `<img onerror=...>` payloads directly in the email field, which was blocked by the browser's native HTML5 `type="email"` client-side validation (not a real security control).
2. Bypassed client-side validation via browser DevTools, changing the input's `type` attribute from `email` to `text`.
3. Submitted various payloads:
   - Raw script tags without RFC 5321 quoting, **rejected** by server-side validation ("Invalid email address").
   - RFC 5321 quoted-string wrapping a `<script>` tag: `"<script>alert(document.domain)</script>"@test.com`, **accepted**, message sent successfully.
4. Logged into the back-office as the store's employee account and opened the resulting Customer Service thread.
5. Used View Page Source to examine exactly how the stored payload was rendered in different locations on the thread detail page.

### 4.4 Findings

Inconsistent output handling of the same stored value was observed across the single thread detail page:

| Rendering location | Behavior observed |
|---|---|
| `<h3 id="reply-form-title">` (heading text) | Raw/unescaped - quotes and content displayed literally |
| `<input type="hidden" name="msg_email" value="...">` | Raw/unescaped - fully intact including backslashes |
| `<strong>` (separate thread, earlier test) | Tag-stripped - angle brackets removed, other characters preserved |
| `<h2>` | Fully HTML-entity-encoded - safe |

This confirms the underlying vulnerability class (CWE-79, improper output encoding) is present and inconsistently applied across the affected view — some locations are genuinely unescaped, which matches the CVE's description.

A follow-up attempt to submit a fresh `<script>`-tag payload specifically to test execution in the confirmed-vulnerable `<h3>` location did not appear in either the thread list or the underlying database, despite receiving what was intended to be a fresh form session. The cause was not conclusively isolated within the testing window, candidate explanations include CSRF token staleness or PrestaShop's thread-grouping behavior (messages from a matching email may append to an existing thread rather than always creating a new one, potentially obscuring the new submission from the query used to check for it).

### Evidence

**Bypassing client-side validation via browser DevTools by changing the input's `type` attribute from `email` to `text`**
![client-side by-pass](/prestashop-security/screenshots/attack-01a-changing-form-type.png)

**Payload entered in Contact Us form**
![payload entered](/prestashop-security/screenshots/attack-01-contact-form-payload.png)

**Accepted submission confirmation**
![payload accepted](/prestashop-security/screenshots/attack-01b-payload-submitted.png)

**Raw HTML excerpts above, captured via View Page Source on the thread detail page**
![Page source](/prestashop-security/screenshots/attack-01bc-page-source-excerpt.png)


### 4.5 Impact
**Confirmed:** an unauthenticated external party can inject content into the admin back-office that is rendered without escaping in at least two page locations. While full script execution (e.g., a firing `alert()`) was not conclusively reproduced within the testing window, the unescaped rendering itself is sufficient to confirm the vulnerability is present on this instance, consistent with the published CVE. Given `HttpOnly` is correctly set on session cookies on this instance (see checklist item 5), the specific "session cookie theft via `document.cookie`" impact described in the original advisory is partially mitigated here, but arbitrary DOM manipulation, credential-phishing overlays, or forced administrative actions via an authenticated context would remain realistic outcomes of a fully weaponized payload.

### 4.6 Mitigation
- **Primary fix**: update PrestaShop to 8.2.6 or later, which patches this vulnerability directly.
- **Defense-in-depth**: ensure consistent output encoding (`|escape:'html':'UTF-8'` Smarty modifier or equivalent) is applied to all user-controllable data rendered in back-office views, not just the specific field patched upstream.
- **Compensating control already in place**: `HttpOnly` cookies reduce (but do not eliminate) the practical impact of any XSS that does succeed.

---

## 5. Vector 2: No Brute-Force Protection on Admin Login

### 5.1 Root Cause
Checklist item 9 confirmed the admin login path has no IP restriction and no WAF in front of it. This vector tests whether the application itself compensates with any rate-limiting or lockout mechanism (CWE-307).

### 5.2 Methodology
1. Submitted repeated failed login attempts against a known valid admin account via the browser UI.
2. Submitted 16 total scripted failed login attempts via `curl`, simulating automated attack behavior:
   ```bash
   for i in {1..8}; do curl -s -o /dev/null -w "%{http_code}\n" -X POST http://<redacted>/admin_secure/index.php \
     --data "controller=AdminLogin&submitLogin=1&email=<redacted>&passwd=wrongpass$i"; done
   ```
3. Observed application behavior across all attempts.

### 5.3 Findings
All 16 attempts (browser + scripted) returned identical `200 OK` responses with a generic "Invalid password" result. No CAPTCHA challenge, progressive delay, temporary account lockout, or any form of alerting was triggered at any point.

**Screenshot**: 
**Repeated failed logins with no lockout behavior**
![failed login attempts](/prestashop-security/screenshots/attack-07-no-lockout.png)

### 5.4 Impact
Combined with checklist item 9's finding (admin panel reachable from `0.0.0.0/0`, no IP restriction, no WAF), this admin login endpoint has **no application-level friction whatsoever** against sustained automated credential-stuffing or brute-force attacks. Current password strength (checklist item 3 - 11 characters, mixed complexity) is the sole barrier protecting this account; there is no defense-in-depth. Combined with the absence of MFA (also item 3), a successful brute-force would grant immediate, unrestricted back-office access.

### 5.5 Mitigation
- Implement account lockout after N failed attempts (e.g., 5), with exponential backoff or a time-based cooldown.
- Add a CAPTCHA (e.g., reCAPTCHA/hCaptcha) after a small number of failed attempts.
- Restrict the admin path to a trusted IP allowlist at the infrastructure level (Apache `<Location>` directive or security-group rule on a dedicated admin-access port) — this alone would eliminate the practical exploitability of this finding even without an application-level fix.
- Deploy a WAF (e.g., the ModSecurity + OWASP CRS pattern used in the DVWA track) capable of rate-limiting login attempts at the proxy layer.
- Add native MFA or a dedicated 2FA module once available, so a compromised password alone is insufficient for access.

---

## 6. Vector 3: Plaintext Credential Interception (No TLS)

### 6.1 Root Cause
Checklist item 8 confirmed the instance serves all traffic over plain HTTP, with port 443 not listening and no TLS certificate present.

### 6.2 Methodology
Packet capture was performed using Wireshark during a normal, legitimate admin login to the back-office, simulating what any network-positioned observer (shared network segment, compromised router, ISP-level interception, etc.) would be able to see without any application-level exploitation.

### 6.3 Findings
The captured traffic contained the admin login POST request with the password field fully visible in plaintext, with no encryption applied at any point in transit.

**Screenshot**
**Wireshark capture showing the plaintext password field**
![Wireshark capture](/prestashop-security/screenshots/attack-08-plaintext-credentials.png)

### 6.4 Impact
This is the most severe and simplest of the three vectors to execute, precisely because it requires no payload crafting, no authentication bypass, and no application-level vulnerability at all, only network position. Any admin login performed over this instance's current configuration is fully compromised the moment an attacker can observe the traffic path, which is a realistic scenario on shared/public networks, compromised intermediate infrastructure, or any on-path interception.

### 6.5 Mitigation
- Configure TLS (Let's Encrypt/Certbot if a domain is available; a self-signed certificate is an acceptable interim step for a lab environment) and enforce HTTP>HTTPS redirection.
- Enable `PS_SSL_ENABLED` in PrestaShop once TLS is live, and set the `Secure` flag on all session cookies (resolves checklist item 5 as a direct consequence).
- The EC2 security group already permits inbound 443 from `0.0.0.0/0` - no infrastructure change is required beyond installing and configuring the certificate.

---

## 7. Findings Considered But Not Used as Attack Vectors

For completeness, the remaining checklist findings were reviewed and deliberately not pursued as standalone attack vectors, since they represent hygiene or resilience gaps rather than directly exploitable weaknesses:

| Checklist item | Reason not used as a vector |
|---|---|
| 2 - Demo catalog data | Reconnaissance/hygiene signal, not directly exploitable |
| 5 - Missing `Secure` cookie flag | Consequence of Vector 3 (no TLS), not an independent vulnerability |
| 6 - Default module set | No specific vulnerable module confirmed |
| 10 - Backup/recovery untested | Affects impact severity and recovery time after a breach, not a way to cause one |

---

## 8. Summary of Findings

| Vector | Status | Severity | Root Checklist Item(s) |
|---|---|---|---|
| 1. Stored XSS (CVE-2026-44212) | Vulnerability class confirmed; full execution PoC inconclusive | Critical (per CVE) | 1 |
| 2. No brute-force protection on admin login | Fully confirmed and reproducible | High | 3, 9 |
| 3. Plaintext credential interception (no TLS) | Fully confirmed | Critical | 8 |

## 9. Overall Conclusion

This lab instance, while benefiting from solid infrastructure-level hardening in other areas (SSH restriction, file permissions, disabled Webservice/API), has three genuine and demonstrable weaknesses stemming directly from an outdated application version, absent authentication friction, and a complete lack of transport encryption. None of these require sophisticated tooling to exploit, a public CVE with documented technique, a handful of scripted login attempts, and a passive packet capture were sufficient to demonstrate real impact. 

**Remediation priority should be:**
1. Enable TLS immediately: it is the simplest fix with the broadest impact reduction.
2. Update PrestaShop to 8.2.6+
3. Add login rate-limiting 
4. Configure lockout policy and restrict admin access by IP.
5. Deploy a web application firewall. 

---

## Author

👩‍💻 **Chukwu PraiseGod**  
Follow my journey: [X](https://x.com/chukwupg) | [LinkedIn](https://linkedin.com/in/chukwupg)  