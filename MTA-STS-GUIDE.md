# MTA-STS Protocol Guide

Comprehensive guide to understanding SMTP MTA Strict Transport Security (MTA-STS) and how it protects email security for livalittle.com.

## Table of Contents

1. [What is MTA-STS?](#what-is-mta-sts)
2. [The Email Security Problem](#the-email-security-problem)
3. [How MTA-STS Works](#how-mta-sts-works)
4. [Protocol Specification](#protocol-specification)
5. [Security Benefits](#security-benefits)
6. [Limitations](#limitations)
7. [MTA-STS vs Other Standards](#mta-sts-vs-other-standards)
8. [Real-World Attack Scenarios](#real-world-attack-scenarios)
9. [Implementation Details](#implementation-details)
10. [Best Practices](#best-practices)

## What is MTA-STS?

**MTA-STS (Mail Transfer Agent Strict Transport Security)** is an email security standard defined in [RFC 8461](https://tools.ietf.org/html/rfc8461) that enables mail service providers to declare their ability to receive TLS-secured connections and to specify whether sending mail servers should refuse to deliver to MX hosts that do not offer TLS with a trusted certificate.

### Key Characteristics

- **Published:** September 2018
- **RFC Number:** 8461
- **Purpose:** Prevent SMTP downgrade attacks
- **Mechanism:** Policy file + DNS TXT record
- **Delivery:** HTTPS (requires valid certificate)
- **Scope:** Server-to-server (SMTP) email delivery

### In Simple Terms

MTA-STS is like HTTPS for email servers. Just as HTTPS ensures your web browser communicates securely with websites, MTA-STS ensures mail servers communicate securely when delivering email to your domain.

## The Email Security Problem

### SMTP's Security Gap

The Simple Mail Transfer Protocol (SMTP) was designed in 1982, long before security was a primary concern. Traditional SMTP has several vulnerabilities:

#### 1. Optional Encryption

SMTP supports encryption via **STARTTLS**, but it's optional:

```
Client: EHLO sender.com
Server: 250-receiver.com Hello
Server: 250-STARTTLS
Client: STARTTLS
[Encryption begins]
```

**Problem:** If a server advertises STARTTLS but an attacker removes it from the response, communication falls back to unencrypted plaintext.

#### 2. No Certificate Validation

Even when STARTTLS is used, traditional SMTP doesn't require certificate validation:

- Self-signed certificates are accepted
- Expired certificates are accepted
- Certificates for wrong domains are accepted

**Problem:** An attacker can intercept traffic with a fake certificate.

#### 3. MITM Downgrade Attacks

An attacker positioned between two mail servers can:

1. Strip STARTTLS capability from server responses
2. Force plaintext communication
3. Read or modify email in transit

```
Sender → [Attacker removes STARTTLS] → Receiver
         └─ Email sent in plaintext ─→
```

### Historical Context

Before MTA-STS, email encryption was:
- **Opportunistic:** Encryption attempted but not required
- **Unverified:** No certificate validation
- **Vulnerable:** Subject to active attacks

## How MTA-STS Works

MTA-STS provides a way for domains to declare that they support TLS and that sending servers should refuse to deliver mail without it.

### The MTA-STS Flow

```
┌──────────────┐                                    ┌──────────────┐
│ Sending Mail │                                    │  Receiving   │
│   Server     │                                    │    Domain    │
│              │                                    │              │
└───────┬──────┘                                    └──────┬───────┘
        │                                                  │
        │ 1. DNS Query: _mta-sts.livalittle.com TXT       │
        │ ─────────────────────────────────────────────>  │
        │                                                  │
        │ 2. Response: "v=STSv1; id=20250206"             │
        │ <─────────────────────────────────────────────  │
        │                                                  │
        │ 3. HTTPS GET:                                    │
        │    https://mta-sts.livalittle.com/.well-known/  │
        │            mta-sts.txt                           │
        │ ─────────────────────────────────────────────>  │
        │                                                  │
        │ 4. Policy File Delivered                         │
        │ <─────────────────────────────────────────────  │
        │                                                  │
        │ 5. Cache Policy (max_age = 24 hours)            │
        │                                                  │
        │ 6. Lookup MX Records: livalittle.com            │
        │ ─────────────────────────────────────────────>  │
        │                                                  │
        │ 7. MX: mail.livalittle.com                      │
        │ <─────────────────────────────────────────────  │
        │                                                  │
        │ 8. Verify: mail.livalittle.com matches          │
        │            policy pattern (*.livalittle.com)     │
        │                                                  │
        │ 9. Connect with REQUIRED TLS                     │
        │ ─────────────────────────────────────────────>  │
        │                                                  │
        │ 10. STARTTLS + Certificate Validation            │
        │ <──────────────────────────────────────────────>│
        │                                                  │
        │ 11. Deliver Email (or FAIL if TLS unavailable)  │
        │ ─────────────────────────────────────────────>  │
        │                                                  │
```

### Step-by-Step Breakdown

#### Step 1: DNS Discovery

Sending server queries for MTA-STS support:

```bash
dig _mta-sts.livalittle.com TXT
```

Response:
```
_mta-sts.livalittle.com. 3600 IN TXT "v=STSv1; id=20250206"
```

**If no TXT record exists:** MTA-STS is not supported; server proceeds with opportunistic TLS.

#### Step 2: Policy Fetch

Server fetches policy via HTTPS (required):

```bash
curl https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

Response:
```
version: STSv1
mode: enforce
mx: *.livalittle.com
max_age: 86400
```

**Certificate validation is required:** Policy must be served over HTTPS with a valid, trusted certificate.

#### Step 3: Policy Interpretation

Server parses the policy:

- **mode: enforce** → TLS is mandatory
- **mx: *.livalittle.com** → Only connect to MX servers matching this pattern
- **max_age: 86400** → Cache this policy for 24 hours

#### Step 4: MX Lookup

Server performs standard MX lookup:

```bash
dig livalittle.com MX
```

Example response:
```
livalittle.com. 3600 IN MX 10 mail.livalittle.com.
```

#### Step 5: Pattern Matching

Server verifies MX hostname matches policy pattern:

```
MX server:      mail.livalittle.com
Policy pattern: *.livalittle.com
Result:         ✓ Match
```

**If no match:** Email delivery fails (in enforce mode) or proceeds without TLS requirements (in testing mode).

#### Step 6: TLS Connection

Server connects to MX server and:

1. Requires STARTTLS support
2. Validates SSL/TLS certificate
3. Ensures certificate is trusted and valid for the MX hostname
4. Refuses connection if requirements not met

#### Step 7: Delivery or Failure

**In enforce mode:**
- ✓ TLS successful → Email delivered
- ✗ TLS fails → Email delivery fails, sender receives bounce

**In testing mode:**
- TLS attempted but not required
- Failures reported via TLS-RPT but email still delivered

## Protocol Specification

### Policy File Format

The MTA-STS policy file follows a strict format:

```
version: STSv1
mode: <mode-value>
mx: <mx-pattern>
max_age: <max-age-value>
```

#### Required Fields

| Field | Required | Format | Description |
|-------|----------|--------|-------------|
| `version` | Yes | `STSv1` | Protocol version (always STSv1) |
| `mode` | Yes | `testing`, `enforce`, or `none` | Policy enforcement level |
| `max_age` | Yes | Integer (seconds) | Cache duration |
| `mx` | Conditional | Hostname pattern | Required if mode ≠ none |

#### Mode Values

**enforce (Maximum Security)**
```
mode: enforce
```
- TLS is strictly required
- Delivery fails if secure connection cannot be established
- Recommended for production once tested

**testing (Gradual Rollout)**
```
mode: testing
```
- TLS is attempted but not required
- Failures are reported (via TLS-RPT) but don't block delivery
- Recommended for initial deployment

**none (Explicit Disable)**
```
mode: none
```
- MTA-STS is explicitly disabled
- No `mx` field required
- Used for planned deactivation

#### MX Patterns

**Wildcard (current configuration):**
```
mx: *.livalittle.com
```
- Matches any subdomain of livalittle.com
- Flexible for infrastructure changes

**Multiple specific servers:**
```
mx: mail1.livalittle.com
mx: mail2.livalittle.com
mx: mail3.livalittle.com
```
- Each server listed explicitly
- More restrictive

**Root domain:**
```
mx: livalittle.com
```
- Only matches exact domain (no subdomains)

**Wildcard limitations:**
```
mx: *.livalittle.com
```
- Does NOT match `livalittle.com` (root)
- Does NOT match `*.mail.livalittle.com` (nested subdomains)

#### Max Age

**Format:** Integer representing seconds

**Common values:**
| Duration | Seconds | Use Case |
|----------|---------|----------|
| 5 minutes | 300 | Testing, development |
| 1 hour | 3600 | Active updates |
| 1 day | 86400 | Standard (current) |
| 1 week | 604800 | Stable production |
| 4 weeks | 2419200 | Very stable |

**Recommendations:**
- Start with short duration (3600) during testing
- Increase to 86400 (1 day) for normal operation
- Use longer durations (604800+) only for very stable infrastructure

**Trade-offs:**
- **Short:** Faster updates, more policy fetches, higher server load
- **Long:** Fewer fetches, slower updates, policy changes take longer to propagate

### DNS TXT Record Format

```
_mta-sts.livalittle.com. IN TXT "v=STSv1; id=20250206"
```

#### Components

**Subdomain:** `_mta-sts.<domain>`
- Prefix with underscore
- Exact name required by RFC 8461

**Version:** `v=STSv1`
- Always `STSv1`
- Case-sensitive

**Policy ID:** `id=<identifier>`
- Unique identifier for policy version
- Can be any string, but date format (YYYYMMDD) recommended
- Change this value whenever policy file is updated
- Used for cache invalidation

**Example evolution:**
```
id=20250206  → Initial deployment
id=20250213  → Changed mode from testing to enforce
id=20250220  → Updated max_age
id=20250227  → Added new MX server
```

### HTTPS Requirements

The policy file MUST be served via HTTPS with:

1. **Valid certificate** from a trusted CA
2. **Correct hostname** (mta-sts.livalittle.com)
3. **No redirects** to different hostnames
4. **HTTP status 200** (not 301, 302, etc. to different hosts)

**Why HTTPS is critical:**
- Prevents attackers from serving fake policies
- Ensures policy integrity
- Leverages existing certificate infrastructure (GitHub Pages uses Let's Encrypt)

## Security Benefits

### 1. Prevents SMTP Downgrade Attacks

**Without MTA-STS:**
```
Sender → EHLO
Attacker → Strips STARTTLS from response
Sender → Sends email in plaintext
Attacker → Reads/modifies email
```

**With MTA-STS (enforce mode):**
```
Sender → EHLO
Attacker → Strips STARTTLS from response
Sender → Checks MTA-STS policy: "TLS required"
Sender → Refuses to deliver email
         └─ Email bounces, attacker gains nothing
```

### 2. Enforces Certificate Validation

**Without MTA-STS:**
- Self-signed certificates accepted
- Expired certificates accepted
- Wrong hostname certificates accepted

**With MTA-STS:**
- Certificate must be from trusted CA
- Certificate must be valid and current
- Certificate must match MX hostname
- Connection refused if validation fails

### 3. Protects Against DNS Spoofing

**Scenario:** Attacker spoofs MX records to point to malicious server

**Without MTA-STS:**
```
Attacker → Spoofs DNS: MX = attacker-mail.com
Sender → Connects to attacker-mail.com
Sender → Accepts any certificate
Attacker → Receives email
```

**With MTA-STS:**
```
Attacker → Spoofs DNS: MX = attacker-mail.com
Sender → Checks MTA-STS: Only *.livalittle.com allowed
Sender → attacker-mail.com doesn't match pattern
Sender → Refuses delivery, email bounces
```

### 4. Provides Long-Term Protection

**Policy caching (max_age = 86400):**
- Sender caches policy for 24 hours
- Even if attacker compromises DNS temporarily
- Sender still enforces cached policy requirements
- Limits exposure window to policy updates

## Limitations

### 1. No End-to-End Encryption

MTA-STS protects **in-transit** between mail servers, but not:

- Email at rest on servers
- Email on recipient's device
- Content accessible to service providers

**For end-to-end encryption, use:**
- PGP/GPG
- S/MIME
- End-to-end encrypted messaging apps

### 2. Sender Dependency

MTA-STS requires the **sending** server to:
- Support MTA-STS protocol
- Properly implement policy checking
- Honor enforce mode directives

**Not all senders support MTA-STS yet.**

### 3. Policy Propagation Delay

Due to caching (max_age):
- Policy changes take up to 24 hours (current setting) to fully propagate
- Emergency changes (like rollback) are delayed
- Short max_age reduces delay but increases server load

### 4. DNS Cache Timing

DNS TXT record changes also subject to TTL:
- Current TTL: 3600 seconds (1 hour)
- Total propagation: DNS TTL + max_age
- Can be up to 25 hours for full policy update

### 5. HTTPS Dependency

Policy delivery requires:
- Valid HTTPS certificate
- Functioning web server (GitHub Pages)
- No HTTPS = MTA-STS fails

**Single point of failure:** If GitHub Pages is down, new senders cannot fetch policy (but existing cache still works).

### 6. No Protection Against Sender Compromise

If attacker compromises the sending server:
- MTA-STS provides no protection
- Email sent from legitimate server with legitimate credentials
- MTA-STS only protects server-to-server transport

## MTA-STS vs Other Standards

### DANE (DNS-Based Authentication of Named Entities)

**DANE (RFC 7672)** is an alternative to MTA-STS.

| Feature | MTA-STS | DANE |
|---------|---------|------|
| **Mechanism** | HTTPS policy file | DNS TLSA records |
| **Certificate** | Standard CA trust | Can use any cert + DNSSEC |
| **Dependency** | HTTPS hosting | DNSSEC |
| **Adoption** | Growing | Limited |
| **Complexity** | Moderate | High |
| **Rollback Speed** | Slow (cache) | Moderate (DNS TTL) |

**Key Difference:**
- MTA-STS leverages existing CA infrastructure
- DANE requires DNSSEC deployment (less common)

**Recommendation:** Use both if possible (defense in depth).

### STARTTLS Everywhere

**STARTTLS Everywhere** (EFF initiative) is a predecessor to MTA-STS.

| Feature | MTA-STS | STARTTLS Everywhere |
|---------|---------|---------------------|
| **Standard** | RFC 8461 (official) | Community project |
| **Discovery** | DNS TXT + HTTPS | Centralized list |
| **Updates** | Self-managed | Submit to EFF |
| **Enforcement** | Policy-based | Hardcoded list |

**MTA-STS supersedes STARTTLS Everywhere.**

### TLS-RPT (TLS Reporting)

**TLS-RPT (RFC 8460)** complements MTA-STS.

```
_smtp._tls.livalittle.com. IN TXT "v=TLSRPTv1; rua=mailto:tls-reports@livalittle.com"
```

**Purpose:**
- Senders report TLS connection results
- Daily aggregate reports
- Includes failures, certificate issues, policy problems

**Relationship:**
- MTA-STS = "What to do" (policy)
- TLS-RPT = "What happened" (reporting)

**Highly recommended to deploy together.**

### DMARC

**DMARC** addresses different concerns:

| Feature | MTA-STS | DMARC |
|---------|---------|-------|
| **Focus** | Transport encryption | Email authentication |
| **Protects Against** | MITM attacks | Spoofing/phishing |
| **Mechanism** | TLS enforcement | SPF + DKIM alignment |

**Use both:** They address different attack vectors.

## Real-World Attack Scenarios

### Scenario 1: Coffee Shop MITM

**Attacker:** Controls WiFi router at coffee shop

**Target:** User sending email via laptop

**Without MTA-STS:**
1. User's mail client connects to SMTP server
2. Attacker intercepts connection
3. Attacker strips STARTTLS from server response
4. User's client sends email in plaintext
5. Attacker reads email content

**With MTA-STS:**
1. Recipient domain has MTA-STS policy (enforce mode)
2. Sender's server checks policy before delivery
3. Policy requires TLS
4. Even if attacker strips STARTTLS
5. Sender refuses to deliver → Email bounces
6. User receives bounce notification and knows something is wrong

### Scenario 2: Nation-State DNS Poisoning

**Attacker:** Controls DNS infrastructure (ISP or nation-state)

**Target:** Email to livalittle.com

**Without MTA-STS:**
1. Attacker poisons DNS, changes MX to attacker-controlled server
2. Sender performs MX lookup, gets attacker's server
3. Sender connects to attacker's server
4. Sender accepts attacker's certificate (no validation)
5. Attacker receives email

**With MTA-STS:**
1. Attacker poisons DNS
2. Sender checks MTA-STS policy: "mx: *.livalittle.com"
3. Attacker's server doesn't match pattern
4. Sender refuses delivery
5. Email bounces, attacker fails

**Additionally:** If attacker tries to change policy:
- Policy served via HTTPS (GitHub Pages)
- Attacker cannot get valid certificate for mta-sts.livalittle.com
- Policy fetch fails, sender uses cached policy

### Scenario 3: Compromised Certificate Authority

**Attacker:** Obtains fraudulent certificate from compromised/rogue CA

**Target:** Intercept email to livalittle.com

**Without MTA-STS:**
- Less relevant (opportunistic TLS doesn't validate certs)

**With MTA-STS:**
1. Attacker gets certificate for attacker-mail.com
2. Attacker spoofs DNS to point to their server
3. Sender checks MTA-STS policy: "mx: *.livalittle.com"
4. attacker-mail.com doesn't match
5. Delivery refused

**Even with fraudulent cert for mail.livalittle.com:**
- Attacker still needs to compromise DNS or network routing
- MTA-STS policy caching provides temporary protection
- Certificate Transparency logs may detect fraudulent cert

## Implementation Details

### For Domain Owners (livalittle.com)

**Requirements:**
1. GitHub Pages (or any HTTPS hosting)
2. DNS access
3. MX servers with TLS support

**Ongoing Maintenance:**
- Monitor email delivery
- Review TLS-RPT reports (if configured)
- Update policy ID when making changes
- Maintain certificate validity on MX servers

### For Sending Mail Servers

To support sending to MTA-STS-enabled domains:

**Implementation checklist:**
1. DNS TXT query capability
2. HTTPS client with certificate validation
3. Policy file parser
4. Policy caching (respect max_age)
5. Pattern matching for MX hostnames
6. TLS connection with certificate validation
7. Failure handling (bounce in enforce mode)

**Major providers with MTA-STS support:**
- Google (Gmail, Workspace)
- Microsoft (Outlook.com, Microsoft 365)
- Apple (iCloud)
- Yahoo
- Postfix 3.4+ (with python-postfix-mta-sts-resolver)
- Exim 4.94+ (with mta-sts support)

### For Security Researchers

**Testing MTA-STS implementations:**

```bash
# Install testing tools
pip install mta-sts

# Query policy
mta-sts-query livalittle.com

# Manual testing
curl https://mta-sts.livalittle.com/.well-known/mta-sts.txt
dig _mta-sts.livalittle.com TXT

# Validate with online tools
# https://aykevl.nl/apps/mta-sts/
```

## Best Practices

### 1. Start with Testing Mode

```
mode: testing
max_age: 3600
```

Deploy in testing mode for 1-2 weeks:
- Monitor delivery with TLS-RPT
- Identify any TLS capability issues
- Fix problems before enforcing

### 2. Use Reasonable Max Age

**Testing phase:** 3600 (1 hour)
- Allows quick policy changes
- Easier to fix issues

**Production:** 86400 (1 day)
- Balance between fetch load and update speed
- Standard recommendation

**Mature deployment:** 604800 (1 week)
- Only for very stable infrastructure
- Reduces load significantly

### 3. Deploy TLS-RPT Alongside MTA-STS

```
_smtp._tls.livalittle.com. IN TXT "v=TLSRPTv1; rua=mailto:tls-reports@livalittle.com"
```

Benefits:
- Visibility into delivery attempts
- Detect TLS failures
- Monitor policy effectiveness
- Identify configuration issues

### 4. Use Date-Based Policy IDs

```
id=20250206  # February 6, 2025
id=20250213  # February 13, 2025
```

Benefits:
- Clear versioning
- Easy to track in logs
- Meaningful at a glance

### 5. Monitor Mail Server Certificates

- Automate certificate renewal (Let's Encrypt)
- Monitor expiration dates
- Test certificate validity regularly
- Use Certificate Transparency monitoring

### 6. Document Changes

Keep changelog of:
- Policy modifications
- Policy ID updates
- DNS changes
- Observed effects on delivery

### 7. Maintain Rollback Plan

Always have documented procedure to:
- Switch to testing mode
- Reduce max_age quickly
- Disable MTA-STS if needed

### 8. Coordinate with Email Provider

If using third-party email (Google, Microsoft):
- Verify they support TLS
- Confirm certificate management
- Check their MTA-STS status
- Align policy with their infrastructure

## Conclusion

MTA-STS is a powerful tool for protecting email in transit:

**Strengths:**
- Prevents SMTP downgrade attacks
- Enforces certificate validation
- Protects against DNS-based MITM
- Leverages existing HTTPS infrastructure

**Limitations:**
- No end-to-end encryption
- Requires sender support
- Policy propagation delays
- HTTPS dependency

**Recommendation:** Deploy MTA-STS as part of comprehensive email security strategy, alongside DMARC, SPF, DKIM, and TLS-RPT.

## Additional Resources

### Official Specifications
- [RFC 8461 - MTA-STS](https://tools.ietf.org/html/rfc8461)
- [RFC 8460 - TLS-RPT](https://tools.ietf.org/html/rfc8460)
- [RFC 7672 - DANE](https://tools.ietf.org/html/rfc7672)
- [RFC 3207 - STARTTLS](https://tools.ietf.org/html/rfc3207)

### Tools & Validators
- [MTA-STS Validator](https://aykevl.nl/apps/mta-sts/)
- [Hardenize Email Security Report](https://www.hardenize.com/)
- [MXToolbox MTA-STS Check](https://mxtoolbox.com/mta-sts.aspx)

### Implementation Guides
- [Google Admin - MTA-STS](https://support.google.com/a/answer/9261504)
- [Microsoft - MTA-STS Support](https://docs.microsoft.com/en-us/microsoft-365/security/office-365-security/mta-sts)

### Related Documentation
- [SETUP.md](SETUP.md) - Deployment instructions
- [DNS-CONFIGURATION.md](DNS-CONFIGURATION.md) - DNS setup details
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Common issues
- [TESTING.md](TESTING.md) - Validation procedures

---

**Last Updated:** 2025-02-06
**RFC Version:** 8461
**Policy Status:** Active (Enforce Mode)
