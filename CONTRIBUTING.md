# Contributing to livalittle MTA-STS

Thank you for your interest in contributing to the MTA-STS email security configuration for livalittle.com!

## Table of Contents

1. [Code of Conduct](#code-of-conduct)
2. [How to Contribute](#how-to-contribute)
3. [Reporting Issues](#reporting-issues)
4. [Suggesting Enhancements](#suggesting-enhancements)
5. [Making Changes](#making-changes)
6. [Documentation Contributions](#documentation-contributions)
7. [Testing Requirements](#testing-requirements)
8. [Commit Guidelines](#commit-guidelines)
9. [Pull Request Process](#pull-request-process)
10. [Security Considerations](#security-considerations)

## Code of Conduct

### Our Pledge

We are committed to providing a welcoming and professional environment for all contributors.

### Our Standards

**Positive behavior:**
- Using welcoming and inclusive language
- Being respectful of differing viewpoints
- Accepting constructive criticism gracefully
- Focusing on what is best for the project and community

**Unacceptable behavior:**
- Harassment, discrimination, or unprofessional conduct
- Publishing others' private information
- Inappropriate or unwelcome advances
- Trolling, insulting, or derogatory comments

## How to Contribute

Contributions are welcome in several areas:

### 1. Policy Configuration
- Improvements to MTA-STS policy settings
- Optimization of max_age or mode values
- MX pattern adjustments

### 2. Documentation
- Fixing typos or unclear instructions
- Adding examples or use cases
- Improving troubleshooting guides
- Translating documentation

### 3. Testing
- Adding test scripts or validation tools
- Improving automated testing
- Reporting test results

### 4. Monitoring
- Contributing monitoring scripts
- Sharing alerting configurations
- Improving health checks

### 5. Issue Reporting
- Reporting bugs or issues
- Suggesting improvements
- Sharing experiences

## Reporting Issues

### Before Creating an Issue

1. **Search existing issues** to avoid duplicates
2. **Check documentation** ([TROUBLESHOOTING.md](TROUBLESHOOTING.md), [SETUP.md](SETUP.md))
3. **Validate your setup** using online tools (see [TESTING.md](TESTING.md))
4. **Gather diagnostic information** (see below)

### Diagnostic Information to Include

```bash
# Run this and include output in your issue
echo "=== MTA-STS Diagnostic Report ===" > issue-report.txt
echo "Date: $(date)" >> issue-report.txt
echo "" >> issue-report.txt
echo "DNS TXT Record:" >> issue-report.txt
dig _mta-sts.livalittle.com TXT >> issue-report.txt
echo "" >> issue-report.txt
echo "DNS CNAME Record:" >> issue-report.txt
dig mta-sts.livalittle.com CNAME >> issue-report.txt
echo "" >> issue-report.txt
echo "Policy Fetch:" >> issue-report.txt
curl -v https://mta-sts.livalittle.com/.well-known/mta-sts.txt >> issue-report.txt 2>&1
echo "" >> issue-report.txt
echo "Policy Content:" >> issue-report.txt
cat .well-known/mta-sts.txt >> issue-report.txt
```

### Issue Template

**Title:** Brief, descriptive title

**Description:**
```
## Summary
Brief description of the issue

## Expected Behavior
What should happen

## Actual Behavior
What actually happens

## Steps to Reproduce
1. Step one
2. Step two
3. Step three

## Environment
- Date/time of issue:
- Browser/tool used:
- Location (country/region):

## Diagnostic Output
[Paste diagnostic output here]

## Additional Context
Any other relevant information
```

### Issue Labels

- `bug` - Something isn't working
- `documentation` - Documentation improvements
- `enhancement` - New feature or improvement
- `question` - Question about usage
- `security` - Security-related issue
- `testing` - Related to testing or validation
- `dns` - DNS configuration issues
- `policy` - Policy file issues

## Suggesting Enhancements

### Enhancement Template

**Title:** Clear description of enhancement

**Description:**
```
## Problem Statement
What problem does this solve?

## Proposed Solution
How should this work?

## Alternatives Considered
What other options did you consider?

## Benefits
- Benefit 1
- Benefit 2

## Drawbacks
- Drawback 1
- Drawback 2

## Implementation Complexity
- Low / Medium / High
- Estimated effort:

## Additional Context
Links, examples, or references
```

## Making Changes

### Development Workflow

1. **Fork the repository**
   ```bash
   # Click "Fork" on GitHub
   git clone https://github.com/YOUR-USERNAME/livalittle.git
   cd livalittle
   ```

2. **Create a branch**
   ```bash
   git checkout -b feature/your-feature-name
   # or
   git checkout -b fix/your-bug-fix
   ```

3. **Make your changes**
   - Edit files as needed
   - Follow style guidelines (see below)

4. **Test your changes**
   - Validate policy file syntax
   - Test locally before pushing
   - Run validation tools

5. **Commit your changes**
   - Follow commit guidelines (see below)

6. **Push to your fork**
   ```bash
   git push origin feature/your-feature-name
   ```

7. **Create a Pull Request**
   - Use the PR template below

### Branch Naming

Use descriptive branch names:

```
feature/add-tls-rpt-config
fix/policy-syntax-error
docs/improve-setup-guide
test/add-validation-script
```

## Documentation Contributions

### Documentation Style Guide

**Formatting:**
- Use Markdown format
- Headers: Use proper hierarchy (H1, H2, H3)
- Code blocks: Include language specification (```bash, ```python)
- Links: Use relative links for internal docs

**Writing Style:**
- Clear, concise language
- Active voice preferred
- Define technical terms
- Include examples where helpful
- Use bullet points for lists

**Structure:**
- Start with overview/summary
- Include Table of Contents for long documents
- Use consistent headings across docs
- End with "Last Updated" date

### Example Documentation Structure

```markdown
# Document Title

Brief description of what this document covers.

## Table of Contents

1. [Section One](#section-one)
2. [Section Two](#section-two)

## Section One

Content here...

### Subsection

More details...

## Section Two

More content...

---

**Last Updated:** YYYY-MM-DD
```

## Testing Requirements

### Before Submitting Changes

**For policy file changes:**
```bash
# 1. Validate syntax
cat .well-known/mta-sts.txt

# Check for:
# - No extra whitespace
# - No special characters
# - Proper line endings
# - All required fields present

# 2. Verify format
grep -E "^version: STSv1$" .well-known/mta-sts.txt
grep -E "^mode: (testing|enforce|none)$" .well-known/mta-sts.txt
grep -E "^mx: " .well-known/mta-sts.txt
grep -E "^max_age: [0-9]+$" .well-known/mta-sts.txt

# 3. Test (after deployment)
curl https://mta-sts.livalittle.com/.well-known/mta-sts.txt
```

**For documentation changes:**
- Check spelling and grammar
- Verify all links work
- Test code examples (if any)
- Ensure consistent formatting

**For scripts/tools:**
- Test on multiple platforms (if applicable)
- Include usage examples
- Add error handling
- Document dependencies

## Commit Guidelines

### Commit Message Format

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Formatting (no code change)
- `refactor`: Code restructuring
- `test`: Adding tests
- `chore`: Maintenance tasks

**Scope (optional):**
- `policy`: Policy file changes
- `dns`: DNS configuration
- `docs`: Documentation
- `test`: Testing

**Examples:**

```
fix(policy): Remove backslashes from policy file

The policy file contained literal backslashes that broke RFC 8461
compliance. Changed to use proper newlines.

Fixes #123
```

```
docs(setup): Add Cloudflare configuration example

Added step-by-step instructions for configuring DNS records in
Cloudflare, including screenshots and common pitfalls.
```

```
feat(monitoring): Add health check script

Created mta-sts-monitor.sh for automated monitoring of DNS and
policy file accessibility. Can be run via cron.
```

### Commit Best Practices

- Write clear, descriptive messages
- Keep subject line under 50 characters
- Use imperative mood ("Add feature" not "Added feature")
- Wrap body at 72 characters
- Reference issues/PRs in footer

## Pull Request Process

### PR Template

**Title:** Clear description of changes

**Description:**
```
## Changes Made
- Change 1
- Change 2
- Change 3

## Motivation
Why are these changes needed?

## Testing
How were these changes tested?

## Related Issues
Fixes #123
Relates to #456

## Checklist
- [ ] Code follows style guidelines
- [ ] Documentation updated
- [ ] Tests pass
- [ ] CHANGELOG.md updated (if applicable)
- [ ] No security vulnerabilities introduced

## Screenshots (if applicable)
[Add screenshots showing before/after]

## Additional Notes
Any other context or information
```

### Review Process

1. **Automated checks** run (if configured)
2. **Manual review** by maintainers
3. **Feedback provided** via comments
4. **Revisions requested** (if needed)
5. **Approval** and merge

### After Your PR is Merged

- Delete your branch (if no longer needed)
- Update your fork
- Consider contributing more!

## Security Considerations

### Security-Sensitive Changes

**Extra care required for:**
- Policy mode changes (testing ↔ enforce)
- MX pattern modifications
- max_age adjustments
- DNS record updates

**Required for security changes:**
1. **Detailed justification** in PR description
2. **Impact assessment** (what breaks if this fails?)
3. **Rollback plan** documented
4. **Extended testing** period
5. **Gradual rollout** if possible

### Security Issue Reporting

**For security vulnerabilities:**

**DO NOT** open a public issue.

Instead:
1. Email security contact (if provided)
2. Provide detailed description
3. Include proof of concept (if safe)
4. Suggest mitigation (if possible)
5. Allow time for fix before disclosure

## Recognition

Contributors will be recognized in:
- Git commit history
- CHANGELOG.md (for significant contributions)
- README.md (for major features)

## Questions?

If you have questions about contributing:
- Check existing documentation
- Review closed issues/PRs
- Open a new issue with `question` label
- Reach out to maintainers

## License

By contributing, you agree that your contributions will be licensed under the MIT License (see [LICENSE](LICENSE)).

## Thank You!

Every contribution, no matter how small, helps improve email security for livalittle.com. Thank you for taking the time to contribute!

---

## Quick Start for Contributors

**First-time contributors:**

```bash
# 1. Fork and clone
git clone https://github.com/YOUR-USERNAME/livalittle.git
cd livalittle

# 2. Create branch
git checkout -b docs/fix-typo

# 3. Make changes
# Edit files...

# 4. Test
cat .well-known/mta-sts.txt  # Verify no errors

# 5. Commit
git add .
git commit -m "docs: Fix typo in SETUP.md"

# 6. Push
git push origin docs/fix-typo

# 7. Create PR on GitHub
```

**Returning contributors:**

```bash
# Update your fork
git checkout main
git pull upstream main
git push origin main

# Create new branch
git checkout -b feature/new-feature

# Make changes, commit, push, PR
```

---

**Last Updated:** 2025-02-06
**Repository:** https://github.com/jdgonzalesdp/livalittle
**Maintainers:** Listed in repository settings
