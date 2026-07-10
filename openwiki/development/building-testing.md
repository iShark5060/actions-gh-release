# Development - Building & Testing

## Overview

This document covers the development workflow for the `actions-gh-release` action, including building, testing, and quality assurance processes.

## Development Environment

### Prerequisites

- **Node.js**: ≥24 (check `.tool-versions`)
- **Package Manager**: pnpm ≥11.10.0
- **Git**: Standard version control

### Initial Setup

```bash
# Clone the repository
git clone https://github.com/iShark5060/actions-gh-release.git
cd actions-gh-release

# Install dependencies
pnpm install

# Verify setup
pnpm run validate
```

## Build Process

### Production Build

```bash
pnpm run build
```

**Output**: `dist/index.js` (minified bundle)

**Build Configuration** (`package.json`):

```json
"build": "esbuild src/index.ts --bundle --platform=node --format=cjs --target=node24 --outfile=dist/index.js --minify"
```

### Development Build

```bash
pnpm run build-debug
```

**Output**: `dist/index.js` (with sourcemaps and preserved names)

**Features**:

- Source maps for debugging
- Function names preserved
- No minification

### Build Configuration Details

- **Target**: Node.js 24 (ES2024 features)
- **Format**: CommonJS (CJS) for GitHub Actions compatibility
- **Bundling**: All dependencies included
- **Minification**: Enabled for production

## Testing

### Test Suite

```bash
# Run all tests
pnpm run test

# Run tests with coverage
pnpm run test:coverage
```

### Test Structure

```
tests/
├── github.test.ts    # 32 tests - GitHub API integration
├── util.test.ts      # 47 tests - Utility functions
├── release.txt       # Test data
└── data/
    └── foo/
        └── bar.txt   # Test data
```

### Test Framework

- **Runner**: Vitest 4.1.10
- **Coverage**: V8 coverage reporter
- **Mocking**: Built-in mocking capabilities
- **Assertions**: Expect API

### Running Specific Tests

```bash
# Run a specific test file
npx vitest run tests/util.test.ts

# Run tests matching a pattern
npx vitest run -t "parseConfig"

# Watch mode (development)
npx vitest
```

### Test Coverage

Current coverage (from `pnpm run test:coverage`):

- **Statements**: 83.87%
- **Branches**: 80.55%
- **Functions**: 84.28%
- **Lines**: 83.87%

**File-specific coverage**:

- `util.ts`: 96.34% statements, 93.42% branches
- `github.ts`: 80.34% statements, 75% branches

## Quality Assurance

### Validation Script

```bash
pnpm run validate
```

Runs the comprehensive validation script `run-quality-checks.mjs` which executes:

1. **Formatting check**: `pnpm run check-format`
2. **Linting**: `pnpm run lint`
3. **Type checking**: `pnpm run typecheck`
4. **Tests**: `pnpm run test`

### Individual Quality Checks

#### 1. Linting

```bash
pnpm run lint          # Check for issues
pnpm run lint:fix      # Auto-fix issues
```

**Linter**: Oxlint with configuration in `.oxlintrc.json`
**Rules**: TypeScript-aware, Unicorn plugin, performance optimizations

#### 2. Formatting

```bash
pnpm run format        # Format code
pnpm run check-format  # Check formatting
```

**Formatter**: Oxfmt with configuration in `.oxfmtrc.json`
**Settings**: 100-120 char width, single quotes, trailing commas

#### 3. Type Checking

```bash
pnpm run typecheck
```

**Configuration**: `tsconfig.json` with strict mode
**Target**: ES2024, module resolution Node

## Development Workflow

### 1. Making Changes

```bash
# Create feature branch
git checkout -b feature/description

# Make code changes
# Edit src/*.ts files

# Run validation
pnpm run validate

# Fix any issues
pnpm run lint:fix
pnpm run format

# Build and test
pnpm run build
pnpm run test
```

### 2. Testing Changes

**Test new features**:

1. Add tests to `tests/*.test.ts`
2. Run tests: `pnpm run test`
3. Verify coverage: `pnpm run test:coverage`

**Test error scenarios**:

- Mock GitHub API errors
- Test retry logic
- Verify error messages

**Test file patterns**:

- Add test files to `tests/data/`
- Test glob pattern matching
- Verify path normalization

### 3. Verifying Build

