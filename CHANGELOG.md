# Changelog

All notable changes to the MTA-STS configuration for livalittle.com will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html) for policy versions.

## [Unreleased]

### Planned
- TLS-RPT configuration for delivery reporting
- Automated monitoring scripts
- DNSSEC implementation

## [1.0.0] - 2025-02-06

### Added
- Complete documentation suite:
  - README.md - Project overview and quick start
  - SETUP.md - Detailed deployment instructions
  - MTA-STS-GUIDE.md - Comprehensive protocol explanation
  - DNS-CONFIGURATION.md - DNS setup guide with provider examples
  - TROUBLESHOOTING.md - Common issues and solutions
  - TESTING.md - Validation and testing procedures
  - CONTRIBUTING.md - Contribution guidelines
  - CHANGELOG.md - This file
  - LICENSE - MIT License
- .gitignore file for repository cleanliness
- Repository infrastructure improvements

### Fixed
- **CRITICAL:** Fixed MTA-STS policy file formatting
  - Removed literal backslashes that were breaking RFC 8461 compliance
  - Removed erroneous closing brace character
  - Policy now uses proper newlines instead of `\` characters
  - File at `.well-known/mta-sts.txt` now RFC-compliant

### Changed
- Policy mode: enforce (strict TLS enforcement)
- Policy max_age: 86400 seconds (24 hours)
- Policy ID: 20250206

## [0.2.0] - 2025-02-06

### Changed
- Updated mta-sts.txt file content
- Modified policy configuration

### Technical Details
- Commit: afda459
- Branch: main

## [0.1.3] - 2025-02-06

### Changed
- Renamed mta-sts.txt to .well-known/mta-sts.txt
- Moved policy file to RFC 8461 required location
- Policy now accessible at: https://mta-sts.livalittle.com/.well-known/mta-sts.txt

### Technical Details
- Commit: e1689eb
- Branch: main
- **Critical:** This change was necessary for RFC compliance

## [0.1.2] - 2025-02-06

### Added
- Created .nojekyll file
- Enabled direct serving of .well-known directory by GitHub Pages
- Bypassed Jekyll processing to allow dot-prefixed directories

### Technical Details
- Commit: c13928c
- Branch: main
- **Why:** Without this file, GitHub Pages ignores `.well-known` directory

## [0.1.1] - 2025-02-06

### Fixed
- Removed RTF extension from mta-sts file
- Changed from mta-sts.rtf to mta-sts.txt
- Corrected file format to plain text

### Technical Details
- Commit: 8d11fe0
- Branch: main

## [0.1.0] - 2025-02-06

### Added
- Initial repository setup
- First version of MTA-STS policy file
- Basic file structure

### Technical Details
- Commit: 24fc141
- Branch: main
- Initial upload

---

## Policy Version History

Track DNS TXT record `id` values to understand policy evolution:

| Date | Policy ID | Mode | Max Age | MX Pattern | Notes |
|------|-----------|------|---------|------------|-------|
| 2025-02-06 | 20250206 | enforce | 86400 | *.livalittle.com | Current (fixed formatting) |
| 2025-02-06 | (various) | enforce | 86400 | *.livalittle.com | Initial deployment (broken) |

## Migration Guide

### From Broken Format to Fixed Format

**Before (v0.2.0):**
```
version: STSv1\
mode: enforce\
mx: *.livalittle.com\
max_age: 86400}
```

**After (v1.0.0):**
```
version: STSv1
mode: enforce
mx: *.livalittle.com
max_age: 86400
```

**Action Required:**
- None - fixed automatically in this update
- DNS `id` should be updated to `20250206` or later

## Deployment Timeline

```
2025-02-06: Initial files uploaded
2025-02-06: Fixed RTF extension
2025-02-06: Added .nojekyll for GitHub Pages
2025-02-06: Moved to .well-known directory
2025-02-06: Updated policy content (broken format)
2025-02-06: Fixed policy formatting (RFC compliant)
2025-02-06: Added complete documentation
```

## Breaking Changes

### v1.0.0
- **Policy file format changed** (fix for RFC 8461 compliance)
- Mail servers that cached the broken policy may experience brief inconsistency
- Recommended: Update DNS `id` to force policy refresh

## Security Advisories

### 2025-02-06: Policy Format Non-Compliance
- **Severity:** Medium
- **Impact:** MTA-STS policy may not be enforced by compliant mail servers
- **Description:** Policy file contained literal backslash characters and closing brace
- **Fixed in:** v1.0.0
- **Recommendation:** Verify mail servers are fetching updated policy

## Future Roadmap

### Version 1.1.0 (Planned)
- [ ] TLS-RPT DNS record configuration
- [ ] TLS reporting email setup
- [ ] Automated monitoring scripts
- [ ] GitHub Actions workflow for validation

### Version 1.2.0 (Planned)
- [ ] DNSSEC implementation
- [ ] Multiple MX server support
- [ ] Backup policy hosting
- [ ] Incident response playbook

### Version 2.0.0 (Future)
- [ ] DANE support (alongside MTA-STS)
- [ ] Advanced monitoring dashboard
- [ ] Automated certificate monitoring
- [ ] Policy version management automation

## Maintenance Schedule

- **Weekly:** Verify policy URL accessibility
- **Monthly:** Review TLS-RPT reports (once configured)
- **Quarterly:** Audit DNS records and configuration
- **Yearly:** Review and update documentation
- **As needed:** Update policy `id` when making changes

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for how to contribute to this project.

## Support

For issues or questions:
- Review documentation in this repository
- Check [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
- Validate with online tools (see [TESTING.md](TESTING.md))
- Open an issue on GitHub

---

**Repository:** https://github.com/jdgonzalesdp/livalittle
**Domain:** livalittle.com
**Policy URL:** https://mta-sts.livalittle.com/.well-known/mta-sts.txt
**Current Version:** 1.0.0
**Last Updated:** 2025-02-06
