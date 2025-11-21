# MTA-STS Policy API Reference

Complete technical specification for the MTA-STS policy file format and HTTP API.

## Table of Contents

1. [Overview](#overview)
2. [HTTP API](#http-api)
3. [Policy File Format](#policy-file-format)
4. [Field Specifications](#field-specifications)
5. [DNS API](#dns-api)
6. [Response Codes](#response-codes)
7. [Error Handling](#error-handling)
8. [Examples](#examples)
9. [Validation Rules](#validation-rules)
10. [RFC Compliance](#rfc-compliance)

## Overview

### API Type

**Classification:** RESTful read-only API (HTTP GET)
**Protocol:** HTTPS (TLS 1.2+)
**Format:** Plain text (text/plain)
**Encoding:** UTF-8
**RFC:** RFC 8461 - SMTP MTA Strict Transport Security

### Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/.well-known/mta-sts.txt` | GET | Retrieve MTA-STS policy |

### Base URL

```
https://mta-sts.livalittle.com
```

**Full policy URL:**
```
https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

### Authentication

**None required** - Public, unauthenticated endpoint

### Rate Limiting

**GitHub Pages:** No documented rate limits
**Caching:** 24-hour client-side caching (via max_age)

## HTTP API

### GET /.well-known/mta-sts.txt

**Retrieves the MTA-STS policy for the domain.**

#### Request

**Method:** GET

**URL:**
```
https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

**Headers:**
```http
GET /.well-known/mta-sts.txt HTTP/1.1
Host: mta-sts.livalittle.com
User-Agent: <mail-server-identifier>
Accept: text/plain
Connection: close
```

**Parameters:** None

**Body:** None

#### Response (Success)

**Status Code:** 200 OK

**Headers:**
```http
HTTP/2 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: <length>
Cache-Control: public, max-age=86400
Date: <date>
```

**Body:**
```
version: STSv1
mode: enforce
mx: *.livalittle.com
max_age: 86400
```

**Content-Type:** MUST be `text/plain` or `text/plain; charset=utf-8`

#### Response (Error)

**404 Not Found:**
```http
HTTP/2 404 Not Found
Content-Type: text/html
```
Policy file does not exist or path is incorrect.

**403 Forbidden:**
```http
HTTP/2 403 Forbidden
```
Access denied (should not occur for properly configured GitHub Pages).

**503 Service Unavailable:**
```http
HTTP/2 503 Service Unavailable
```
GitHub Pages temporarily unavailable.

### TLS Requirements

**Minimum TLS Version:** TLS 1.2
**Recommended:** TLS 1.3

**Certificate Requirements:**
- MUST be from publicly trusted CA
- MUST be valid (not expired)
- MUST match hostname: `mta-sts.livalittle.com`
- MUST include complete certificate chain

**Cipher Suites:** Modern, secure suites only

### HTTP Redirects

**301/302 Redirects:** Allowed to HTTPS on same hostname
**Redirects to different hostname:** NOT allowed (policy fetch fails)

**Example (allowed):**
```
http://mta-sts.livalittle.com/.well-known/mta-sts.txt
  → 301 → https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

**Example (not allowed):**
```
https://mta-sts.livalittle.com/.well-known/mta-sts.txt
  → 301 → https://example.com/.well-known/mta-sts.txt
  (Different hostname - FAILS)
```

## Policy File Format

### Syntax

**Format:** Key-value pairs, one per line
**Separator:** Colon and space (`: `)
**Line Ending:** LF (`\n`) or CRLF (`\r\n`)
**Encoding:** UTF-8
**Maximum Size:** Not specified in RFC, recommended < 64 KB

### Structure

```abnf
policy     = version-field mode-field mx-field+ max-age-field
version-field = "version" ":" SP "STSv1" CRLF
mode-field = "mode" ":" SP ("testing" / "enforce" / "none") CRLF
mx-field   = "mx" ":" SP domain CRLF
max-age-field = "max_age" ":" SP 1*DIGIT CRLF
```

### Example (Minimal)

```
version: STSv1
mode: enforce
mx: mail.example.com
max_age: 86400
```

### Example (Multiple MX)

```
version: STSv1
mode: enforce
mx: mail1.example.com
mx: mail2.example.com
mx: mail3.example.com
max_age: 604800
```

### Example (Wildcard MX)

```
version: STSv1
mode: enforce
mx: *.example.com
max_age: 86400
```

## Field Specifications

### version

**Required:** Yes
**Type:** String (literal)
**Value:** `STSv1`
**Case-Sensitive:** Yes

**Description:**
Specifies the MTA-STS protocol version. Currently, only `STSv1` exists.

**Format:**
```
version: STSv1
```

**Validation:**
- MUST be exactly `STSv1` (case-sensitive)
- MUST appear exactly once
- MUST be the first field (by convention, not RFC requirement)

**Examples:**

✓ **Valid:**
```
version: STSv1
```

✗ **Invalid:**
```
version: stsv1          # Wrong case
version: STSv2          # Non-existent version
version:STSv1           # Missing space after colon
version : STSv1         # Space before colon
```

### mode

**Required:** Yes
**Type:** Enum
**Values:** `testing`, `enforce`, `none`
**Case-Sensitive:** Yes

**Description:**
Defines the policy enforcement level.

**Format:**
```
mode: <value>
```

**Values:**

| Value | Description | Behavior |
|-------|-------------|----------|
| `testing` | Testing mode | TLS attempted but not required; failures reported |
| `enforce` | Enforcement mode | TLS required; delivery fails if TLS unavailable |
| `none` | Disabled | MTA-STS explicitly disabled |

**Detailed Behavior:**

**`testing` mode:**
- Mail servers SHOULD attempt TLS
- Failures do not prevent delivery
- Failures MAY be reported via TLS-RPT
- Use for initial deployment and testing

**`enforce` mode:**
- Mail servers MUST use TLS
- Delivery MUST fail if TLS is unavailable or fails validation
- Provides maximum security
- Use for production

**`none` mode:**
- MTA-STS is explicitly disabled
- Mail servers revert to opportunistic TLS
- Use for planned deactivation
- `mx` field not required when mode is `none`

**Validation:**
- MUST be one of: `testing`, `enforce`, `none`
- MUST appear exactly once
- Case-sensitive

**Examples:**

✓ **Valid:**
```
mode: testing
mode: enforce
mode: none
```

✗ **Invalid:**
```
mode: strict            # Invalid value
mode: Enforce           # Wrong case
mode:enforce            # Missing space
```

### mx

**Required:** Conditional (required if mode ≠ none)
**Type:** String (hostname or pattern)
**Multiple:** Yes (can appear multiple times)
**Case-Sensitive:** No (DNS is case-insensitive)

**Description:**
Specifies authorized MX hostnames. Supports wildcards.

**Format:**
```
mx: <hostname-or-pattern>
```

**Hostname Rules:**
- Valid DNS hostname
- May include wildcard (`*`) as leftmost label
- Wildcard matches exactly one label
- No internationalized domain names (IDN) in punycode form

**Wildcard Matching:**

**Pattern:** `*.example.com`
- Matches: `mail.example.com` ✓
- Matches: `smtp.example.com` ✓
- Does NOT match: `example.com` ✗ (no subdomain)
- Does NOT match: `sub.mail.example.com` ✗ (nested subdomain)

**Pattern:** `mail.example.com`
- Matches: `mail.example.com` ✓
- Does NOT match: `smtp.example.com` ✗ (different subdomain)

**Multiple MX Entries:**
```
mx: mail1.example.com
mx: mail2.example.com
mx: backup.example.com
```

All MX servers for the domain MUST match at least one pattern.

**Validation:**
- MUST be valid DNS hostname or pattern
- MAY appear multiple times
- If mode is `testing` or `enforce`, MUST appear at least once
- If mode is `none`, MAY be omitted

**Examples:**

✓ **Valid:**
```
mx: mail.example.com
mx: *.example.com
mx: mail-01.example.com
mx: mail.sub.example.com
```

✗ **Invalid:**
```
mx: *example.com        # Wildcard without dot
mx: **.example.com      # Multiple wildcards
mx: mail.*.example.com  # Wildcard not leftmost
mx: example.com:25      # Port number (not allowed)
mx:mail.example.com     # Missing space
```

### max_age

**Required:** Yes
**Type:** Integer (seconds)
**Range:** 0 to 2^31 - 1 (2,147,483,647)
**Unit:** Seconds

**Description:**
Specifies how long the policy should be cached by sending mail servers.

**Format:**
```
max_age: <seconds>
```

**Recommended Values:**

| Duration | Seconds | Use Case |
|----------|---------|----------|
| 5 minutes | 300 | Testing, frequent changes |
| 1 hour | 3600 | Active development |
| 1 day | 86400 | Standard (recommended) |
| 1 week | 604800 | Stable production |
| 4 weeks | 2419200 | Very stable infrastructure |

**Considerations:**

**Short max_age (< 3600):**
- Pro: Faster policy updates
- Con: More DNS/HTTPS queries
- Con: Higher server load
- Use: Testing phase

**Medium max_age (3600-86400):**
- Pro: Balance between updates and efficiency
- Con: Policy changes take time to propagate
- Use: Standard production

**Long max_age (> 86400):**
- Pro: Reduced query load
- Con: Very slow policy updates
- Use: Extremely stable infrastructure only

**Validation:**
- MUST be a non-negative integer
- MUST appear exactly once
- SHOULD be ≥ 86400 for production (not enforced)

**Examples:**

✓ **Valid:**
```
max_age: 86400
max_age: 3600
max_age: 604800
max_age: 0              # Valid but not recommended
```

✗ **Invalid:**
```
max_age: -1             # Negative value
max_age: 86400.5        # Decimal (must be integer)
max_age: 1d             # Non-numeric
max_age:86400           # Missing space
max_age: 86400s         # Unit suffix (not allowed)
```

## DNS API

### TXT Record Specification

**Purpose:** Advertise MTA-STS support and policy version

**Record Format:**
```
_mta-sts.livalittle.com. IN TXT "v=STSv1; id=<policy-id>"
```

**Fields:**

#### v (Version)

**Required:** Yes
**Type:** String (literal)
**Value:** `STSv1`
**Case-Sensitive:** Yes

#### id (Policy ID)

**Required:** Yes
**Type:** String (alphanumeric)
**Max Length:** Not specified (keep under 64 characters)
**Format:** Alphanumeric, hyphens, underscores

**Purpose:**
Uniquely identifies policy version. Change when policy file changes.

**Recommended Format:** `YYYYMMDD` (date-based)

**Examples:**
```
id=20250206           # Date-based (recommended)
id=2025-02-06         # Date with hyphens
id=v1.2.3             # Version number
id=abc123             # Arbitrary string
```

**When to Update:**
- Policy file content changes
- Mode changes
- MX patterns change
- max_age changes

**Why it matters:**
Mail servers use this to detect policy updates and invalidate cache.

### DNS Query

**Query Type:** TXT
**Query Name:** `_mta-sts.<domain>`

**Example:**
```bash
dig _mta-sts.livalittle.com TXT
```

**Response:**
```
_mta-sts.livalittle.com. 3600 IN TXT "v=STSv1; id=20250206"
```

### CNAME Record Specification

**Purpose:** Point policy hostname to hosting provider

**Record Format:**
```
mta-sts.livalittle.com. IN CNAME jdgonzalesdp.github.io.
```

**Requirements:**
- MUST point to hosting provider (GitHub Pages)
- MUST NOT coexist with other record types for same name
- Target MUST serve valid HTTPS with trusted certificate

## Response Codes

### HTTP Status Codes

| Code | Meaning | Client Action |
|------|---------|---------------|
| 200 | OK | Parse and cache policy |
| 301 | Moved Permanently | Follow redirect (same host only) |
| 302 | Found | Follow redirect (same host only) |
| 304 | Not Modified | Use cached policy |
| 400 | Bad Request | Ignore policy, use opportunistic TLS |
| 403 | Forbidden | Ignore policy, use opportunistic TLS |
| 404 | Not Found | Ignore policy, use opportunistic TLS |
| 500 | Internal Server Error | Retry later, use cached policy |
| 503 | Service Unavailable | Retry later, use cached policy |

### DNS Response Codes

| Code | Meaning | Client Action |
|------|---------|---------------|
| NOERROR | Record found | Proceed to fetch policy |
| NXDOMAIN | Domain does not exist | No MTA-STS support |
| SERVFAIL | DNS server error | Retry, use cached policy |
| REFUSED | Query refused | No MTA-STS support |

## Error Handling

### Policy Fetch Failures

**Scenario:** HTTP request fails or returns non-200 status

**Mail Server Behavior:**
1. If policy is cached → Use cached policy
2. If no cache → Fall back to opportunistic TLS
3. If in enforce mode (cached) → Still enforce requirements

**Example:**
```
Mail Server: Fetch policy from HTTPS
GitHub: 503 Service Unavailable
Mail Server: Check cache
Cache: Has policy from yesterday (still valid)
Mail Server: Use cached policy, enforce TLS
```

### Invalid Policy Syntax

**Scenario:** Policy file has syntax errors

**Mail Server Behavior:**
1. Reject invalid policy
2. If previous valid policy cached → Use cached policy
3. If no cache → Fall back to opportunistic TLS
4. MAY report error via TLS-RPT

**Example:**
```
Policy file contains:
  version: STSv1
  mode: invalid_value    ← Invalid mode
  mx: *.example.com
  max_age: 86400

Mail Server: Parse policy
Mail Server: Invalid mode detected
Mail Server: Reject policy
Mail Server: Check cache
Cache: Has old valid policy
Mail Server: Use cached policy
```

### Certificate Errors

**Scenario:** HTTPS certificate is invalid, expired, or doesn't match

**Mail Server Behavior:**
1. MUST reject policy fetch
2. Use cached policy if available
3. Fall back to opportunistic TLS if no cache

**Example:**
```
Mail Server: Connect to mta-sts.example.com
Server: Presents expired certificate
Mail Server: Certificate validation fails
Mail Server: ABORT policy fetch
Mail Server: Use cached policy or fall back to opportunistic TLS
```

### DNS Failures

**Scenario:** Cannot resolve _mta-sts TXT record

**Mail Server Behavior:**
1. If temporary failure (SERVFAIL) → Retry
2. If NXDOMAIN → No MTA-STS support
3. If cached policy → Continue using cache

## Examples

### Example 1: Minimal Policy

```
version: STSv1
mode: testing
mx: mail.livalittle.com
max_age: 3600
```

**Use case:** Initial testing deployment

### Example 2: Production Policy (Current)

```
version: STSv1
mode: enforce
mx: *.livalittle.com
max_age: 86400
```

**Use case:** Production, wildcard MX, 24-hour cache

### Example 3: Multiple MX Servers

```
version: STSv1
mode: enforce
mx: mail1.livalittle.com
mx: mail2.livalittle.com
mx: backup.livalittle.com
max_age: 604800
```

**Use case:** Explicit MX list, 1-week cache

### Example 4: Third-Party Email Provider

```
version: STSv1
mode: enforce
mx: *.google.com
mx: *.googlemail.com
max_age: 86400
```

**Use case:** Google Workspace

### Example 5: Disabled Policy

```
version: STSv1
mode: none
max_age: 300
```

**Use case:** Temporarily disabling MTA-STS

### Example 6: Complex Multi-Provider

```
version: STSv1
mode: enforce
mx: *.livalittle.com
mx: *.google.com
mx: *.protection.outlook.com
max_age: 86400
```

**Use case:** Hybrid email setup (own servers + cloud providers)

## Validation Rules

### Policy File Validation

**Checklist:**

- [ ] File is plain text (UTF-8)
- [ ] Contains `version: STSv1` (exactly)
- [ ] Contains `mode:` with valid value
- [ ] Contains `max_age:` with integer value
- [ ] Contains at least one `mx:` (if mode ≠ none)
- [ ] No extra whitespace around values
- [ ] No backslashes or special characters
- [ ] Proper line endings (LF or CRLF)
- [ ] File size < 64 KB
- [ ] No duplicate fields (except mx)

### DNS Validation

**TXT Record:**
- [ ] Name is `_mta-sts.<domain>`
- [ ] Contains `v=STSv1`
- [ ] Contains `id=<value>`
- [ ] Semicolon separator between parameters
- [ ] TTL is reasonable (recommended: 3600)

**CNAME Record:**
- [ ] Name is `mta-sts.<domain>`
- [ ] Points to hosting provider
- [ ] No conflicting A/AAAA records

### HTTP Validation

- [ ] HTTPS required (not HTTP)
- [ ] Valid certificate from trusted CA
- [ ] Certificate matches hostname
- [ ] Returns 200 status code
- [ ] Content-Type is text/plain
- [ ] No redirects to different hostname

## RFC Compliance

### RFC 8461 Requirements

**MUST:**
- Serve policy over HTTPS with valid certificate
- Use exactly `STSv1` for version
- Include version, mode, max_age fields
- Include mx field if mode is testing or enforce
- Use one of: testing, enforce, none for mode
- max_age be a non-negative integer

**SHOULD:**
- Use max_age ≥ 86400 for production
- Implement TLS-RPT (RFC 8460) for reporting

**MAY:**
- Include multiple mx fields
- Use wildcards in mx patterns

### Non-Standard Extensions

**None implemented**

This implementation strictly follows RFC 8461 with no custom extensions.

### Future Compatibility

If RFC 8461 is updated or new versions are released:
- Current policies will continue to work
- New version field would be introduced (e.g., STSv2)
- Backward compatibility maintained

## API Changelog

### v1.0 (Current)

- Initial implementation
- RFC 8461 compliance
- mode: enforce
- mx: *.livalittle.com
- max_age: 86400

---

**Document Version:** 1.0
**Last Updated:** 2025-02-06
**RFC:** RFC 8461
**Specification:** https://tools.ietf.org/html/rfc8461
