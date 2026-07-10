# Development - Contribution Guide

## Overview

This guide outlines the contribution process, core rules, and development practices for the `actions-gh-release` action. Following these guidelines ensures maintainability and consistency.

## Core Rules

### 1. Prefer Narrow Behavior Fixes

**Do**: Small, focused changes that address specific issues
**Don't**: Large refactors or architectural changes without discussion

**Example**: Fixing a specific glob pattern issue is better than rewriting all file handling.

### 2. Reproduce Current Behavior First

**Always**: Reproduce the existing behavior on `main` before making changes
**Verify**: Changes don't break existing functionality unintentionally

**Process**:

1. Check out `main` branch
2. Run tests: `pnpm run test`
3. Understand current behavior
4. Make changes
5. Verify behavior preserved

### 3. Separate GitHub Platform Behavior

**Distinguish**: Action behavior vs. GitHub API/platform behavior
**Document**: When behavior is imposed by GitHub, not the action

**Example**: Asset name normalization is GitHub's behavior, documented but not changed.

### 4. Be Careful with Parsing Changes

**Special attention**: `files` input parsing, path handling, Windows compatibility
**Test thoroughly**: Cross-platform testing for file operations

**Risk areas**:

- Glob pattern parsing
- Path separator handling (`/` vs `\`)
- Brace expansion (`{a,b}` patterns)
- Special character escaping

## Contribution Process

### 1. Before Starting

- **Check issues**: Avoid duplicating work
- **Discuss changes**: For significant changes, open discussion first
- **Understand scope**: Small, focused changes preferred

### 2. Development Workflow

```bash
# 1. Fork and clone
git clone https://github.com/your-username/actions-gh-release.git
cd actions-gh-release

# 2. Create feature branch
git checkout -b fix/description

# 3. Install dependencies
pnpm install

# 4. Make changes
# Edit src/*.ts, tests/*.test.ts

# 5. Run validation
pnpm run validate

# 6. Fix issues
pnpm run lint:fix
pnpm run format

# 7. Build and test
pnpm run build
pnpm run test

# 8. Commit changes
git add .
git commit -m "fix: description of changes"

# 9. Push and create PR
git push origin fix/description
```

### 3. Pull Request Requirements

**Required checks**:

- [ ] All tests pass (`pnpm run test`)
- [ ] Code is formatted (`pnpm run check-format`)
- [ ] Linting passes (`pnpm run lint`)
- [ ] Type checking passes (`pnpm run typecheck`)
- [ ] Bundle builds successfully (`pnpm run build`)
- [ ] Bundle is fresh (matches source changes)

**PR description**:

- What changes were made
- Why they were needed
- How they were tested
- Any breaking changes

## Contract Sync

When behavior changes, update all related artifacts:

### 1. Documentation

- **`README.md`**: User-facing documentation
- **`action.yml`**: Action metadata and inputs
- **This OpenWiki**: Internal documentation

### 2. Tests

- **Add new tests**: For new features
- **Update existing tests**: For behavior changes
- **Test edge cases**: Error scenarios, race conditions

### 3. Build Artifacts

```bash
# Regenerate bundle
pnpm run build

# Verify bundle
node --check dist/index.js
```

### 4. Verification Checklist

```bash
# Run full validation
pnpm run validate

# Manual verification
git diff --no-ext-diff dist/index.js  # Should show changes
```

## Code Organization

### Module Responsibilities

Keep behavior-specific logic in appropriate modules:

#### `src/github.ts`

- GitHub API interactions
- Release semantics
- Race condition handling
- Error recovery logic

#### `src/util.ts`

- Configuration parsing
- File path operations
- Input validation
- Utility functions

#### `src/index.ts`

- Orchestration only
- Main workflow coordination
- Error handling setup
- Output setting

**Avoid**: Adding feature-specific code to `src/index.ts`. Use `src/github.ts` or `src/util.ts` instead.

### Interface Design

**Use interfaces** for testability:

```typescript
// Good: Interface allows mocking
interface Releaser {
  release(config: Config): Promise<ReleaseResult>;
}

// Use in production
class GitHubReleaser implements Releaser {
  // Implementation
}
```

## Testing Guidelines

### Test Categories

#### 1. Unit Tests

- **Location**: `tests/util.test.ts`
- **Scope**: Individual functions
- **Examples**: `parseConfig()`, `paths()`, `normalizeTagName()`

#### 2. Integration Tests

- **Location**: `tests/github.test.ts`
- **Scope**: GitHub API interactions
- **Examples**: Release creation, asset upload, error handling

#### 3. Error Scenario Tests

- **Focus**: Failure cases
- **Examples**: Rate limiting, permission errors, race conditions
- **Mocking**: Mock GitHub API responses

### Test Patterns

#### Mocking GitHub API

```typescript
// Use consistent mocking patterns
const mockOctokit = {
  rest: {
    repos: {
      createRelease: vi.fn(),
      listReleases: vi.fn(),
      // ... other methods
    },
  },
};
```

#### Testing Error Handling

```typescript
// Test retry logic
mockOctokit.rest.repos.createRelease
  .mockRejectedValueOnce({ status: 429 })
  .mockResolvedValueOnce({ data: { id: 1 } });

await expect(release(config)).resolves.toBeDefined();
expect(mockOctokit.rest.repos.createRelease).toHaveBeenCalledTimes(2);
```

#### Testing Race Conditions

```typescript
// Test duplicate release handling
mockOctokit.rest.repos.createRelease
  .mockRejectedValueOnce({
    status: 422,
    response: { data: { errors: [{ code: 'already_exists' }] } },
  })
  .mockResolvedValueOnce({ data: { id: 1 } });
```

## Code Review Guidelines

### What to Look For

1. **Behavior changes**: Intended and documented
2. **Error handling**: Proper retry and recovery
3. **Test coverage**: New code has tests
4. **Documentation**: Updated where needed
5. **Performance**: No unnecessary overhead
6. **Security**: No token exposure or vulnerabilities

### Review Checklist

- [ ] Changes are focused and minimal
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] No breaking changes unintentionally
- [ ] Error handling appropriate
- [ ] Cross-platform considerations
- [ ] Bundle regenerated if needed

## Common Pitfalls

### 1. Breaking Windows Compatibility

**Issue**: Assuming `/` path separators
**Fix**: Use `path` module functions
**Example**:

```typescript
// Bad
const fullPath = `${workingDirectory}/${filePath}`;

// Good
import { join } from 'path';
const fullPath = join(workingDirectory, filePath);
```

### 2. Overlooking Race Conditions

**Issue**: Assuming single workflow execution
**Fix**: Design for concurrency
**Example**: Check for existing releases before creating

### 3. Inadequate Error Testing

**Issue**: Only testing happy path
**Fix**: Test failure scenarios
**Example**: Mock API failures, network errors

### 4. Forgetting Contract Sync

**Issue**: Changing behavior without updating docs/tests
**Fix**: Update all related artifacts
**Checklist**: README, action.yml, tests, bundle

## Development Environment

### Recommended Tools

- **Editor**: VS Code with TypeScript support
- **Terminal**: Shell with Git
- **Node.js**: Version in `.tool-versions`
- **Git**: Latest stable

### Editor Configuration

**VS Code Settings**:

```json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll": "always"
  },
  "typescript.preferences.importModuleSpecifier": "relative"
}
```

**Extensions**:

- ESLint/oxlint
- Prettier/oxfmt
- GitLens
- GitHub Actions

## Version Management

### Semantic Versioning

- **Major**: Breaking changes
- **Minor**: New features (backward compatible)
- **Patch**: Bug fixes

### Breaking Changes

**Require**:

1. Major version bump
2. Clear documentation
3. Migration guide if needed
4. Deprecation period if possible

**Examples of breaking changes**:

- Removing an input
- Changing default behavior
- Requiring new permissions

## Communication

### Issue Reporting

**Include**:

1. Steps to reproduce
2. Expected behavior
3. Actual behavior
4. Workflow examples
5. Error messages

**Template**:

````markdown
## Description

Brief description of the issue

## Steps to Reproduce

1. Use this workflow...
2. Observe...

## Expected Behavior

What should happen

## Actual Behavior

What actually happens

## Workflow Example

```yaml
# Minimal reproducing workflow
```
````

## Environment

- Action version: v1.0.0
- Runner: ubuntu-latest
- GitHub: (if relevant)

````

### Discussion Before Implementation
**When to discuss**:
- Significant behavior changes
- New feature proposals
- Breaking changes
- Architectural changes

**Where to discuss**:
- GitHub Issues
- Pull Request description
- (If available) Discussions tab

## Maintenance

### Dependency Updates
```bash
# Check for updates
pnpm run deps

# Update dependencies
pnpm run deps:update

# Test after updates
pnpm run validate
````

### Regular Maintenance Tasks

1. **Update dependencies**: Regular `pnpm run deps:update`
2. **Review open issues**: Triage and prioritize
3. **Check test coverage**: Maintain >80% coverage
4. **Verify documentation**: Keep current
5. **Monitor bundle size**: Watch for growth

## Getting Help

### Resources

1. **Existing documentation**: README.md, this OpenWiki
2. **Source code**: Well-documented TypeScript
3. **Test suite**: Examples of usage patterns
4. **Git history**: Previous changes and patterns

### Questions to Ask

1. Is this a GitHub platform limitation?
2. Does this break existing workflows?
3. Are there race conditions to consider?
4. Is Windows compatibility affected?
5. Are tests needed for this change?

## Related Documentation

- [Building & Testing](building-testing.md) - Development workflow
- [Release Process](release-process.md) - Release management
- [Architecture Core Concepts](../architecture/core-concepts.md) - Code structure
- [Usage Examples](../usage/examples.md) - Action usage patterns
