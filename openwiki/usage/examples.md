# Usage - Examples

## Overview

This document provides practical examples of using the `actions-gh-release` action in various scenarios. These examples cover common patterns, edge cases, and best practices.

## Basic Examples

### 1. Simple Tag-Based Release

```yaml
name: Release
on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v7
      - name: Create Release
        uses: iShark5060/actions-gh-release@v1
```

### 2. Release with Generated Notes

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    generate_release_notes: true
```

### 3. Upload Single Asset

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    files: build/app.zip
```

## Advanced Examples

### 1. Multiple Asset Uploads with Patterns

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    files: |
      dist/*.zip
      dist/*.tar.gz
      build/reports/*.pdf
      "special[chars].txt"  # Quote patterns with special characters
```

### 2. Custom Release Name and Body

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    name: 'Awesome Release ${{ github.ref_name }}'
    body: |
      ## What's New
      - Feature: Added dark mode
      - Fix: Resolved memory leak

      ## Breaking Changes
      - API endpoint `/v1/old` removed

      See full changelog at CHANGELOG.md
```

### 3. Release Body from File

```yaml
- name: Generate Changelog
  run: echo "## Changes since last release" > release-notes.md

- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    body_path: release-notes.md
    append_body: true # Append to any existing body
```

## Workflow Integration Examples

### 1. Build and Release Pipeline

```yaml
name: Build and Release
on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - name: Build
        run: |
          npm install
          npm run build
          zip -r dist/app.zip dist/
      - name: Test
        run: npm test
      - name: Release
        uses: iShark5060/actions-gh-release@v1
        with:
          files: dist/app.zip
          generate_release_notes: true
```

### 2. Multi-Platform Builds

```yaml
name: Multi-Platform Release
on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  build-linux:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: make build-linux
      - uses: actions/upload-artifact@v4
        with:
          name: linux-binary
          path: build/linux/app

  build-windows:
    runs-on: windows-latest
    steps:
      - uses: actions/checkout@v7
      - run: build-windows.cmd
      - uses: actions/upload-artifact@v4
        with:
          name: windows-binary
          path: build/windows/app.exe

  release:
    needs: [build-linux, build-windows]
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v7
      - uses: actions/download-artifact@v4
        with:
          name: linux-binary
      - uses: actions/download-artifact@v4
        with:
          name: windows-binary
      - name: Create Release
        uses: iShark5060/actions-gh-release@v1
        with:
          files: |
            app
            app.exe
```

## Conditional Examples

### 1. Prerelease Detection

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    prerelease: ${{
      contains(github.ref_name, 'alpha') ||
      contains(github.ref_name, 'beta') ||
      contains(github.ref_name, 'rc') ||
      contains(github.ref_name, '-')
    }}
```

### 2. Manual Release with Inputs

```yaml
name: Manual Release
on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version tag (e.g., v1.2.3)'
        required: true
      draft:
        description: 'Create as draft'
        type: boolean
        default: false

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - name: Create Tag
        run: |
          git tag ${{ inputs.version }}
          git push origin ${{ inputs.version }}
      - name: Create Release
        uses: iShark5060/actions-gh-release@v1
        with:
          tag_name: ${{ inputs.version }}
          draft: ${{ inputs.draft }}
```

### 3. Branch-Based Release Strategy

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    draft: ${{ github.ref_name != 'main' && github.ref_name != 'master' }}
    prerelease: ${{ github.ref_name != 'main' && github.ref_name != 'master' }}
```

## Error Handling Examples

### 1. Fail on Missing Files

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    files: dist/*.zip
    fail_on_unmatched_files: true # Fail if no zip files
```

### 2. Continue on Missing Files

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    files: |
      required.zip
      optional-*.zip
    fail_on_unmatched_files: false # Warn but continue
```

### 3. Skip Existing Assets

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    files: artifacts/*.jar
    overwrite_files: false # Skip if asset already exists
```

## Complex Pattern Examples

### 1. Excluding Files

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    files: |
      dist/*.@(zip|tar.gz)
      !dist/*.debug.*  # Exclude debug files
```

### 2. Brace Expansion

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    files: |
      build/{linux,windows,macos}/app
      docs/{README,CHANGELOG}.md
```

### 3. Working Directory Relative Paths

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    working_directory: ${{ github.workspace }}/build
    files: |
      *.zip
      reports/*.pdf
```

## Integration Examples

### 1. With Discussions

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    discussion_category_name: 'Announcements'
```

### 2. Cross-Repository Release

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    repository: 'organization/repository'
    token: ${{ secrets.CROSS_REPO_TOKEN }}
```

### 3. Specific Commit Target

```yaml
- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    target_commitish: 'production' # Tag will point to production branch
```

## Testing Examples

### 1. Test Release (Draft)

```yaml
- name: Test Release
  if: github.event_name == 'pull_request'
  uses: iShark5060/actions-gh-release@v1
  with:
    draft: true
    tag_name: 'test-pr-${{ github.event.pull_request.number }}'
    files: test-artifacts/*.zip
```

### 2. Dry Run Pattern

```yaml
- name: Check Files
  run: |
    echo "Files that would be uploaded:"
    ls -la dist/*.zip || true

- name: Create Draft Release
  if: github.ref_type == 'tag'
  uses: iShark5060/actions-gh-release@v1
  with:
    draft: true
    files: dist/*.zip

- name: Publish Release
  if: github.ref_type == 'tag' && github.event_name != 'pull_request'
  run: |
    # Manually publish the draft
    echo "Release would be published"
```

## Real-World Examples

### 1. Node.js Package Release

```yaml
name: Node.js Release
on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm test
      - run: npm run build
      - name: Create Release
        uses: iShark5060/actions-gh-release@v1
        with:
          files: |
            dist/*.js
            package.json
            README.md
          generate_release_notes: true
```

### 2. Docker Image Release

```yaml
name: Docker Release
on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - name: Build Docker image
        run: docker build -t app:${{ github.ref_name }} .
      - name: Save Docker image
        run: docker save app:${{ github.ref_name }} | gzip > app-${{ github.ref_name }}.tar.gz
      - name: Create Release
        uses: iShark5060/actions-gh-release@v1
        with:
          files: app-${{ github.ref_name }}.tar.gz
```

### 3. Documentation Release

```yaml
name: Docs Release
on:
  push:
    tags:
      - 'docs-v*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - name: Build Documentation
        run: |
          npm install
          npm run docs:build
          tar -czf docs.tar.gz docs/
      - name: Create Release
        uses: iShark5060/actions-gh-release@v1
        with:
          tag_name: ${{ github.ref_name }}
          name: 'Documentation ${{ github.ref_name }}'
          body_path: docs/CHANGELOG.md
          files: docs.tar.gz
```

## Troubleshooting Examples

### 1. Debugging File Patterns

```yaml
- name: Debug Files
  run: |
    echo "Working directory: $(pwd)"
    echo "Files matching pattern:"
    shopt -s nullglob
    for pattern in "dist/*.zip" "build/*.tar.gz"; do
      echo "Pattern: $pattern"
      ls -la $pattern 2>/dev/null || echo "No matches"
    done

- name: Create Release
  uses: iShark5060/actions-gh-release@v1
  with:
    files: |
      dist/*.zip
      build/*.tar.gz
```

### 2. Checking Permissions

```yaml
- name: Verify Token
  run: |
    # Check token permissions
    curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
      https://api.github.com/repos/${{ github.repository }} | jq .permissions

- name: Create Release
  uses: iShark5060/actions-gh-release@v1
```

## Best Practice Examples

### 1. Comprehensive Release

```yaml
- name: Create Comprehensive Release
  uses: iShark5060/actions-gh-release@v1
  with:
    name: 'Release ${{ github.ref_name }}'
    body_path: CHANGELOG.md
    generate_release_notes: true
    discussion_category_name: 'Releases'
    files: |
      dist/*.zip
      dist/*.tar.gz
      dist/*.exe
      dist/*.deb
      dist/*.rpm
      checksums.txt
      release-notes.pdf
    make_latest: true
```

### 2. Minimal Permissions

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write # Minimum required permission
    steps:
      - uses: actions/checkout@v7
      - uses: iShark5060/actions-gh-release@v1
```

## Related Documentation

- [Configuration](configuration.md) - Complete input reference
- [Permissions](permissions.md) - Security and token guidance
- [GitHub API Integration](../integration/github-api.md) - API interaction details
