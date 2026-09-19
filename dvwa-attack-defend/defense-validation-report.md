# Defense Validation Report: DVWA SQL Injection & XSS, ModSecurity WAF

## 1. Objective
Demonstrate exploitation of SQL Injection and Cross-Site Scripting (XSS) vulnerabilities in DVWA, deploy a ModSecurity Web Application Firewall (WAF) to mitigate them, and validate that the deployed WAF blocks the previously successful attacks without altering legitimate application behavior.

## 2. Environment / Scope
- **Target application**: DVWA (`ghcr.io/digininja/dvwa:latest`), Docker container `dvwa`, security level Low.
- **Database**: MariaDB 10, container `dvwa-db`, internal-only (not exposed to the WAF or network).
- **WAF**: `owasp/modsecurity-crs:nginx` (Nginx + ModSecurity v3 + OWASP Core Rule Set v4, paranoia level 1), container `modsec-waf`, publicly bound to port 80 on the lab network.
- **Network**: Docker bridge network `dvwa-net`, on an Ubuntu server sitting on an isolated host-only lab network (no direct internet exposure).
- **Scope boundary**: testing was confined to this self-hosted lab environment; no external or third-party systems were targeted.

---

## 3. Methodology
1. **Baseline attack (no WAF):** Executed SQL Injection and XSS against DVWA directly, confirming both vulnerabilities were exploitable under default (Low security) configuration.

2. **WAF deployment:** Stood up ModSecurity + OWASP CRS as a reverse proxy in front of DVWA, configured with custom rules targeting the specific payloads used in step 1, in addition to the CRS baseline ruleset.

3. **Detection baseline:** Ran the WAF in `DetectionOnly` mode and re-executed the same payloads, confirming the rules matched (logged) without altering DVWA's responses, establishing that detection logic was accurate before activating enforcement.

4. **Enforcement & re-test:** Switched the WAF to blocking mode (`SecRuleEngine On`) and re-executed the identical payloads, confirming they were now blocked (`403 Forbidden`).

This staged approach **(attack - detect - validate detection - enforce - re-test)** was deliberate: it avoids the risk of enabling blocking rules that either don't fire on the intended traffic or, worse, block legitimate traffic, which is a real concern with WAFs in production.

---

## 4. Findings

### 4.1 SQL Injection
- **Vector**: `id` parameter, DVWA SQL Injection module (GET-based).
- **Technique**: Column count enumerated via `ORDER BY` incrementing (`1' ORDER BY 2 #` succeeded, `1' ORDER BY 3 #` returned a fatal error, confirming a 2-column query) followed by `UNION SELECT user,password FROM users-- -` to extract the full user table, including password hashes.
- **Impact (pre-mitigation)**: Full disclosure of application user credentials via a single unauthenticated-parameter injection point.

![DVWA user table](/dvwa-attack-defend/screenshots/dvwa-sqli-4.png)

### 4.2 Cross-Site Scripting
- **Vector**: Reflected XSS module (name field) and Stored XSS module (guestbook message field).
- **Technique**: `<script>alert("Your device is infected")</script>` injected in both contexts; reflected payload executed immediately on response, stored payload persisted and executed on every subsequent page load or refresh.
- **Impact (pre-mitigation)**: Arbitrary JavaScript execution in victims' browsers; demonstrated potential for session/cookie theft via a redirect-based exfiltration payload.

![XSS Reflected](/dvwa-attack-defend/screenshots/xss-reflected.png)

---

## 5. Mitigation Applied
- Deployed OWASP CRS v4 as the baseline ruleset (SQLi rule family 942..., XSS rule family 941...).

- Added custom rules (`custom-dvwa-rules.conf`) directly targeting the demonstrated attack patterns: `ORDER BY` enumeration, `UNION SELECT` extraction, tautology-based bypass (`OR 1=1`), `<script>` tag injection, inline event-handler XSS, and `javascript:` URI XSS.

- Rules deployed via the CRS "before-CRS" hook point, evaluated ahead of the base ruleset.

## 6. Validation (Re-test Results)
| Attack | Pre-WAF | Post-WAF (DetectionOnly) | Post-WAF (Enforced) |
|---|---|---|---|
| SQLi - `ORDER BY` enumeration | `200 OK`, exploited | `200 OK`, logged (not blocked) | `403 Forbidden`, blocked |
| SQLi - `UNION SELECT` extraction | `200 OK`, data exfiltrated | `200 OK`, logged (not blocked) | `403 Forbidden`, blocked |
| XSS - reflected `<script>` | Alert executed | `200 OK`, logged (not blocked) | `403 Forbidden`, blocked |
| XSS - stored `<script>` | Alert persisted | `200 OK`, logged (not blocked) | `403 Forbidden`, blocked |

Full request/response evidence and log excerpts: see `04-retest-results.md` and `screenshots/`.

---

## 7. Notable Implementation Issues (Troubleshooting)
**Full detail in** `03-waf-config/setup-notes.md`. 

### Summarized:
- A host-only lab network with no internet route initially blocked pulling the WAF image; resolved with a temporary NAT adapter, removed once the image was cached.

- An incorrect mount path meant custom rules were never actually loaded by the WAF for the first deployment attempt, caught by monitoring the rule count in the container's startup log.

- An audit log path override pointed at a file that was never created, silently suppressing all logging; resolved by reverting to the image's stdout default.

- A leftover `SecRuleEngine On` directive inside the custom rules file caused the WAF to block during what was meant to be a non-blocking detection baseline; identified by inspecting the rendered configuration directly inside the container and commented it out.

These are included deliberately: each represents a case where the WAF *appeared* to be doing the right thing (or not) for the wrong reason, and required verifying actual container state (config files, logs, rule counts) rather than trusting the intended configuration.

---

## 8. Conclusion
The deployed ModSecurity WAF, combining OWASP CRS with targeted custom rules, successfully blocks the SQL Injection and XSS attack techniques demonstrated against the unprotected DVWA baseline, with detection validated independently before enforcement was enabled. The staged detect-then-enforce methodology provides confidence that the blocking observed in Phase 3 reflects accurate rule matching rather than incidental over-blocking.

## Author

👩‍💻 **Chukwu PraiseGod**  
Follow my journey: [X](https://x.com/chukwupg) | [LinkedIn](https://linkedin.com/in/chukwupg)  