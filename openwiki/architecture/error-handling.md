# Architecture - Error Handling

## Overview

The `actions-gh-release` action implements comprehensive error handling designed for reliability in GitHub Actions environments. This includes retry logic, race condition detection, and graceful degradation.

## Error Categories

### 1. User Configuration Errors

**Cause**: Invalid inputs or missing requirements
**Examples**:

- Missing tag for non-draft release
- Invalid glob patterns
- Missing required permissions

**Handling Strategy**:

- Immediate failure with clear error messages
- No retry attempts
- Exit code 1

### 2. Transient Network Errors

**Cause**: Network issues, rate limiting, temporary GitHub API issues
**Examples**:

- Rate limiting (429)
- Network timeouts
- 5xx server errors

**Handling Strategy**:

- Exponential backoff retry
- Built-in Octokit plugin support
- Maximum retry limits

### 3. Race Conditions

**Cause**: Concurrent workflow executions
**Examples**:

- Duplicate release creation (422)
- Asset deleted during upload (404)
- Concurrent modification conflicts

**Handling Strategy**:

- Detection via error codes
- Recovery through reconciliation
- Retry with find-then-act pattern

### 4. Permission/Authorization Errors

**Cause**: Insufficient token permissions
**Examples**:

- Repository access denied (403)
- Invalid token (401)
- Repository not found (404)

**Handling Strategy**:

- Immediate failure
- Clear permission guidance
- No retry attempts

## Retry Implementation

### Octokit Plugin Configuration (`src/index.ts:34-50`)

```typescript
const gh = getOctokit(config.github_token, {
  throttle: {
    onRateLimit: (retryAfter, options) => {
      console.warn(`Request quota exhausted for request ${options.method} ${options.url}`);
      if (options.request.retryCount === 0) {
        console.log(`Retrying after ${retryAfter} seconds!`);
        return true; // Retry once
      }
    },
    onAbuseLimit: (retryAfter, options) => {
      console.warn(`Abuse detected for request ${options.method} ${options.url}`);
    },
  },
});
```

### Custom Retry Logic (`src/github.ts`)

#### Release Creation Retry (lines 552-590)

```typescript
const retryOptions = {
  retries: 3,
  factor: 2,
  minTimeout: 500,
  maxTimeout: 3000,
};

await retry(async (bail) => {
  try {
    return await this.createRelease(params);
  } catch (error) {
    if (isDuplicateReleaseError(error)) {
      // Race condition: another workflow created it
      throw error; // Bail will be called by retry
    }
    if (isPermissionError(error)) {
      bail(error); // Don't retry permission errors
    }
    throw error; // Retry other errors
  }
}, retryOptions);
```

## Race Condition Detection & Recovery

### 1. Duplicate Release Creation

**Detection**: 422 error with "already_exists" message
**Recovery**:

1. Catch the 422 error
2. List releases to find the duplicate
3. Return existing release instead
4. Continue with asset uploads

**Source**: `src/github.ts:552-615`

### 2. Concurrent Asset Modification

**Detection**: 404 when updating existing asset
**Recovery**:

1. Asset was deleted by another workflow
2. Treat as new upload instead of update
3. Continue with upload

**Source**: `src/github.ts:424-429`

```typescript
if (error.status === 404 && overwrite) {
  // Asset disappeared between list and upload
  // Upload as new asset
  return this.uploadAsset(release, filePath, assetName);
}
```

### 3. Immutable Release Repositories

**Detection**: Repository rules block certain operations
**Recovery**:

1. Create as draft instead of published
2. Upload assets to draft
3. Finalize only if allowed by rules

**Source**: `src/github.ts:626-684`

## Error Helper Functions

### `isDuplicateReleaseError()` (`src/github.ts:918-925`)

```typescript
function isDuplicateReleaseError(error: any): boolean {
  return (
    error.status === 422 &&
    error.response?.data?.errors?.some((e: any) => e.code === 'already_exists')
  );
}
```

### `isTagCreationBlockedError()` (`src/github.ts:945-952`)

```typescript
function isTagCreationBlockedError(error: any): boolean {
  return error.status === 403 && error.response?.data?.message?.includes('tag creation blocked');
}
```

### `isImmutableReleaseAssetUploadFailure()` (`src/github.ts:933-942`)

```typescript
function isImmutableReleaseAssetUploadFailure(error: any): boolean {
  return (
    error.status === 422 &&
    error.response?.data?.errors?.some((e: any) => e.code === 'invalid' && e.field === 'draft')
  );
}
```

## Error Logging & User Communication

### Console Output Patterns

- **`⚠️` prefix**: User configuration errors
- **`🤔` prefix**: Warnings (unmatched patterns)
- **`Request quota exhausted`**: Rate limiting
- **`Retrying after X seconds!`**: Retry in progress

### Error Messages

**Clear Guidance**:

```typescript
throw new Error(`⚠️ GitHub Releases requires a tag`);
// vs.
throw new Error(`Tag required`);
```

**Context Included**:

```typescript
console.warn(`Pattern '${pattern}' does not match any files.`);
```

## Testing Error Scenarios

### Test Coverage (`tests/github.test.ts`)

- **Rate Limiting**: Mocked 429 responses
- **Duplicate Releases**: 422 "already_exists" errors
- **Permission Errors**: 403/401 responses
- **Network Failures**: Simulated timeouts

### Test Patterns

```typescript
// Mock rate limiting
mockOctokit.rest.repos.createRelease.mockRejectedValueOnce({
  status: 429,
  response: { headers: { 'x-ratelimit-reset': '123' } },
});

// Test retry behavior
await expect(release(config)).rejects.toThrow();
expect(mockOctokit.rest.repos.createRelease).toHaveBeenCalledTimes(2);
```

## Failure Modes & Recovery

### 1. Partial Upload Failure

**Scenario**: Some assets upload successfully, others fail
**Behavior**: Continue with successful uploads, fail action at end
**Output**: Partial success with error message

### 2. Release Created but Uploads Fail

**Scenario**: Release created successfully, but all uploads fail
**Behavior**: Release remains (draft or published), action fails
**Recovery**: Manual intervention or re-run workflow

### 3. Network Partition Mid-Operation

**Scenario**: Network fails during multi-asset upload
**Behavior**: Retry logic attempts recovery, fails if unrecoverable
**State**: Possibly partially uploaded assets

## Configuration for Error Behavior

### `fail_on_unmatched_files`

- `false` (default): Warn on unmatched patterns, continue
- `true`: Fail action on any unmatched pattern

### `overwrite_files`

- `true` (default): Delete and re-upload existing assets
- `false`: Skip existing assets, continue with others

### `preserve_order`

- `false` (default): Parallel uploads (faster, less predictable)
- `true`: Sequential uploads (slower, ordered)

## Best Practices for Error Handling

### When Extending the Action

1. **Use Existing Patterns**: Follow established error handling conventions
2. **Test Error Paths**: Add tests for new error scenarios
3. **Clear Messages**: Provide actionable error information
4. **Consider Recovery**: Design for automatic recovery where possible

### When Using the Action

1. **Monitor Logs**: Check for warning messages
2. **Set Timeouts**: GitHub Actions has default timeouts
3. **Use Retries**: Consider workflow-level retry for critical releases
4. **Check Permissions**: Ensure token has required scopes

## Related Documentation

- [Core Concepts](core-concepts.md) - Architecture overview
- [Workflow Flow](workflow-flow.md) - Execution sequence
- [GitHub API Integration](../integration/github-api.md) - API error patterns
