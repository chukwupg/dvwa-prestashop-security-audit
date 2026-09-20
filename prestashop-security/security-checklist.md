# PrestaShop Security Checklist: Completed

**Target**: PrestaShop instance on AWS (EC2 + RDS)

**Date assessed**: 19/09/2026

**Environment**: This assessment was performed against the live PrestaShop lab instance originally deployed on aws (see [`prestashop-aws-deployment`](https://github.com/chukwupg/prestashop-aws-deployment/) for original deployment steps). No new instance was stood up for this assignment.

> Redaction note: password, real email address, and the RDS endpoint hostname are redacted below and should never be published regardless of environment. The public EC2 IP and the personal SSH source IP used during testing are also redacted, since both are live/identifying even though this is a lab rather than a production system, publishing a real, internet-reachable IP with a documented unpatched CVE invites unsolicited scanning. Non-identifying details (container names, internal architecture, etc.) are left as-is.

---

## Architecture Summary
- **EC2 (Ubuntu)**: Apache + PHP, hosts PrestaShop application
- **RDS (MySQL)**: `prestashop_db`, restricted to EC2-only access
- **Security Groups**: HTTP (80) and HTTPS (443) open to `0.0.0.0/0`; SSH restricted to a single trusted `/32` IP on a non-default port
- **Admin path**: renamed from default `/admin/` to a custom, non-guessable path
- **Install directory**: removed post-installation

---

## Checklist Results

### 1. Ensure application and modules are up to date
**Status: ❌ FAIL**

- Current version: **PrestaShop 8.2.1** (confirmed via Admin Panel > Advanced Parameters > Information)
- Current patched release for this branch: **8.2.6**
- The 8.2.6 release is a **critical security patch** fixing a stored XSS vulnerability in the back-office Customer Service view (GHSA-w9f3-qc75-qgx9)
- Nearly all installed modules flagged "Upgrade available" in Module Manager, consistent with the outdated core

**Remediation**: Update to 8.2.6 or later as soon as possible. Until patched, treat back-office Customer Service access as elevated risk.

**Evidence:** Admin Panel > Advanced Parameters > Information showing version 8.2.1

![prestashop version info](/prestashop-security/screenshots/01-version-info.png.png)

---

### 2. Remove or disable default/demo accounts and sample data
**Status: ⚠️ PARTIAL**

- **Accounts**: PASS 

    Only one employee account exists (`<redacted>@gmail.com`), active, not a generic/default address

- **Catalog data**: FAIL 
    
    All 19 products in the catalog are unmodified default PrestaShop demo items (reference codes `demo_1`-`demo_21`, e.g. "Hummingbird printed t-shirt," "Mountain fox notebook"). No real store inventory present.

**Remediation**: Bulk-delete demo products via Catalog → Products before this instance is used beyond lab/training purposes.

**Evidence:** Catalog > Products showing all 19 default demo items
![catalog](/prestashop-security/screenshots/02-demo-products-catalog.png)

---

### 3. Verify strong admin password and MFA where possible
**Status: ⚠️ PARTIAL**

- **Password**: PASS 
    
    11 characters, mixed upper/lowercase, numbers, and 3 symbols. Reasonable strength, though under the stricter 12+ character bar some policies recommend.

- **MFA**: FAIL 

    No native 2FA section available on employee account settings; no 2FA/authentication module installed in this PrestaShop version.

**Remediation**: Consider extending password to 12+ characters. Since core lacks 2FA in this version, evaluate a dedicated 2FA module from the PrestaShop Marketplace, or enforce access control at the infrastructure layer (IP allowlist on the admin path, see item 9).

**Evidence:** Employee edit page showing no 2FA section available

![employee edit page](/prestashop-security/screenshots/03-employee-edit-no-2fa.png)

---

### 4. Review file/folder permissions and disable directory listing

**Status: ✅ PASS**

- No world-writable directories or files found under the web root
- Key directories (`/var/www/html`, `/config`, admin folder) owned by `www-data:www-data`, permissions `755`/`644` — standard, least-privilege configuration
- Directory listing confirmed disabled from outside: `/modules/`, `/img/`, `/download/` all return `302 Found` redirects, not raw file listings

**Remediation**: None required.

**Evidence**: terminal output of `find`/`ls`/`stat` commands and `curl -I` results showing 302 redirects
![file and folder permissions](/prestashop-security/screenshots/04-permissions-and-listing.png)

---

### 5. Confirm secure session cookie settings (HttpOnly, Secure, SameSite)
**Status: ⚠️ PARTIAL**

- `HttpOnly`: PASS

    Present on all session cookies (storefront and admin)

- `SameSite`: PASS 
    Set to `Lax` consistently

- `Secure`: FAIL
    Absent on all cookies, because the site is served over plain HTTP; this flag cannot be meaningfully set without TLS

**Remediation**: Once TLS is enabled (item 8), configure PrestaShop/Apache to enforce the `Secure` flag on all cookies.

**Note**: Because `HttpOnly` is correctly set, the stored-XSS vulnerability identified in item 1 is partially mitigated at the cookie level, a successful XSS payload could not read the session cookie via `document.cookie`. This does not fix the underlying vulnerability but is a genuine defense-in-depth factor.

**Evidence:** `curl -v` output showing Set-Cookie headers with HttpOnly/SameSite flags (token values redacted)
![http response header](/prestashop-security/screenshots/05-cookie-headers.png)

---

### 6. Disable unnecessary modules/plugins
**Status: ⚠️ PARTIAL**

- No individually high-risk modules identified
- Module set reflects an unmodified default install (stats blocks, theme/navigation modules, cross-selling, contact form, etc.) - consistent with the demo catalog finding in item 2
- Three payment modules enabled simultaneously (Bank Transfer, Cash on Delivery, Payments by Check), none requiring live credentials
- "Distribution API Client" module enabled - used for native module updates.

**Remediation**: Update core + modules together (covered by item 1's fix). Disable "Distribution API Client" if native auto-update isn't in use. Reduce to a single real payment module before production use.

**Evidence:** Module Manager list showing enabled modules and "Upgrade available" flags
![module manager](/prestashop-security/screenshots/06-module-manager-gui.png)

---

### 7. Validate secure configuration for payment and API endpoints
**Status: ✅ PASS**

- Webservice/API confirmed disabled in the admin panel
- `/api/` endpoint returns `404 Not Found` - not routed, no exposed attack surface
- No live payment gateway in use; current payment modules (Bank Transfer, COD, Check) require no API credentials

**Remediation**: None required currently. Re-verify this item if a real payment gateway is added. API key scoping and endpoint authentication would then need dedicated testing.

**Evidencce:

**Advanced Parameters > Webservice showing it toggled off**
![webservice disabled](/prestashop-security/screenshots/07-webservice-disabled.png)

**`/api/` endpoint returns 404 Not Found**
![api endpoint](/prestashop-security/screenshots/07-webservice-api-check.png)

---

### 8. Ensure TLS is configured and enforced
**Status: ❌ FAIL**

- HTTP (port 80): reachable, `200 OK`
- HTTPS (port 443): connection times out — **not listening at all**
- No TLS certificate present anywhere on the server for this application (only the standard OS CA trust bundle, unrelated to serving TLS)
- Security group already permits inbound 443 from `0.0.0.0/0` — no infrastructure change needed once a cert is in place

**Remediation**: Configure TLS via Let's Encrypt/Certbot (requires a domain name) or a self-signed certificate for lab purposes. Enforce HTTP→HTTPS redirect and enable `PS_SSL_ENABLED` once live.

**Evidence:** terminal output of `curl -I https://` timing out and `ss -tlnp` showing empty response indicating port 443 not listening
![http response header](/prestashop-security/screenshots/08-https-timeout.png)

---

### 9. Check for exposed admin paths and use IP restrictions or WAF rules
**Status: ⚠️ PARTIAL**

- Default `/admin/` path correctly returns `404 Not Found` — admin folder renamed as part of prior hardening
- However: the renamed admin path is reachable from `0.0.0.0/0`, same as the public storefront — no IP allowlist or additional access restriction specific to admin access
- No WAF (ModSecurity, AWS WAF, or otherwise) deployed in front of this instance

**Remediation**: Restrict the admin path to trusted IP(s) via Apache `<Location>` directive or a dedicated security-group rule. Deploy a WAF fronting Apache (the same ModSecurity + OWASP CRS pattern used for DVWA in Phase 2 could be adapted here).

**Note**: This finding, combined with item 1 (outdated version with a known critical stored-XSS CVE), forms a coherent and realistic attack narrative — an internet-reachable, unrestricted admin panel running a version with a documented vulnerability.

**Evidence:** 

**`curl -I /admin/` 404 result**
![admin-path](/prestashop-security/screenshots/09-admin-path-404.png)

**EC2 security group inbound rules**
![EC2 inbound rules](/prestashop-security/screenshots/09-ec2-sg-inbound-rules.png)

---

### 10. Backup and recovery verification
**Status: ⚠️ PARTIAL**

- RDS automated backups: enabled
    
    - 1-day retention
    - Backup window 10:25–10:55 UTC
- EC2/EBS snapshots (application files): none found, zero snapshot coverage
- Restore process: never tested end-to-end

**Remediation**: Increase RDS backup retention (7–35 days depending on requirements). Set up scheduled EBS snapshots for the application volume. Perform and document at least one full test restore.

**Screenshot**: `screenshots/10-rds-backup-config.png` — `aws rds describe-db-instances` output or RDS console Maintenance & backups tab

---

## Summary Table

| # | Item | Status |
|---|---|---|
| 1 | App/modules up to date | ❌ FAIL |
| 2 | Remove default/demo accounts & data | ⚠️ PARTIAL |
| 3 | Strong admin password & MFA | ⚠️ PARTIAL |
| 4 | File/folder permissions & directory listing | ✅ PASS |
| 5 | Secure session cookie settings | ⚠️ PARTIAL |
| 6 | Disable unnecessary modules | ⚠️ PARTIAL |
| 7 | Secure payment/API config | ✅ PASS |
| 8 | TLS configured and enforced | ❌ FAIL |
| 9 | Exposed admin paths / IP restriction / WAF | ⚠️ PARTIAL |
| 10 | Backup and recovery verification | ⚠️ PARTIAL |

**2 Pass / 6 Partial / 2 Fail**

## Planned Attack Simulation 

Findings from items 1 and 9 will be used as the basis for the attack simulation: exploiting the documented stored-XSS vulnerability (GHSA-w9f3-qc75-qgx9) present in the current unpatched version (8.2.1), against an admin panel with no IP restriction or WAF protection. See [`attack-simulation-report.md`](/prestashop-security/attack-simulation-report.md) for methodology, evidence, and post-attack mitigation.