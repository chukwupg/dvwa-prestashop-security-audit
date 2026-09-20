# DVWA & Prestashop Security Audit

A two part web application security project. Part one demonstrates SQL Injection and XSS exploitation against DVWA, then deploys and validates a ModSecurity WAF to block both. 

Part two is a full security assessment of a live PrestaShop deployment on AWS, including a completed hardening checklist and a documented attack simulation that confirms a critical, unpatched stored XSS vulnerability, plus missing TLS and login rate limiting.

## Objective

Perform and block web based attacks against DVWA using SQL Injection and Cross Site Scripting, configure a ModSecurity WAF to stop them, and re test to confirm effectiveness. Additionally, complete a security checklist for a deployed PrestaShop instance and simulate a real attack against it, documenting findings and mitigation.

## Repository Structure

```
dvwa-prestashop-security-audit/
├── README.md
├── dvwa-attack-defend/
│   ├── 01-sqli-attack.md
│   ├── 02-xss-attack.md
│   ├── 03-waf-config/
│   │   ├── docker-compose-waf.yml
│   │   ├── custom-rules/
│   │   │   └── custom-dvwa-rules.conf
│   │   └── setup-notes.md
│   ├── 04-retest-results.md
│   ├── defense-validation-report.md
│   └── screenshots/
├── prestashop-security/
│   ├── security-checklist.md
│   ├── attack-simulation-report.md
│   └── screenshots/
└── Deliverable.pdf
```

## Lab Environment

### DVWA
Deployed via Docker on an Ubuntu server, originally set up during Assignment 2 of my Cybersecurity training at Bincom. 

ModSecurity plus OWASP Core Rule Set reverse proxy (also containerized) and placed in front of it for this assignment.

### PestaShop
A live AWS deployment (EC2 running Apache and PHP, RDS running MySQL), also originally set up in a previous lab I did but reused here for the checklist and attack simulation.

For original deployment steps, see [`prestashop-aws-deployment`](https://github.com/chukwupg/prestashop-aws-deployment/)


## Summary of Work

### Part 1, DVWA Attack and Defend
1. Executed SQL Injection (column enumeration via ORDER BY, data extraction via UNION SELECT) and Cross Site Scripting (reflected and stored) against an unprotected DVWA instance.
2. Deployed ModSecurity with the OWASP Core Rule Set plus custom rules targeting the exact payloads used, in front of DVWA as a reverse proxy.
3. Validated detection in DetectionOnly mode before enabling enforcement, then confirmed both attacks were blocked once the WAF was switched to blocking mode.
4. Full methodology, evidence, and troubleshooting notes are in [`dvwa-attack-defend/`](/dvwa-attack-defend/).

### Part 2, PrestaShop Security Checklist and Attack Simulation
1. Completed a ten item security checklist against the live PrestaShop instance, covering version currency, default accounts and data, password and MFA posture, file permissions, cookie settings, module hygiene, payment and API configuration, TLS, admin path exposure, and backup and recovery.
2. Identified three genuinely exploitable weaknesses from the checklist findings and demonstrated each: a stored XSS vulnerability (CVE 2026 44212) present due to an outdated application version, the complete absence of brute force protection on the admin login, and plaintext credential interception due to the lack of TLS.
3. Full checklist and attack simulation report are in [`prestashop-security/`](/prestashop-security/).


## Deliverables

- Screenshots of attack success and block (DVWA), in [`dvwa-attack-defend/screenshots/`](/dvwa-attack-defend/screenshots/)
- Configured WAF rule file, in [`dvwa-attack-defend/03-waf-config/`](/dvwa-attack-defend/03-waf-config/)
- Short report on defense validation, [`defense-validation-report.md`](/dvwa-attack-defend/defense-validation-report.md)
- Completed PrestaShop security checklist, [`security-checklist.md`](/prestashop-security/security-checklist.md)
- PrestaShop attack simulation report, [`attack-simulation-report.md`](/prestashop-security/attack-simulation-report.md)
- Consolidated Word document for submission, [`Assignment-3-Deliverable.docx`]

## Status

Mitigation and re test for the PrestaShop attack simulation findings (patching to 8.2.6, enabling TLS, adding login rate limiting) are planned as a follow up and will be added to this repository once completed.

## Notes on Redaction

Real IP addresses, database endpoints, personal email addresses, and passwords used during testing have been redacted from all documents in this repository. Only non identifying architecture and configuration details are shown.

## Author

👩‍💻 **Chukwu PraiseGod**  
Follow my journey: [X](https://x.com/chukwupg) | [LinkedIn](https://linkedin.com/in/chukwupg)  