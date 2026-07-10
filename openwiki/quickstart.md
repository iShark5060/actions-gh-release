# GH Release Action - Quickstart

## Overview

`actions-gh-release` is a GitHub Action for creating and managing GitHub Releases. It's a maintained fork of [`softprops/action-gh-release`](https://github.com/softprops/action-gh-release) with updated dependencies, modern tooling, and enhanced reliability features.

**Key Features:**

- 🚀 **Tag-based releases** with automatic gating
- 📦 **Asset uploads** with glob pattern matching
- 🛡️ **Robust error handling** with retry logic
- 🔄 **Race condition prevention** for concurrent workflows
- 🏗️ **Draft and prerelease** support
- 🔧 **Modern TypeScript** with comprehensive testing

## Quick Usage

```yaml
name: Release
on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - name: Create Release
        uses: iShark5060/actions-gh-release@v1
        with:
          files: |
            dist/*.zip
            build/*.tar.gz
          generate_release_notes: true
```

## Core Concepts

This action orchestrates GitHub Releases through three main phases:

1. **Release Creation/Update** - Creates new releases or updates existing ones
2. **Asset Upload** - Uploads files with pattern matching and error recovery
3. **Finalization** - Publishes drafts and sets outputs

## Documentation Sections

### [Architecture](architecture/)

- **Core Concepts**: Releaser interface, configuration, error handling
- **Workflow Flow**: Step-by-step execution flow
- **Error Recovery**: Retry logic and race condition management

### [Usage](usage/)

- **Configuration**: Inputs, outputs, and environment variables
- **Examples**: Common patterns and use cases
- **Permissions**: Required token scopes and security considerations

### [Development](development/)

- **Building & Testing**: Local development workflow
- **Contribution Guide**: Core rules and verification steps
- **Release Process**: CI/CD automation and version management

### [Integration](integration/)

- **GitHub API**: Octokit integration and rate limiting
- **File Handling**: Glob patterns and path normalization

### [Operations](operations/)

- **CI/CD**: GitHub Actions workflows and validation
- **Security**: Supply chain attestation and dependency management

## Version Compatibility

| Version | Node.js | Status    |
| ------- | ------- | --------- |
| v1.x    | ≥24     | ✅ Active |
| v2.6.2  | 20      | ⚠️ Legacy |

**Important:** Always reference published version tags (e.g., `@v1`). The bundled action code is only committed to release tags.

## Getting Help

- **Issues**: Check existing issues before reporting
- **Documentation**: This OpenWiki provides comprehensive guidance
- **Source**: The code is well-documented and tested

## Quick Links

- [GitHub Repository](https://github.com/iShark5060/actions-gh-release)
- [Original Fork](https://github.com/softprops/action-gh-release)
- [GitHub Actions Marketplace](https://github.com/marketplace/actions/gh-release)
