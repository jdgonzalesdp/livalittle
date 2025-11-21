# livalittle.com - MTA-STS Email Security Policy

## Overview

This repository hosts the **MTA-STS (Mail Transfer Agent Strict Transport Security)** policy file for the `livalittle.com` domain. MTA-STS is a security standard (RFC 8461) that ensures email sent to livalittle.com is always transmitted over encrypted TLS connections, protecting against man-in-the-middle attacks and email interception.

## What is MTA-STS?

MTA-STS is an email security protocol that:
- **Prevents downgrade attacks** on email encryption
- **Ensures TLS encryption** for all incoming mail
- **Protects against SMTP MITM attacks** by verifying certificates
- **Works alongside DANE** for enhanced email security

When a mail server sends email to @livalittle.com, it:
1. Checks DNS for MTA-STS support
2. Fetches the policy from this repository (via GitHub Pages)
3. Enforces TLS encryption based on the policy
4. Fails email delivery if secure connection cannot be established

## Repository Structure

```
livalittle/
├── .well-known/
│   └── mta-sts.txt           # MTA-STS policy file (RFC 8461 compliant)
├── .nojekyll                  # GitHub Pages configuration
├── README.md                  # This file
├── SETUP.md                   # Deployment and configuration guide
├── ARCHITECTURE.md            # System architecture and design
├── HOW-IT-WORKS.md            # Functional explanation of the system
├── DEPLOYMENT-FLOW.md         # Deployment processes and workflows
├── API-REFERENCE.md           # Policy file API specification
├── MTA-STS-GUIDE.md          # Detailed MTA-STS protocol documentation
├── DNS-CONFIGURATION.md       # DNS setup instructions
├── TROUBLESHOOTING.md         # Common issues and solutions
├── TESTING.md                 # How to test and validate the policy
├── CHANGELOG.md               # Version history
├── CONTRIBUTING.md            # Contribution guidelines
└── LICENSE                    # License information
```

## Current Policy

The MTA-STS policy for livalittle.com is configured as follows:

- **Version:** STSv1
- **Mode:** enforce (strict enforcement, email delivery fails if TLS unavailable)
- **MX Servers:** *.livalittle.com (all mail servers under this domain)
- **Cache Duration:** 86400 seconds (24 hours)

## Quick Start

### Prerequisites

1. **Domain:** livalittle.com
2. **GitHub Pages:** Enabled and configured
3. **DNS Access:** Ability to create TXT records
4. **HTTPS Subdomain:** mta-sts.livalittle.com pointing to GitHub Pages

### Minimal Setup Steps

1. **Enable GitHub Pages** on this repository
   - Settings → Pages → Source: Deploy from branch (main)
   - Custom domain: mta-sts.livalittle.com

2. **Configure DNS Records**
   ```
   # MTA-STS discovery record
   _mta-sts.livalittle.com. IN TXT "v=STSv1; id=20250206"

   # CNAME for policy hosting
   mta-sts.livalittle.com. IN CNAME jdgonzalesdp.github.io.
   ```

3. **Verify Policy Access**
   ```bash
   curl https://mta-sts.livalittle.com/.well-known/mta-sts.txt
   ```

4. **Test with Online Tools**
   - https://aykevl.nl/apps/mta-sts/
   - https://www.hardenize.com/

See [SETUP.md](SETUP.md) for detailed instructions.

## Documentation

### Getting Started
| Document | Description |
|----------|-------------|
| [SETUP.md](SETUP.md) | Complete deployment and configuration guide |
| [HOW-IT-WORKS.md](HOW-IT-WORKS.md) | How the system works (functional explanation) |
| [MTA-STS-GUIDE.md](MTA-STS-GUIDE.md) | In-depth explanation of MTA-STS protocol |

