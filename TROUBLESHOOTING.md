# MTA-STS Troubleshooting Guide

Comprehensive troubleshooting guide for diagnosing and resolving MTA-STS issues with livalittle.com.

## Table of Contents

1. [Quick Diagnostics](#quick-diagnostics)
2. [DNS Issues](#dns-issues)
3. [Policy File Issues](#policy-file-issues)
4. [GitHub Pages Issues](#github-pages-issues)
5. [Certificate Issues](#certificate-issues)
6. [Email Delivery Issues](#email-delivery-issues)
7. [Validation Tool Errors](#validation-tool-errors)
8. [Performance Issues](#performance-issues)
9. [Security Concerns](#security-concerns)
10. [Emergency Procedures](#emergency-procedures)

## Quick Diagnostics

Run these commands to quickly identify common issues:

### One-Line Health Check

```bash
echo "=== MTA-STS Health Check ===" && \
echo "DNS TXT Record:" && dig +short _mta-sts.livalittle.com TXT && \
echo "DNS CNAME Record:" && dig +short mta-sts.livalittle.com CNAME && \
echo "HTTPS Status:" && curl -s -o /dev/null -w "HTTP %{http_code}\n" https://mta-sts.livalittle.com/.well-known/mta-sts.txt && \
echo "Policy Content:" && curl -s https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

### Expected Output

```
=== MTA-STS Health Check ===
DNS TXT Record:
"v=STSv1; id=20250206"
DNS CNAME Record:
jdgonzalesdp.github.io.
HTTPS Status:
HTTP 200
Policy Content:
version: STSv1
mode: enforce
mx: *.livalittle.com
max_age: 86400
```

### Quick Validation

```bash
# Use online validator (copy output URL)
echo "Check your MTA-STS status at:"
echo "https://aykevl.nl/apps/mta-sts/?domain=livalittle.com"
```

## DNS Issues

### Issue 1: DNS TXT Record Not Found

**Symptoms:**
```bash
$ dig _mta-sts.livalittle.com TXT
# No answer section or empty response
```

**Diagnosis:**

```bash
# Check authoritative nameservers
dig livalittle.com NS

# Query authoritative server directly
dig @<nameserver> _mta-sts.livalittle.com TXT
```

**Causes:**
1. Record not created yet
2. Typo in record name (`mta-sts` vs `_mta-sts`)
3. DNS propagation delay
4. Record in wrong zone/domain

**Solutions:**

**Check record in DNS provider:**
- Log in to DNS management console
- Search for `_mta-sts` record
- Verify it's in the correct zone (livalittle.com)

**Verify record format:**
```
Name:  _mta-sts  (or _mta-sts.livalittle.com, depending on provider)
Type:  TXT
Value: v=STSv1; id=20250206
```

**Wait for propagation:**
- Check TTL of previous record (if any)
- Wait at least TTL duration
- Check multiple nameservers:
  ```bash
  dig @8.8.8.8 _mta-sts.livalittle.com TXT
  dig @1.1.1.1 _mta-sts.livalittle.com TXT
  ```

**Clear local DNS cache:**
```bash
# macOS
sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder

# Linux
sudo systemd-resolve --flush-caches

# Windows
ipconfig /flushdns
```

### Issue 2: Invalid TXT Record Format

**Symptoms:**
- Validators report "invalid policy format"
- MTA-STS not recognized

**Common formatting errors:**

**Missing version:**
```
# ✗ Wrong
id=20250206

# ✓ Correct
v=STSv1; id=20250206
```

**Wrong case:**
```
# ✗ Wrong
v=stsv1; id=20250206
v=STSV1; id=20250206

# ✓ Correct
v=STSv1; id=20250206
```

**Missing semicolon:**
```
# ✗ Wrong
v=STSv1 id=20250206

# ✓ Correct
v=STSv1; id=20250206
```

**Extra spaces:**
```
# ✗ Wrong (extra spaces around equals)
v = STSv1 ; id = 20250206

# ✓ Correct
v=STSv1; id=20250206
```

**Solutions:**

```bash
# Verify current value
dig +short _mta-sts.livalittle.com TXT

# Should return exactly:
"v=STSv1; id=20250206"
```

Update in DNS provider to match exact format.

### Issue 3: CNAME Record Not Resolving

**Symptoms:**
```bash
$ dig mta-sts.livalittle.com CNAME
# No CNAME in answer section
```

**Diagnosis:**

```bash
# Check what record exists
dig mta-sts.livalittle.com ANY

# Check CNAME specifically
dig mta-sts.livalittle.com CNAME +trace
```

**Causes:**
1. CNAME not created
2. CNAME target incorrect
3. Conflicting A/AAAA records
4. DNS propagation delay

**Solutions:**

**Verify CNAME configuration:**
```
Name:  mta-sts
Type:  CNAME
Value: jdgonzalesdp.github.io.
```

**Check for conflicting records:**
```bash
# Look for A or AAAA records
dig mta-sts.livalittle.com A
dig mta-sts.livalittle.com AAAA
```

**If A/AAAA records exist:**
- Delete them (CNAME cannot coexist with other record types)
- Keep only CNAME

**Verify target:**
```bash
# CNAME should point to GitHub Pages
dig +short mta-sts.livalittle.com CNAME
# Should return: jdgonzalesdp.github.io.

# Final resolution should be GitHub Pages IP
dig +short mta-sts.livalittle.com A
# Should return GitHub Pages IP (e.g., 185.199.108.153)
```

### Issue 4: Multiple TXT Records

**Symptoms:**
- Inconsistent validator results
- Old policy ID showing up

**Diagnosis:**

```bash
# List all TXT records
dig _mta-sts.livalittle.com TXT

# May show multiple records:
"v=STSv1; id=20250206"
"v=STSv1; id=20250101"  # Old record
```

**Solutions:**

1. Log in to DNS provider
2. Find all `_mta-sts` TXT records
3. Delete old/duplicate records
4. Keep only the current record
5. Wait for TTL expiration

## Policy File Issues

### Issue 5: Policy File Not Accessible (404 Error)

**Symptoms:**
```bash
$ curl https://mta-sts.livalittle.com/.well-known/mta-sts.txt
404: Not Found
```

**Diagnosis:**

```bash
# Check GitHub Pages status
curl -I https://jdgonzalesdp.github.io/livalittle/.well-known/mta-sts.txt

# Check if .nojekyll exists
ls -la /home/user/livalittle/.nojekyll
```

**Causes:**
1. `.nojekyll` file missing
2. File in wrong location
3. GitHub Pages not enabled
4. GitHub Pages deployment failed
5. Branch mismatch

**Solutions:**

**Ensure .nojekyll exists:**
```bash
cd /home/user/livalittle
touch .nojekyll
git add .nojekyll
git commit -m "Add .nojekyll for GitHub Pages"
git push
```

**Verify file location:**
```bash
ls -la .well-known/mta-sts.txt
# File must be at: .well-known/mta-sts.txt
```

**Check GitHub Pages settings:**
1. Go to repository settings → Pages
2. Verify:
   - Source: Deploy from branch
   - Branch: main (or your default branch)
   - Folder: / (root)
   - Custom domain: mta-sts.livalittle.com
   - Enforce HTTPS: ✓ checked

**Wait for deployment:**
- GitHub Pages deploys within 5-10 minutes
- Check deployment status in Actions tab
- Wait, then retry

### Issue 6: Policy File Has Wrong Content

**Symptoms:**
- Validators report syntax errors
- Old policy content displayed

**Diagnosis:**

```bash
# Fetch current policy
curl https://mta-sts.livalittle.com/.well-known/mta-sts.txt

# Compare with local file
cat .well-known/mta-sts.txt
```

**Causes:**
1. Changes not committed/pushed
2. GitHub Pages cache
3. CDN caching
4. Wrong branch deployed

**Solutions:**

**Verify local changes are committed:**
```bash
git status
git log -1 .well-known/mta-sts.txt
```

**If uncommitted:**
```bash
git add .well-known/mta-sts.txt
git commit -m "Update MTA-STS policy"
git push origin main
```

**Wait for GitHub Pages deployment:**
```bash
# Check every 30 seconds
watch -n 30 curl -s https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

**Check which branch is deployed:**
- Repository settings → Pages → Check branch
- Ensure it matches your working branch

### Issue 7: Policy Syntax Errors

**Symptoms:**
- Validators report "invalid policy"
- MTA-STS not enforced

**Common syntax errors:**

**Missing required field:**
```
# ✗ Missing version
mode: enforce
mx: *.livalittle.com
max_age: 86400

# ✓ Correct
version: STSv1
mode: enforce
mx: *.livalittle.com
max_age: 86400
```

**Wrong mode value:**
```
# ✗ Wrong
mode: strict

# ✓ Correct (one of these)
mode: testing
mode: enforce
mode: none
```

**Invalid max_age:**
```
# ✗ Wrong (not a number)
max_age: 1 day

# ✓ Correct
max_age: 86400
```

**Extra characters:**
```
# ✗ Wrong (backslashes from previous version)
version: STSv1\
mode: enforce\

# ✓ Correct
version: STSv1
mode: enforce
```

**Solutions:**

**Validate format:**
```bash
cat .well-known/mta-sts.txt
```

**Should exactly match:**
```
version: STSv1
mode: enforce
mx: *.livalittle.com
max_age: 86400
```

**No extra whitespace, no backslashes, no special characters.**

## GitHub Pages Issues

### Issue 8: HTTPS Certificate Error

**Symptoms:**
```bash
$ curl https://mta-sts.livalittle.com/.well-known/mta-sts.txt
SSL certificate problem: unable to get local issuer certificate
```

**Diagnosis:**

```bash
# Check certificate
openssl s_client -connect mta-sts.livalittle.com:443 -servername mta-sts.livalittle.com

# Look for:
# - Issuer: Let's Encrypt
# - Subject: mta-sts.livalittle.com
# - Validity dates
```

**Causes:**
1. Let's Encrypt certificate not provisioned yet
2. Custom domain not configured correctly
3. "Enforce HTTPS" not enabled
4. DNS not pointing to GitHub Pages

**Solutions:**

**Wait for certificate provisioning:**
- Can take up to 20 minutes after DNS setup
- GitHub automatically provisions via Let's Encrypt
- Check "Enforce HTTPS" status in Pages settings

**Verify DNS:**
```bash
# CNAME should resolve to GitHub Pages
dig +short mta-sts.livalittle.com
# Should eventually resolve to GitHub Pages IP
```

**Re-save custom domain:**
1. Go to repository settings → Pages
2. Remove custom domain
3. Save
4. Re-add custom domain: mta-sts.livalittle.com
5. Save
6. Wait for certificate provisioning

**Check "Enforce HTTPS":**
- Must be checked
- If grayed out, wait for certificate

### Issue 9: GitHub Pages Not Updating

**Symptoms:**
- Changes pushed but old content served
- Deployment successful but policy unchanged

**Diagnosis:**

```bash
# Check last commit
git log -1

# Check GitHub Pages deployment time
# (via GitHub Actions or repository Insights → Traffic)

# Compare timestamps
```

**Causes:**
1. GitHub Pages caching
2. CDN caching
3. Browser caching
4. Wrong branch deployed

**Solutions:**

**Force cache bypass:**
```bash
# Add cache-busting parameter
curl "https://mta-sts.livalittle.com/.well-known/mta-sts.txt?$(date +%s)"

# Use different browser/incognito
```

**Verify deployment:**
1. Go to repository → Actions tab
2. Check latest "pages build and deployment" workflow
3. Ensure it succeeded
4. Check timestamp

**Re-trigger deployment:**
```bash
# Make a trivial change and push
git commit --allow-empty -m "Trigger Pages rebuild"
git push
```

**Check branch:**
- Settings → Pages → Verify correct branch is selected

## Certificate Issues

### Issue 10: MX Server Certificate Invalid

**Symptoms:**
- Email delivery failures
- TLS-RPT reports certificate errors
- Sender logs show TLS handshake failures

**Diagnosis:**

```bash
# Check MX records
dig livalittle.com MX

# Test SMTP TLS (replace with your MX server)
openssl s_client -connect mail.livalittle.com:25 -starttls smtp
```

**Causes:**
1. Expired certificate on mail server
2. Self-signed certificate
3. Wrong hostname in certificate
4. Certificate not trusted

**Solutions:**

**Check certificate validity:**
```bash
echo | openssl s_client -connect mail.livalittle.com:25 -starttls smtp 2>/dev/null | openssl x509 -noout -dates
```

**Renew certificate:**
- Use Let's Encrypt (free, automated)
- Install certbot on mail server
- Configure auto-renewal

**Verify certificate matches hostname:**
```bash
echo | openssl s_client -connect mail.livalittle.com:25 -starttls smtp 2>/dev/null | openssl x509 -noout -subject
```

Subject should include: `CN=mail.livalittle.com`

### Issue 11: Policy Fetch Certificate Error

**Symptoms:**
- Policy URL shows certificate warning
- Validators can't fetch policy

**Diagnosis:**

```bash
# Check policy URL certificate
curl -v https://mta-sts.livalittle.com/.well-known/mta-sts.txt 2>&1 | grep -i "certificate"
```

**Causes:**
1. GitHub Pages certificate not provisioned
2. Custom domain misconfigured
3. DNS pointing to wrong location

**Solutions:**

See [Issue 8: HTTPS Certificate Error](#issue-8-https-certificate-error)

## Email Delivery Issues

### Issue 12: Email Bouncing After MTA-STS Deployment

**Symptoms:**
- Emails from some senders bounce
- Bounce message mentions TLS or certificate errors
- Delivery failures correlate with MTA-STS activation

**Diagnosis:**

**Check bounce messages:**
- Look for keywords: "TLS", "certificate", "MTA-STS", "policy"
- Note which senders are affected

**Review TLS-RPT reports (if configured):**
```bash
# Check email at tls-reports@livalittle.com
# Look for failure reasons
```

**Test SMTP TLS manually:**
```bash
# Test from external server
openssl s_client -connect mail.livalittle.com:25 -starttls smtp
```

**Causes:**
1. MX server doesn't support TLS
2. MX server certificate invalid
3. MX hostname doesn't match policy pattern
4. Firewall blocking port 587/465

**Solutions:**

**Temporarily switch to testing mode:**
```
version: STSv1
mode: testing
mx: *.livalittle.com
max_age: 86400
```

**Update DNS `id`:**
```
v=STSv1; id=20250206_testing
```

**Fix underlying TLS issues:**
1. Install valid certificate on MX server
2. Enable TLS in mail server config
3. Test TLS connectivity
4. Switch back to enforce mode once fixed

### Issue 13: MX Server Doesn't Match Policy

**Symptoms:**
- Validators report MX mismatch
- Some email bounces

**Diagnosis:**

```bash
# Check MX records
dig livalittle.com MX

# Check policy pattern
curl -s https://mta-sts.livalittle.com/.well-known/mta-sts.txt | grep "^mx:"
```

**Example problem:**
```
MX record:    mail.example.com
Policy mx:    *.livalittle.com
Result:       ✗ No match
```

**Causes:**
1. Using third-party mail provider
2. MX records point to different domain
3. Policy pattern too restrictive

**Solutions:**

**Update policy to match MX servers:**

**For third-party providers:**
```
# Google Workspace
mx: *.google.com
mx: *.googlemail.com

# Microsoft 365
mx: *.mail.protection.outlook.com

# Proofpoint
mx: *.pphosted.com
```

**Or use multiple MX patterns:**
```
version: STSv1
mode: enforce
mx: *.livalittle.com
mx: *.google.com
max_age: 86400
```

**Update policy file, commit, push, and change DNS `id`.**

## Validation Tool Errors

### Issue 14: "Policy ID Not Changed"

**Symptoms:**
- Validator warns: "Policy ID should be updated"
- Policy content changed but ID unchanged

**Cause:**
- Updated policy file but forgot to update DNS TXT `id`

**Solution:**

**Update DNS TXT record:**
```
Old: v=STSv1; id=20250206
New: v=STSv1; id=20250207  (or current date)
```

**Wait for DNS propagation, then retest.**

### Issue 15: "Unable to Fetch Policy"

**Symptoms:**
- Validator error: "Could not retrieve policy"
- Policy URL returns 404 or timeout

**Diagnosis:**

```bash
# Test policy fetch manually
curl -v https://mta-sts.livalittle.com/.well-known/mta-sts.txt

# Check DNS resolution
dig mta-sts.livalittle.com

# Check GitHub Pages status
curl -I https://jdgonzalesdp.github.io/livalittle/
```

**Solutions:**

See [Issue 5: Policy File Not Accessible](#issue-5-policy-file-not-accessible-404-error)

### Issue 16: "DNS Record Not Found"

**Symptoms:**
- Validator error: "No MTA-STS DNS record found"
- TXT query returns empty

**Solutions:**

See [Issue 1: DNS TXT Record Not Found](#issue-1-dns-txt-record-not-found)

## Performance Issues

### Issue 17: Slow Policy Fetch

**Symptoms:**
- Policy URL loads slowly (>5 seconds)
- Intermittent timeouts

**Diagnosis:**

```bash
# Measure response time
time curl -s https://mta-sts.livalittle.com/.well-known/mta-sts.txt >/dev/null

# Check from multiple locations
# https://www.webpagetest.org/
```

**Causes:**
1. GitHub Pages temporary slowness
2. Geographic distance from GitHub CDN
3. Network issues

**Solutions:**

**Usually temporary:**
- GitHub Pages is generally fast
- Wait and retry

**If persistent:**
- Check GitHub Status: https://www.githubstatus.com/
- Consider alternative hosting (advanced)

### Issue 18: High DNS Query Volume

**Symptoms:**
- DNS costs increasing
- Provider throttling queries

**Cause:**
- Short TTL on DNS records causing frequent queries

**Solution:**

**Increase TTL:**
```
Current: 300 (5 minutes)
Recommended: 3600 (1 hour)
Stable: 86400 (1 day)
```

**Trade-off:** Longer TTL = slower updates but fewer queries.

## Security Concerns

### Issue 19: Suspected DNS Hijacking

**Symptoms:**
- DNS records changed without authorization
- Unexpected policy ID changes
- Unknown MX servers appearing

**Immediate Actions:**

1. **Reset DNS provider credentials**
   - Change password immediately
   - Enable 2FA if not already enabled
   - Review account access logs

2. **Verify current DNS records**
   ```bash
   dig _mta-sts.livalittle.com TXT
   dig mta-sts.livalittle.com CNAME
   dig livalittle.com MX
   ```

3. **Restore correct records**
   ```
   _mta-sts.livalittle.com TXT:   v=STSv1; id=20250206
   mta-sts.livalittle.com CNAME:  jdgonzalesdp.github.io.
   ```

4. **Enable DNSSEC** (if available)
   - Prevents DNS record tampering
   - Configuration varies by provider

5. **Monitor for further changes**
   - Set up DNS monitoring alerts
   - Regular audits of DNS records

### Issue 20: Suspected Policy File Compromise

**Symptoms:**
- Policy content changed in repository
- Unauthorized commits
- Unknown policy served

**Immediate Actions:**

1. **Secure GitHub account**
   - Change password
   - Enable 2FA
   - Review authorized applications
   - Check account activity

2. **Review commit history**
   ```bash
   git log --all --oneline .well-known/mta-sts.txt
   ```

3. **Revert unauthorized changes**
   ```bash
   git revert <bad-commit-hash>
   git push origin main
   ```

4. **Verify policy content**
   ```bash
   cat .well-known/mta-sts.txt
   ```

5. **Update DNS `id` to force refresh**
   ```
   v=STSv1; id=20250206_restored
   ```

## Emergency Procedures

### Emergency: Disable MTA-STS Immediately

**When to use:**
- Critical email delivery issues
- MX server completely down
- Need to bypass MTA-STS immediately

**Steps:**

1. **Update policy to `mode: none`**
   ```
   version: STSv1
   mode: none
   max_age: 300
   ```

2. **Commit and push**
   ```bash
   git add .well-known/mta-sts.txt
   git commit -m "Emergency: Disable MTA-STS"
   git push origin main
   ```

3. **Update DNS `id` immediately**
   ```
   v=STSv1; id=20250206_disabled
   ```

4. **Wait for propagation**
   - DNS: TTL duration (3600 seconds = 1 hour)
   - Policy cache: max_age (now 300 seconds = 5 minutes)
   - Total: Up to 1 hour 5 minutes

**Note:** Due to caching, immediate disablement is not possible. This is a security feature.

### Emergency: Rollback to Testing Mode

**When to use:**
- Delivery issues but not critical
- Need time to investigate
- Want reporting but not enforcement

**Steps:**

1. **Update policy to `mode: testing`**
   ```
   version: STSv1
   mode: testing
   mx: *.livalittle.com
   max_age: 3600
   ```

2. **Commit and push**
   ```bash
   git add .well-known/mta-sts.txt
   git commit -m "Rollback to testing mode"
   git push origin main
   ```

3. **Update DNS `id`**
   ```
   v=STSv1; id=20250206_testing
   ```

4. **Monitor TLS-RPT reports**
   - Identify root cause
   - Fix issues
   - Switch back to enforce mode when ready

## Getting Help

### Before Asking for Help

Gather this information:

```bash
# Run diagnostic script
echo "=== MTA-STS Diagnostic Report ===" > mta-sts-report.txt
echo "Date: $(date)" >> mta-sts-report.txt
echo "" >> mta-sts-report.txt
echo "DNS TXT:" >> mta-sts-report.txt
dig _mta-sts.livalittle.com TXT >> mta-sts-report.txt
echo "" >> mta-sts-report.txt
echo "DNS CNAME:" >> mta-sts-report.txt
dig mta-sts.livalittle.com CNAME >> mta-sts-report.txt
echo "" >> mta-sts-report.txt
echo "MX Records:" >> mta-sts-report.txt
dig livalittle.com MX >> mta-sts-report.txt
echo "" >> mta-sts-report.txt
echo "Policy Fetch:" >> mta-sts-report.txt
curl -v https://mta-sts.livalittle.com/.well-known/mta-sts.txt >> mta-sts-report.txt 2>&1
echo "" >> mta-sts-report.txt
echo "Local Policy File:" >> mta-sts-report.txt
cat .well-known/mta-sts.txt >> mta-sts-report.txt

cat mta-sts-report.txt
```

### Support Resources

1. **Validation Tools:**
   - https://aykevl.nl/apps/mta-sts/
   - https://www.hardenize.com/

2. **Documentation:**
   - [README.md](README.md)
   - [SETUP.md](SETUP.md)
   - [MTA-STS-GUIDE.md](MTA-STS-GUIDE.md)
   - [DNS-CONFIGURATION.md](DNS-CONFIGURATION.md)

3. **Official Specification:**
   - RFC 8461: https://tools.ietf.org/html/rfc8461

4. **Community:**
   - GitHub Issues (this repository)
   - Email security forums
   - Server Fault / Stack Exchange

### Escalation Path

1. **Review documentation** (this file and others)
2. **Use validation tools** to identify specific errors
3. **Check GitHub Pages status** (https://www.githubstatus.com/)
4. **Consult DNS provider support** for DNS issues
5. **Review mail server logs** for delivery issues
6. **Open GitHub issue** with diagnostic report

## Checklist for Common Problems

Use this checklist to systematically troubleshoot:

- [ ] DNS TXT record exists and has correct format
- [ ] DNS CNAME record points to GitHub Pages
- [ ] GitHub Pages is enabled and deploying
- [ ] `.nojekyll` file exists in repository
- [ ] Policy file is at `.well-known/mta-sts.txt`
- [ ] Policy file has correct syntax (no extra characters)
- [ ] HTTPS access works (status 200)
- [ ] Certificate is valid (Let's Encrypt)
- [ ] MX records match policy pattern
- [ ] MX servers support TLS
- [ ] MX servers have valid certificates
- [ ] DNS `id` updated after policy changes
- [ ] DNS propagated globally
- [ ] Validated with online tools

---

**Last Updated:** 2025-02-06
**Version:** 1.0
**For:** livalittle.com MTA-STS deployment