```bash
# Build production bundle
pnpm run build

# Verify bundle works
node --check dist/index.js

# Test bundle execution
node -e "console.log('Bundle valid')" dist/index.js
```

## Debugging

### Debug Build

```bash
pnpm run build-debug
```

Creates debuggable bundle with source maps.

### Test Debugging

```bash
# Run tests with debug output
npx vitest run --reporter=verbose

# Debug specific test
npx vitest run -t "test name" --reporter=verbose
```

### Runtime Debugging

1. **Local testing**: Create test workflow in separate repo
2. **Action-tester**: Use `act` for local GitHub Actions testing
3. **Console output**: Add `console.log` for debugging (remove before commit)

## Common Development Tasks

### Adding New Input

1. **Update `action.yml`**: Add input definition
2. **Update `src/util.ts`**: Add to `Config` interface and `parseConfig()`
3. **Update tests**: Add test cases
4. **Update documentation**: Add to configuration docs

### Modifying GitHub API Interaction

1. **Update `src/github.ts`**: Modify `GitHubReleaser` methods
2. **Add error handling**: Consider retry logic
3. **Update tests**: Mock API responses
4. **Test race conditions**: Concurrent execution scenarios

### Changing File Handling

1. **Update `src/util.ts`**: Modify `paths()` or `parseInputFiles()`
2. **Consider Windows compatibility**: Path separators
3. **Update tests**: Add cross-platform tests
4. **Verify glob patterns**: Test edge cases

## Performance Testing

### Bundle Size

```bash
# Check bundle size
ls -lh dist/index.js

# Production: ~791KB (minified)
# Debug: ~1.2MB (with sourcemaps)
```

### Test Performance

```bash
# Time test execution
time pnpm run test

# Check memory usage
node --max-old-space-size=512 node_modules/.bin/vitest run
```

### API Rate Limit Testing

Test retry logic by mocking rate limit responses:

```typescript
// In tests
mockOctokit.rest.repos.createRelease.mockRejectedValueOnce({
  status: 429,
  response: { headers: { 'x-ratelimit-reset': '123' } },
});
```

## CI Integration

### Local CI Simulation

```bash
# Run full validation (like CI)
pnpm run validate

# Build and verify (like CI)
pnpm run build
node --check dist/index.js

# Check dist freshness
git diff --no-ext-diff dist/index.js
```

### Pre-commit Hooks

Consider adding Git hooks for:

- Formatting check
- Linting
- Test run
- Build verification

## Troubleshooting Development Issues

### Common Issues

#### 1. TypeScript Errors

```bash
# Clear and rebuild
rm -rf node_modules dist
pnpm install
pnpm run typecheck
```

#### 2. Test Failures

- **Mock issues**: Reset mocks between tests
- **Timing issues**: Use `vi.useFakeTimers()`
- **Async issues**: Ensure proper async/await handling

#### 3. Build Failures

```bash
# Check esbuild version
npx esbuild --version

# Clean rebuild
rm dist/index.js
pnpm run build
```

#### 4. Validation Failures

Run individual checks to isolate:

```bash
pnpm run check-format  # Formatting
pnpm run lint          # Linting
pnpm run typecheck     # TypeScript
pnpm run test          # Tests
```

## Best Practices

### 1. Code Quality

- **Follow existing patterns**: Consistency matters
- **Add tests**: New features need tests
- **Document changes**: Update relevant documentation
- **Keep bundle small**: Monitor dependencies

### 2. Testing

- **Test error paths**: Don't just test happy paths
- **Mock external APIs**: Don't call real GitHub API
- **Test race conditions**: Concurrent execution scenarios
- **Cross-platform**: Consider Windows/Unix differences

### 3. Performance

- **Stream files**: Don't buffer large files
- **Limit retries**: Avoid infinite loops
- **Parallel uploads**: Default behavior for speed
- **Cache appropriately**: Consider build caching

### 4. Maintenance

- **Update dependencies**: Regular `pnpm deps:update`
- **Monitor bundle size**: Keep eye on growth
- **Review error handling**: Ensure robustness
- **Backward compatibility**: Consider existing users

## Related Documentation

- [Contribution Guide](contribution-guide.md) - Core rules and process
- [Release Process](release-process.md) - CI/CD and release workflow
- [Architecture Core Concepts](../architecture/core-concepts.md) - Code structure
