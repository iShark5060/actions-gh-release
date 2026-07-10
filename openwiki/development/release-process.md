# Development - Release Process

## Overview

This document describes the release process for the `actions-gh-release` action, including CI/CD automation, version management, and quality gates.

## Release Philosophy

### Version Strategy

- **Semantic Versioning**: Follows `MAJOR.MINOR.PATCH`
- **Backward Compatibility**: Minor versions maintain compatibility
- **Breaking Changes**: Require major version bump

### Release Channels

- **Stable**: Tagged releases (`v1.0.0`)
- **Pre-release**: Tagged with suffix (`v1.0.0-rc.1`)
- **Development**: `main` branch (unstable)

## CI/CD Pipeline

### 1. Continuous Integration (CI)

**Workflow**: `.github/workflows/ci.yml`
**Triggers**: Push to `main`, pull requests, manual dispatch

**Key Steps**:

1. **Setup**: pnpm, Node.js 24
2. **Validation**: Full quality check suite
3. **Build**: Production bundle
4. **Dist Freshness Check**: Ensure `dist/index.js` matches source
5. **PR Feedback**: Comment on stale dist bundles

### 2. Release Automation

**Workflow**: `.github/workflows/release.yml`
**Trigger**: Release published event

**Key Steps**:

1. **Checkout**: Release tag
2. **Build**: Production bundle
3. **Attestation**: Supply chain security
4. **Publish**: Push to release tag

## Release Preparation

### 1. Pre-release Checklist

```bash
# 1. Ensure main is clean
git status

# 2. Run full validation
pnpm run validate

# 3. Update version (if needed)
# Check package.json version
# Update CHANGELOG.md

# 4. Create and push tag
git tag v1.0.0
git push origin v1.0.0
```

### 2. Version Bumping

**Manual Process**:

1. Update `package.json` version
2. Update `CHANGELOG.md`
3. Commit changes
4. Create tag

**Automatic Process**:

- Use GitHub's release workflow
- Automatic changelog generation via `.github/release.yml`

### 3. Changelog Generation

**Configuration**: `.github/release.yml`
**Categories**:

- Breaking Changes
- Features
- Bug Fixes
- Other Changes

**Label Mapping**:

- `breaking`: Breaking Changes
- `enhancement`: Features
- `bug`: Bug Fixes
- Default: Other Changes

**Excluded Labels**: `ignore-for-release`, `github-actions`

## Release Workflow Details

### CI Workflow (`.github/workflows/ci.yml`)

#### Validation Phase

```yaml
- name: Validate
  run: pnpm run validate
```

**Includes**:

- Formatting check (`check-format`)
- Linting (`lint`)
- Type checking (`typecheck`)
- Tests (`test`)

#### Build Phase

```yaml
- name: Build action
  run: pnpm run build
```

**Produces**: `dist/index.js` (minified bundle)

#### Dist Freshness Check

```yaml
- name: Verify action bundle
  run: node --check dist/index.js

- name: Check dist freshness
  run: |
    if git diff --no-ext-diff --quiet dist/index.js; then
      echo "✅ dist/index.js is up to date"
    else
      echo "❌ dist/index.js is stale"
      git diff --no-ext-diff dist/index.js | head -100
      exit 1
    fi
```

**Purpose**: Ensure compiled bundle matches source changes
**Failure**: Blocks merge if dist is stale

#### PR Feedback

```yaml
- name: Comment on PR if dist is stale
  if: github.event_name == 'pull_request' && failure()
  uses: actions/github-script@v7
  with:
    script: |
      // Posts comment about stale dist bundle
```

### Release Workflow (`.github/workflows/release.yml`)

#### Attestation Phase

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

**Purpose**: Supply chain security
**Creates**: Signed attestations for artifacts

#### Publishing Phase

```yaml
- name: Publish to release tag
  uses: iShark5060/actions-build-and-tag@v1
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
```

**Action**: `iShark5060/actions-build-and-tag@v1`
**Purpose**: Push built artifacts to release tag

#### Source Tagging

```yaml
# Creates {tag}-src tag for provenance
```

## Manual Release Process

### 1. Creating a Release

```bash
# 1. Prepare release
git checkout main
git pull origin main

# 2. Update version (if not using GitHub UI)
npm version patch  # or minor/major
# Or manually update package.json

# 3. Update CHANGELOG.md
# Add entries for changes since last release

# 4. Commit changes
git add package.json CHANGELOG.md
git commit -m "chore: release v1.0.1"

# 5. Create and push tag
git tag v1.0.1
git push origin v1.0.1
```

### 2. Using GitHub UI

1. Go to Repository → Releases
2. Click "Draft a new release"
3. Select tag (create new if needed)
4. GitHub auto-generates changelog based on `.github/release.yml`
5. Publish release

### 3. Pre-release Testing

