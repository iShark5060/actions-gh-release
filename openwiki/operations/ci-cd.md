# Operations - CI/CD

## Overview

This document describes the Continuous Integration and Continuous Deployment (CI/CD) workflows for the `actions-gh-release` action, including validation, testing, and release automation.

## CI Workflow (`.github/workflows/ci.yml`)

### Purpose

Ensure code quality and prevent regressions on every change to `main` branch or pull request.

### Triggers

```yaml
on:
  workflow_dispatch: # Manual trigger
  push:
    branches: [main] # Push to main
  pull_request: # Any pull request
```

### Concurrency Control

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true
```

**Prevents**: Multiple runs for same branch/PR
**Benefit**: Saves CI resources, avoids conflicts

### Job: `validate`

**Runs on**: `ubuntu-latest`
**Timeout**: 10 minutes
**Permissions**: Read-only (no writes needed)

#### Steps

##### 1. Checkout

```yaml
- uses: actions/checkout@v7
  with:
    persist-credentials: false # Security: don't persist token
```

##### 2. Setup pnpm

```yaml
- uses: pnpm/action-setup@v6
```

**Ensures**: pnpm ≥11.10.0 available

##### 3. Setup Node.js

```yaml
- uses: actions/setup-node@v6
  with:
    node-version: ${{ env.NODE_VERSION }} # 24
    cache: pnpm
```

**Cache**: pnpm dependencies for faster runs

##### 4. Install Dependencies

```yaml
- run: pnpm install --frozen-lockfile
```

**`--frozen-lockfile`**: Ensures lockfile is up-to-date

##### 5. Validate

```yaml
- run: pnpm run validate
```

**Runs**: `run-quality-checks.mjs` script
**Includes**: Formatting, linting, type checking, tests

##### 6. Build Action

```yaml
- run: pnpm run build
```

**Produces**: `dist/index.js` (production bundle)

##### 7. Verify Action Bundle

```yaml
- run: node --check dist/index.js
```

**Checks**: Bundle syntax is valid JavaScript

##### 8. Check Dist Freshness

```yaml
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

**Critical**: Ensures compiled bundle matches source changes
**Failure**: Blocks merge if dist is stale

##### 9. PR Feedback (Conditional)

```yaml
- name: Comment on PR if dist is stale
  if: github.event_name == 'pull_request' && failure()
  uses: actions/github-script@v7
  with:
    script: |
      // Posts helpful comment about regenerating bundle
```

**Only on**: Pull requests with stale dist
**Purpose**: Guide contributors to fix issue

## Release Workflow (`.github/workflows/release.yml`)

### Purpose

Automate release publication with supply chain security.

### Trigger

```yaml
on:
  release:
    types: [published]
```

**When**: Release is published via GitHub UI or API

### Job: `release`

**Runs on**: `ubuntu-latest`
**Permissions**: Write access needed for attestations

#### Steps

##### 1. Checkout Release

```yaml
- uses: actions/checkout@v7
  with:
    ref: ${{ github.event.release.tag_name }}
```

**Checks out**: The release tag (not main branch)

##### 2. Setup Environment

```yaml
- uses: pnpm/action-setup@v6
- uses: actions/setup-node@v6
  with:
    node-version: ${{ env.NODE_VERSION }}
    cache: pnpm
- run: pnpm install --frozen-lockfile
```

**Same as CI**: Consistent environment

##### 3. Build Action

```yaml
- run: pnpm run build
```

**Produces**: Release bundle

##### 4. Attest Artifacts

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

**Supply Chain Security**: Creates signed attestations
**Proves**: Artifacts came from this repository/version
**Subjects**: Action metadata and compiled bundle

##### 5. Publish to Release Tag

```yaml
- name: Publish to release tag
  uses: iShark5060/actions-build-and-tag@v1
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
```

**Action**: Custom action for publishing
**Purpose**: Push built artifacts to release tag

##### 6. Create Source Tag

```yaml
# Implicit: creates {tag}-src tag for provenance
```

**Traceability**: Links release to source commit

## Quality Checks (`run-quality-checks.mjs`)

### Purpose

Orchestrate all quality checks in proper order.

### Implementation

**File**: `/run-quality-checks.mjs`
**Type**: ES Module JavaScript
**Dependencies**: Built-in Node.js modules

### Check Sequence

1. **Formatting**: `pnpm run check-format`
2. **Linting**: `pnpm run lint`
3. **Type Checking**: `pnpm run typecheck`
4. **Tests**: `pnpm run test`

### Error Handling

- **Early exit**: First failure stops script
- **Clear messages**: Which check failed
- **Exit codes**: Non-zero on failure

### Runtime Preflight

**File**: `scripts/runtime-preflight.mjs`
**Checks**:

- Node.js version ≥24
- pnpm version ≥11.10.0
- Required tools available

