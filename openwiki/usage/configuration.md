# Usage - Configuration

## Overview

This document details all configuration options for the `actions-gh-release` action. Understanding these inputs, outputs, and environment variables is essential for effective usage.

## Inputs

All inputs are optional unless specified. Inputs are passed via the `with` key in your workflow.

### Release Configuration

| Input                    | Type    | Default           | Description                                                                                                                                |
| ------------------------ | ------- | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| `body`                   | string  | -                 | Note-worthy description of changes in release                                                                                              |
| `body_path`              | string  | -                 | Path to load note-worthy description of changes in release from                                                                            |
| `name`                   | string  | tag name          | Gives the release a custom name. Defaults to tag name                                                                                      |
| `tag_name`               | string  | `github.ref_name` | Gives a tag name. Defaults to github.ref_name. `refs/tags/<name>` values are normalized to `<name>`                                        |
| `draft`                  | boolean | `false`           | Keeps the release as a draft. When reusing an existing draft release, set this to `true` to keep it draft; omit it to publish after upload |
| `prerelease`             | boolean | `false`           | Identify the release as a prerelease                                                                                                       |
| `generate_release_notes` | boolean | `false`           | Whether to automatically generate the name and body for this release                                                                       |
| `previous_tag`           | string  | -                 | When `generate_release_notes` is enabled, use this tag as GitHub's `previous_tag_name` comparison base                                     |
| `append_body`            | boolean | `false`           | Append to existing body instead of overwriting it                                                                                          |
| `make_latest`            | string  | -                 | Specifies whether this release should be set as the latest release. Can be `true`, `false`, or `legacy`                                    |

### Asset Configuration

| Input                     | Type    | Default        | Description                                                    |
| ------------------------- | ------- | -------------- | -------------------------------------------------------------- |
| `files`                   | string  | -              | Newline-delimited list of path globs for asset files to upload |
| `working_directory`       | string  | workspace root | Base directory to resolve `files` globs against                |
| `overwrite_files`         | boolean | `true`         | Overwrite existing files with the same name                    |
| `fail_on_unmatched_files` | boolean | `false`        | Fails if any of the `files` globs match nothing                |
| `preserve_order`          | boolean | `false`        | Upload artifacts sequentially in the provided order            |

### Repository Configuration

| Input                      | Type   | Default        | Description                                                                               |
| -------------------------- | ------ | -------------- | ----------------------------------------------------------------------------------------- |
| `repository`               | string | current repo   | Repository to make releases against, in `<owner>/<repo>` format                           |
| `token`                    | string | `github.token` | Authorized GitHub token or PAT                                                            |
| `target_commitish`         | string | -              | Commitish value that determines where the Git tag is created from                         |
| `discussion_category_name` | string | -              | If specified, a discussion of the specified category is created and linked to the release |

## Outputs

The action sets these outputs that can be used in subsequent steps:

| Output       | Type   | Description                                                 |
| ------------ | ------ | ----------------------------------------------------------- |
| `url`        | string | URL to the Release HTML Page                                |
| `id`         | string | Release ID                                                  |
| `upload_url` | string | URL for uploading assets to the release                     |
| `assets`     | string | JSON array containing information about each uploaded asset |

### Example Output Usage

```yaml
steps:
  - name: Create Release
    id: release
    uses: iShark5060/actions-gh-release@v1
    with:
      files: dist/*.zip

  - name: Print Release URL
    run: echo "Release created at ${{ steps.release.outputs.url }}"
```

## Environment Variables

The action reads these environment variables automatically:

| Variable       | Source         | Purpose                           |
| -------------- | -------------- | --------------------------------- |
| `GITHUB_REF`   | GitHub Context | Current branch/tag reference      |
| `GITHUB_TOKEN` | GitHub Context | Default authentication token      |
| `INPUT_*`      | GitHub Actions | All inputs prefixed with `INPUT_` |

## Detailed Input Descriptions

### `files` Input

**Format**: Newline-delimited list of glob patterns
**Examples**:

```yaml
files: |
  dist/*.zip
  build/*.tar.gz
  "special[file].txt"  # Quotes for special characters
```

**Features**:

