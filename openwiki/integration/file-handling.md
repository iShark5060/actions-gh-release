# Integration - File Handling

## Overview

This document details how the `actions-gh-release` action handles files, including glob pattern matching, path resolution, file uploads, and cross-platform compatibility.

## File Input Processing

### `files` Input Parsing

**Source**: `src/util.ts:136-190` (`parseInputFiles()`)

**Format**: Newline-delimited list of glob patterns

```yaml
files: |
  dist/*.zip
  build/*.tar.gz
  "special[chars].txt"
```

**Parsing Logic**:

1. Split by newlines
2. Trim whitespace
3. Filter empty lines
4. Handle quoted patterns
5. Brace-aware splitting

### Special Characters

**Quoting**: Patterns with special characters need quotes

```yaml
files: |
  "file[name].txt"      # Square brackets
  "file{1,2}.txt"       # Braces (if not using expansion)
  "file*.txt"           # Asterisk (literal)
```

**Brace Expansion**: Supported but requires careful parsing

```yaml
files: |
  build/{linux,windows}/app  # Expands to two patterns
```

## Glob Pattern Matching

### Implementation

**Library**: `glob` package (version 13.0.6)
**Function**: `src/util.ts:83-105` (`paths()`)

```typescript
import { glob } from 'glob';

async function paths(patterns: string[], cwd: string): Promise<string[]> {
  const matches = await Promise.all(patterns.map((pattern) => glob(pattern, { cwd, nodir: true })));
  return Array.from(new Set(matches.flat())); // Deduplicate
}
```

### Supported Patterns

| Pattern | Meaning                     | Example         |
| ------- | --------------------------- | --------------- |
| `*`     | Any characters (except `/`) | `*.zip`         |
| `**`    | Any directories             | `**/*.js`       |
| `?`     | Single character            | `file?.txt`     |
| `[]`    | Character class             | `file[0-9].txt` |
| `{}`    | Brace expansion             | `{a,b}.txt`     |
| `!`     | Negation                    | `!*.debug.*`    |

### Working Directory

**Default**: GitHub workspace root
**Custom**: Set via `working_directory` input
**Resolution**: Patterns resolved relative to working directory

## Path Normalization

### Cross-Platform Paths

**Source**: `src/util.ts:192-210` (`normalizePaths()`)

**Windows Considerations**:

