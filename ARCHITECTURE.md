# System Architecture

Technical architecture documentation for the livalittle.com MTA-STS email security system.

## Table of Contents

1. [System Overview](#system-overview)
2. [Architecture Diagram](#architecture-diagram)
3. [Components](#components)
4. [Data Flow](#data-flow)
5. [Infrastructure](#infrastructure)
6. [Network Architecture](#network-architecture)
7. [Security Architecture](#security-architecture)
8. [Deployment Architecture](#deployment-architecture)
9. [Scalability & Performance](#scalability--performance)
10. [Dependencies](#dependencies)

## System Overview

### What This System Does

This repository implements an **MTA-STS policy server** for the domain `livalittle.com`. It functions as a distributed email security enforcement system that:

1. **Publishes email security requirements** via a static policy file
2. **Serves the policy over HTTPS** to requesting mail servers
3. **Enforces TLS encryption** for all incoming email to @livalittle.com
4. **Prevents downgrade attacks** on email security

### System Type

**Classification:** Static Configuration Infrastructure
**Deployment Model:** Serverless (GitHub Pages)
**Architecture Pattern:** Policy-as-Code
**Protocol:** RFC 8461 (MTA-STS)

### Key Characteristics

- **Stateless:** No database, no session management
- **Read-Only:** Policy is consumed, not modified by clients
- **Highly Available:** Leverages GitHub's CDN infrastructure
- **Self-Contained:** All configuration in repository
- **Version Controlled:** All changes tracked in Git

## Architecture Diagram

### High-Level System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         INTERNET                                    │
│                                                                     │
│  ┌──────────────┐         ┌──────────────┐      ┌──────────────┐  │
│  │   Sending    │         │  DNS Servers │      │  Receiving   │  │
│  │ Mail Server  │         │   (Public)   │      │ Mail Server  │  │
│  │              │         │              │      │(livalittle)  │  │
│  └──────┬───────┘         └───────┬──────┘      └──────────────┘  │
│         │                         │                                │
└─────────┼─────────────────────────┼────────────────────────────────┘
          │                         │
          │ ❶ DNS Query             │ ❷ DNS Response
          │   _mta-sts.livalittle   │   "v=STSv1; id=20250206"
          │                         │
          ▼                         ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      DNS LAYER                                      │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │  DNS Provider (Cloudflare / Route53 / Other)              │    │
│  │                                                            │    │
│  │  Records:                                                  │    │
│  │  • _mta-sts.livalittle.com TXT "v=STSv1; id=20250206"    │    │
│  │  • mta-sts.livalittle.com CNAME jdgonzalesdp.github.io.  │    │
│  └────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
          │
          │ ❸ HTTPS GET Request
          │   https://mta-sts.livalittle.com/.well-known/mta-sts.txt
          │
          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   CONTENT DELIVERY LAYER                            │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │              GitHub Pages CDN (Global)                     │    │
│  │                                                            │    │
│  │  • TLS Termination (Let's Encrypt Certificate)           │    │
│  │  • Content Caching                                        │    │
│  │  • DDoS Protection                                        │    │
│  │  • Geographic Distribution                                │    │
│  └────────────┬───────────────────────────────────────────────┘    │
└───────────────┼─────────────────────────────────────────────────────┘
                │
                │ ❹ File Request
                │
                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                   APPLICATION LAYER                                 │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │           GitHub Pages Static Server                       │    │
│  │                                                            │    │
│  │  Serves: /.well-known/mta-sts.txt                         │    │
│  │  From: main branch, root directory                        │    │
│  └────────────┬───────────────────────────────────────────────┘    │
└───────────────┼─────────────────────────────────────────────────────┘
                │
                │ ❺ Reads from Repository
                │
                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     STORAGE LAYER                                   │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │          Git Repository (jdgonzalesdp/livalittle)         │    │
│  │                                                            │    │
│  │  File: .well-known/mta-sts.txt                           │    │
│  │  ┌──────────────────────────────────────────────┐        │    │
│  │  │ version: STSv1                               │        │    │
│  │  │ mode: enforce                                │        │    │
│  │  │ mx: *.livalittle.com                         │        │    │
│  │  │ max_age: 86400                               │        │    │
│  │  └──────────────────────────────────────────────┘        │    │
│  │                                                            │    │
│  │  Control: .nojekyll (disables Jekyll processing)         │    │
│  └────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Interaction Flow

```
Mail Server          DNS              GitHub Pages        Repository
    │                 │                     │                  │
    │ ❶ Query TXT     │                     │                  │
    ├────────────────>│                     │                  │
    │                 │                     │                  │
    │ ❷ Return TXT    │                     │                  │
    │<────────────────┤                     │                  │
    │                 │                     │                  │
    │ ❸ HTTPS GET .well-known/mta-sts.txt  │                  │
    ├───────────────────────────────────────>│                  │
    │                 │                     │                  │
    │                 │          ❹ Read File│                  │
    │                 │                     ├─────────────────>│
    │                 │                     │                  │
    │                 │          ❺ File Content                │
    │                 │                     │<─────────────────┤
    │                 │                     │                  │
    │ ❻ Return Policy (200 OK)             │                  │
    │<───────────────────────────────────────┤                  │
    │                 │                     │                  │
    │ ❼ Cache Policy (24 hours)            │                  │
    │                 │                     │                  │
    │ ❽ Enforce TLS when sending email     │                  │
    │                 │                     │                  │
```

## Components

### 1. Git Repository

**Purpose:** Version-controlled storage of policy configuration

**Technology:** Git (GitHub)

**Key Files:**
- `.well-known/mta-sts.txt` - Policy file (THE core file)
- `.nojekyll` - GitHub Pages control file
- `README.md` - Documentation hub
- `CHANGELOG.md` - Version history

**Functions:**
- Store policy configuration
- Track changes over time
- Enable rollback capability
- Collaborate on updates

**Location:** `https://github.com/jdgonzalesdp/livalittle`

### 2. GitHub Pages (Static Hosting)

**Purpose:** HTTPS web server for policy delivery

**Technology:** GitHub Pages (Jekyll bypass mode)

**Configuration:**
- **Source:** main branch, root directory
- **Custom Domain:** mta-sts.livalittle.com
- **HTTPS:** Enforced (Let's Encrypt)
- **Jekyll:** Disabled via `.nojekyll`

**Functions:**
- Serve `.well-known/mta-sts.txt` via HTTPS
- TLS termination
- Content caching
- Automatic deployment on git push

**Endpoint:** `https://mta-sts.livalittle.com/.well-known/mta-sts.txt`

### 3. DNS Infrastructure

**Purpose:** Policy discovery and hostname resolution

**Provider:** User-configured (Cloudflare, Route 53, etc.)

**Records:**

**TXT Record (Policy Discovery):**
```
Name:  _mta-sts.livalittle.com
Type:  TXT
Value: "v=STSv1; id=20250206"
TTL:   3600
```

**CNAME Record (Policy Hosting):**
```
Name:  mta-sts.livalittle.com
Type:  CNAME
Value: jdgonzalesdp.github.io.
TTL:   3600
```

**Functions:**
- Announce MTA-STS support
- Version policy via `id` parameter
- Resolve policy hostname to GitHub Pages
- Cache DNS responses

### 4. Let's Encrypt (Certificate Authority)

**Purpose:** SSL/TLS certificate for HTTPS

**Technology:** Let's Encrypt (automated by GitHub)

**Certificate:**
- **Subject:** mta-sts.livalittle.com
- **Issuer:** Let's Encrypt Authority
- **Validity:** 90 days (auto-renewed)

**Functions:**
- Provide trusted certificate
- Enable HTTPS
- Automatic renewal

### 5. MTA-STS Policy File

**Purpose:** Define email security requirements

**File:** `.well-known/mta-sts.txt`

**Format:** Plain text, RFC 8461 compliant

**Content:**
```
version: STSv1
mode: enforce
mx: *.livalittle.com
max_age: 86400
```

**Functions:**
- Declare TLS requirement
- Specify authorized mail servers
- Set cache duration
- Define enforcement mode

### 6. Mail Servers (External)

**Purpose:** Consume and enforce policy

**Type:** Third-party SMTP servers (Gmail, Outlook, etc.)

**Functions:**
- Query DNS for MTA-STS support
- Fetch policy via HTTPS
- Validate policy syntax
- Cache policy
- Enforce TLS requirements
- Report via TLS-RPT (optional)

## Data Flow

### Request Flow (Policy Fetch)

```
1. External mail server prepares to send email to user@livalittle.com

2. Mail server checks for MTA-STS support:
   DNS Query: _mta-sts.livalittle.com TXT
   ↓
   Response: "v=STSv1; id=20250206"
   ✓ MTA-STS is supported

3. Mail server fetches policy:
   HTTPS GET: https://mta-sts.livalittle.com/.well-known/mta-sts.txt
   ↓
   DNS Resolution: mta-sts.livalittle.com → jdgonzalesdp.github.io → [GitHub IP]
   ↓
   TLS Handshake: Verify Let's Encrypt certificate
   ↓
   HTTP Request: GET /.well-known/mta-sts.txt
   ↓
   GitHub Pages: Read from repository
   ↓
   HTTP Response: 200 OK + policy content
   ↓
   Mail server receives:
   version: STSv1
   mode: enforce
   mx: *.livalittle.com
   max_age: 86400

4. Mail server parses and caches policy for 86400 seconds (24 hours)

5. Mail server looks up MX records:
   DNS Query: livalittle.com MX
   ↓
   Response: 10 mail.livalittle.com

6. Mail server validates: "mail.livalittle.com" matches "*.livalittle.com" ✓

7. Mail server enforces TLS:
   SMTP Connection: mail.livalittle.com:25
   ↓
   STARTTLS required (mode: enforce)
   ↓
   TLS Handshake + Certificate Validation
   ↓
   If success: Deliver email
   If failure: Bounce email (enforce mode)
```

### Update Flow (Policy Changes)

```
1. Developer updates .well-known/mta-sts.txt locally
   ↓
2. Commit changes to Git
   ↓
3. Push to GitHub repository
   ↓
4. GitHub triggers Pages deployment
   ↓
5. GitHub Pages updates static files (5-10 minutes)
   ↓
6. New policy available at HTTPS endpoint
   ↓
7. Developer updates DNS TXT record:
   id=20250206 → id=20250207
   ↓
8. DNS propagates (TTL: 3600 seconds = 1 hour)
   ↓
9. Mail servers detect new policy ID
   ↓
10. Mail servers fetch updated policy
    ↓
11. Mail servers cache new policy (max_age: 86400 seconds)
    ↓
12. New policy fully effective after: DNS TTL + max_age ≈ 25 hours
```

## Infrastructure

### Hosting Infrastructure

**Provider:** GitHub Pages

**Characteristics:**
- **Type:** Static file hosting
- **CDN:** Fastly (GitHub's CDN provider)
- **Geographic Distribution:** Global edge locations
- **Uptime SLA:** 99.9% (GitHub Pages SLA)
- **Bandwidth:** Unlimited (within GitHub's fair use)
- **Storage:** Git repository size limits (1GB soft limit)

**Benefits:**
- Zero cost
- Global CDN
- Automatic HTTPS
- High availability
- DDoS protection

**Limitations:**
- Static content only
- No server-side processing
- No custom backend
- Deployment delay (5-10 minutes)

### DNS Infrastructure

**Provider:** User Choice (Cloudflare, AWS Route 53, Google Cloud DNS, etc.)

**Requirements:**
- Support for TXT records
- Support for CNAME records
- Configurable TTL values
- Reliable resolution (99.9%+ uptime)

**Recommended Features:**
- DNSSEC support
- DDoS protection
- Global anycast network
- Low latency (<50ms)

### Certificate Infrastructure

**Provider:** Let's Encrypt (via GitHub Pages)

**Characteristics:**
- **Type:** Domain Validation (DV) certificate
- **Validity:** 90 days
- **Renewal:** Automatic (GitHub handles)
- **Chain:** Let's Encrypt → DST Root CA X3 / ISRG Root X1
- **Trust:** Trusted by all major browsers and mail servers

## Network Architecture

### Request Path

```
Mail Server → [Internet] → DNS Resolver → DNS Provider → [Response]
                    ↓
Mail Server → [Internet] → GitHub CDN Edge → GitHub Origin → Repository
```

### Geographic Distribution

**GitHub Pages CDN Locations (examples):**
- North America: Multiple PoPs
- Europe: Multiple PoPs
- Asia: Multiple PoPs
- Oceania: Multiple PoPs
- South America: Multiple PoPs

**Benefits:**
- Low latency globally
- High availability
- DDoS mitigation

### Network Protocols

| Layer | Protocol | Purpose |
|-------|----------|---------|
| Application | HTTP/2, HTTP/1.1 | Policy delivery |
| Security | TLS 1.2, TLS 1.3 | Encryption |
| Transport | TCP | Reliable delivery |
| Network | IPv4, IPv6 | Addressing |
| DNS | DNS over UDP/TCP | Name resolution |

## Security Architecture

### Security Layers

```
┌─────────────────────────────────────────────────────────┐
│  Layer 1: Transport Security (TLS)                      │
│  • HTTPS required for policy delivery                   │
│  • Let's Encrypt certificate                            │
│  • TLS 1.2+ required                                    │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Layer 2: DNS Security                                  │
│  • DNSSEC (optional but recommended)                    │
│  • Registrar lock                                       │
│  • 2FA on DNS provider                                  │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Layer 3: Repository Security                           │
│  • GitHub authentication                                │
│  • Branch protection (optional)                         │
│  • Signed commits (optional)                            │
│  • Access control                                       │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Layer 4: Policy Validation                             │
│  • RFC 8461 compliance                                  │
│  • Syntax validation                                    │
│  • Pattern matching for MX                              │
└─────────────────────────────────────────────────────────┘
```

### Attack Surface

**Potential Attack Vectors:**

1. **DNS Compromise**
   - Mitigation: DNSSEC, registrar lock, 2FA

2. **GitHub Account Compromise**
   - Mitigation: 2FA, SSH keys, audit logs

3. **Man-in-the-Middle (Policy Fetch)**
   - Mitigation: HTTPS required, certificate validation

4. **Policy Cache Poisoning**
   - Mitigation: max_age limits, policy ID versioning

5. **DDoS on Policy Endpoint**
   - Mitigation: GitHub CDN, rate limiting

### Security Controls

| Control | Implemented | Optional |
|---------|-------------|----------|
| HTTPS (TLS) | ✓ Required | - |
| Certificate Validation | ✓ Required | - |
| DNS TXT Record | ✓ Required | - |
| Version Control (Git) | ✓ Implemented | - |
| DNSSEC | - | ✓ Recommended |
| 2FA (GitHub) | - | ✓ Recommended |
| 2FA (DNS) | - | ✓ Recommended |
| Branch Protection | - | ✓ Optional |
| Signed Commits | - | ✓ Optional |

## Deployment Architecture

### CI/CD Pipeline

```
Developer → Local Changes → Git Commit → Git Push
                                           ↓
                                    GitHub Repository
                                           ↓
                                  GitHub Actions (Optional)
                                           ↓
                                    Trigger Pages Build
                                           ↓
                                  Pages Build & Deploy
                                           ↓
                                    CDN Cache Update
                                           ↓
                                  Policy Live (5-10 min)
```

### Deployment Stages

1. **Development:** Local repository, changes made
2. **Commit:** Changes committed to Git
3. **Push:** Pushed to GitHub repository
4. **Build:** GitHub Pages builds static site
5. **Deploy:** New content deployed to CDN
6. **Propagate:** CDN caches update globally
7. **Live:** New policy available

### Rollback Capability

```
Emergency Rollback:
1. git revert <commit-hash>
2. git push origin main
3. Wait for Pages deployment (5-10 min)
4. Update DNS id to force cache refresh
5. Wait for DNS + Policy cache expiry (up to 25 hours)

Alternative (Fast):
1. Update policy mode to "testing" or "none"
2. Push changes
3. Update DNS id immediately
4. Effective in: Pages deploy + DNS TTL + max_age
```

## Scalability & Performance

### Performance Characteristics

**Policy File:**
- Size: ~100 bytes
- Parse time: <1ms
- Transfer time: <50ms (global average)

**DNS Lookups:**
- TXT record query: 10-50ms
- CNAME resolution: 10-50ms
- Total DNS time: 20-100ms

**HTTPS Request:**
- TLS handshake: 50-200ms
- HTTP GET: 10-50ms
- Total HTTPS time: 60-250ms

**Total Policy Fetch Time:** 80-350ms (global average)

### Scalability

**Horizontal Scaling:**
- GitHub CDN handles automatically
- No manual scaling required
- Unlimited concurrent requests (within fair use)

**Caching:**
- **Mail Server Cache:** 86400 seconds (24 hours)
- **DNS Cache:** 3600 seconds (1 hour)
- **CDN Cache:** GitHub manages

**Capacity:**
- Requests/second: Unlimited (GitHub CDN)
- Geographic distribution: Global
- Failover: Automatic (GitHub infrastructure)

### Bottlenecks & Mitigation

| Bottleneck | Impact | Mitigation |
|------------|--------|------------|
| GitHub Pages Down | Policy unavailable to new senders | Cached policies still valid |
| DNS Propagation | Slow policy updates | Use shorter max_age during updates |
| Let's Encrypt Renewal | Temporary cert errors | GitHub auto-renews |
| Repository Size | Slower clones | Keep repo minimal |

## Dependencies

### External Dependencies

```
┌─────────────────────────────────────────────────────────┐
│  GitHub.com                                             │
│  • Repository hosting                                   │
│  • Version control                                      │
│  • CI/CD (Pages build)                                  │
│  • Dependency: CRITICAL                                 │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  GitHub Pages                                           │
│  • Static hosting                                       │
│  • CDN distribution                                     │
│  • HTTPS serving                                        │
│  • Dependency: CRITICAL                                 │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  Let's Encrypt                                          │
│  • SSL/TLS certificates                                 │
│  • Automatic renewal                                    │
│  • Dependency: CRITICAL                                 │
└─────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────┐
│  DNS Provider (User Choice)                             │
│  • DNS hosting                                          │
│  • Record management                                    │
│  • Dependency: CRITICAL                                 │
└─────────────────────────────────────────────────────────┘
```

### Dependency Risk Assessment

| Dependency | Risk Level | Mitigation |
|------------|------------|------------|
| GitHub | Low | 99.9% uptime, policy caching |
| GitHub Pages | Low | Policy caching reduces impact |
| Let's Encrypt | Low | Auto-renewal, GitHub manages |
| DNS Provider | Medium | Choose reliable provider, monitor |
| Mail Servers | N/A | External, not controllable |

### Vendor Lock-in

**GitHub Pages:** Medium lock-in
- Migration path: Any static hosting (Netlify, Vercel, Cloudflare Pages)
- Portability: High (just static files)
- Effort to migrate: Low (update DNS CNAME)

**Let's Encrypt:** No lock-in
- Any certificate authority works
- GitHub manages automatically

**DNS Provider:** No lock-in
- Standard DNS records
- Easy to transfer

## Monitoring & Observability

### Key Metrics

**Availability:**
- Policy endpoint uptime
- DNS resolution success rate
- Certificate validity

**Performance:**
- Policy fetch latency
- DNS query time
- HTTPS response time

**Usage:**
- Policy fetch frequency
- Geographic distribution
- User agent analysis

### Monitoring Points

```
┌─────────────────┐
│  DNS Monitor    │ → Query _mta-sts.livalittle.com TXT
└─────────────────┘

┌─────────────────┐
│  HTTPS Monitor  │ → GET https://mta-sts.livalittle.com/.well-known/mta-sts.txt
└─────────────────┘

┌─────────────────┐
│  Cert Monitor   │ → Check certificate expiration
└─────────────────┘

┌─────────────────┐
│  Policy Monitor │ → Validate policy syntax
└─────────────────┘
```

## Disaster Recovery

### Failure Scenarios

**1. GitHub Pages Outage**
- Impact: New policy fetches fail
- Duration: Temporary (GitHub resolves)
- Mitigation: Cached policies still valid (24h)
- Recovery: Automatic when GitHub recovers

**2. DNS Provider Outage**
- Impact: No policy discovery
- Duration: Depends on provider
- Mitigation: Cached policies still valid
- Recovery: Automatic when DNS recovers

**3. Certificate Expiration**
- Impact: Policy fetch fails (HTTPS required)
- Duration: Until renewed
- Mitigation: GitHub auto-renews
- Recovery: Automatic

**4. Policy File Corruption**
- Impact: Invalid policy served
- Duration: Until fixed
- Mitigation: Git rollback
- Recovery: Minutes (git revert + push)

### Business Continuity

**RTO (Recovery Time Objective):** 10 minutes
- Git revert + GitHub Pages deploy

**RPO (Recovery Point Objective):** Last commit
- All changes in Git history

**Backup Strategy:**
- Git repository is the backup
- Clone repository locally
- Multiple team members have access

---

**Document Version:** 1.0
**Last Updated:** 2025-02-06
**Architecture Review:** Required when adding new components or changing infrastructure