- Supports glob patterns (`*`, `**`, `?`, `[]`)
- `~/` expands to runner home directory
- Brace-aware parsing (handles `{a,b}` patterns)
- Both `\` and `/` path separators accepted on Windows

### `token` Input

**Default Behavior**: Uses `github.token` when omitted
**Override**: Explicit non-empty token overrides `GITHUB_TOKEN`
**Empty String**: Passing `""` treats token as unset

**Required Permissions**:

- `contents: write` - Create releases and upload assets
- `contents: read` - Read repository contents (for tag creation)

### `tag_name` Normalization

The action automatically normalizes tag names:

- `refs/tags/v1.0.0` → `v1.0.0`
- `v1.0.0` → `v1.0.0` (unchanged)

### `body` vs `body_path`

- **`body`**: Direct string content
- **`body_path`**: Path to file containing release notes
- **Both**: Can be used together (body appended to file content)

### `draft` Behavior

**Creating new drafts**: Set `draft: true`
**Updating existing drafts**:

- Keep as draft: Set `draft: true`
- Publish after upload: Omit `draft` or set `draft: false`

**Immutable release repositories**: Always creates as draft first

### `make_latest` Options

- **`true`**: Set as latest release
- **`false`**: Do not set as latest
- **`legacy`**: GitHub decides based on version comparison
- **unset**: GitHub API default behavior

## Configuration Examples

### Minimal Configuration

```yaml
- uses: iShark5060/actions-gh-release@v1
  if: github.ref_type == 'tag'
```

### Full Configuration

```yaml
- uses: iShark5060/actions-gh-release@v1
  with:
    tag_name: ${{ github.ref_name }}
    name: 'Release ${{ github.ref_name }}'
    body: "## What's Changed\n- Feature A\n- Bug Fix B"
    draft: false
    prerelease: ${{ contains(github.ref_name, '-') }}
    files: |
      dist/*.zip
      build/*.tar.gz
    working_directory: ${{ github.workspace }}
    overwrite_files: true
    fail_on_unmatched_files: false
    generate_release_notes: true
    append_body: false
    make_latest: true
```

### Conditional Prerelease

```yaml
- uses: iShark5060/actions-gh-release@v1
  with:
    prerelease: ${{ contains(github.ref_name, 'alpha') || contains(github.ref_name, 'beta') || contains(github.ref_name, 'rc') }}
```

## Validation Rules

### Tag Requirement

Unless `draft: true`, the action requires:

- `tag_name` input is set, OR
- `GITHUB_REF` points to a tag (`github.ref_type == 'tag'`)

### File Pattern Validation

When `files` is provided:

1. Patterns are expanded in `working_directory`
2. Unmatched patterns trigger warnings or errors
3. Control with `fail_on_unmatched_files`

### Token Validation

- Non-empty token must be valid GitHub token
- Token must have required permissions
- Invalid tokens cause immediate failure

## Common Configuration Patterns

### 1. Tag-Only Releases

```yaml
on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release:
    steps:
      - uses: iShark5060/actions-gh-release@v1
```

### 2. Branch with Manual Tag

```yaml
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version tag'
        required: true

jobs:
  release:
    steps:
      - uses: actions/checkout@v7
      - run: git tag ${{ inputs.version }}
      - run: git push origin ${{ inputs.version }}
      - uses: iShark5060/actions-gh-release@v1
        with:
          tag_name: ${{ inputs.version }}
```

### 3. Draft Release for Review

```yaml
- uses: iShark5060/actions-gh-release@v1
  with:
    draft: true
    files: artifacts/*.zip
```

## Troubleshooting Configuration

### Common Issues

1. **No tag error**: Ensure `draft: true` or tag exists
2. **Permission errors**: Check token has `contents: write`
3. **Unmatched files**: Verify paths relative to `working_directory`
4. **Asset not uploaded**: Check `overwrite_files` setting

### Debug Tips

1. **Enable Step Debugging**: Add `env: ACTIONS_STEP_DEBUG: true`
2. **Check Expanded Inputs**: Use `echo` to print variables
3. **Verify File Existence**: List files before release step
4. **Test with Dry Run**: Use `draft: true` for testing

## Related Documentation

- [Examples](examples.md) - Common use cases and patterns
- [Permissions](permissions.md) - Token requirements and security
- [Architecture Core Concepts](../architecture/core-concepts.md) - Configuration model internals