## Dist Bundle Management

### Policy

**Bundle location**: `dist/index.js`
**Checked into**:

- Release tags (always)
- `main` branch (for CI freshness checks)

**Users must**: Reference published version tags, not `@main`

### Freshness Check Rationale

**Problem**: Forgetting to regenerate bundle after source changes
**Solution**: CI fails if bundle doesn't match source
**Benefit**: Prevents version mismatches

### Bundle Generation

```bash
# Production build
pnpm run build

# Development build (with sourcemaps)
pnpm run build-debug
```

## Configuration Files

### TypeScript (`tsconfig.json`)

```json
{
  "compilerOptions": {
    "target": "ES2024",
    "module": "ES2022",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  }
}
```

### Linting (`.oxlintrc.json`)

**Linter**: Oxlint with TypeScript support
**Rules**: Performance-focused, security-aware

### Formatting (`.oxfmtrc.json`)

**Formatter**: Oxfmt
**Settings**: 100-120 char width, single quotes

### Dependabot (`.github/dependabot.yml`)

**Updates**: Daily for npm and GitHub Actions
**Grouping**: Efficient update management
**Cooldown**: 1 day between updates

## Release Configuration (`.github/release.yml`)

### Purpose

Configure automatic changelog generation.

### Categories

```yaml
changelog:
  categories:
    - title: Breaking Changes
      labels:
        - breaking
    - title: Features
      labels:
        - enhancement
    - title: Bug Fixes
      labels:
        - bug
    - title: Other Changes
      labels:
        - '*'
```

### Excluded Labels

```yaml
exclude:
  labels:
    - ignore-for-release
    - github-actions
```

## Performance Optimizations

### Caching

```yaml
cache: pnpm # Node.js setup step
```

**Benefit**: Faster dependency installation
**Key**: pnpm lockfile ensures consistency

### Concurrency

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true
```

**Prevents**: Wasted CI minutes on obsolete runs

### Timeouts

```yaml
timeout-minutes: 10 # Validation job
```

**Prevents**: Hung jobs consuming resources

## Security Considerations

### Permission Model

**CI job**: Read-only (`contents: read`)
**Release job**: Write for attestations
**Principle**: Least privilege

### Token Security

```yaml
persist-credentials: false # Checkout step
```

**Prevents**: Accidental token persistence

### Supply Chain Security

**Attestations**: Prove artifact provenance
**Signed releases**: GitHub signed tags
**Dependency scanning**: Regular updates

## Monitoring and Metrics

### Workflow Success Rate

**Monitor**: CI pass/fail rate
**Alert**: On frequent failures
**Goal**: >95% success rate

### Build Times

**Typical**: 2-5 minutes for CI
**Goal**: Keep under 10 minutes
**Optimization**: Caching, parallel steps

### Bundle Size

**Current**: ~791KB (minified)
**Monitor**: Growth over time
**Goal**: Reasonable growth with features

## Troubleshooting CI/CD

### Common Issues

#### 1. CI Fails: Dist is Stale

**Symptoms**: "dist/index.js is stale" error
**Fix**:

```bash
pnpm run build
git add dist/index.js
git commit -m "chore: regenerate bundle"
```

#### 2. CI Fails: Dependency Issues

**Symptoms**: `pnpm install` fails
**Check**:

- Lockfile consistency
- Network connectivity
- pnpm version compatibility

#### 3. Release Workflow Fails

**Symptoms**: Release not published
**Check**:

- Token permissions
- Repository settings
- Attestation service availability

#### 4. Tests Flaky

**Symptoms**: Intermittent test failures
**Investigate**:

- Timing issues
- Mock state leakage
- Resource constraints

### Debugging Workflows

```yaml
# Enable step debugging
env:
  ACTIONS_STEP_DEBUG: true
  ACTIONS_RUNNER_DEBUG: true
```

**Add to workflow**: For detailed logging
**Remove after debugging**: Security consideration

## Maintenance Tasks

### Regular Updates

1. **Dependencies**: `pnpm run deps:update` weekly
2. **Node.js**: Update `.tool-versions` when needed
3. **Actions**: Update action versions in workflows
4. **Configuration**: Review CI/CD config quarterly

### Performance Review

1. **Build times**: Monitor for degradation
2. **Bundle size**: Check growth trends
3. **Test coverage**: Maintain >80%
4. **CI costs**: Monitor GitHub Actions minutes

### Security Scanning

1. **Dependencies**: Regular vulnerability scans
2. **Code scanning**: GitHub CodeQL if enabled
3. **Secret scanning**: Monitor for exposed secrets
4. **Permission review**: Audit workflow permissions

## Related Documentation

- [Release Process](../development/release-process.md) - Release management
- [Security Operations](security.md) - Security practices
- [Building & Testing](../development/building-testing.md) - Development workflow
