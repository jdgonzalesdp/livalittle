# DNS Configuration Guide

Complete DNS configuration guide for MTA-STS email security on livalittle.com.

## Table of Contents

1. [Overview](#overview)
2. [Required DNS Records](#required-dns-records)
3. [Record Configuration Details](#record-configuration-details)
4. [Provider-Specific Instructions](#provider-specific-instructions)
5. [DNS Propagation](#dns-propagation)
6. [Verification](#verification)
7. [Advanced Configuration](#advanced-configuration)
8. [Troubleshooting DNS Issues](#troubleshooting-dns-issues)
9. [Security Considerations](#security-considerations)

## Overview

MTA-STS requires two DNS records:

1. **CNAME record:** Points policy hosting subdomain to GitHub Pages
2. **TXT record:** Enables MTA-STS discovery and policy versioning

Both records are critical for MTA-STS functionality.

## Required DNS Records

### Record Summary

| Type | Name | Value | TTL | Purpose |
|------|------|-------|-----|---------|
| CNAME | `mta-sts` | `jdgonzalesdp.github.io.` | 3600 | Policy hosting |
| TXT | `_mta-sts` | `"v=STSv1; id=20250206"` | 3600 | Policy discovery |

### Additional Recommended Records

| Type | Name | Value | TTL | Purpose |
|------|------|-------|-----|---------|
| TXT | `_smtp._tls` | `"v=TLSRPTv1; rua=mailto:tls-reports@livalittle.com"` | 3600 | TLS reporting |
| MX | `@` | `10 mail.livalittle.com.` | 3600 | Mail routing |

## Record Configuration Details

### 1. CNAME Record (Policy Hosting)

**Purpose:** Points the MTA-STS subdomain to GitHub Pages for policy file hosting.

#### Configuration

```
Type:  CNAME
Name:  mta-sts
Host:  mta-sts.livalittle.com (fully qualified)
Value: jdgonzalesdp.github.io.
TTL:   3600
```

#### Important Notes

**Trailing Dot:**
- Include the trailing dot: `jdgonzalesdp.github.io.`
- Some DNS providers add it automatically
- Without it, some systems may append your domain

**CNAME Restrictions:**
- Cannot coexist with other record types for same name
- Cannot be used for apex/root domain (@)
- Only for subdomains (mta-sts in this case)

**Common Formats by Provider:**

**Format 1 (Separate fields):**
```
Name:  mta-sts
Value: jdgonzalesdp.github.io.
```

**Format 2 (Full hostname in name):**
```
Name:  mta-sts.livalittle.com
Value: jdgonzalesdp.github.io.
```

**Format 3 (Relative name):**
```
Name:  mta-sts
Value: jdgonzalesdp.github.io
```

Use the format your DNS provider supports.

### 2. TXT Record (MTA-STS Discovery)

**Purpose:** Announces MTA-STS support and policy version.

#### Configuration

```
Type:  TXT
Name:  _mta-sts
Host:  _mta-sts.livalittle.com (fully qualified)
Value: v=STSv1; id=20250206
TTL:   3600
```

#### Value Format

**Syntax:**
```
v=STSv1; id=<identifier>
```

**Components:**

**v=STSv1**
- Version identifier (required)
- Always `STSv1` (case-sensitive)
- No other versions exist

**id=<identifier>**
- Policy version identifier
- Can be any string (alphanumeric, hyphens, underscores)
- Recommended format: `YYYYMMDD` (date-based)
- Must change when policy file changes
- Used for cache invalidation

**Examples:**
```
v=STSv1; id=20250206        # Date-based (recommended)
v=STSv1; id=2025-02-06      # Alternative date format
v=STSv1; id=version_123     # Version number
v=STSv1; id=abc123def456    # Hash-based
```

#### Quotes in TXT Records

**Depends on DNS provider:**

**With quotes (most common):**
```
"v=STSv1; id=20250206"
```

**Without quotes (some providers):**
```
v=STSv1; id=20250206
```

**Both are valid;** some providers add quotes automatically.

#### Important Notes

**Underscore Prefix:**
- Name must start with underscore: `_mta-sts`
- Required by RFC 8461
- Indicates this is a special-purpose record

**Semicolon Separator:**
- Separate parameters with semicolon: `; `
- Space after semicolon is optional but recommended

**Case Sensitivity:**
- `v=STSv1` is case-sensitive
- `id` value is case-insensitive

### 3. TXT Record (TLS Reporting - Optional but Recommended)

**Purpose:** Receive reports about TLS connection attempts and failures.

#### Configuration

```
Type:  TXT
Name:  _smtp._tls
Host:  _smtp._tls.livalittle.com (fully qualified)
Value: v=TLSRPTv1; rua=mailto:tls-reports@livalittle.com
TTL:   3600
```

#### Value Format

**Syntax:**
```
v=TLSRPTv1; rua=<reporting-uri>
```

**Components:**

**v=TLSRPTv1**
- TLS-RPT version (required)
- Always `TLSRPTv1`

**rua=<uri>**
- Report URI (where to send reports)
- Format: `mailto:email@domain.com`
- Multiple URIs supported (comma-separated)

**Examples:**
```
v=TLSRPTv1; rua=mailto:tls-reports@livalittle.com
v=TLSRPTv1; rua=mailto:reports@livalittle.com,mailto:backup@livalittle.com
v=TLSRPTv1; rua=https://example.com/tls-reports
```

#### Report Format

Daily aggregate JSON reports sent by supporting mail servers:
- TLS connection successes/failures
- Certificate validation issues
- Policy fetch problems
- MTA-STS policy details

### 4. MX Records (Existing - Must Match Policy)

**Purpose:** Define mail servers for livalittle.com.

#### Configuration

```
Type:     MX
Name:     @ (or livalittle.com)
Priority: 10
Value:    mail.livalittle.com.
TTL:      3600
```

#### Important: Policy Alignment

Your MX records must match the pattern in `.well-known/mta-sts.txt`:

**Current policy:**
```
mx: *.livalittle.com
```

**Matches:**
- `mail.livalittle.com` ✓
- `smtp.livalittle.com` ✓
- `mx1.livalittle.com` ✓
- `anything.livalittle.com` ✓

**Does NOT match:**
- `livalittle.com` ✗ (no subdomain)
- `mail.example.com` ✗ (different domain)
- `sub.mail.livalittle.com` ✗ (nested subdomain)

**If using third-party email providers:**

**Google Workspace:**
```
mx: *.google.com
mx: *.googlemail.com
```

**Microsoft 365:**
```
mx: *.mail.protection.outlook.com
```

**Adjust policy file accordingly.**

## Provider-Specific Instructions

### Cloudflare

#### CNAME Record

1. Log in to Cloudflare dashboard
2. Select livalittle.com domain
3. Navigate to DNS → Records
4. Click "Add record"
5. Configure:
   - Type: `CNAME`
   - Name: `mta-sts`
   - Target: `jdgonzalesdp.github.io`
   - Proxy status: **DNS only** (gray cloud, not proxied)
   - TTL: `Auto` or `3600`
6. Click "Save"

**Important:** Disable Cloudflare proxy (orange cloud) for this record. MTA-STS requires direct access to GitHub Pages.

#### TXT Record

1. Click "Add record"
2. Configure:
   - Type: `TXT`
   - Name: `_mta-sts`
   - Content: `v=STSv1; id=20250206`
   - TTL: `Auto` or `3600`
3. Click "Save"

### AWS Route 53

#### CNAME Record

1. Open Route 53 console
2. Select Hosted Zone: livalittle.com
3. Click "Create record"
4. Configure:
   - Record name: `mta-sts`
   - Record type: `CNAME`
   - Value: `jdgonzalesdp.github.io`
   - TTL: `3600`
   - Routing policy: `Simple routing`
5. Click "Create records"

#### TXT Record

1. Click "Create record"
2. Configure:
   - Record name: `_mta-sts`
   - Record type: `TXT`
   - Value: `"v=STSv1; id=20250206"` (include quotes)
   - TTL: `3600`
3. Click "Create records"

### Google Cloud DNS

#### Using Console

1. Navigate to Cloud DNS
2. Select livalittle.com zone
3. Click "Add standard"

**CNAME:**
- DNS name: `mta-sts.livalittle.com.`
- Resource record type: `CNAME`
- TTL: `3600`
- Canonical name: `jdgonzalesdp.github.io.`

**TXT:**
- DNS name: `_mta-sts.livalittle.com.`
- Resource record type: `TXT`
- TTL: `3600`
- TXT data: `"v=STSv1; id=20250206"`

#### Using gcloud CLI

```bash
# CNAME record
gcloud dns record-sets create mta-sts.livalittle.com. \
  --zone=livalittle-zone \
  --type=CNAME \
  --ttl=3600 \
  --rrdatas=jdgonzalesdp.github.io.

# TXT record
gcloud dns record-sets create _mta-sts.livalittle.com. \
  --zone=livalittle-zone \
  --type=TXT \
  --ttl=3600 \
  --rrdatas='"v=STSv1; id=20250206"'
```

### GoDaddy

#### CNAME Record

1. Log in to GoDaddy
2. Navigate to DNS Management for livalittle.com
3. Click "Add" → Select "CNAME"
4. Configure:
   - Host: `mta-sts`
   - Points to: `jdgonzalesdp.github.io`
   - TTL: `1 Hour` (3600 seconds)
5. Click "Save"

#### TXT Record

1. Click "Add" → Select "TXT"
2. Configure:
   - Host: `_mta-sts`
   - TXT Value: `v=STSv1; id=20250206`
   - TTL: `1 Hour`
3. Click "Save"

**Note:** GoDaddy automatically adds the domain suffix.

### Namecheap

#### CNAME Record

1. Log in to Namecheap
2. Navigate to Domain List → Manage → Advanced DNS
3. Click "Add New Record"
4. Configure:
   - Type: `CNAME Record`
   - Host: `mta-sts`
   - Value: `jdgonzalesdp.github.io.`
   - TTL: `Automatic` or `1 hour`
5. Click checkmark to save

#### TXT Record

1. Click "Add New Record"
2. Configure:
   - Type: `TXT Record`
   - Host: `_mta-sts`
   - Value: `v=STSv1; id=20250206`
   - TTL: `Automatic`
3. Click checkmark to save

## DNS Propagation

### Expected Timelines

| Stage | Duration | Notes |
|-------|----------|-------|
| DNS provider update | Immediate - 5 min | Changes saved in provider's system |
| Local cache clear | 0 - TTL | Based on previous record's TTL |
| Global propagation | 1 - 24 hours | Depends on TTL and caching |
| Full propagation | Up to 48 hours | Rare; most within 1-4 hours |

### Factors Affecting Propagation

**TTL (Time To Live):**
- Previous record's TTL must expire first
- New record's TTL takes effect after
- Lower TTL = faster updates but more queries

**DNS Caching:**
- ISP DNS caches
- Local resolver caches
- Browser DNS caches
- OS DNS caches

**Geographic Distribution:**
- Different regions update at different rates
- Some countries have slower DNS infrastructure

### Checking Propagation

**Command-line tools:**

```bash
# Check specific nameserver
dig @8.8.8.8 _mta-sts.livalittle.com TXT
dig @1.1.1.1 _mta-sts.livalittle.com TXT

# Check CNAME
dig mta-sts.livalittle.com CNAME

# Check from multiple nameservers
dig @8.8.8.8 _mta-sts.livalittle.com TXT +short
dig @1.1.1.1 _mta-sts.livalittle.com TXT +short
dig @208.67.222.222 _mta-sts.livalittle.com TXT +short
```

**Online tools:**

- https://www.whatsmydns.net/
- https://dnschecker.org/
- https://mxtoolbox.com/SuperTool.aspx

**Check both records:**
- https://www.whatsmydns.net/#CNAME/mta-sts.livalittle.com
- https://www.whatsmydns.net/#TXT/_mta-sts.livalittle.com

### Clearing Local Cache

**Windows:**
```cmd
ipconfig /flushdns
```

**macOS:**
```bash
sudo dscacheutil -flushcache
sudo killall -HUP mDNSResponder
```

**Linux:**
```bash
sudo systemd-resolve --flush-caches  # systemd-resolved
sudo /etc/init.d/nscd restart        # nscd
sudo service dnsmasq restart         # dnsmasq
```

**Chrome browser:**
```
Navigate to: chrome://net-internals/#dns
Click: "Clear host cache"
```

## Verification

### 1. Verify CNAME Record

```bash
dig mta-sts.livalittle.com CNAME +short
```

Expected output:
```
jdgonzalesdp.github.io.
```

### 2. Verify TXT Record

```bash
dig _mta-sts.livalittle.com TXT +short
```

Expected output:
```
"v=STSv1; id=20250206"
```

### 3. Verify HTTPS Resolution

```bash
# Should resolve to GitHub Pages IP
dig mta-sts.livalittle.com A

# Test HTTPS access
curl -I https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

Expected:
```
HTTP/2 200
```

### 4. Complete Validation

```bash
# One-liner to check everything
echo "CNAME:" && dig +short mta-sts.livalittle.com CNAME && \
echo "TXT:" && dig +short _mta-sts.livalittle.com TXT && \
echo "HTTPS:" && curl -s -o /dev/null -w "%{http_code}" https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

Expected output:
```
CNAME:
jdgonzalesdp.github.io.
TXT:
"v=STSv1; id=20250206"
HTTPS:
200
```

### 5. Use MTA-STS Validators

**Online validators:**
- https://aykevl.nl/apps/mta-sts/ (enter: livalittle.com)
- https://www.hardenize.com/ (enter: livalittle.com)
- https://mxtoolbox.com/mta-sts.aspx (enter: livalittle.com)

These tools check:
- DNS TXT record presence and format
- CNAME record resolution
- HTTPS policy fetch
- Policy file syntax
- Certificate validity

## Advanced Configuration

### DNSSEC

**Purpose:** Cryptographically sign DNS records to prevent tampering.

**Benefit for MTA-STS:**
- Protects against DNS cache poisoning
- Ensures TXT record integrity
- Complements MTA-STS security

**Setup varies by provider:**

**Cloudflare:** Automatic when enabled
**Route 53:** Manual KSK/ZSK configuration
**Google Cloud DNS:** Managed DNSSEC available

**Configuration:**
1. Enable DNSSEC in DNS provider
2. Copy DS records
3. Add DS records to parent zone (domain registrar)
4. Verify with: `dig +dnssec _mta-sts.livalittle.com TXT`

### CAA Records

**Purpose:** Specify which Certificate Authorities can issue certificates for your domain.

**Recommended for MTA-STS:**
```
livalittle.com. 3600 IN CAA 0 issue "letsencrypt.org"
livalittle.com. 3600 IN CAA 0 issuewild "letsencrypt.org"
```

**Why:** GitHub Pages uses Let's Encrypt, so explicitly allowing it prevents unauthorized certificate issuance.

### Multiple MTA-STS Subdomains

**Standard setup:**
```
mta-sts.livalittle.com → GitHub Pages
```

**With backup:**
```
mta-sts.livalittle.com     → Primary (GitHub Pages)
mta-sts-backup.livalittle.com → Backup (alternative hosting)
```

**Note:** RFC 8461 only supports single `mta-sts` subdomain. This is for manual failover only.

### Short TTL During Testing

During initial deployment, use shorter TTLs for faster iteration:

```
Type:  TXT
Name:  _mta-sts
Value: v=STSv1; id=20250206
TTL:   300  # 5 minutes
```

**Increase to 3600 (1 hour) once stable.**

## Troubleshooting DNS Issues

### Issue: DNS Not Propagating

**Symptoms:**
- Records not visible after 24 hours
- Inconsistent results from different nameservers

**Solutions:**
1. Verify records in DNS provider console
2. Check for typos in record names/values
3. Ensure records are not in "draft" state
4. Contact DNS provider support
5. Try different DNS checkers

### Issue: CNAME Not Resolving to GitHub Pages

**Symptoms:**
- CNAME exists but HTTPS doesn't work
- Certificate errors

**Solutions:**
1. Verify CNAME target: `jdgonzalesdp.github.io.` (with dot)
2. Check GitHub Pages is enabled
3. Wait for Let's Encrypt certificate provisioning (can take 20 minutes)
4. Verify "Enforce HTTPS" is checked in GitHub Pages settings
5. Check for conflicting A/AAAA records on `mta-sts` subdomain

### Issue: TXT Record Format Incorrect

**Symptoms:**
- Validators report "invalid format"
- MTA-STS not detected

**Solutions:**
1. Check for extra spaces: `v=STSv1; id=20250206` (space after semicolon)
2. Verify case: `STSv1` (not `stsv1` or `STSV1`)
3. Check quotes: Some providers require quotes, some don't
4. Ensure no linebreaks in TXT value
5. Verify underscore prefix: `_mta-sts` (not `mta-sts`)

### Issue: Multiple TXT Records

**Symptoms:**
- Conflicting TXT values
- Validators show wrong/old version

**Solutions:**
1. List all TXT records for `_mta-sts`:
   ```bash
   dig _mta-sts.livalittle.com TXT
   ```
2. Delete duplicate/old records
3. Ensure only one MTA-STS TXT record exists

### Issue: Cloudflare Proxying

**Symptoms:**
- CNAME record exists but policy fetch fails
- Certificate mismatch errors

**Solutions:**
1. Disable Cloudflare proxy (orange cloud → gray cloud)
2. Set to "DNS only" mode for `mta-sts` subdomain
3. Wait for DNS propagation

**Why:** Cloudflare proxy intercepts traffic, presenting Cloudflare's certificate instead of GitHub Pages' certificate, breaking MTA-STS.

## Security Considerations

### DNS Hijacking Protection

**Risks:**
- Attacker compromises DNS provider
- Attacker changes TXT record ID to force policy refetch
- Attacker serves malicious policy (requires HTTPS compromise too)

**Mitigations:**
- Enable 2FA on DNS provider account
- Use strong, unique passwords
- Enable DNSSEC
- Monitor DNS changes (alerts)
- Use DNS provider with audit logs

### Minimize DNS TTL for Security?

**Lower TTL pros:**
- Faster policy updates
- Quicker incident response
- Reduced cache poisoning window

**Lower TTL cons:**
- More DNS queries
- Higher DNS costs (some providers)
- More load on DNS infrastructure

**Recommendation:**
- **Testing:** 300-600 seconds (5-10 minutes)
- **Production:** 3600 seconds (1 hour)
- **Stable:** 7200-86400 seconds (2 hours - 1 day)

### Preventing Policy Rollback Attacks

**Attack scenario:**
- Attacker compromises DNS
- Changes TXT `id` to old value
- Senders fetch old policy (potentially weaker)

**Mitigations:**
- Never reuse policy IDs
- Use incrementing/date-based IDs
- Monitor DNS for unexpected changes
- Keep policy in enforce mode

### DNS Provider Security

**Choose providers with:**
- Two-factor authentication (2FA)
- DNSSEC support
- Audit logging
- DDoS protection
- API access controls
- SOC 2 certification

**Avoid:**
- Providers without 2FA
- Free DNS with no security features
- Providers with history of compromises

## Summary Checklist

Before going live, verify:

- [ ] CNAME record: `mta-sts.livalittle.com` → `jdgonzalesdp.github.io.`
- [ ] TXT record: `_mta-sts.livalittle.com` → `"v=STSv1; id=20250206"`
- [ ] DNS propagated globally (check with multiple tools)
- [ ] HTTPS access works: `https://mta-sts.livalittle.com/.well-known/mta-sts.txt`
- [ ] MX records match policy pattern (*.livalittle.com)
- [ ] Validated with online MTA-STS checker
- [ ] TLS-RPT record configured (optional but recommended)
- [ ] DNSSEC enabled (optional but recommended)
- [ ] DNS provider account secured with 2FA

## Additional Resources

- [SETUP.md](SETUP.md) - Complete deployment guide
- [MTA-STS-GUIDE.md](MTA-STS-GUIDE.md) - Protocol deep dive
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Common issues
- [TESTING.md](TESTING.md) - Validation procedures

## DNS Tools Reference

### Command-Line Tools

```bash
# Query tools
dig
nslookup
host
drill

# DNS debugging
dig +trace _mta-sts.livalittle.com TXT
dig +dnssec _mta-sts.livalittle.com TXT

# Specific nameserver
dig @8.8.8.8 _mta-sts.livalittle.com TXT
dig @1.1.1.1 _mta-sts.livalittle.com TXT
```

### Online Tools

- **General DNS:** https://www.whatsmydns.net/
- **MX Records:** https://mxtoolbox.com/
- **DNSSEC:** https://dnssec-analyzer.verisignlabs.com/
- **MTA-STS Validator:** https://aykevl.nl/apps/mta-sts/
- **Email Security:** https://www.hardenize.com/

---

**Last Updated:** 2025-02-06
**Domain:** livalittle.com
**DNS Provider:** (Your provider here)
**DNSSEC Status:** (Enabled/Disabled)
