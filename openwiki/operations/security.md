# Operations - Security

## Overview

This document details the security practices, considerations, and implementations for the `actions-gh-release` action, covering supply chain security, token management, and vulnerability prevention.

## Supply Chain Security

### Artifact Attestation

**Implementation**: `.github/workflows/release.yml`
**Purpose**: Prove artifact provenance and integrity

```yaml
- name: Attest action.yml and dist/index.js
  uses: actions/attest@v3
  with:
    subjects: |
      action.yml
      dist/index.js
    push-to-registry: true
    predicate-type: https://github.com/actions/attestation/predicates/v0.1
```

**What's attested**:

1. `action.yml` - Action metadata
2. `dist/index.js` - Compiled bundle

**Benefits**:

- Verifies artifacts came from this repository
- Prevents tampering during distribution
- Enables downstream verification

### Source Tagging

**Pattern**: `{tag}-src` tags created during release
**Purpose**: Link release artifacts to source commit
**Traceability**: Full provenance from source to artifact

### Bundle Freshness Checks

**CI Check**: Validates `dist/index.js` matches source
**Prevents**: Using stale or mismatched bundles
**Enforcement**: CI fails if bundle is stale

## Token Security

### Token Types and Scopes

#### 1. Default Token (`github.token`)

**Characteristics**:

- Automatically provided by GitHub
- Scoped to current repository
- Expires after workflow run
- Least privilege principle

**Required permissions**:

- `contents: write` - Create releases, upload assets
- `contents: read` - Read repository contents

#### 2. Personal Access Tokens (PATs)

**When used**:

- Cross-repository releases
- Extended permissions needed
- Service accounts

**Security considerations**:

- Store in GitHub Secrets
- Regular rotation (90 days recommended)
- Principle of least privilege
- Monitor usage

#### 3. GitHub App Tokens

**Characteristics**:

- Fine-grained permissions
- Repository-specific
- Automatic rotation

### Token Handling in Code

#### Input Validation

```typescript
// src/util.ts - parseConfig
github_token: env.INPUT_TOKEN || env.GITHUB_TOKEN || '',
```

**Behavior**:

- Empty string treated as "unset"
- Falls back to `github.token`
- Explicit token overrides default

#### Security Practices

1. **Never log tokens**: Console output sanitized
2. **No persistence**: `persist-credentials: false` in checkout
3. **Input validation**: Validate token format if possible
4. **Error messages**: Don't expose token details

### Permission Configuration

#### Minimal Permissions

```yaml
jobs:
  release:
    permissions:
      contents: write # Minimum required
```

**Benefits**:

- Reduces attack surface
- Follows principle of least privilege
- Prevents accidental misuse

#### Environment-specific Permissions

```yaml
jobs:
  release:
    environment: production
    permissions:
      contents: write
      attestations: write # For attestations
```

## Dependency Security

### Dependency Management

#### Regular Updates

**Script**: `pnpm run deps:update`
**Frequency**: Weekly or as needed
**Process**:

1. Check for updates
2. Update dependencies
3. Run validation
4. Test thoroughly

#### Dependabot Configuration

**File**: `.github/dependabot.yml`
**Features**:

- Daily dependency checks
- Grouped updates for efficiency
- Cooldown periods
- Ignore rules for breaking changes

#### Security-focused Updates

```yaml
# Block potentially breaking type changes
- package-ecosystem: 'npm'
  ignore:
    - dependency-name: '@types/node'
      versions: ['>=25.0.0']
```

### Vulnerability Scanning

#### Built-in GitHub Features

1. **Dependabot alerts**: Automatic vulnerability detection
2. **Code scanning**: If CodeQL enabled
3. **Secret scanning**: Detects exposed secrets

#### Manual Checks

```bash
# Check for known vulnerabilities
pnpm audit

# Update vulnerable dependencies
pnpm update --audit
```

### Dependency Pinning

#### Action References

**Best practice**: Pin to major version

```yaml
# Good
uses: iShark5060/actions-gh-release@v1

# Better (specific)
uses: iShark5060/actions-gh-release@v1.0.1

# Risky
uses: iShark5060/actions-gh-release@main
```

#### Internal Dependencies

**Lockfile**: `pnpm-lock.yaml` ensures reproducible installs
**Frozen lockfile**: CI uses `--frozen-lockfile` to detect changes

## Input Validation and Sanitization

### File Path Security

**Risk**: Path traversal attacks
**Prevention**: `working_directory` scope limits

```typescript
// Patterns resolved within working_directory
const matches = await glob(pattern, { cwd: workingDirectory });
```

### Tag Name Validation

**Normalization**: Remove `refs/tags/` prefix
**No additional validation**: GitHub validates tag names

### Repository Name Validation