- Accepts both `\` and `/` separators
- Normalizes to `/` for consistency
- Handles drive letters (`C:\` → `C:/`)

```typescript
function normalizePaths(paths: string[]): string[] {
  return paths.map((path) => path.replace(/\\/g, '/'));
}
```

### Home Directory Expansion

**Pattern**: `~/path/to/file`
**Expansion**: Expands to runner's home directory
**Implementation**: Handled by `glob` library

### Tag Name Normalization

**Source**: `src/util.ts:57-65` (`normalizeTagName()`)

```typescript
function normalizeTagName(tagName: string): string {
  return tagName.replace(/^refs\/tags\//, '');
}
```

**Examples**:

- `refs/tags/v1.0.0` → `v1.0.0`
- `v1.0.0` → `v1.0.0` (unchanged)

## File Upload Process

### Upload Flow (`src/github.ts:306-498`)

1. **Parse patterns** → Get file list
2. **Deduplicate** → Remove duplicates
3. **Check existing** → List current assets
4. **Delete if overwrite** → Remove existing assets
5. **Upload** → Stream file to GitHub
6. **Handle errors** → Retry on failures

### Streaming Implementation

**Key**: Don't buffer large files in memory
**Implementation**: `src/github.ts:455-498`

```typescript
import { createReadStream, statSync } from 'fs';

const fileStats = statSync(filePath);
const fileStream = createReadStream(filePath);

await octokit.rest.repos.uploadReleaseAsset({
  url: uploadUrl,
  data: fileStream, // Stream, not buffer
  name: assetName,
  headers: {
    'content-type': mimeType,
    'content-length': fileStats.size,
  },
});
```

### MIME Type Detection

**Library**: `mime-types` (version 3.0.2)
**Usage**: Automatically detect content type
**Fallback**: `application/octet-stream`

```typescript
import { lookup } from 'mime-types';

const mimeType = lookup(filePath) || 'application/octet-stream';
```

## Asset Name Handling

### File Name vs Asset Label

**File Name**: Actual filename on disk
**Asset Label**: Display name in GitHub UI
**GitHub Behavior**: May normalize special characters

### Name Alignment

**Source**: `src/github.ts:471-485` (`alignAssetName()`)
**Purpose**: Attempt to restore original filename when GitHub normalizes it

```typescript
function alignAssetName(name: string): string {
  // GitHub replaces spaces with dots
  return name.replace(/\./g, ' ');
}
```

### Special Character Handling

**GitHub Normalization**:

- Spaces → Dots
- Other special characters → May be encoded
- Unicode → Generally preserved

**Action Strategy**:

1. Upload with original name
2. If GitHub returns different name, try to match via label
3. Log warning if normalization occurs

## Error Handling for Files

### Unmatched Patterns

**Detection**: `src/util.ts:107-134` (`unmatchedPatterns()`)
**Behavior**: Controlled by `fail_on_unmatched_files`

```typescript
const unmatched = await unmatchedPatterns(patterns, cwd);
if (unmatched.length > 0) {
  if (failOnUnmatched) {
    throw new Error(`⚠️ Pattern '${pattern}' does not match any files.`);
  } else {
    console.warn(`🤔 Pattern '${pattern}' does not match any files.`);
  }
}
```

### File Not Found

**During upload**: File disappears between listing and upload
**Recovery**: Fail individual upload, continue with others
**Logging**: Error message for specific file

### Permission Errors

**Reading files**: Insufficient permissions
**Handling**: Fail with clear error message
**Common on Windows**: File in use, locked by another process

## Performance Considerations

### Parallel vs Sequential Uploads

**Default**: Parallel (`Promise.all`)
**Option**: Sequential (`preserve_order: true`)

```typescript
if (config.input_preserve_order) {
  // Sequential
  for (const filePath of filePaths) {
    await uploadFile(filePath);
  }
} else {
  // Parallel
  await Promise.all(filePaths.map((filePath) => uploadFile(filePath)));
}
```

### Large File Handling

**Streaming**: Essential for large files
**Memory**: Constant memory usage regardless of file size
**Progress**: No built-in progress reporting

### Network Considerations

**Timeout**: GitHub API may timeout for very large files
**Retry**: Built-in retry for network errors
**Chunking**: Not implemented (GitHub accepts streaming)

## Cross-Platform Testing

### Test Data

**Location**: `tests/data/`
**Structure**:

```
tests/data/
└── foo/
    └── bar.txt
```

**Purpose**: Test file pattern matching with actual files

### Platform-Specific Tests

**Patterns tested**:

- Basic glob patterns
- Relative paths
- Absolute paths (where safe)
- Special characters

**Windows-specific**:

- Backslash path separators
- Drive letters
- File locking behavior

## Configuration Options

### `fail_on_unmatched_files`

**Default**: `false` (warn but continue)
**True**: Fail action if any pattern doesn't match
**Use case**: Strict validation for CI/CD pipelines

### `overwrite_files`

**Default**: `true` (delete and re-upload)
**False**: Skip existing assets
**Use case**: Incremental uploads, avoid duplication

### `preserve_order`

**Default**: `false` (parallel uploads)
**True**: Upload in pattern order
**Use case**: Ordered assets, dependency chains

### `working_directory`

**Default**: GitHub workspace root
**Custom**: Base for pattern resolution
**Use case**: Build artifacts in subdirectory

## Common File Patterns

### Build Artifacts

```yaml
files: |
  dist/*.zip
  build/*.tar.gz
  packages/*.@(deb|rpm)
```

### Documentation

```yaml
files: |
  README.md
  CHANGELOG.md
  docs/*.pdf
```

### Checksums

```yaml
files: |
  *.sha256
  *.md5
  checksums.txt
```

### Platform-Specific

```yaml
files: |
  build/linux/app
  build/windows/app.exe
  build/macos/app
```

## Troubleshooting File Issues

### Debugging Pattern Matching

```bash
# Test patterns manually
node -e "
const { glob } = require('glob');
glob('dist/*.zip', { cwd: process.cwd() }).then(console.log);
"
```

### Checking File Existence

```yaml
- name: List files
  run: |
    echo "Working directory: $(pwd)"
    echo "Files matching dist/*.zip:"
    ls -la dist/*.zip 2>/dev/null || echo "No matches"
```

### Permission Issues

**Symptoms**: "Permission denied" errors
**Check**:

- File permissions on runner
- Whether file is open/locked
- Runner user permissions

### Path Resolution Issues

**Common mistakes**:

- Relative paths from wrong directory
- Missing `working_directory` setting
- Incorrect pattern syntax

**Debug**:

```yaml
- name: Debug paths
  run: |
    echo "Workspace: ${{ github.workspace }}"
    echo "PWD: $(pwd)"
    find . -name "*.zip" -type f | head -10
```

## Security Considerations

### Path Traversal Prevention

**Implementation**: Patterns resolved within `working_directory`
**Safety**: Cannot access files outside designated directory

### File Content Safety

**No inspection**: Action doesn't inspect file contents
**Upload only**: Streams files directly to GitHub
**Size limits**: GitHub enforces 2GB per asset limit

### Token in File Names

**Risk**: Accidental token exposure in filenames
**Mitigation**: No special handling, user responsibility

## Related Documentation

- [GitHub API](github-api.md) - Upload endpoint details
- [Configuration](../usage/configuration.md) - File input configuration
- [Examples](../usage/examples.md) - File pattern examples
- [Error Handling](../architecture/error-handling.md) - File error recovery