```bash
# Create pre-release tag
git tag v1.0.0-rc.1
git push origin v1.0.0-rc.1

# Test in downstream workflows
# Monitor release workflow completion

# When ready, create final release
git tag v1.0.0
git push origin v1.0.0
```

## Quality Gates

### Pre-release Checks

1. **All tests pass**: `pnpm run test`
2. **Code quality**: `pnpm run validate` passes
3. **Bundle builds**: `pnpm run build` successful
4. **Dist freshness**: Bundle matches source
5. **Documentation**: README and action.yml updated

### Post-release Verification

1. **Release workflow**: Completes successfully
2. **Attestations**: Created and published
3. **Artifacts**: Available on release tag
4. **Downstream testing**: Verify in test workflows

## Version Compatibility

### Node.js Requirements

| Version | Node.js | Status    |
| ------- | ------- | --------- |
| v1.x    | ≥24     | ✅ Active |
| v2.6.2  | 20      | ⚠️ Legacy |

**Note**: v3 requires Node.js 24 runtime. Users needing Node 20 compatibility should stay on v2.6.2.

### Breaking Change Policy

**Requires**:

1. Major version bump
2. Clear documentation
3. Migration guide if needed
4. Announcement in release notes

**Examples**:

- Removing input parameters
- Changing default behavior
- Requiring new permissions

## Dependency Management

### Regular Updates

```bash
# Check for updates
pnpm run deps

# Update dependencies
pnpm run deps:update

# Test after updates
pnpm run validate
```

### Dependabot Configuration

**File**: `.github/dependabot.yml`
**Updates**:

- Daily checks for npm packages
- Daily checks for GitHub Actions
- Grouped updates for efficiency
- Cooldown period: 1 day

### Special Handling

```yaml
# Block @types/node >=25.0.0
- package-ecosystem: 'npm'
  ignore:
    - dependency-name: '@types/node'
      versions: ['>=25.0.0']
```

## Troubleshooting Releases

### Common Issues

#### 1. Release Workflow Fails

**Symptoms**: Release workflow doesn't complete
**Check**:

- GitHub token permissions
- Repository release settings
- Network connectivity

**Fix**:

```yaml
# Ensure proper permissions
permissions:
  contents: write
  attestations: write
```

#### 2. Dist Freshness Check Fails

**Symptoms**: CI fails with "dist/index.js is stale"
**Cause**: Bundle not regenerated after source changes

**Fix**:

```bash
# Regenerate bundle
pnpm run build

# Commit changes
git add dist/index.js
git commit -m "chore: regenerate bundle"
```

#### 3. Attestation Fails

**Symptoms**: Attest step fails
**Check**:

- GitHub token has `attestations: write` permission
- Subjects exist (`action.yml`, `dist/index.js`)
- Network access to registry

#### 4. Tag Already Exists

**Symptoms**: Can't push tag
**Fix**:

```bash
# Delete local tag
git tag -d v1.0.0

# Delete remote tag
git push origin --delete v1.0.0

# Recreate and push
git tag v1.0.0
git push origin v1.0.0
```

## Release Artifacts

### What's Released

1. **Source code**: Tagged commit
2. **Action bundle**: `dist/index.js`
3. **Action metadata**: `action.yml`
4. **Attestations**: Supply chain proofs

### Bundle Management

**Policy**: Bundle (`dist/index.js`) is checked into:

- Release tags (always)
- `main` branch (for CI freshness checks)

**Users must**: Reference published version tags (e.g., `@v1`), not `@main`

## Security Considerations

### Supply Chain Security

1. **Attestations**: Prove artifact provenance
2. **Signed tags**: Git signed commits/tags
3. **Dependency scanning**: Regular updates
4. **Token management**: Least privilege principle

### Release Signing

Consider adding GPG signing:

```bash
# Create signed tag
git tag -s v1.0.0 -m "Release v1.0.0"

# Push signed tag
git push origin v1.0.0
```

## Rollback Procedure

### If Release Has Issues

1. **Identify problem**: Which version has issue
2. **Create fix**: Develop patch on `main`
3. **Release patch**: New patch version
4. **Document workaround**: In release notes

### Emergency Rollback

```bash
# Delete problematic release
# Note: Use cautiously, breaks downstream references

# Create fixed release
git tag v1.0.2
git push origin v1.0.2
```

## Monitoring and Metrics

### Release Health

1. **Workflow success rate**: Monitor CI/CD failures
2. **Downstream issues**: Watch for issues referencing releases
3. **Usage metrics**: GitHub Actions usage data

### Quality Metrics

- **Test coverage**: Maintain >80%
- **Bundle size**: Monitor growth
- **Dependency freshness**: Regular updates
- **Issue resolution time**: Responsive maintenance

## Related Documentation

- [Building & Testing](building-testing.md) - Development workflow
- [Contribution Guide](contribution-guide.md) - Development practices
- [CI/CD Operations](../operations/ci-cd.md) - Workflow details
- [Security Operations](../operations/security.md) - Security practices
