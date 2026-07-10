# Usage - Permissions

## Overview

This document details the permission requirements, token configurations, and security considerations for the `actions-gh-release` action. Proper permission configuration is essential for the action to function correctly and securely.

## Token Requirements

### Default Token (`github.token`)

By default, the action uses the automatically provided `github.token`. This token has permissions scoped to the repository where the workflow runs.

**Required Scopes for `github.token`**:

- `contents: write` - Create releases, upload assets, create tags
- `contents: read` - Read repository contents (for tag operations)

### Custom Tokens

You can provide a custom token via the `token` input:

```yaml
- uses: iShark5060/actions-gh-release@v1
  with:
    token: ${{ secrets.MY_CUSTOM_TOKEN }}
```

**When to use custom tokens**:

1. **Cross-repository releases**: Releasing to a different repository
2. **Extended permissions**: Need additional scopes beyond default
3. **Service accounts**: Using a dedicated service account token
4. **Personal Access Tokens (PAT)**: For specific use cases

## Permission Configuration

### GitHub Actions Permissions

Configure permissions at the workflow or job level:

#### Minimal Permissions (Recommended)

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write # Minimum required
    steps:
      - uses: iShark5060/actions-gh-release@v1
```

#### Default Permissions

If no permissions are specified, GitHub uses default permissions which include `contents: write` for the `github.token`.

#### Read-Only Workflow

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read # Read-only for build steps
    steps:
      # Build steps here

  release:
    runs-on: ubuntu-latest
    needs: [build]
    permissions:
      contents: write # Write only for release step
    steps:
      - uses: iShark5060/actions-gh-release@v1
```

## Token Scopes and Limitations

### GitHub App Tokens

If using a GitHub App token, ensure it has:

- Repository permissions: `Contents` → `Read & Write`
- No additional scopes required

### Personal Access Tokins (PAT)

**Required Scopes**:

- `repo` (full control) - For private repositories
- `public_repo` - For public repositories only

**Note**: PATs have broader permissions than `github.token`. Use with caution.

### Fine-Grained PATs (Beta)

If using fine-grained PATs:

- Repository access: Select target repository
- Permissions: `Contents` → `Read and write`

## Permission Errors and Solutions

### Common Error: 403 Forbidden

**Symptoms**:

- `Resource not accessible by integration`
- `tag creation blocked by repository rule`

**Solutions**:

1. **Check workflow permissions**: Ensure `contents: write` is granted
2. **Use PAT for tag creation**: `github.token` may lack tag creation permissions
3. **Repository rules**: Check if repository rules block tag creation

### Error: 401 Unauthorized

**Causes**:

- Invalid or expired token
- Token doesn't have required scopes

**Solutions**:

1. **Verify token validity**: Check if token is still valid
2. **Regenerate token**: Create new token with correct scopes
3. **Check token format**: Ensure proper token formatting

## Security Best Practices

### 1. Principle of Least Privilege

Grant only the permissions needed:

```yaml
permissions:
  contents: write # Just enough for releases
```

### 2. Token Management

- **Use `github.token` by default**: It's automatically scoped and rotated
- **Store custom tokens in secrets**: Never hardcode tokens
- **Rotate tokens regularly**: Especially PATs and service account tokens

### 3. Repository Rules Considerations

If your repository has rules enabled:

- **Tag creation rules**: May require specific permissions
- **Branch protection**: May affect release targeting
- **Immutable releases**: Requires draft-first workflow

### 4. Cross-Repository Security

When releasing to other repositories:

```yaml
- uses: iShark5060/actions-gh-release@v1
  with:
    repository: 'org/other-repo'
    token: ${{ secrets.OTHER_REPO_TOKEN }} # Token with access to other repo
```

## Permission Scenarios

### Scenario 1: Simple Release (Same Repository)

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v7
      - uses: iShark5060/actions-gh-release@v1
```

### Scenario 2: Release with Custom Tag Creation

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v7
      - name: Create Release
        uses: iShark5060/actions-gh-release@v1
        with:
          target_commitish: 'main'
          # If tag creation fails with github.token, use PAT:
          # token: ${{ secrets.RELEASE_PAT }}
```

