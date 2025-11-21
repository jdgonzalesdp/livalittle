# MTA-STS Testing and Validation Guide

Comprehensive guide for testing and validating MTA-STS implementation for livalittle.com.

## Table of Contents

1. [Pre-Deployment Testing](#pre-deployment-testing)
2. [DNS Testing](#dns-testing)
3. [Policy File Testing](#policy-file-testing)
4. [End-to-End Testing](#end-to-end-testing)
5. [Online Validation Tools](#online-validation-tools)
6. [Command-Line Testing](#command-line-testing)
7. [SMTP TLS Testing](#smtp-tls-testing)
8. [Automated Testing](#automated-testing)
9. [Production Monitoring](#production-monitoring)
10. [Testing Checklist](#testing-checklist)

## Pre-Deployment Testing

Before enabling MTA-STS, verify these prerequisites:

### 1. Test MX Server TLS Support

```bash
# Get MX records
dig livalittle.com MX +short

# Test STARTTLS on each MX server (replace with actual MX host)
openssl s_client -connect mail.livalittle.com:25 -starttls smtp
```

**Expected output includes:**
```
CONNECTED
depth=2 ...
verify return:1
...
New, TLSv1.2, Cipher is ECDHE-RSA-AES256-GCM-SHA384
...
250 HELP
```

**Look for:**
- Successful connection
- TLS version 1.2 or higher
- Valid certificate chain
- Proper cipher suite

### 2. Test MX Server Certificate

```bash
# Check certificate details
echo | openssl s_client -connect mail.livalittle.com:25 -starttls smtp 2>/dev/null | openssl x509 -noout -text

# Check certificate validity dates
echo | openssl s_client -connect mail.livalittle.com:25 -starttls smtp 2>/dev/null | openssl x509 -noout -dates

# Check certificate subject
echo | openssl s_client -connect mail.livalittle.com:25 -starttls smtp 2>/dev/null | openssl x509 -noout -subject
```

**Verify:**
- Certificate is not expired
- Certificate is from trusted CA (not self-signed)
- Subject matches MX hostname
- Certificate chain is complete

### 3. Verify GitHub Pages Deployment

```bash
# Test file access before DNS changes
curl -v https://jdgonzalesdp.github.io/livalittle/.well-known/mta-sts.txt
```

**Expected:**
- HTTP 200 status
- Correct policy content
- Valid HTTPS certificate

## DNS Testing

### 1. Test TXT Record

```bash
# Query TXT record
dig _mta-sts.livalittle.com TXT +short

# Expected output:
"v=STSv1; id=20250206"
```

### 2. Test TXT Record Format

```bash
# Detailed query
dig _mta-sts.livalittle.com TXT

# Should show:
# - ANSWER SECTION with TXT record
# - Correct format: v=STSv1; id=<identifier>
# - No extra spaces or characters
```

### 3. Test CNAME Record

```bash
# Query CNAME
dig mta-sts.livalittle.com CNAME +short

# Expected output:
jdgonzalesdp.github.io.
```

### 4. Test Full Resolution Chain

```bash
# Follow CNAME chain to final IP
dig mta-sts.livalittle.com +short

# Should show GitHub Pages IP addresses, e.g.:
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

### 5. Test DNS from Multiple Nameservers

```bash
# Google DNS
dig @8.8.8.8 _mta-sts.livalittle.com TXT +short

# Cloudflare DNS
dig @1.1.1.1 _mta-sts.livalittle.com TXT +short

# Quad9
dig @9.9.9.9 _mta-sts.livalittle.com TXT +short

# All should return the same result
```

### 6. Test DNS Propagation

```bash
# Check propagation status
# Use online tool: https://www.whatsmydns.net/#TXT/_mta-sts.livalittle.com

# Or check multiple locations via command line
for ns in 8.8.8.8 1.1.1.1 9.9.9.9 208.67.222.222; do
  echo "Nameserver $ns:"
  dig @$ns _mta-sts.livalittle.com TXT +short
  echo ""
done
```

### 7. Test DNSSEC (if enabled)

```bash
# Query with DNSSEC validation
dig _mta-sts.livalittle.com TXT +dnssec

# Look for:
# - ad flag (authenticated data)
# - RRSIG record (signature)
```

## Policy File Testing

### 1. Test HTTPS Access

```bash
# Basic fetch
curl https://mta-sts.livalittle.com/.well-known/mta-sts.txt

# Expected output:
version: STSv1
mode: enforce
mx: *.livalittle.com
max_age: 86400
```

### 2. Test HTTP Status Code

```bash
# Check status code only
curl -s -o /dev/null -w "%{http_code}\n" https://mta-sts.livalittle.com/.well-known/mta-sts.txt

# Expected:
200
```

### 3. Test HTTPS Certificate

```bash
# Check certificate details
curl -v https://mta-sts.livalittle.com/.well-known/mta-sts.txt 2>&1 | grep -i "subject:\|issuer:\|expire"

# Verify:
# - Issuer: Let's Encrypt
# - Subject: mta-sts.livalittle.com
# - Not expired
```

### 4. Test Policy Syntax

```bash
# Fetch and validate format
curl -s https://mta-sts.livalittle.com/.well-known/mta-sts.txt | while IFS=: read key value; do
  key=$(echo $key | xargs)
  value=$(echo $value | xargs)
  echo "$key: $value"
done

# Should show clean key-value pairs:
# version: STSv1
# mode: enforce
# mx: *.livalittle.com
# max_age: 86400
```

### 5. Test Policy Content Type

```bash
# Check Content-Type header
curl -I https://mta-sts.livalittle.com/.well-known/mta-sts.txt | grep -i content-type

# Expected:
Content-Type: text/plain; charset=utf-8
```

### 6. Test No Redirects

```bash
# Check for redirects
curl -L -v https://mta-sts.livalittle.com/.well-known/mta-sts.txt 2>&1 | grep "< HTTP\|< Location"

# Should only show:
# < HTTP/2 200
# No Location headers (no redirects)
```

### 7. Test Cache Headers

```bash
# Check cache-related headers
curl -I https://mta-sts.livalittle.com/.well-known/mta-sts.txt | grep -i "cache\|age\|etag"

# Note GitHub Pages caching behavior
```

## End-to-End Testing

### 1. Manual MTA-STS Flow Test

Simulate complete MTA-STS discovery and validation:

```bash
#!/bin/bash

DOMAIN="livalittle.com"

echo "=== MTA-STS End-to-End Test for $DOMAIN ==="
echo ""

# Step 1: DNS Discovery
echo "1. Checking DNS TXT record..."
TXT=$(dig +short _mta-sts.$DOMAIN TXT)
echo "   Found: $TXT"

if [[ $TXT == *"STSv1"* ]]; then
  echo "   ✓ TXT record is valid"
else
  echo "   ✗ TXT record is invalid or missing"
  exit 1
fi
echo ""

# Step 2: Policy Fetch
echo "2. Fetching policy via HTTPS..."
POLICY=$(curl -s https://mta-sts.$DOMAIN/.well-known/mta-sts.txt)
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" https://mta-sts.$DOMAIN/.well-known/mta-sts.txt)
echo "   HTTP Status: $HTTP_CODE"

if [ "$HTTP_CODE" = "200" ]; then
  echo "   ✓ Policy fetch successful"
else
  echo "   ✗ Policy fetch failed"
  exit 1
fi
echo ""

# Step 3: Parse Policy
echo "3. Parsing policy..."
echo "$POLICY" | while IFS=: read key value; do
  echo "   $key:$value"
done
echo ""

# Step 4: MX Lookup
echo "4. Checking MX records..."
MX=$(dig +short $DOMAIN MX)
echo "$MX" | while read priority mx; do
  echo "   Priority $priority: $mx"
done
echo ""

# Step 5: Pattern Matching
echo "5. Verifying MX matches policy pattern..."
PATTERN=$(echo "$POLICY" | grep "^mx:" | cut -d: -f2 | xargs)
echo "   Policy pattern: $PATTERN"

echo "$MX" | while read priority mx; do
  mx=$(echo $mx | sed 's/\.$//')
  if [[ $mx == $PATTERN ]]; then
    echo "   ✓ $mx matches"
  elif [[ $PATTERN == \*.* ]]; then
    base_domain=$(echo $PATTERN | sed 's/\*\.//')
    if [[ $mx == *$base_domain ]]; then
      echo "   ✓ $mx matches wildcard pattern"
    else
      echo "   ✗ $mx does NOT match"
    fi
  else
    echo "   ✗ $mx does NOT match"
  fi
done

echo ""
echo "=== Test Complete ==="
```

Save as `test-mta-sts.sh` and run:
```bash
chmod +x test-mta-sts.sh
./test-mta-sts.sh
```

### 2. Test with Python Script

```python
#!/usr/bin/env python3

import dns.resolver
import requests
import sys

def test_mta_sts(domain):
    print(f"=== Testing MTA-STS for {domain} ===\n")

    # Test 1: DNS TXT Record
    print("1. Testing DNS TXT record...")
    try:
        txt_record = f"_mta-sts.{domain}"
        answers = dns.resolver.resolve(txt_record, 'TXT')
        for rdata in answers:
            txt_value = rdata.to_text().strip('"')
            print(f"   Found: {txt_value}")
            if "STSv1" in txt_value:
                print("   ✓ Valid MTA-STS TXT record")
            else:
                print("   ✗ Invalid TXT record")
                return False
    except Exception as e:
        print(f"   ✗ DNS lookup failed: {e}")
        return False

    print()

    # Test 2: Policy Fetch
    print("2. Testing policy fetch...")
    try:
        policy_url = f"https://mta-sts.{domain}/.well-known/mta-sts.txt"
        response = requests.get(policy_url, timeout=10)
        print(f"   HTTP Status: {response.status_code}")

        if response.status_code == 200:
            print("   ✓ Policy fetch successful")
            print("\n   Policy content:")
            for line in response.text.strip().split('\n'):
                print(f"   {line}")
        else:
            print("   ✗ Policy fetch failed")
            return False
    except Exception as e:
        print(f"   ✗ HTTPS request failed: {e}")
        return False

    print()

    # Test 3: MX Records
    print("3. Testing MX records...")
    try:
        mx_records = dns.resolver.resolve(domain, 'MX')
        for rdata in mx_records:
            print(f"   Priority {rdata.preference}: {rdata.exchange}")
        print("   ✓ MX records found")
    except Exception as e:
        print(f"   ✗ MX lookup failed: {e}")
        return False

    print("\n=== All tests passed ===")
    return True

if __name__ == "__main__":
    domain = "livalittle.com"
    success = test_mta_sts(domain)
    sys.exit(0 if success else 1)
```

Save as `test_mta_sts.py` and run:
```bash
pip install dnspython requests
python3 test_mta_sts.py
```

## Online Validation Tools

### 1. MTA-STS Validator (Recommended)

**URL:** https://aykevl.nl/apps/mta-sts/

**Steps:**
1. Open URL in browser
2. Enter: `livalittle.com`
3. Click "Check"

**What it tests:**
- DNS TXT record presence and format
- Policy HTTPS fetch
- Policy syntax validation
- MX record matching
- Certificate validity

**Expected result:** All checks should be green/passing.

### 2. Hardenize

**URL:** https://www.hardenize.com/

**Steps:**
1. Open URL
2. Enter: `livalittle.com`
3. Click "Scan"

**What it tests:**
- Complete email security posture
- MTA-STS configuration
- DANE (if configured)
- TLS-RPT
- DMARC, SPF, DKIM
- Certificate analysis

**Review:** Email Security section for MTA-STS status.

### 3. MXToolbox MTA-STS Check

**URL:** https://mxtoolbox.com/mta-sts.aspx

**Steps:**
1. Open URL
2. Enter: `livalittle.com`
3. Click "MTA-STS Lookup"

**What it tests:**
- DNS discovery
- Policy retrieval
- Basic syntax validation

### 4. Internet.nl (Netherlands)

**URL:** https://internet.nl/

**Steps:**
1. Select "Email Test"
2. Enter: `postmaster@livalittle.com` (or any @livalittle.com address)
3. Run test

**What it tests:**
- Comprehensive email security
- MTA-STS support
- DANE support
- IPv6 connectivity
- DNSSEC

## Command-Line Testing

### Using mta-sts-query (Python Tool)

**Installation:**
```bash
pip install mta-sts
```

**Usage:**
```bash
# Query policy for domain
mta-sts-query livalittle.com

# Expected output:
Policy:
  version: STSv1
  mode: enforce
  max_age: 86400
  mx:
    - *.livalittle.com
```

**Advanced usage:**
```bash
# Show detailed information
mta-sts-query --verbose livalittle.com

# Test specific policy URL
mta-sts-query --policy-url https://mta-sts.livalittle.com/.well-known/mta-sts.txt livalittle.com
```

### Using Postfix MTA-STS Resolver

**Installation (Debian/Ubuntu):**
```bash
sudo apt install postfix-mta-sts-resolver
```

**Testing:**
```bash
# Query policy
mta-sts-query livalittle.com

# Test with specific MX
mta-sts-query --mx mail.livalittle.com livalittle.com
```

## SMTP TLS Testing

### Test MX Server TLS Capability

```bash
# Get MX record
MX=$(dig +short livalittle.com MX | awk '{print $2}' | head -1 | sed 's/\.$//')

echo "Testing SMTP TLS for: $MX"

# Connect and test STARTTLS
(
  sleep 1; echo "EHLO test.example.com"
  sleep 1; echo "STARTTLS"
  sleep 2; echo "QUIT"
) | openssl s_client -connect $MX:25 -starttls smtp -quiet 2>&1 | grep -E "250|220|TLS|Cipher"
```

### Test Certificate Matching

```bash
# Extract certificate subject
MX=$(dig +short livalittle.com MX | awk '{print $2}' | head -1 | sed 's/\.$//')

echo "MX server: $MX"
echo "Certificate subject:"
echo | openssl s_client -connect $MX:25 -starttls smtp 2>/dev/null | openssl x509 -noout -subject

# Subject should match or include $MX hostname
```

### Test Multiple MX Servers

```bash
#!/bin/bash

DOMAIN="livalittle.com"

echo "=== Testing TLS for all MX servers of $DOMAIN ==="
echo ""

dig +short $DOMAIN MX | sort -n | while read priority mx; do
  mx=$(echo $mx | sed 's/\.$//')
  echo "Testing MX (priority $priority): $mx"

  # Test STARTTLS
  timeout 10 bash -c "(echo 'QUIT' | openssl s_client -connect $mx:25 -starttls smtp 2>&1)" > /tmp/tls_test.txt

  if grep -q "CONNECTED" /tmp/tls_test.txt; then
    echo "  ✓ Connection successful"
  else
    echo "  ✗ Connection failed"
    continue
  fi

  if grep -q "Verify return code: 0" /tmp/tls_test.txt; then
    echo "  ✓ Certificate valid"
  else
    echo "  ⚠ Certificate verification issue"
    grep "Verify return code" /tmp/tls_test.txt
  fi

  echo ""
done

rm -f /tmp/tls_test.txt
```

## Automated Testing

### Continuous Monitoring Script

```bash
#!/bin/bash

# mta-sts-monitor.sh
# Run this script periodically (e.g., via cron) to monitor MTA-STS health

DOMAIN="livalittle.com"
LOG_FILE="/var/log/mta-sts-monitor.log"
ALERT_EMAIL="admin@livalittle.com"

log() {
  echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOG_FILE"
}

alert() {
  log "ALERT: $1"
  echo "$1" | mail -s "MTA-STS Alert for $DOMAIN" "$ALERT_EMAIL"
}

# Test 1: DNS TXT Record
log "Checking DNS TXT record..."
TXT=$(dig +short _mta-sts.$DOMAIN TXT | tr -d '"')
if [[ $TXT == *"STSv1"* ]]; then
  log "✓ DNS TXT record OK: $TXT"
else
  alert "DNS TXT record missing or invalid"
  exit 1
fi

# Test 2: Policy Fetch
log "Checking policy file..."
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" https://mta-sts.$DOMAIN/.well-known/mta-sts.txt)
if [ "$HTTP_CODE" = "200" ]; then
  log "✓ Policy fetch OK"
else
  alert "Policy fetch failed with HTTP $HTTP_CODE"
  exit 1
fi

# Test 3: Policy Content
log "Validating policy content..."
POLICY=$(curl -s https://mta-sts.$DOMAIN/.well-known/mta-sts.txt)
if [[ $POLICY == *"version: STSv1"* ]] && [[ $POLICY == *"mode: enforce"* ]]; then
  log "✓ Policy content valid"
else
  alert "Policy content invalid"
  exit 1
fi

# Test 4: Certificate Expiration
log "Checking certificate expiration..."
CERT_END=$(echo | openssl s_client -connect mta-sts.$DOMAIN:443 -servername mta-sts.$DOMAIN 2>/dev/null | openssl x509 -noout -enddate | cut -d= -f2)
CERT_END_EPOCH=$(date -d "$CERT_END" +%s)
NOW_EPOCH=$(date +%s)
DAYS_UNTIL_EXPIRY=$(( ($CERT_END_EPOCH - $NOW_EPOCH) / 86400 ))

if [ $DAYS_UNTIL_EXPIRY -gt 30 ]; then
  log "✓ Certificate valid for $DAYS_UNTIL_EXPIRY days"
elif [ $DAYS_UNTIL_EXPIRY -gt 7 ]; then
  log "⚠ Certificate expires in $DAYS_UNTIL_EXPIRY days"
else
  alert "Certificate expires in $DAYS_UNTIL_EXPIRY days!"
fi

log "All checks passed"
```

**Setup cron job:**
```bash
# Run every hour
0 * * * * /path/to/mta-sts-monitor.sh
```

### GitHub Actions Workflow

Create `.github/workflows/mta-sts-test.yml`:

```yaml
name: MTA-STS Validation

on:
  push:
    paths:
      - '.well-known/mta-sts.txt'
  schedule:
    - cron: '0 */6 * * *'  # Every 6 hours
  workflow_dispatch:

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Validate Policy Syntax
        run: |
          echo "Checking policy file syntax..."

          # Check required fields
          grep -q "^version: STSv1" .well-known/mta-sts.txt || (echo "Missing version"; exit 1)
          grep -q "^mode: " .well-known/mta-sts.txt || (echo "Missing mode"; exit 1)
          grep -q "^max_age: " .well-known/mta-sts.txt || (echo "Missing max_age"; exit 1)

          # Check mode value
          MODE=$(grep "^mode: " .well-known/mta-sts.txt | cut -d: -f2 | xargs)
          if [[ "$MODE" != "testing" ]] && [[ "$MODE" != "enforce" ]] && [[ "$MODE" != "none" ]]; then
            echo "Invalid mode: $MODE"
            exit 1
          fi

          echo "✓ Policy syntax valid"

      - name: Install mta-sts tool
        run: pip install mta-sts

      - name: Test MTA-STS (after deployment)
        if: github.ref == 'refs/heads/main'
        run: |
          echo "Waiting for GitHub Pages deployment..."
          sleep 60

          echo "Testing MTA-STS..."
          mta-sts-query livalittle.com || echo "Note: May fail if DNS not updated"
```

## Production Monitoring

### Key Metrics to Monitor

1. **Policy Availability**
   - HTTP 200 response rate
   - Response time
   - Certificate validity

2. **DNS Health**
   - TXT record accessibility
   - CNAME resolution
   - Query response time

3. **Email Delivery**
   - Bounce rate
   - Delivery failures
   - TLS connection failures

4. **TLS-RPT Reports**
   - Successful vs failed deliveries
   - Certificate validation errors
   - Policy fetch issues

### Setting Up Uptime Monitoring

**Use services like:**
- UptimeRobot (free tier available)
- Pingdom
- StatusCake
- Datadog

**Monitor URL:**
```
https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

**Check frequency:** Every 5-15 minutes

**Alerts:**
- HTTP status != 200
- Response time > 5 seconds
- Certificate expiring within 14 days

### Log Analysis

**Review GitHub Pages access logs (if available):**
- Policy fetch frequency
- Geographic distribution
- User agents (mail servers)

**Review mail server logs:**
- STARTTLS usage
- TLS connection failures
- Certificate errors

## Testing Checklist

Use this checklist before going live:

### DNS Configuration
- [ ] TXT record `_mta-sts.livalittle.com` exists
- [ ] TXT record format: `v=STSv1; id=20250206`
- [ ] CNAME record `mta-sts.livalittle.com` exists
- [ ] CNAME points to `jdgonzalesdp.github.io.`
- [ ] DNS propagated globally (check 3+ nameservers)
- [ ] No duplicate/conflicting records

### Policy File
- [ ] File at `.well-known/mta-sts.txt`
- [ ] File syntax correct (no extra characters)
- [ ] HTTPS access returns 200
- [ ] Certificate valid (Let's Encrypt)
- [ ] No redirects to different hostname
- [ ] Content-Type: text/plain

### MX Servers
- [ ] MX records match policy pattern
- [ ] All MX servers support STARTTLS
- [ ] All MX servers have valid certificates
- [ ] Certificates not expiring soon (>30 days)
- [ ] TLS version 1.2 or higher

### Validation
- [ ] Tested with aykevl.nl validator
- [ ] Tested with Hardenize
- [ ] Tested with MXToolbox
- [ ] Command-line validation passed
- [ ] End-to-end test script passed

### GitHub Pages
- [ ] GitHub Pages enabled
- [ ] Custom domain configured
- [ ] Enforce HTTPS enabled
- [ ] `.nojekyll` file exists
- [ ] Latest commit deployed

### Monitoring
- [ ] Uptime monitoring configured
- [ ] Alert email configured
- [ ] TLS-RPT configured (optional)
- [ ] Monitoring script set up

### Documentation
- [ ] Policy ID documented
- [ ] Deployment date recorded
- [ ] Rollback procedure accessible
- [ ] Contact information updated

## Post-Deployment Testing

After deployment, continue monitoring for:

**First 24 hours:**
- Check bounce rate every 2 hours
- Monitor for delivery issues
- Review any error reports

**First week:**
- Daily validation with online tools
- Review TLS-RPT reports (if configured)
- Check certificate status
- Monitor email delivery metrics

**Ongoing:**
- Weekly validation checks
- Monthly certificate expiration checks
- Quarterly policy review
- Update policy ID when making changes

## Troubleshooting Test Failures

If tests fail, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for detailed solutions.

## Additional Resources

- [SETUP.md](SETUP.md) - Deployment guide
- [MTA-STS-GUIDE.md](MTA-STS-GUIDE.md) - Protocol details
- [DNS-CONFIGURATION.md](DNS-CONFIGURATION.md) - DNS setup
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Common issues

---

**Last Updated:** 2025-02-06
**Test Coverage:** DNS, HTTPS, Policy, SMTP, End-to-End
**Recommended Testing Frequency:** Daily (first week), Weekly (ongoing)
