# Deployment Flow Documentation

Complete documentation of deployment workflows, processes, and automation for the livalittle.com MTA-STS system.

## Table of Contents

1. [Deployment Overview](#deployment-overview)
2. [Development Workflow](#development-workflow)
3. [Deployment Pipeline](#deployment-pipeline)
4. [Change Management](#change-management)
5. [Release Process](#release-process)
6. [Rollback Procedures](#rollback-procedures)
7. [Automation](#automation)
8. [Monitoring Deployment](#monitoring-deployment)
9. [Deployment Scenarios](#deployment-scenarios)
10. [Best Practices](#best-practices)

## Deployment Overview

### What Gets Deployed

This system deploys **static configuration files** to **GitHub Pages**:

**Primary artifact:**
- `.well-known/mta-sts.txt` - MTA-STS policy file

**Supporting files:**
- `.nojekyll` - GitHub Pages configuration
- Documentation (optional, not served to mail servers)

**Configuration:**
- DNS records (deployed separately to DNS provider)

### Deployment Model

**Type:** Continuous Deployment (CD)
**Trigger:** Git push to main branch
**Platform:** GitHub Pages (serverless)
**Automation:** GitHub Actions (built-in)
**Rollback:** Git revert + re-deploy

### Deployment Frequency

**Typical:** Very infrequent
- Policy changes: Quarterly or as needed
- Documentation updates: As needed
- Emergency fixes: Immediate

**Reason:** MTA-STS policies are stable configuration, not active code.

## Development Workflow

### Local Development

```
┌─────────────────────────────────────────────────────────┐
│  Step 1: Clone Repository                               │
│  $ git clone https://github.com/jdgonzalesdp/livalittle│
│  $ cd livalittle                                        │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Step 2: Create Feature Branch                          │
│  $ git checkout -b update/change-to-enforce             │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Step 3: Make Changes                                   │
│  $ vim .well-known/mta-sts.txt                         │
│  [Edit policy file]                                     │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Step 4: Validate Changes                               │
│  $ cat .well-known/mta-sts.txt                         │
│  $ grep -E "^(version|mode|mx|max_age):" ...           │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Step 5: Commit Changes                                 │
│  $ git add .well-known/mta-sts.txt                     │
│  $ git commit -m "Update mode to enforce"               │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Step 6: Push to GitHub                                 │
│  $ git push origin update/change-to-enforce             │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Step 7: Create Pull Request                            │
│  Open PR on GitHub                                      │
│  Request review (optional)                              │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Step 8: Merge to Main                                  │
│  After approval, merge PR                               │
└─────────────────────────────────────────────────────────┘
```

### Branching Strategy

**Main branch:**
- `main` - Production-ready code
- Always deployable
- Protected (optional)

**Feature branches:**
- `feature/<description>` - New features
- `fix/<description>` - Bug fixes
- `update/<description>` - Policy updates
- `docs/<description>` - Documentation

**Example branches:**
```
feature/add-tls-rpt
fix/policy-syntax-error
update/enforce-mode
docs/improve-setup
```

## Deployment Pipeline

### Automatic Deployment (GitHub Pages)

```
┌──────────────────────────────────────────────────────────────┐
│  Developer                                                   │
│  $ git push origin main                                      │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  GitHub Repository                                           │
│  • Receives push                                             │
│  • Triggers Pages build                                      │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  GitHub Pages Build                                          │
│  1. Checkout main branch                                     │
│  2. Check for .nojekyll (found ✓)                           │
│  3. Skip Jekyll processing                                   │
│  4. Copy files to CDN                                        │
│  Duration: 30 seconds - 10 minutes                           │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  GitHub CDN                                                  │
│  • Files deployed to edge locations                          │
│  • Cache invalidation                                        │
│  • Global distribution                                       │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  Live                                                        │
│  https://mta-sts.livalittle.com/.well-known/mta-sts.txt     │
│  Serving new policy ✓                                       │
└──────────────────────────────────────────────────────────────┘
```

### Deployment Timeline

| Event | Time | Cumulative |
|-------|------|------------|
| Git push | T+0s | 0s |
| GitHub receives | T+1s | 1s |
| Pages build starts | T+5s | 5s |
| Build completes | T+30s | 30s |
| CDN deployment starts | T+35s | 35s |
| First edge location live | T+2min | 2min |
| Most edges live | T+5min | 5min |
| All edges live | T+10min | 10min |

**Average deployment time:** 5-10 minutes

### Manual DNS Deployment

```
┌──────────────────────────────────────────────────────────────┐
│  Step 1: Log in to DNS Provider                              │
│  (Cloudflare, Route 53, etc.)                                │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  Step 2: Update TXT Record                                   │
│  Name: _mta-sts.livalittle.com                               │
│  Old:  "v=STSv1; id=20250206"                                │
│  New:  "v=STSv1; id=20250207"                                │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  Step 3: Save Changes                                        │
│  DNS provider updates records                                │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  Step 4: Wait for Propagation                                │
│  TTL: 3600 seconds (1 hour)                                  │
│  Check: dig _mta-sts.livalittle.com TXT                      │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  Step 5: Verify                                              │
│  $ dig @8.8.8.8 _mta-sts.livalittle.com TXT                  │
│  Should show new id=20250207                                 │
└──────────────────────────────────────────────────────────────┘
```

**DNS propagation time:** 1-4 hours (typically 1 hour)

## Change Management

### Types of Changes

#### Low-Risk Changes
- Documentation updates
- README improvements
- Adding test scripts

**Impact:** None on email security
**Testing:** Review only
**Approval:** Single developer
**Rollback:** Not critical

#### Medium-Risk Changes
- max_age adjustments
- Adding MX patterns (non-breaking)
- mode: enforce → testing (relaxing)

**Impact:** Reduced security temporarily
**Testing:** Validation tools + manual testing
**Approval:** Technical review
**Rollback:** Available, low urgency

#### High-Risk Changes
- mode: testing → enforce (tightening)
- Removing MX patterns
- Major policy restructuring

**Impact:** Potential email delivery failures
**Testing:** Extensive validation + staging period
**Approval:** Multiple reviewers + stakeholder approval
**Rollback:** Critical path documented

### Change Request Process

```
┌──────────────────────────────────────────────────────────────┐
│  1. Proposal                                                 │
│  • Document change rationale                                 │
│  • Assess risk level                                         │
│  • Identify rollback plan                                    │
└────────────────────┬─────────────────────────────────────────┘
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  2. Review                                                   │
│  • Technical review (syntax, RFC compliance)                 │
│  • Security review (impact assessment)                       │
│  • Stakeholder approval (for high-risk)                      │
└────────────────────┬─────────────────────────────────────────┘
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  3. Testing                                                  │
│  • Local validation                                          │
│  • Online validators                                         │
│  • Staging deployment (if available)                         │
└────────────────────┬─────────────────────────────────────────┘
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  4. Scheduling                                               │
│  • Choose deployment window                                  │
│  • Notify stakeholders                                       │
│  • Prepare monitoring                                        │
└────────────────────┬─────────────────────────────────────────┘
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  5. Deployment                                               │
│  • Execute deployment steps                                  │
│  • Update DNS                                                │
│  • Monitor for issues                                        │
└────────────────────┬─────────────────────────────────────────┘
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  6. Verification                                             │
│  • Validate deployment                                       │
│  • Check email delivery                                      │
│  • Review metrics                                            │
└────────────────────┬─────────────────────────────────────────┘
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  7. Documentation                                            │
│  • Update CHANGELOG.md                                       │
│  • Document lessons learned                                  │
│  • Close change request                                      │
└──────────────────────────────────────────────────────────────┘
```

## Release Process

### Standard Release (Policy Update)

**Pre-Release:**

```bash
# 1. Validate current state
curl https://mta-sts.livalittle.com/.well-known/mta-sts.txt
dig _mta-sts.livalittle.com TXT +short

# 2. Create release branch
git checkout -b release/v1.1.0

# 3. Update policy file
vim .well-known/mta-sts.txt

# 4. Update CHANGELOG
vim CHANGELOG.md
# Add release notes

# 5. Validate syntax
cat .well-known/mta-sts.txt | grep -E "^(version|mode|mx|max_age):"

# 6. Commit
git add .well-known/mta-sts.txt CHANGELOG.md
git commit -m "Release v1.1.0: Update to enforce mode"

# 7. Create PR
git push origin release/v1.1.0
# Open PR on GitHub
```

**Release:**

```bash
# 8. Merge to main
# Via GitHub PR merge

# 9. Tag release
git checkout main
git pull
git tag -a v1.1.0 -m "Release v1.1.0: Enforce mode"
git push origin v1.1.0

# 10. Wait for GitHub Pages deployment
sleep 600  # Wait 10 minutes

# 11. Verify deployment
curl https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

**Post-Release:**

```bash
# 12. Update DNS
# Log in to DNS provider
# Change: id=20250206 to id=20250207

# 13. Verify DNS
dig @8.8.8.8 _mta-sts.livalittle.com TXT

# 14. Monitor
# Check email delivery metrics
# Review bounce rates
# Check TLS-RPT reports (if configured)

# 15. Document
# Update release notes
# Notify stakeholders
```

### Emergency Release (Hotfix)

**For critical issues requiring immediate deployment:**

```bash
# 1. Create hotfix branch from main
git checkout main
git pull
git checkout -b hotfix/critical-policy-fix

# 2. Make fix
vim .well-known/mta-sts.txt

# 3. Validate
cat .well-known/mta-sts.txt

# 4. Commit
git add .well-known/mta-sts.txt
git commit -m "Hotfix: Fix policy syntax error"

# 5. Push directly to main (emergency only)
git push origin hotfix/critical-policy-fix

# 6. Create PR and merge immediately
# Or push directly to main if authorized

# 7. Update DNS immediately
# Change policy ID

# 8. Monitor closely
# Watch for issues
```

## Rollback Procedures

### Scenario 1: Policy File Error (Fast Rollback)

**Symptom:** Invalid policy file deployed

**Steps:**

```bash
# 1. Identify bad commit
git log --oneline .well-known/mta-sts.txt

# 2. Revert to last good version
git revert <bad-commit-hash>

# 3. Push revert
git push origin main

# 4. Wait for Pages deployment
# Monitor: https://mta-sts.livalittle.com/.well-known/mta-sts.txt

# 5. Update DNS to previous ID
# Revert DNS TXT record to old ID

# 6. Verify
curl https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

**Time to rollback:** 10-15 minutes

### Scenario 2: Email Delivery Issues (Policy Rollback)

**Symptom:** Emails bouncing after policy change

**Steps:**

```bash
# Option A: Revert policy file (as above)

# Option B: Change mode to testing (faster mitigation)
# 1. Edit policy
vim .well-known/mta-sts.txt
# Change: mode: enforce → mode: testing

# 2. Commit and push
git add .well-known/mta-sts.txt
git commit -m "Emergency: Rollback to testing mode"
git push origin main

# 3. Update DNS immediately
# Change: id=20250207 → id=20250207_rollback

# 4. Monitor
# Check if emails start delivering
```

**Time to mitigation:** 15-30 minutes
**Time to full effect:** DNS TTL + max_age (up to 25 hours)

### Scenario 3: DNS Issues

**Symptom:** DNS records incorrect or inaccessible

**Steps:**

```bash
# 1. Verify DNS records
dig _mta-sts.livalittle.com TXT
dig mta-sts.livalittle.com CNAME

# 2. Log in to DNS provider
# Check record configuration

# 3. Restore correct records
# TXT: "v=STSv1; id=20250206"
# CNAME: jdgonzalesdp.github.io.

# 4. Wait for propagation
# Check multiple nameservers

# 5. Verify
dig @8.8.8.8 _mta-sts.livalittle.com TXT
dig @1.1.1.1 _mta-sts.livalittle.com TXT
```

**Time to rollback:** 1-4 hours (DNS propagation)

## Automation

### GitHub Actions Workflow (Optional)

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy MTA-STS Policy

on:
  push:
    branches:
      - main
    paths:
      - '.well-known/mta-sts.txt'
  pull_request:
    paths:
      - '.well-known/mta-sts.txt'

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Validate Policy Syntax
        run: |
          # Check required fields
          grep -q "^version: STSv1$" .well-known/mta-sts.txt || exit 1
          grep -q "^mode: " .well-known/mta-sts.txt || exit 1
          grep -q "^max_age: " .well-known/mta-sts.txt || exit 1

          # Check mode value
          MODE=$(grep "^mode: " .well-known/mta-sts.txt | cut -d: -f2 | xargs)
          if [[ "$MODE" != "testing" ]] && [[ "$MODE" != "enforce" ]] && [[ "$MODE" != "none" ]]; then
            echo "Invalid mode: $MODE"
            exit 1
          fi

          echo "✓ Policy syntax valid"

      - name: Check for Common Errors
        run: |
          # Check for backslashes
          if grep -q '\\' .well-known/mta-sts.txt; then
            echo "Error: Policy contains backslashes"
            exit 1
          fi

          # Check for extra whitespace
          if grep -q $'\t' .well-known/mta-sts.txt; then
            echo "Error: Policy contains tabs"
            exit 1
          fi

          echo "✓ No common errors found"

  deploy:
    needs: validate
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Wait for Pages Deployment
        run: |
          echo "GitHub Pages will deploy automatically"
          echo "Check status at: https://github.com/${{ github.repository }}/deployments"

      - name: Notify Deployment
        run: |
          echo "::notice::Policy deployed to GitHub Pages"
          echo "::notice::Update DNS TXT record if policy changed"
```

### Pre-Commit Hook (Local Validation)

Create `.git/hooks/pre-commit`:

```bash
#!/bin/bash

# Pre-commit hook for MTA-STS policy validation

POLICY_FILE=".well-known/mta-sts.txt"

if git diff --cached --name-only | grep -q "$POLICY_FILE"; then
  echo "Validating MTA-STS policy..."

  # Check required fields
  if ! grep -q "^version: STSv1$" "$POLICY_FILE"; then
    echo "Error: Missing or invalid version"
    exit 1
  fi

  if ! grep -q "^mode: " "$POLICY_FILE"; then
    echo "Error: Missing mode field"
    exit 1
  fi

  if ! grep -q "^max_age: " "$POLICY_FILE"; then
    echo "Error: Missing max_age field"
    exit 1
  fi

  # Check for backslashes
  if grep -q '\\' "$POLICY_FILE"; then
    echo "Error: Policy contains backslashes"
    exit 1
  fi

  echo "✓ Policy validation passed"
fi

exit 0
```

Make it executable:
```bash
chmod +x .git/hooks/pre-commit
```

## Monitoring Deployment

### Post-Deployment Checks

**Immediate (T+10 minutes):**

```bash
# 1. HTTPS Access
curl https://mta-sts.livalittle.com/.well-known/mta-sts.txt

# 2. HTTP Status
curl -I https://mta-sts.livalittle.com/.well-known/mta-sts.txt | grep "HTTP"

# 3. Content Validation
curl -s https://mta-sts.livalittle.com/.well-known/mta-sts.txt | grep "mode:"
```

**Short-term (T+1 hour):**

```bash
# 1. DNS Propagation
dig @8.8.8.8 _mta-sts.livalittle.com TXT
dig @1.1.1.1 _mta-sts.livalittle.com TXT

# 2. Online Validation
# Visit: https://aykevl.nl/apps/mta-sts/?domain=livalittle.com

# 3. Certificate Check
openssl s_client -connect mta-sts.livalittle.com:443 -servername mta-sts.livalittle.com </dev/null
```

**Medium-term (T+24 hours):**

```bash
# 1. Email Delivery Metrics
# Check bounce rate
# Review delivery logs

# 2. TLS-RPT Reports (if configured)
# Check for failures
# Review certificate errors

# 3. Usage Statistics
# GitHub Pages analytics
# DNS query counts (if available)
```

## Deployment Scenarios

### Scenario A: First-Time Deployment

See [SETUP.md](SETUP.md) for complete initial deployment guide.

### Scenario B: Policy Update (Testing → Enforce)

```
1. Update policy file locally
2. Validate syntax
3. Commit and push to main
4. Wait for GitHub Pages (10 min)
5. Verify HTTPS endpoint
6. Update DNS TXT id
7. Wait for DNS propagation (1 hour)
8. Monitor email delivery (24-48 hours)
9. Verify with online tools
10. Update documentation
```

### Scenario C: Emergency Disable

```
1. Update mode to "none"
2. Set max_age to 300 (5 minutes)
3. Commit and push
4. Update DNS id immediately
5. Wait for Pages deployment
6. Verify policy served
7. Monitor for improvement
8. Document incident
```

### Scenario D: MX Pattern Change

```
1. Verify new MX records exist
2. Update policy mx: pattern
3. Test pattern matching locally
4. Commit and push
5. Update DNS id
6. Monitor for delivery issues
7. Verify pattern matches all MX servers
8. Document change
```

## Best Practices

### Pre-Deployment

- [ ] Validate policy syntax locally
- [ ] Test with online validators
- [ ] Review change impact
- [ ] Document rollback plan
- [ ] Schedule deployment window
- [ ] Notify stakeholders

### During Deployment

- [ ] Monitor deployment progress
- [ ] Verify HTTPS endpoint
- [ ] Update DNS within deployment window
- [ ] Check validation tools
- [ ] Document deployment time

### Post-Deployment

- [ ] Verify policy accessible
- [ ] Check DNS propagation
- [ ] Monitor email delivery
- [ ] Review metrics/logs
- [ ] Update CHANGELOG.md
- [ ] Close deployment ticket

### General Guidelines

1. **Never skip validation** - Always validate policy syntax
2. **Update DNS ID** - Change ID when policy changes
3. **Monitor delivery** - Watch email metrics after changes
4. **Document everything** - Update CHANGELOG and docs
5. **Test in testing mode first** - Before switching to enforce
6. **Have rollback ready** - Know how to revert quickly
7. **Communicate changes** - Notify team/stakeholders
8. **Schedule carefully** - Avoid high-traffic periods

---

**Document Version:** 1.0
**Last Updated:** 2025-02-06
**Related:** See SETUP.md for initial deployment, ARCHITECTURE.md for system design
