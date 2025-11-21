# How It Works

A comprehensive functional explanation of how the livalittle.com MTA-STS email security system operates.

## Table of Contents

1. [The Big Picture](#the-big-picture)
2. [What Happens When Email is Sent](#what-happens-when-email-is-sent)
3. [Step-by-Step Operation](#step-by-step-operation)
4. [Real-World Example](#real-world-example)
5. [What This System Prevents](#what-this-system-prevents)
6. [What This System Does NOT Do](#what-this-system-does-not-do)
7. [Component Interactions](#component-interactions)
8. [Caching and Performance](#caching-and-performance)
9. [Failure Modes](#failure-modes)
10. [Lifecycle of a Policy Update](#lifecycle-of-a-policy-update)

## The Big Picture

### What This System Is

This repository is an **email security enforcement system** that protects incoming email to `@livalittle.com` addresses. It works by:

1. **Publishing** security requirements in a file (`.well-known/mta-sts.txt`)
2. **Serving** that file over HTTPS when requested
3. **Announcing** its existence via DNS
4. **Instructing** sending mail servers to use encrypted connections

### The Three-Second Explanation

**When someone sends email to you@livalittle.com:**
1. Their mail server checks if you support MTA-STS (via DNS)
2. If yes, it downloads your security policy (via HTTPS)
3. It then MUST use encrypted TLS connection to deliver the email
4. If it can't establish a secure connection, the email bounces

**Without this system:** Email could be sent unencrypted or intercepted.

**With this system:** Email MUST be encrypted or it won't be delivered.

## What Happens When Email is Sent

### The Complete Flow (Non-Technical)

```
Alice (Gmail) wants to send email to bob@livalittle.com

┌─────────────────────────────────────────────────────────────┐
│  Step 1: Gmail's server prepares to send the email          │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 2: Gmail asks "Does livalittle.com support MTA-STS?"  │
│          (Checks DNS for _mta-sts.livalittle.com)           │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 3: DNS responds "Yes! Version ID: 20250206"           │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 4: Gmail fetches the security policy from:            │
│          https://mta-sts.livalittle.com/.well-known/        │
│          mta-sts.txt                                         │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 5: GitHub Pages serves the policy file:               │
│                                                              │
│          version: STSv1                                      │
│          mode: enforce                                       │
│          mx: *.livalittle.com                                │
│          max_age: 86400                                      │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 6: Gmail reads the policy:                            │
│          • TLS is REQUIRED (mode: enforce)                   │
│          • Only send to *.livalittle.com mail servers        │
│          • Remember this for 24 hours (max_age: 86400)       │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 7: Gmail looks up where to send the email             │
│          (Checks MX record for livalittle.com)               │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 8: DNS responds "Send to mail.livalittle.com"         │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 9: Gmail verifies mail.livalittle.com matches         │
│          the pattern *.livalittle.com ✓                      │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 10: Gmail connects to mail.livalittle.com             │
│           AND REQUIRES encrypted TLS connection              │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  Step 11A: If TLS works → Email delivered ✓                 │
│  Step 11B: If TLS fails → Email bounces ✗                   │
└─────────────────────────────────────────────────────────────┘
```

## Step-by-Step Operation

### Phase 1: Policy Discovery (DNS)

**What happens:**
Sending mail server queries DNS for MTA-STS support.

**Technical details:**
```bash
# Query sent by mail server
dig _mta-sts.livalittle.com TXT

# Response from DNS
"v=STSv1; id=20250206"
```

**What this means:**
- `v=STSv1` → MTA-STS version 1 is supported
- `id=20250206` → Policy version identifier (helps with caching)

**Outcome:**
- If record exists → Continue to Phase 2
- If record doesn't exist → No MTA-STS, use opportunistic TLS

### Phase 2: Hostname Resolution (DNS)

**What happens:**
Mail server looks up where the policy file is hosted.

**Technical details:**
```bash
# Query sent by mail server
dig mta-sts.livalittle.com CNAME

# Response from DNS
jdgonzalesdp.github.io.

# Further resolution
dig jdgonzalesdp.github.io A

# Final IP addresses (GitHub Pages)
185.199.108.153
185.199.109.153
185.199.110.153
185.199.111.153
```

**What this means:**
- `mta-sts.livalittle.com` → Points to GitHub Pages
- GitHub Pages hosts the policy file

**Outcome:**
Mail server now knows where to fetch the policy.

### Phase 3: Policy Fetch (HTTPS)

**What happens:**
Mail server downloads the policy file via secure HTTPS.

**Technical details:**
```bash
# HTTPS request sent by mail server
GET /.well-known/mta-sts.txt HTTP/1.1
Host: mta-sts.livalittle.com
```

**Security checks performed:**
1. ✓ HTTPS connection required (not HTTP)
2. ✓ Valid certificate (from Let's Encrypt)
3. ✓ Certificate matches hostname (mta-sts.livalittle.com)
4. ✓ No redirects to different hostnames

**Response received:**
```
HTTP/2 200 OK
Content-Type: text/plain; charset=utf-8

version: STSv1
mode: enforce
mx: *.livalittle.com
max_age: 86400
```

**Outcome:**
Mail server has the policy file.

### Phase 4: Policy Parsing

**What happens:**
Mail server parses and validates the policy.

**Validation performed:**
- ✓ `version: STSv1` → Correct version
- ✓ `mode: enforce` → TLS is mandatory
- ✓ `mx: *.livalittle.com` → Mail server pattern
- ✓ `max_age: 86400` → Cache for 24 hours
- ✓ Syntax is correct (no extra characters, proper format)

**What the policy means:**
- **mode: enforce** → TLS connection is REQUIRED, fail if unavailable
- **mx: *.livalittle.com** → Only connect to mail servers under livalittle.com
- **max_age: 86400** → Use this policy for 86,400 seconds (24 hours) before checking again

**Outcome:**
Mail server understands the security requirements.

### Phase 5: Policy Caching

**What happens:**
Mail server stores the policy in memory/cache.

**Cache details:**
- Duration: 86,400 seconds (24 hours)
- Key: `livalittle.com` + policy ID `20250206`
- Storage: Mail server's local cache

**Benefit:**
- Next email to @livalittle.com doesn't need to fetch policy again
- Reduces DNS/HTTPS queries
- Faster email delivery
- Protection even if policy URL is temporarily unavailable

**Outcome:**
Policy cached and ready for enforcement.

### Phase 6: MX Lookup

**What happens:**
Mail server looks up actual mail servers for livalittle.com.

**Technical details:**
```bash
# Query sent by mail server
dig livalittle.com MX

# Response
livalittle.com. 3600 IN MX 10 mail.livalittle.com.
```

**What this means:**
- Priority 10: Preference order (lower = higher priority)
- mail.livalittle.com: The actual mail server to connect to

**Outcome:**
Mail server knows the destination.

### Phase 7: Pattern Matching

**What happens:**
Mail server verifies MX hostname matches the policy pattern.

**Matching logic:**
```
Policy pattern:  *.livalittle.com
MX hostname:     mail.livalittle.com

Check: Does "mail.livalittle.com" match "*.livalittle.com"?
Result: ✓ YES (wildcard * matches "mail")
```

**Examples:**
- `mail.livalittle.com` → ✓ Matches
- `smtp.livalittle.com` → ✓ Matches
- `mx1.livalittle.com` → ✓ Matches
- `livalittle.com` → ✗ Doesn't match (no subdomain)
- `mail.example.com` → ✗ Doesn't match (different domain)

**Outcome:**
- If match → Continue to Phase 8
- If no match → Email fails (in enforce mode)

### Phase 8: TLS Connection

**What happens:**
Mail server establishes encrypted connection to mail.livalittle.com.

**Connection steps:**
```
1. TCP connection to mail.livalittle.com:25
2. SMTP handshake: EHLO sender.com
3. Server advertises: 250-STARTTLS
4. Client initiates: STARTTLS
5. TLS handshake begins
6. Certificate exchange and validation
7. Encrypted channel established
```

**Certificate validation (enforced by MTA-STS):**
- ✓ Certificate is from trusted CA (not self-signed)
- ✓ Certificate is valid (not expired)
- ✓ Certificate matches hostname (mail.livalittle.com)
- ✓ Certificate chain is complete

**In enforce mode:**
- If TLS succeeds → Continue to Phase 9
- If TLS fails → Email bounces immediately

**Outcome:**
Secure connection established.

### Phase 9: Email Delivery

**What happens:**
Email is transmitted over the encrypted connection.

**Transmission:**
```
MAIL FROM:<alice@gmail.com>
RCPT TO:<bob@livalittle.com>
DATA
Subject: Hello
[Email content encrypted in transit]
.
250 OK Message accepted
```

**Security guarantees:**
- ✓ Email content encrypted during transmission
- ✓ Protection against eavesdropping
- ✓ Protection against tampering
- ✓ Certificate validated (prevents MITM)

**Outcome:**
Email successfully delivered securely.

## Real-World Example

### Scenario: Alice sends email to Bob

**Setup:**
- Alice: alice@gmail.com (Gmail user)
- Bob: bob@livalittle.com (Your domain)
- Gmail's mail servers support MTA-STS

**Detailed Flow:**

#### T+0ms: Alice clicks "Send"
- Gmail server: mail.google.com
- Recipient: bob@livalittle.com
- Email queued for delivery

#### T+10ms: DNS Query for MTA-STS
```
Gmail → DNS: "Does livalittle.com support MTA-STS?"
Query: _mta-sts.livalittle.com TXT
```

#### T+30ms: DNS Response
```
DNS → Gmail: "Yes! v=STSv1; id=20250206"
Gmail: "MTA-STS is supported, let's fetch the policy"
```

#### T+40ms: Policy Fetch Initiated
```
Gmail → GitHub Pages: HTTPS GET /.well-known/mta-sts.txt
Host: mta-sts.livalittle.com
```

#### T+50ms: TLS Handshake
```
Gmail ↔ GitHub Pages: TLS handshake
Gmail: Verify GitHub's Let's Encrypt certificate ✓
Gmail: Hostname matches: mta-sts.livalittle.com ✓
```

#### T+70ms: Policy Downloaded
```
GitHub Pages → Gmail:
version: STSv1
mode: enforce
mx: *.livalittle.com
max_age: 86400
```

#### T+75ms: Policy Parsed
```
Gmail: Parsing policy...
Gmail: Mode is "enforce" → TLS is REQUIRED
Gmail: MX pattern: *.livalittle.com
Gmail: Cache for 86400 seconds
Gmail: Policy cached ✓
```

#### T+80ms: MX Lookup
```
Gmail → DNS: "Where should I send email for livalittle.com?"
Query: livalittle.com MX
```

#### T+100ms: MX Response
```
DNS → Gmail: "Send to mail.livalittle.com (priority 10)"
```

#### T+105ms: Pattern Validation
```
Gmail: Checking if "mail.livalittle.com" matches "*.livalittle.com"
Gmail: ✓ Match confirmed
```

#### T+110ms: Connect to Mail Server
```
Gmail → mail.livalittle.com: TCP SYN (port 25)
mail.livalittle.com → Gmail: TCP SYN-ACK
Gmail: TCP connection established
```

#### T+130ms: SMTP Handshake
```
Gmail → mail.livalittle.com: EHLO mail.google.com
mail.livalittle.com → Gmail: 250-Hello, 250-STARTTLS, 250-PIPELINING
```

#### T+140ms: STARTTLS Required
```
Gmail: Policy requires TLS, initiating STARTTLS
Gmail → mail.livalittle.com: STARTTLS
mail.livalittle.com → Gmail: 220 Ready to start TLS
```

#### T+150ms: TLS Handshake
```
Gmail ↔ mail.livalittle.com: TLS handshake
Gmail: Verify mail.livalittle.com certificate ✓
Gmail: Certificate from trusted CA ✓
Gmail: Certificate valid (not expired) ✓
Gmail: Encrypted channel established ✓
```

#### T+200ms: Email Transmission
```
Gmail → mail.livalittle.com: MAIL FROM:<alice@gmail.com>
mail.livalittle.com → Gmail: 250 OK
Gmail → mail.livalittle.com: RCPT TO:<bob@livalittle.com>
mail.livalittle.com → Gmail: 250 OK
Gmail → mail.livalittle.com: DATA
Gmail → mail.livalittle.com: [Encrypted email content]
mail.livalittle.com → Gmail: 250 OK Message accepted for delivery
```

#### T+250ms: Success
```
Gmail: Email delivered successfully ✓
Alice: Sees "Message sent" confirmation
Bob: Will receive email in inbox
```

**Total time:** ~250ms (0.25 seconds)

**What happened differently because of MTA-STS:**
1. Policy was fetched and validated
2. TLS was REQUIRED (not optional)
3. Certificate was validated (mandatory)
4. If TLS failed, email would have bounced instead of being sent insecurely

## What This System Prevents

### Attack 1: STARTTLS Stripping (Downgrade Attack)

**Without MTA-STS:**
```
Gmail → mail.livalittle.com: EHLO gmail.com
Attacker intercepts response, removes STARTTLS capability
Gmail: No STARTTLS available, sending unencrypted
Attacker: Reads email in plaintext ✓ (attacker wins)
```

**With MTA-STS:**
```
Gmail: Policy says TLS is REQUIRED
Gmail → mail.livalittle.com: EHLO gmail.com
Attacker intercepts response, removes STARTTLS capability
Gmail: No STARTTLS, but policy requires it
Gmail: REJECT email delivery (follows policy)
Sender: Receives bounce message
Attacker: Gets nothing ✗ (attacker fails)
```

### Attack 2: DNS Spoofing (Redirect to Malicious Server)

**Without MTA-STS:**
```
Attacker: Poisons DNS, changes MX to attacker.com
Gmail → DNS: "Where to send mail for livalittle.com?"
DNS (poisoned): "Send to attacker-mail.com"
Gmail → attacker-mail.com: Here's the email
Attacker: Receives email ✓ (attacker wins)
```

**With MTA-STS:**
```
Attacker: Poisons DNS, changes MX to attacker.com
Gmail: Already has cached policy: "mx: *.livalittle.com"
Gmail → DNS: "Where to send mail for livalittle.com?"
DNS (poisoned): "Send to attacker-mail.com"
Gmail: Checking if "attacker-mail.com" matches "*.livalittle.com"
Gmail: ✗ No match, REJECT delivery
Sender: Receives bounce message
Attacker: Gets nothing ✗ (attacker fails)
```

### Attack 3: Man-in-the-Middle with Fake Certificate

**Without MTA-STS:**
```
Gmail → mail.livalittle.com
Attacker intercepts connection
Attacker: Presents self-signed certificate
Gmail: Self-signed cert, but no requirement to validate
Gmail: Sends email anyway (opportunistic TLS)
Attacker: Decrypts and reads email ✓ (attacker wins)
```

**With MTA-STS:**
```
Gmail: Policy requires valid certificate
Gmail → mail.livalittle.com
Attacker intercepts connection
Attacker: Presents self-signed certificate
Gmail: Validates certificate
Gmail: ✗ Self-signed, not from trusted CA
Gmail: REJECT delivery (policy enforcement)
Sender: Receives bounce message
Attacker: Gets nothing ✗ (attacker fails)
```

## What This System Does NOT Do

### Limitation 1: No End-to-End Encryption

**What MTA-STS protects:**
```
Alice's Gmail → [Encrypted] → Bob's livalittle.com mailbox
```

**What MTA-STS does NOT protect:**
```
Alice writes email → Stored on Gmail servers (unencrypted to Google)
Bob receives email → Stored on livalittle.com servers (readable by server admin)
Bob reads email → On his device (readable if device compromised)
```

**For end-to-end encryption, use:**
- PGP/GPG
- S/MIME
- ProtonMail or similar

### Limitation 2: Sender Must Support MTA-STS

**Supported senders:**
- Gmail (Google) ✓
- Outlook.com (Microsoft) ✓
- Yahoo ✓
- Major providers ✓

**Unsupported senders:**
- Small ISPs/mail servers that don't implement MTA-STS
- Old/legacy mail systems
- Custom mail servers without MTA-STS support

**Result:** If sender doesn't support MTA-STS, they'll use regular opportunistic TLS (less secure).

### Limitation 3: Policy Updates Take Time

**Scenario:** You update the policy from "testing" to "enforce"

**Timeline:**
```
T+0:      Update policy file, commit, push
T+10min:  GitHub Pages deploys new policy
T+1hr:    DNS record updated (new id)
T+2hr:    Some mail servers notice new DNS id
T+25hr:   All cached policies expire (max_age: 86400s)
T+25hr+:  New policy fully effective
```

**Implication:** Changes don't take effect immediately.

### Limitation 4: No Protection of Outgoing Email

**This system protects:**
- Email TO @livalittle.com ✓

**This system does NOT protect:**
- Email FROM @livalittle.com ✗

**For outgoing protection:**
- Configure MTA-STS on the SENDING side
- That's your outbound mail server's responsibility

## Component Interactions

### The Four Key Components

```
┌──────────────────┐
│  1. DNS Server   │ ← Announces policy existence
└────────┬─────────┘
         │
         ↓ (points to)
┌──────────────────┐
│ 2. GitHub Pages  │ ← Hosts and serves policy file
└────────┬─────────┘
         │
         ↓ (serves)
┌──────────────────┐
│ 3. Policy File   │ ← Defines security requirements
└────────┬─────────┘
         │
         ↓ (enforced by)
┌──────────────────┐
│ 4. Mail Servers  │ ← Consume and enforce policy
└──────────────────┘
```

### Interaction Timeline

```
Deployment Phase:
  Repository (Git) → GitHub Pages → Policy Live

Discovery Phase:
  Mail Server → DNS → "MTA-STS supported"

Fetch Phase:
  Mail Server → GitHub Pages → Policy Downloaded

Enforcement Phase:
  Mail Server → Recipient Mail Server → TLS Required
```

## Caching and Performance

### Three-Level Caching

**Level 1: Mail Server Policy Cache**
- Duration: 86,400 seconds (24 hours)
- Invalidation: Policy ID change in DNS
- Benefit: Fastest delivery, no DNS/HTTPS queries needed

**Level 2: DNS Cache**
- Duration: 3,600 seconds (1 hour) - TTL
- Invalidation: TTL expiration
- Benefit: Fast policy discovery

**Level 3: GitHub CDN Cache**
- Duration: GitHub manages (typically minutes to hours)
- Invalidation: Automatic by GitHub
- Benefit: Fast policy delivery globally

### Performance Impact

**First email to @livalittle.com (cold cache):**
```
DNS TXT query:      ~30ms
DNS CNAME query:    ~30ms
HTTPS GET request:  ~100ms
Total overhead:     ~160ms
```

**Subsequent emails (warm cache):**
```
Cache hit: 0ms additional overhead
```

**Cache hit rate:** Very high (24-hour policy cache)

## Failure Modes

### Failure 1: GitHub Pages Down

**Symptom:** Policy file unreachable

**Impact:**
- **Cached policies:** Still work (up to 24 hours old)
- **New senders:** Cannot fetch policy, fall back to opportunistic TLS
- **Delivery:** Continues but with reduced security for new senders

**Duration:** Temporary, GitHub resolves outages quickly

**Mitigation:** Long max_age provides window of cached protection

### Failure 2: DNS Outage

**Symptom:** Cannot resolve _mta-sts.livalittle.com

**Impact:**
- **Cached policies:** Still work
- **New senders:** Cannot discover MTA-STS, use opportunistic TLS
- **Delivery:** Continues but without MTA-STS protection for new senders

**Duration:** Depends on DNS provider

**Mitigation:** Choose reliable DNS provider, monitor uptime

### Failure 3: Certificate Expired

**Symptom:** GitHub Pages certificate invalid

**Impact:**
- Policy fetch fails (HTTPS required)
- Cached policies still work
- New senders cannot fetch policy

**Duration:** Until renewed

**Mitigation:** GitHub auto-renews Let's Encrypt certificates

### Failure 4: Invalid Policy Syntax

**Symptom:** Malformed policy file

**Impact:**
- Sending servers reject policy
- Fall back to opportunistic TLS or cached old policy
- Email delivery may fail (depends on sender)

**Duration:** Until policy fixed

**Mitigation:**
- Validation before commit
- Git rollback capability
- Testing in "mode: testing" first

## Lifecycle of a Policy Update

### Scenario: Changing mode from "testing" to "enforce"

```
Day 0, 00:00: Current state
  - Policy: mode: testing
  - DNS id: 20250206
  - All mail servers have cached policy

Day 1, 10:00: Developer updates policy
  - Edit .well-known/mta-sts.txt
  - Change: mode: testing → mode: enforce
  - Commit: git commit -m "Switch to enforce mode"
  - Push: git push origin main

Day 1, 10:05: GitHub Pages deploys
  - New policy live at HTTPS endpoint
  - Old policy still in mail server caches

Day 1, 10:10: Update DNS
  - Change: id=20250206 → id=20250207
  - Purpose: Signal policy change to mail servers
  - DNS TTL: 3600 seconds (1 hour)

Day 1, 11:10: DNS fully propagated
  - All DNS servers have new id=20250207
  - Mail servers checking DNS see new ID

Day 1, 11:10+: Mail servers react
  - Gmail: "Policy ID changed (20250206→20250207), fetch new policy"
  - Outlook: "Policy ID changed, fetch new policy"
  - Others: Gradually detect change

Day 1, 11:15: First servers fetch new policy
  - Download: mode: enforce
  - Cache: For next 24 hours
  - Enforce: TLS now required

Day 2, 11:00: Most servers have new policy
  - Old cached policies expired (24hr max_age)
  - Majority now enforce TLS

Day 3, 00:00: Full rollout complete
  - All cached policies expired
  - All mail servers using new enforce mode
  - TLS enforced globally
```

**Total propagation time:** ~36-48 hours
- GitHub Pages: 5-10 minutes
- DNS TTL: 1 hour
- Policy cache: 24 hours
- Safety margin: +12 hours

---

**Document Version:** 1.0
**Last Updated:** 2025-02-06
**Audience:** Technical and non-technical readers
**Related:** See ARCHITECTURE.md for technical architecture details