### Technical Reference
| Document | Description |
|----------|-------------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | System architecture and component design |
| [API-REFERENCE.md](API-REFERENCE.md) | Complete API specification for policy file |
| [DEPLOYMENT-FLOW.md](DEPLOYMENT-FLOW.md) | Deployment workflows and processes |
| [DNS-CONFIGURATION.md](DNS-CONFIGURATION.md) | DNS record setup and management |

### Operations
| Document | Description |
|----------|-------------|
| [TESTING.md](TESTING.md) | Validation and testing procedures |
| [TROUBLESHOOTING.md](TROUBLESHOOTING.md) | Common issues and debugging |
| [CHANGELOG.md](CHANGELOG.md) | Version history and updates |
| [CONTRIBUTING.md](CONTRIBUTING.md) | How to contribute to this repository |

## Policy Modes Explained

This repository uses **enforce mode**, which provides maximum security:

| Mode | Behavior | Security Level | Use Case |
|------|----------|----------------|----------|
| `testing` | TLS preferred but not required | Low | Initial deployment, testing |
| `enforce` | TLS required, delivery fails if unavailable | **High** | Production (current) |
| `none` | Policy explicitly disabled | None | Deactivation |

## Security Considerations

### Protection Provided
- Prevents SMTP downgrade attacks
- Ensures certificate validation
- Protects against DNS spoofing (when combined with DNSSEC)
- Enforces encryption for email in transit

### Limitations
- Does not protect email at rest
- Requires both sender and receiver support
- 24-hour cache means policy changes take time to propagate
- Relies on HTTPS (GitHub Pages) for policy delivery

## Maintenance

### When to Update the Policy

Update the `id` in the DNS TXT record when you:
- Change the policy mode
- Modify MX server patterns
- Update max_age duration
- Make any changes to the policy file

### Policy Update Workflow

1. Update `.well-known/mta-sts.txt` in this repository
2. Commit and push changes (GitHub Pages deploys automatically)
3. Wait 5-10 minutes for GitHub Pages to update
4. Update DNS TXT record with new `id` value
5. Test the new policy with validation tools
6. Monitor email delivery for 24 hours

## Monitoring

### Recommended Monitoring

- **SMTP TLS-RPT Reports:** Configure TLS reporting (RFC 8460)
- **Email Delivery:** Monitor bounce rates and delivery failures
- **Policy Accessibility:** Regular checks that policy URL is accessible
- **Certificate Validity:** Ensure GitHub Pages certificate is valid

### TLS-RPT Configuration

Add this DNS record to receive delivery reports:
```
_smtp._tls.livalittle.com. IN TXT "v=TLSRPTv1; rua=mailto:tls-reports@livalittle.com"
```

## Technology Stack

- **Hosting:** GitHub Pages (static file hosting)
- **Protocol:** RFC 8461 (SMTP MTA Strict Transport Security)
- **Format:** Plain text policy file
- **DNS:** TXT records for policy discovery
- **Security:** HTTPS for policy delivery

## Support

### Validation Tools
- [MTA-STS Validator](https://aykevl.nl/apps/mta-sts/)
- [Hardenize](https://www.hardenize.com/)
- [MXToolbox MTA-STS Check](https://mxtoolbox.com/mta-sts.aspx)

### Email Security Standards
- [RFC 8461 - MTA-STS](https://tools.ietf.org/html/rfc8461)
- [RFC 8460 - TLS-RPT](https://tools.ietf.org/html/rfc8460)
- [RFC 7672 - DANE](https://tools.ietf.org/html/rfc7672)

## Contributing

Contributions are welcome! Please read [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

## License

See [LICENSE](LICENSE) file for details.

## Contact

For issues or questions related to email delivery to @livalittle.com, please contact the domain administrator.

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history and updates.

---

**Repository:** github.com/jdgonzalesdp/livalittle
**Domain:** livalittle.com
**Policy URL:** https://mta-sts.livalittle.com/.well-known/mta-sts.txt
**Last Updated:** 2025-02-06