### Scenario 3: Cross-Organization Release

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - name: Release to Other Org
        uses: iShark5060/actions-gh-release@v1
        with:
          repository: 'other-org/repository'
          token: ${{ secrets.OTHER_ORG_TOKEN }}
```

### Scenario 4: Immutable Release Repository

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v7
      - name: Create Draft Release
        uses: iShark5060/actions-gh-release@v1
        with:
          draft: true # Required for immutable releases
          files: dist/*.zip
        # Draft will be published automatically if allowed by rules
```

## Advanced Permission Configurations

### Job-Level Permission Inheritance

```yaml
permissions:
  contents: write
  issues: read
  pull-requests: read

jobs:
  test:
    runs-on: ubuntu-latest
    # Inherits permissions from workflow

  release:
    runs-on: ubuntu-latest
    # Can override if needed
    permissions:
      contents: write
```

### Environment Secrets

Use environment-specific tokens:

```yaml
jobs:
  release:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: iShark5060/actions-gh-release@v1
        with:
          token: ${{ secrets.GITHUB_TOKEN }} # Auto-scoped to environment
```

### Token Input Behavior

```yaml
- uses: iShark5060/actions-gh-release@v1
  with:
    # Different token behaviors:

    # 1. Default (uses github.token)
    # token: ${{ github.token }}

    # 2. Custom token (overrides default)
    token: ${{ secrets.CUSTOM_TOKEN }}

    # 3. Empty string (treats as unset)
    # token: ''
```

## Troubleshooting Permissions

### Diagnostic Steps

1. **Check workflow permissions**:

   ```yaml
   - name: Debug Permissions
     run: |
       echo "Event: ${{ github.event_name }}"
       echo "Ref: ${{ github.ref }}"
       echo "Ref type: ${{ github.ref_type }}"
   ```

2. **Test token access**:

   ```bash
   curl -H "Authorization: token $TOKEN" \
     https://api.github.com/repos/owner/repo
   ```

3. **Check repository rules**:
   - Visit repository settings → Rules
   - Check tag creation rules
   - Review branch protection rules

### Common Issues and Fixes

#### Issue: Tag creation blocked

**Error**: `tag creation blocked by repository rule`
**Fix**:

```yaml
- uses: iShark5060/actions-gh-release@v1
  with:
    # Use PAT with bypass permissions
    token: ${{ secrets.RELEASE_BYPASS_TOKEN }}
    # Or create draft first
    draft: true
```

#### Issue: Insufficient contents permission

**Error**: `Resource not accessible by integration`
**Fix**:

```yaml
jobs:
  release:
    permissions:
      contents: write # Explicitly grant permission
```

#### Issue: Cross-repository access denied

**Error**: `Not Found` or `Forbidden`
**Fix**:

```yaml
- uses: iShark5060/actions-gh-release@v1
  with:
    repository: 'owner/repo'
    token: ${{ secrets.TOKEN_WITH_ACCESS }} # Token with access to target repo
```

## Security Recommendations

### 1. Token Storage

- **Use GitHub Secrets**: Never expose tokens in logs
- **Environment-specific**: Different tokens for different environments
- **Regular rotation**: Especially for long-lived tokens

### 2. Permission Auditing

- **Review workflow permissions**: Regularly audit who can release
- **Monitor release activity**: Watch for unexpected releases
- **Use required reviewers**: For critical releases

### 3. Repository Configuration

- **Protect tags**: Use tag protection rules
- **Require reviews**: For release workflows
- **Limit permissions**: Grant write only where needed

### 4. Action Pinning

```yaml
- uses: iShark5060/actions-gh-release@v1 # Use major version
# or
- uses: iShark5060/actions-gh-release@v1.0.1 # Use specific version
```

**Always pin to major version or specific tag** to avoid unexpected changes.

## Related Documentation

- [Configuration](configuration.md) - Token input configuration
- [Examples](examples.md) - Permission usage examples
- [GitHub API Integration](../integration/github-api.md) - API authentication details
