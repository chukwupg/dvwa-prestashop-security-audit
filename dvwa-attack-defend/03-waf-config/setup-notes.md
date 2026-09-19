# WAF Setup Notes: ModSecurity + OWASP CRS in front of DVWA
**Bincom Training: Assignment 3, Phase 2**

## 1. Architecture

```
Attacker/Browser → modsec-waf (Nginx + ModSecurity + OWASP CRS, port 80)
                        │
                        └──► dvwa (Apache/PHP, internal port 80, host debug port 8081)
                                 │
                                 └──► dvwa-db (MariaDB, internal port 3306)
```

All three containers run on a shared Docker bridge network (`dvwa-net`) on an Ubuntu server (host-only lab network, no direct internet access). `modsec-waf` is the only container publicly exposed on the lab network, on port 80. `dvwa` retains a host-only debug port (`8081`) for direct access when isolating issues from the WAF layer.

---

## 2. Deployment Steps

1. Confirmed existing `dvwa` and `dvwa-db` containers (carried over from Assignment 2 [`dvwa-vulnerability-assessment-lab`](https://github.com/chukwupg/dvwa-vulnerability-assessment-lab)) on network `dvwa-net`.
2. Freed host port 80 by rebinding `dvwa`'s port mapping to `8081`, since port 80 was needed for the WAF:
   ```bash
   docker rm -f dvwa
   docker run -d --name dvwa --network dvwa-net -p 8081:80 -e DB_SERVER=dvwa-db ghcr.io/digininja/dvwa:latest
   ```
3. Deployed `modsec-waf` (image: `owasp/modsecurity-crs:nginx`) via `docker-compose-waf.yml`, joined to the same `dvwa-net` network, proxying to `dvwa` by container name.
4. Verified end-to-end connectivity: DVWA login page loads via `http://192.168.98.10/` (port 80, through the WAF), confirmed via browser and `curl -v` (response headers show `Server: nginx` from the WAF, with DVWA's PHP session cookies passed through, proving the WAF is genuinely proxying, not just coincidentally reachable).

### Evidence

**Existing dvwa and dvwa-db containers deployed on Ubuntu server**
![Existing lab setup](/dvwa-attack-defend/screenshots/existing_dvwa-and-dvwa-db.png)

**DVWA port rebinding**
![DVWA port rebinding](/dvwa-attack-defend/screenshots/rebinding-dvwa-port-mapping.png)

**`modsec-waf` deployed** 
![modsec-waf](/dvwa-attack-defend/screenshots/modsec-waf-deployed.png)

**DVWA accessible via `modsec-waf` port 80**
![DVWA](/dvwa-attack-defend/screenshots/dvwa-accessible-through-waf.png) 

**HTTP response header proving the WAF is proxying**
![HTTP response header](/dvwa-attack-defend/screenshots/http-response-headers-confirming-firewall.png)

---

## 3. Configuration

- **Base ruleset**: OWASP Core Rule Set v4 (bundled with the image), paranoia level 1.
- **Custom rules**: `custom-dvwa-rules.conf`, targeting the exact attack patterns used in Phase 1 - `ORDER BY` column enumeration, `UNION SELECT` extraction, tautology-based bypass (`1' OR '1'='1`), `<script>` injection, inline event-handler XSS, and `javascript:` URI XSS.
- **Rule engine mode**: toggled between `DetectionOnly` (baseline validation) and `On` (enforcement), controlled via the `MODSEC_RULE_ENGINE` environment variable in `docker-compose-waf.yml`.


**For the Custom rule see** [`custom-dvwa-rule.conf`](/dvwa-attack-defend/03-waf-config/custom-rules/custom-dvwa-rules.conf)

**For toggling the rule engine mode, edit** [`docker-compose-waf.yml`](/dvwa-attack-defend/03-waf-config/docker-compose-waf.yml)

### Evidence

**Rule engine mode DetectOnly**
![DetectOnly mode](/dvwa-attack-defend/screenshots/waf-detectiononly.png)

**Rule engine mode DetectOnly Confirmation**
![DetectOnly](/dvwa-attack-defend/screenshots/modsec-waf-output-showing-detectonly.png)

**Rule engine mode On**
![On mode](/dvwa-attack-defend/screenshots/waf-blocking-on.png)

**Rule engine mode On Confirmation**
![On](/dvwa-attack-defend/screenshots/modsec-waf-output-showing-on.png)

---

## 4. Troubleshooting Log

Faced Several configuration issues during setup, each corrected before the baseline/blocking tests were considered valid:

**a) DNS resolution failure pulling the WAF image**
The lab server sits on a host-only network with no internet route. `docker compose up -d` failed resolving `registry-1.docker.io` ("server misbehaving") when pulling `owasp/modsecurity-crs:nginx`. Resolved by temporarily attaching a second NAT-mode network adapter to the VM for internet access during the image pull only. The host-only adapter used for all lab traffic remained unchanged and isolated throughout; the NAT adapter was removed once the image was cached locally.

**b) Custom rules never loaded**
Initial mount pointed the custom rules file at a `custom/` subdirectory (`/etc/modsecurity.d/owasp-crs/rules/custom/`) that the image does not scan. Confirmed via the loaded rule count in the container's startup log staying at the CRS base count (849) with no increase. Corrected by mounting the custom rules file at one of the image's actual hook points instead:
```yaml
volumes:
  - ./custom-rules/custom-dvwa-rules.conf:/etc/modsecurity.d/owasp-crs/rules/REQUEST-900-EXCLUSION-RULES-BEFORE-CRS.conf
```
This file is loaded before the CRS ruleset runs, despite its "exclusion rules" naming, it accepts full `SecRule` definitions, not just exclusions.

**c) Audit log silently not writing**
`SecAuditLog` was explicitly overridden to a file path (`/var/log/modsecurity/audit.log`) that nothing in the container ever created, so ModSecurity had no valid audit destination, no errors were surfaced, entries simply never appeared. Fixed by commenting out the override entirely and letting the image's actual default (`/dev/stdout`) apply, so audit entries stream directly into `docker logs`.

**d) DetectionOnly baseline was blocking anyway**
After fixing (b) and (c), test payloads still returned `403 Forbidden` even with `MODSEC_RULE_ENGINE=DetectionOnly` set in the compose file. 

  **Root cause:** the custom rules file itself contained a hardcoded `SecRuleEngine On` directive, left over from an earlier draft where the file was written to stand alone. Because this file loads after the main config applies the compose file's env-var-driven setting, it silently overrode `DetectionOnly` back to enforcement. 
  
  **Resolution:** Commented out the directive `SecRuleEngine On` from the custom rules file, leaving `SecRuleEngine` controlled solely by the compose file's `MODSEC_RULE_ENGINE` variable.

---

## 5. Validation

- **DetectionOnly mode**: SQLi (`ORDER BY` enumeration) and XSS (`<script>` reflected) payloads returned `200 OK` from DVWA (unmodified, as if no WAF were present), while `docker logs modsec-waf` showed matching rule IDs logged for each request, confirming detection logic was correct before enforcement was enabled.

- **Enforcement mode** (`MODSEC_RULE_ENGINE=On`): the same payloads returned `403 Forbidden`, with the same rule IDs logged as the disruptive-action trigger.

This progression (detect without blocking, then confirm blocking only after validating detection) is intentional: it demonstrates the rules fire on the correct traffic before enforcement risk is introduced, rather than blocking blindly and hoping the coverage is accurate.
