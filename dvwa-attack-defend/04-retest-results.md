# Phase 3: Re-test Results (Post-WAF)

## Objective
Re-run the exact SQL Injection and XSS payloads from Phase 1 against DVWA, now fronted by the configured ModSecurity WAF, to confirm the attacks that previously succeeded are now blocked.

## Test Conditions
- Target: `http://192.168.98.10/` (port 80, routed through `modsec-waf`)
- WAF rule engine: `On` (enforcement mode)
- Rules in effect: OWASP CRS v4 (paranoia level 1) + custom rules (`custom-dvwa-rules.conf`)
- DVWA security level: Low (unchanged from Phase 1, to isolate the WAF as the only variable)

## Results

### **SQL Injection**

| Payload | Pre-WAF (Phase 1) | Post-WAF (Phase 3) |
|---|---|---|
| `1' ORDER BY 2 #` | `200 OK` - column count enumerated | `403 Forbidden` - blocked, matched custom rule (ORDER BY enumeration) |
| `1' ORDER BY 3 #` | `200 OK` - fatal error confirmed 2-column query | `403 Forbidden` - blocked |
| `1' UNION SELECT user,password FROM users-- -` | `200 OK` - user table dumped | `403 Forbidden` - blocked, matched custom rule (UNION SELECT) and/or CRS SQLi rules (942xxx family) |

### Evidence

**`docker logs` excerpt showing the 403 + matched rule ID for the first ORDER By enumeration**
![ORDER BY enumeration blocked](/dvwa-attack-defend/screenshots/docker-log-showing-blocked-sqli-attack-1.png)

**`docker logs` excerpt showing the 403 + matched rule ID for the second ORDER By enumeration**
![ORDER BY enumeration blocked](/dvwa-attack-defend/screenshots/docker-log-showing-blocked-sqli-attack-2.png)

**`docker logs` excerpt showing the 403 + matched rule ID for the UNION SELECT request** 
![UNION SELECT request blocked](/dvwa-attack-defend/screenshots/docker-log-showing-blocked-sqli-attack-3.png)

**Browser showing 403 response body for the first ORDER BY enumeration**
![First ORDER BY 403](/dvwa-attack-defend/screenshots/dvwa-sqli-blocked-1.png)

**Browser showing 403 response body for the second ORDER BY enumeration**
![Second ORDER BY 403](/dvwa-attack-defend/screenshots/dvwa-sqli-blocked-2.png)

**Browser showing 403 response body for the UINION SELECT request**
![UNION SELECT 403](/dvwa-attack-defend/screenshots/dvwa-sqli-blocked-3.png)

### **Cross-Site Scripting (XSS)**

| Payload | Pre-WAF (Phase 1) | Post-WAF (Phase 3) |
|---|---|---|
| `<script>alert("Your device is infected")</script>` (Reflected XSS module) | Alert box rendered | `403 Forbidden` - blocked, matched custom `<script>` tag rule |
| `<script>alert("Your device is infected")</script>` (Stored XSS / guestbook) | Alert fired on every page load | `403 Forbidden` - blocked at submission, payload never stored |

### Evidence

**`docker logs` excerpt showing the 403 + matched rule ID for the XSS (reflected) request**
![XSS blocked](/dvwa-attack-defend/screenshots/docker-log-showing-blocked-reflected-xss-attack.png)

**`docker logs` excerpt showing the 403 + matched rule ID for the XSS (stored) request**
![XSS blocked](/dvwa-attack-defend/screenshots/docker-log-showing-blocked-stored-xss-attack.png)

**403 page returned instead of the reflected alert**
![Blocked xss 1](/dvwa-attack-defend/screenshots/reflected-xss-attack-blocked.png)

**403 page returned instead of the persistent stored alert**
![Blocked xss 2](/dvwa-attack-defend/screenshots/stored-xss-attack-blocked.png)


## Rule Engine Confirmation

```
$ docker exec -it modsec-waf grep -r "SecRuleEngine" /etc/nginx/modsecurity.d/
SecRuleEngine On
```

### Evidence

**Rule engine mode On Confirmation**
![On](/dvwa-attack-defend/screenshots/modsec-waf-output-showing-on.png)

## Observations

- All three tested attack techniques (column enumeration, UNION-based extraction, script-tag injection) were successfully blocked once the WAF was switched from `DetectionOnly` to `On`.
- Detection was validated independently in Phase 2b (`DetectionOnly` mode) before enforcement was enabled, confirming the rules matched the correct traffic rather than blocking indiscriminately.
- [Optional, if tested: note here whether case-variation or encoding-based evasion attempts (`UnIoN SeLeCt`, double-URL-encoding) succeeded or were still caught — useful honesty for the defense validation report.]

## Conclusion

The configured ModSecurity WAF (custom rules + OWASP CRS baseline) successfully mitigates the SQL Injection and XSS vectors demonstrated against the unprotected DVWA instance in Phase 1. 

See [`defense-validation-report.md`](/dvwa-attack-defend/defense-validation-report.md) for the full methodology, impact assessment, and mitigation summary.