**Format**: `owner/repo`
**Validation**: Basic format check in API calls
**GitHub enforcement**: API validates repository existence

## Error Handling Security

### Safe Error Messages

**Do not expose**:

- Token values
- Internal file paths
- System details

**Example safe error**:

```typescript
throw new Error(`⚠️ GitHub Releases requires a tag`);
// Not: throw new Error(`Token ${token} invalid`);
```

### Logging Security

**Console output**:

- No sensitive data
- Generic error messages
- Actionable guidance without details

**Debug mode**: `ACTIONS_STEP_DEBUG: true` shows more details (use cautiously)

## Cross-Repository Security

### When Releasing to Other Repositories

**Requirements**:

1. Token with access to target repository
2. Explicit `repository` input
3. Proper permission scopes

**Security considerations**:

- Token stored in secrets
- Repository validation
- Permission auditing

### Repository Rules Impact

**Immutable releases**: Affects release strategy
**Tag protection**: May require specific permissions
**Branch protection**: Affects `target_commitish`

## Runtime Security

### Node.js Security

**Version**: ≥24 (security updates)
**Sandbox**: GitHub Actions runner isolation
**Memory limits**: Built-in constraints

### File System Security

**Working directory isolation**: Patterns can't escape
**File permissions**: Respect runner permissions
**Temporary files**: No sensitive data written

### Network Security

**HTTPS only**: All GitHub API calls use HTTPS
**Certificate validation**: Built into Node.js
**No external calls**: Only GitHub API endpoints

## Security Monitoring

### Activity Monitoring

**What to monitor**:

- Release activity patterns
- Token usage anomalies
- Failed authentication attempts
- Permission errors

### Alerting Considerations

**Potential alerts**:

- Multiple failed release attempts
- Unusual repository targets
- Permission escalation attempts
- Dependency vulnerability detection

### Audit Logs

**GitHub Audit Log**: Tracks repository events
**Action usage**: GitHub provides usage metrics
**Workflow runs**: History of all executions

## Incident Response

### Security Incident Types

#### 1. Exposed Token

**Response**:

1. Immediately revoke token
2. Rotate all related tokens
3. Audit recent releases
4. Monitor for misuse

#### 2. Vulnerable Dependency

**Response**:

1. Update to fixed version
2. Run security scans
3. Verify no exploitation
4. Release patch if needed

#### 3. Malicious Release

**Response**:

1. Delete malicious release
2. Revoke compromised tokens
3. Investigate source
4. Strengthen security

### Recovery Procedures

#### Token Compromise

```bash
# 1. Revoke token (GitHub UI)
# 2. Rotate secrets
# 3. Audit releases
# 4. Update workflows
```

#### Dependency Issue

```bash
# 1. Update package.json
# 2. Update lockfile
# 3. Run security scan
# 4. Release update
```

## Best Practices for Users

### 1. Token Management

```yaml
# Store tokens in secrets
env:
  RELEASE_TOKEN: ${{ secrets.RELEASE_TOKEN }}

# Use minimal permissions
permissions:
  contents: write
```

### 2. Action References

```yaml
# Pin to major version
uses: iShark5060/actions-gh-release@v1

# Or specific version
uses: iShark5060/actions-gh-release@v1.0.1
```

### 3. Input Validation

```yaml
# Validate in workflow
- name: Check files exist
  run: test -f dist/app.zip

- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    files: dist/app.zip
```

### 4. Environment Segregation

```yaml
jobs:
  test-release:
    environment: test
    # Different token, permissions

  production-release:
    environment: production
    # More restricted
```

## Development Security

### Code Review Security Checklist

- [ ] No token logging
- [ ] Input validation
- [ ] Path traversal prevention
- [ ] Error message safety
- [ ] Dependency updates
- [ ] Permission requirements

### Pre-commit Security Checks

**Consider adding**:

- Secret detection
- Dependency vulnerability scanning
- Code quality with security focus

### Security Testing

**Test scenarios**:

- Invalid token handling
- Permission denial
- Path traversal attempts
- Large file handling
- Concurrent execution

## Compliance Considerations

### GitHub Actions Security

Follows [GitHub Actions security best practices](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

### Supply Chain Levels for Software Artifacts (SLSA)

**Current level**: SLSA 1+ (with attestations)
**Potential improvements**:

- Build reproducibility
- Provenance linking
- Additional signatures

### Open Source Security

**SBOM generation**: Consider adding Software Bill of Materials
**VEX statements**: Vulnerability Exploitability eXchange

## Related Documentation

- [CI/CD Operations](ci-cd.md) - Attestation implementation
- [Permissions](../usage/permissions.md) - Token and permission details
- [GitHub API](../integration/github-api.md) - API security considerations
- [Release Process](../development/release-process.md) - Secure release workflow
