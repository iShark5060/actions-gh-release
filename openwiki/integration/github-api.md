# Integration - GitHub API

## Overview

This document details how the `actions-gh-release` action integrates with GitHub's REST API, including authentication, rate limiting, error handling, and specific API endpoints used.

## API Client Setup

### Octokit Initialization

**Source**: `src/index.ts:34-50`

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

### Plugins Used

1. **`@octokit/plugin-throttling`**: Built-in to `@actions/github`
   - Automatic rate limit handling
   - Exponential backoff
   - Abuse detection

2. **`@octokit/plugin-retry`**: Built-in to `@actions/github`
   - Retry on network errors
   - Retry on certain HTTP status codes
   - Configurable retry count

## API Endpoints Used

### 1. Releases Endpoints

#### Create a Release

**Endpoint**: `POST /repos/{owner}/{repo}/releases`
**Usage**: `src/github.ts:888-972`

```typescript
await octokit.rest.repos.createRelease({
  owner,
  repo,
  tag_name: tagName,
  name: name,
  body: body,
  draft: draft,
  prerelease: prerelease,
  target_commitish: targetCommitish,
  generate_release_notes: generateReleaseNotes,
  discussion_category_name: discussionCategoryName,
  make_latest: makeLatest,
});
```

#### Get a Release by Tag

**Endpoint**: `GET /repos/{owner}/{repo}/releases/tags/{tag}`
**Alternative**: List all releases and filter (used in `src/github.ts`)

#### List Releases

**Endpoint**: `GET /repos/{owner}/{repo}/releases`
**Usage**: Paginated listing to find existing releases

```typescript
const response = await octokit.rest.repos.listReleases({
  owner,
  repo,
  per_page: 100,
  page: 1,
});
```

#### Update a Release

**Endpoint**: `PATCH /repos/{owner}/{repo}/releases/{release_id}`
**Usage**: Finalizing drafts, updating metadata

```typescript
await octokit.rest.repos.updateRelease({
  owner,
  repo,
  release_id: releaseId,
  draft: false, // Publish draft
  // ... other fields
});
```

### 2. Release Assets Endpoints

#### Upload a Release Asset

**Endpoint**: `POST {upload_url}`
**Special**: Uses upload URL from release response
**Usage**: `src/github.ts:455-498`

```typescript
await octokit.rest.repos.uploadReleaseAsset({
  url: uploadUrl,
  data: fileStream,
  name: assetName,
  label: assetLabel,
  headers: {
    'content-type': mimeType,
    'content-length': fileStats.size,
  },
});
```

#### List Release Assets

**Endpoint**: `GET /repos/{owner}/{repo}/releases/{release_id}/assets`
**Usage**: Check for existing assets before upload

```typescript
const assets = await octokit.rest.repos.listReleaseAssets({
  owner,
  repo,
  release_id: releaseId,
  per_page: 100,
});
```

#### Delete a Release Asset

**Endpoint**: `DELETE /repos/{owner}/{repo}/releases/{release_id}/assets/{asset_id}`
**Usage**: When `overwrite_files: true`

```typescript
await octokit.rest.repos.deleteReleaseAsset({
  owner,
  repo,
  release_id: releaseId,
  asset_id: assetId,
});
```

### 3. Repository Endpoints

#### Get Repository

**Endpoint**: `GET /repos/{owner}/{repo}`
**Usage**: Validate repository access

## Rate Limiting Strategy

### GitHub Rate Limits

- **Unauthenticated**: 60 requests/hour
- **Authenticated**: 5,000 requests/hour
- **Secondary rate limits**: Abuse detection

### Action Implementation

#### 1. Built-in Throttling

The `@octokit/plugin-throttling` plugin automatically:

- Queues requests when near limit
- Waits for reset time
- Retries once on rate limit

#### 2. Request Batching

- **List releases**: Fetches up to 100 per page
- **List assets**: Fetches up to 100 per page
- **Parallel uploads**: Controlled concurrency

#### 3. Error Recovery

```typescript
onRateLimit: (retryAfter, options) => {
  console.warn(`Request quota exhausted for request ${options.method} ${options.url}`);
  if (options.request.retryCount === 0) {
    console.log(`Retrying after ${retryAfter} seconds!`);
    return true; // Retry once
  }
};
```

## Authentication

### Token Types

1. **`github.token`**: Default, auto-provided
   - Scoped to repository
   - Expires after workflow run
   - Limited permissions

2. **Personal Access Token (PAT)**: Custom token
   - Broader permissions
   - Can access other repositories
   - Longer lived

3. **GitHub App Token**: Installation token
   - Fine-grained permissions
   - Repository-specific

### Token Handling

```typescript
// In parseConfig (src/util.ts)
github_token: env.INPUT_TOKEN || env.GITHUB_TOKEN || '',

// Empty string means "unset"
if (!config.github_token) {
  // Use default behavior
}
```

## Error Handling Patterns

### 1. Duplicate Release (422)

**Detection**: `error.status === 422` with `already_exists` error code
**Recovery**: Find existing release, continue with it
**Source**: `src/github.ts:918-925`

