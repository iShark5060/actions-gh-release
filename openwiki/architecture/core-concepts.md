# Architecture - Core Concepts

## Overview

The `actions-gh-release` action follows a clean, layered architecture with strong separation of concerns. The design prioritizes reliability, testability, and maintainability.

## Architecture Layers

### 1. Orchestration Layer (`src/index.ts`)

**Purpose**: Coordinates the entire release workflow
**Key Responsibilities**:

- Parse configuration from environment variables
- Validate inputs and preconditions
- Initialize GitHub API client with plugins
- Coordinate release, upload, and finalization steps
- Handle errors and set action outputs

### 2. GitHub API Layer (`src/github.ts`)

**Purpose**: Abstracts GitHub Releases API interactions
**Key Components**:

#### `Releaser` Interface

```typescript
interface Releaser {
  release(config: Config): Promise<ReleaseResult>;
  upload(release: Release, config: Config): Promise<Asset[]>;
  finalizeRelease(release: Release, config: Config): Promise<void>;
  listReleaseAssets(release: Release, config: Config): Promise<Asset[]>;
}
```

#### `GitHubReleaser` Class

- Concrete implementation using Octokit REST API
- Handles rate limiting with `@octokit/plugin-throttling`
- Implements retry logic with `@octokit/plugin-retry`
- Manages pagination for listing releases/assets

### 3. Utility Layer (`src/util.ts`)

**Purpose**: Provides parsing, validation, and file operations
**Key Functions**:

- `parseConfig()` - Maps environment variables to typed configuration
- `paths()` - Resolves glob patterns to file paths
- `unmatchedPatterns()` - Validates file pattern matches
- `normalizeTagName()` - Strips `refs/tags/` prefix
- `releaseBody()` - Reads release notes from file or input

## Configuration Model

The `Config` interface (defined in `src/util.ts`) represents all action inputs:

```typescript
interface Config {
  // GitHub context
  github_ref: string;
  github_token: string;

  // Release configuration
  input_tag_name?: string;
  input_body?: string;
  input_body_path?: string;
  input_name?: string;
  input_draft: boolean;
  input_prerelease: boolean;

  // Asset configuration
  input_files?: string;
  input_working_directory: string;
  input_fail_on_unmatched_files: boolean;
  input_preserve_order: boolean;
  input_overwrite_files: boolean;

  // Repository configuration
  input_repository?: string;
  input_target_commitish?: string;
  input_discussion_category_name?: string;
  input_generate_release_notes: boolean;
  input_previous_tag: string;
  input_append_body: boolean;
  input_make_latest?: string;
}
```

## Key Design Patterns

### 1. Dependency Injection

The `Releaser` interface allows for easy testing by mocking GitHub API interactions. The `GitHubReleaser` is the production implementation.

### 2. Error Recovery

- **Retry Logic**: Automatic retry for rate limiting (once) and certain API errors
- **Race Condition Handling**: Detection and recovery for concurrent release operations
- **Partial Success**: Continues with other uploads when one fails (configurable)

### 3. Concurrency Management

- **Parallel Uploads**: Default behavior for performance
- **Sequential Option**: `preserve_order` input for ordered uploads
- **Race Prevention**: Checks for existing releases/assets before operations

### 4. Immutable Release Support

Special handling for repositories with "immutable releases" enabled:

- Creates draft releases for asset uploads
- Publishes only after all uploads complete
- Handles repository rule violations gracefully

## Data Flow

1. **Input Parsing** → Environment variables → `Config` object
2. **Validation** → Tag checks, file pattern validation
3. **Release Creation** → Find/create release, handle duplicates
4. **Asset Upload** → Match files, upload with error handling
5. **Finalization** → Publish draft, set outputs
6. **Output** → Release URL, ID, upload URL, asset info

## Error Handling Strategy

### 1. User Errors

- Missing tag for non-draft releases
- Invalid glob patterns
- Permission errors (403)

### 2. Transient Errors

- Rate limiting (429) - automatic retry
- Network failures - retry with backoff
- Concurrent modification (422) - race detection

### 3. Fatal Errors

- Invalid token (401)
- Repository not found (404)
- Validation failures

## Source Files

| File            | Purpose          | Key Exports                                                    |
| --------------- | ---------------- | -------------------------------------------------------------- |
| `src/index.ts`  | Main entry point | `run()` function                                               |
| `src/github.ts` | GitHub API layer | `GitHubReleaser`, `release()`, `upload()`, `finalizeRelease()` |
| `src/util.ts`   | Utilities        | `parseConfig()`, `paths()`, `normalizeTagName()`               |

## Constraints & Limitations

1. **Tag Requirement**: Requires a tag unless creating a draft release
2. **Release Note Size**: GitHub limits release notes to 125,000 characters
3. **Asset Name Normalization**: GitHub may rename assets with special characters
4. **Concurrency Limits**: GitHub API has rate limits and concurrency constraints

## Related Documentation

- [Workflow Flow](workflow-flow.md) - Step-by-step execution
- [Error Handling](error-handling.md) - Retry logic and recovery strategies
- [GitHub API Integration](../integration/github-api.md) - API usage details
