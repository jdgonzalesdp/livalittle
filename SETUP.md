# MTA-STS Setup Guide

Complete step-by-step guide for deploying MTA-STS email security for livalittle.com.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [GitHub Pages Setup](#github-pages-setup)
3. [DNS Configuration](#dns-configuration)
4. [Policy File Deployment](#policy-file-deployment)
5. [Verification](#verification)
6. [Going Live](#going-live)
7. [Rollback Procedure](#rollback-procedure)

## Prerequisites

### Required Access

- **GitHub Repository Access:** Admin access to this repository
- **DNS Management:** Access to livalittle.com DNS settings
- **Email Server Access:** Understanding of current MX records

### Required Knowledge

- Basic understanding of DNS records (TXT, CNAME, MX)
- Familiarity with SMTP and email delivery
- Basic command line skills for testing

### Pre-Deployment Checklist

- [ ] All MX servers for livalittle.com support TLS 1.2 or higher
- [ ] Valid SSL/TLS certificates installed on all mail servers
- [ ] DNS provider supports TXT records
- [ ] GitHub Pages can be enabled on this repository
- [ ] Backup of current DNS configuration
- [ ] Email monitoring system in place

## GitHub Pages Setup

### Step 1: Enable GitHub Pages

1. Navigate to repository settings:
   ```
   https://github.com/jdgonzalesdp/livalittle/settings/pages
   ```

2. Configure source:
   - **Source:** Deploy from a branch
   - **Branch:** `main` (or your default branch)
   - **Folder:** `/ (root)`

3. Click "Save"

### Step 2: Configure Custom Domain

1. In the GitHub Pages settings, find "Custom domain"

2. Enter the MTA-STS subdomain:
   ```
   mta-sts.livalittle.com
   ```

3. Click "Save"

4. Wait for DNS check (may take a few minutes)

5. **Important:** Check "Enforce HTTPS" once DNS is propagated
   - MTA-STS **requires** HTTPS
   - GitHub provides free SSL via Let's Encrypt

### Step 3: Verify .nojekyll File

This file should already exist in the repository root. Verify:

```bash
ls -la .nojekyll
```

If missing, create it:

```bash
touch .nojekyll
git add .nojekyll
git commit -m "Add .nojekyll for GitHub Pages"
git push
```

**Why this matters:** Without `.nojekyll`, GitHub Pages uses Jekyll processing, which ignores directories starting with `.` (like `.well-known`), breaking MTA-STS.

## DNS Configuration

### Step 1: Create CNAME Record for Policy Hosting

Point the MTA-STS subdomain to GitHub Pages:

```
Type:  CNAME
Name:  mta-sts
Value: jdgonzalesdp.github.io.
TTL:   3600 (or your preference)
```

**Important:** Include the trailing dot in `jdgonzalesdp.github.io.`

### Step 2: Wait for DNS Propagation

Check DNS propagation:

```bash
# Using dig
dig mta-sts.livalittle.com CNAME

# Using nslookup
nslookup -type=CNAME mta-sts.livalittle.com

# Check from multiple locations
# https://www.whatsmydns.net/#CNAME/mta-sts.livalittle.com
```

Expected result:
```
mta-sts.livalittle.com. 3600 IN CNAME jdgonzalesdp.github.io.
```

### Step 3: Verify HTTPS Access

Once DNS propagates and GitHub Pages enables HTTPS:

```bash
curl -I https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

Expected response:
```
HTTP/2 200
content-type: text/plain; charset=utf-8
...
```

**Troubleshooting:** If you get a certificate error, wait a few more minutes for Let's Encrypt provisioning.

### Step 4: Create MTA-STS Discovery Record

This is the **critical** DNS record that enables MTA-STS:

```
Type:  TXT
Name:  _mta-sts
Value: "v=STSv1; id=20250206"
TTL:   3600
```

**Important Parameters:**

- **v=STSv1:** Version identifier (required)
- **id=20250206:** Policy version ID (format: YYYYMMDD or any unique identifier)
  - Change this ID whenever you update the policy
  - Mail servers cache policy based on this ID
  - Use date format: YYYYMMDD (e.g., 20250206 for Feb 6, 2025)

### Step 5: Verify DNS TXT Record

```bash
dig _mta-sts.livalittle.com TXT

# Or more specifically
dig +short _mta-sts.livalittle.com TXT
```

Expected output:
```
"v=STSv1; id=20250206"
```

## Policy File Deployment

### Step 1: Review Policy Configuration

The policy file is located at `.well-known/mta-sts.txt`:

```
version: STSv1
mode: enforce
mx: *.livalittle.com
max_age: 86400
```

### Step 2: Understanding Policy Parameters

| Parameter | Current Value | Description |
|-----------|---------------|-------------|
| `version` | STSv1 | Protocol version (always STSv1) |
| `mode` | enforce | Policy enforcement level |
| `mx` | *.livalittle.com | Mail server pattern (supports wildcards) |
| `max_age` | 86400 | Cache duration in seconds (24 hours) |

### Step 3: Customizing the Policy (If Needed)

#### Mode Options

**Testing Mode** (for initial deployment):
```
version: STSv1
mode: testing
mx: *.livalittle.com
max_age: 86400
```
- TLS is attempted but not required
- Failures are reported but don't block delivery
- Recommended for initial rollout

**Enforce Mode** (current/production):
```
version: STSv1
mode: enforce
mx: *.livalittle.com
max_age: 86400
```
- TLS is **required**
- Delivery fails if secure connection cannot be established
- Maximum security

**None Mode** (to disable):
```
version: STSv1
mode: none
max_age: 86400
```
- Explicitly disables MTA-STS
- Use for planned deactivation

#### MX Pattern Options

**Wildcard (current):**
```
mx: *.livalittle.com
```
- Matches all mail servers under livalittle.com
- Recommended for flexibility

**Specific servers:**
```
mx: mail.livalittle.com
mx: mail2.livalittle.com
```
- List each mail server explicitly
- Multiple `mx:` lines allowed
- More restrictive

**Third-party providers:**
```
mx: *.google.com
mx: *.protection.outlook.com
```
- For services like Google Workspace, Microsoft 365
- Use provider's documented patterns

#### Max Age Considerations

| Duration | Seconds | Use Case |
|----------|---------|----------|
| 1 hour | 3600 | Testing, frequent changes |
| 24 hours | 86400 | Standard (current) |
| 1 week | 604800 | Stable production |
| 1 month | 2592000 | Very stable infrastructure |

**Trade-offs:**
- **Shorter:** Faster policy updates, more frequent fetches
- **Longer:** Less server load, slower policy changes

### Step 4: Commit Policy Changes

If you modified the policy:

```bash
git add .well-known/mta-sts.txt
git commit -m "Update MTA-STS policy configuration"
git push origin main
```

### Step 5: Wait for GitHub Pages Deployment

GitHub Pages typically deploys within 5-10 minutes:

```bash
# Check deployment status via GitHub Actions
# or monitor the file directly:

watch -n 10 curl -s https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

## Verification

### Step 1: Manual Policy Fetch

```bash
curl https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

Expected output:
```
version: STSv1
mode: enforce
mx: *.livalittle.com
max_age: 86400
```

### Step 2: Check HTTPS Requirements

MTA-STS requires valid HTTPS:

```bash
# Check certificate
openssl s_client -connect mta-sts.livalittle.com:443 -servername mta-sts.livalittle.com

# Verify no redirects
curl -L -v https://mta-sts.livalittle.com/.well-known/mta-sts.txt 2>&1 | grep "< HTTP"
```

### Step 3: Online Validation Tools

Use these services to validate your setup:

1. **MTA-STS Validator**
   - URL: https://aykevl.nl/apps/mta-sts/
   - Enter: livalittle.com
   - Check: DNS record, policy fetch, syntax validation

2. **Hardenize**
   - URL: https://www.hardenize.com/
   - Enter: livalittle.com
   - Review: Complete email security report

3. **MXToolbox**
   - URL: https://mxtoolbox.com/mta-sts.aspx
   - Enter: livalittle.com
   - Verify: Policy accessibility

### Step 4: Command-Line Validation

Using `mta-sts-query` tool (if installed):

```bash
# Install (Python)
pip install mta-sts

# Query policy
mta-sts-query livalittle.com

# Expected output shows policy details
```

### Step 5: DNS Propagation Check

```bash
# Check from multiple nameservers
dig @8.8.8.8 _mta-sts.livalittle.com TXT
dig @1.1.1.1 _mta-sts.livalittle.com TXT
dig @208.67.222.222 _mta-sts.livalittle.com TXT

# Use online tools
# https://www.whatsmydns.net/#TXT/_mta-sts.livalittle.com
```

## Going Live

### Pre-Launch Checklist

- [ ] GitHub Pages is serving policy file via HTTPS
- [ ] DNS TXT record is propagated globally
- [ ] Policy syntax validated with online tools
- [ ] MX servers support TLS 1.2+
- [ ] SSL certificates on mail servers are valid
- [ ] Monitoring/logging is in place
- [ ] Rollback procedure documented

### Recommended Rollout Strategy

#### Phase 1: Testing Mode (Week 1)

1. Deploy with `mode: testing`
2. Monitor email delivery
3. Review any TLS-RPT reports (if configured)
4. Verify no delivery issues

#### Phase 2: Enforce Mode (Week 2+)

1. Change `mode: enforce`
2. Update DNS `id` value (e.g., id=20250213)
3. Monitor delivery closely for 48 hours
4. Review bounce rates and delivery reports

#### Immediate Enforcement (Current Setup)

The repository is already configured with `mode: enforce`. If this is your first deployment:

**Consider temporarily switching to testing mode:**

```
version: STSv1
mode: testing
mx: *.livalittle.com
max_age: 86400
```

Then follow the phased rollout above.

### Monitoring During Rollout

Monitor these metrics:

1. **Email Delivery Rate**
   - Compare before/after MTA-STS activation
   - Look for increased bounce rates

2. **TLS Connection Failures**
   - Check mail server logs
   - Review any delivery errors

3. **Policy Fetch Logs**
   - GitHub Pages access logs (if available)
   - Look for unusual patterns

4. **Certificate Warnings**
   - Ensure no certificate validation failures
   - Monitor SSL/TLS certificate expiration

## Rollback Procedure

If you need to disable MTA-STS quickly:

### Emergency Rollback (Immediate)

Update DNS TXT record to disable:

```
Type:  TXT
Name:  _mta-sts
Value: "v=STSv1; id=20250206_disabled"
```

Change policy to `mode: none`:

```
version: STSv1
mode: none
max_age: 300
```

**Note:** Due to caching (max_age), full rollback takes up to 24 hours with current settings.

### Planned Deactivation

1. Change `mode: enforce` to `mode: testing`
2. Update DNS `id` value
3. Wait 24 hours (or current max_age duration)
4. Monitor for issues
5. If clear, change to `mode: none`
6. Update DNS `id` again
7. Wait another 24 hours
8. Remove DNS TXT record if desired

### Quick Reference: Rollback Timeline

| Action | Time to Take Effect |
|--------|---------------------|
| DNS TXT record update | 5-60 minutes (depends on TTL) |
| GitHub Pages deployment | 5-10 minutes |
| Policy cache expiration | Up to 24 hours (current max_age) |
| Full rollback complete | 24 hours + DNS TTL |

## Post-Deployment

### Regular Maintenance Tasks

- **Monthly:** Verify policy URL is accessible
- **Quarterly:** Review TLS-RPT reports
- **Yearly:** Audit MX server TLS capabilities
- **As needed:** Update policy `id` when making changes

### Updating the Policy

When you need to make changes:

1. Update `.well-known/mta-sts.txt`
2. Commit and push to GitHub
3. Wait for GitHub Pages deployment (5-10 min)
4. Update DNS TXT record with new `id` value
5. Test with validation tools
6. Monitor email delivery

### TLS-RPT Setup (Optional but Recommended)

To receive delivery reports, add this DNS record:

```
Type:  TXT
Name:  _smtp._tls
Value: "v=TLSRPTv1; rua=mailto:tls-reports@livalittle.com"
TTL:   3600
```

You'll receive daily aggregate reports about TLS delivery attempts.

## Troubleshooting

For detailed troubleshooting, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

### Quick Fixes

**Policy not accessible via HTTPS:**
- Check GitHub Pages is enabled
- Verify "Enforce HTTPS" is checked
- Wait for Let's Encrypt certificate provisioning

**DNS record not propagating:**
- Verify correct record format
- Check TTL settings
- Use multiple DNS checkers

**Email delivery failures:**
- Check MX server TLS support
- Verify SSL certificates are valid
- Review mail server logs
- Consider switching to `mode: testing`

## Additional Resources

- [MTA-STS-GUIDE.md](MTA-STS-GUIDE.md) - Protocol deep dive
- [DNS-CONFIGURATION.md](DNS-CONFIGURATION.md) - Advanced DNS setup
- [TESTING.md](TESTING.md) - Comprehensive testing guide
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Common issues and solutions

## Support

For questions or issues:
- Review documentation in this repository
- Check RFC 8461: https://tools.ietf.org/html/rfc8461
- Use validation tools listed above
- Contact your DNS provider or email service provider

---

**Last Updated:** 2025-02-06
**Policy Version:** 20250206
**Status:** Production (Enforce Mode)