### 2. Rate Limiting (429)

**Detection**: `error.status === 429`
**Recovery**: Automatic retry via plugin
**Logging**: Warning message with retry time

### 3. Permission Errors (403, 401)

**Detection**: `error.status === 403` or `401`
**Recovery**: Fail immediately, no retry
**User guidance**: Check token permissions

### 4. Not Found (404)

**Scenarios**:

- Repository not found
- Release not found
- Asset not found (during update)

**Handling**: Context-specific recovery or failure

## Pagination Strategy

### For List Operations

```typescript
async function listAllReleases(octokit, owner, repo) {
  let page = 1;
  let allReleases = [];

  while (true) {
    const response = await octokit.rest.repos.listReleases({
      owner,
      repo,
      per_page: 100,
      page,
    });

    if (response.data.length === 0) break;

    allReleases.push(...response.data);
    page++;
  }

  return allReleases;
}
```

### Performance Considerations

- **Page size**: 100 (GitHub maximum)
- **Memory**: Releases stored in memory during search
- **Network**: Multiple requests for many releases

## Concurrency Management

### Parallel Uploads

**Default**: All files uploaded concurrently
**Benefit**: Faster for multiple small files
**Risk**: Rate limiting, GitHub API limits

### Sequential Uploads

**Option**: `preserve_order: true`
**Use case**: Ordered uploads, debugging
**Implementation**: `Promise.all` vs sequential `for` loop

### Race Condition Handling

**Scenario**: Multiple workflows creating same release
**Solution**: Find-then-create pattern with retry

## API Response Processing

### Release Data Structure

```typescript
interface Release {
  id: number;
  html_url: string;
  upload_url: string;
  draft: boolean;
  // ... other fields
}

interface ReleaseResult {
  release: Release;
  created: boolean; // Whether newly created
}
```

### Asset Data Structure

```typescript
interface Asset {
  id: number;
  name: string;
  label: string;
  size: number;
  browser_download_url: string;
  // ... other fields
}
```

## GitHub API Constraints

### 1. Release Note Size Limit

**Limit**: 125,000 characters
**Handling**: Truncation in `src/github.ts`

```typescript
if (body && body.length > 125000) {
  body = body.substring(0, 125000) + '\n\n... (truncated)';
}
```

### 2. Asset Size Limits

**Limit**: 2GB per asset (GitHub constraint)
**Handling**: Stream-based upload, no buffering

### 3. File Name Normalization

**Behavior**: GitHub may rename files with special characters
**Handling**: Attempt to restore via `label` field
**Source**: `src/github.ts:471-485`

### 4. Immutable Releases

**Repository rule**: Prevents modifying published releases
**Handling**: Create as draft, publish after uploads
**Detection**: 422 error with specific error code

## Testing API Interactions

### Mocking Strategy

**Location**: `tests/github.test.ts`
**Approach**: Mock Octokit methods

```typescript
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

### Test Scenarios

1. **Happy path**: Successful API calls
2. **Rate limiting**: Mock 429 responses
3. **Duplicate releases**: Mock 422 errors
4. **Permission errors**: Mock 403/401 responses
5. **Network failures**: Mock timeouts, errors

### Mock Implementation Examples

```typescript
// Mock successful release creation
mockOctokit.rest.repos.createRelease.mockResolvedValue({
  data: {
    id: 123,
    html_url: 'https://github.com/owner/repo/releases/tag/v1.0.0',
    upload_url: 'https://uploads.github.com/...',
    draft: false,
  },
});

// Mock rate limiting
mockOctokit.rest.repos.createRelease.mockRejectedValueOnce({
  status: 429,
  headers: { 'x-ratelimit-reset': '1234567890' },
});
```

## Performance Optimizations

### 1. Request Minimization

- List releases once, cache results
- Batch asset listing when possible
- Avoid unnecessary API calls

### 2. Streaming Uploads

- Don't buffer large files in memory
- Use Node.js streams for file uploads
- Progress reporting via console

### 3. Connection Reuse

- Octokit manages HTTP connection pooling
- Keep-alive connections
- Connection reuse across requests

## Security Considerations

### 1. Token Security

- Never log tokens
- Use GitHub secrets
- Principle of least privilege

### 2. Input Validation

- Validate repository names
- Sanitize file paths
- Check token format

### 3. Error Message Safety

- Don't expose internal details
- Provide actionable guidance
- Log appropriately for debugging

## Monitoring and Logging

### API Call Logging

- **Rate limiting**: Warning messages
- **Retries**: Log retry attempts
- **Errors**: Detailed error context
- **Success**: Minimal logging for success

### Performance Metrics

- **Request count**: Monitor API usage
- **Response times**: Identify slow endpoints
- **Error rates**: Track API reliability

## Related Documentation

- [Error Handling](../architecture/error-handling.md) - Detailed error recovery
- [Configuration](../usage/configuration.md) - Token and input configuration
- [File Handling](file-handling.md) - Asset upload implementation
