# GH Release

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE)
[![CI](https://img.shields.io/github/actions/workflow/status/iShark5060/actions-gh-release/ci.yml?style=flat-square&label=CI)](https://github.com/iShark5060/actions-gh-release/actions/workflows/ci.yml)
![Node](https://img.shields.io/badge/Node-%3E%3D24-339933?logo=node.js&logoColor=white&style=flat-square)
![TypeScript](https://img.shields.io/badge/TypeScript-7.x-3178C6?logo=typescript&logoColor=white&style=flat-square)
[![Cursor](https://img.shields.io/badge/Cursor-IDE-141414?logo=cursor&logoColor=white&style=flat-square)](https://cursor.com)

<<<<<<< Updated upstream
Creates GitHub Releases and uploads assets. Maintained fork of [softprops/action-gh-release](https://github.com/softprops/action-gh-release). This fork’s published major is `@v1` (Node 24) — not upstream `v2` / `v3`.
=======
Create GitHub Releases from a workflow on Linux, Windows, and macOS. Tag a build, attach the zip, done. Same action on every runner so a Windows native release and a Linux one look identical.

This is a maintained fork of [softprops/action-gh-release](https://github.com/softprops/action-gh-release) by Doug Tangren (MIT License).

> **Always reference a published version tag** (e.g. `@v1`). The bundled action code (`dist/index.js`) is only committed to release tags, so referencing `@main` will not work.

- [🤸 Usage](#-usage)
  - [🚥 Limit releases to pushes to tags](#-limit-releases-to-pushes-to-tags)
  - [⬆️ Uploading release assets](#️-uploading-release-assets)
  - [📝 External release notes](#-external-release-notes)
  - [💅 Customizing](#-customizing)
    - [inputs](#inputs)
    - [outputs](#outputs)
    - [environment variables](#environment-variables)
  - [Permissions](#permissions)

## 🤸 Usage

### 🚥 Limit releases to pushes to tags

Typically usage of this action involves adding a step to a build that
is gated pushes to git tags. You may find `step.if` field helpful in accomplishing this
as it maximizes the reuse value of your workflow for non-tag pushes.

`v3` requires a GitHub Actions runtime that supports Node 24. If you still need the
last Node 20-compatible line, stay on `v2.6.2`.

Below is a simple example of `step.if` tag gating
>>>>>>> Stashed changes

```yaml
- uses: iShark5060/actions-gh-release@v1
  if: github.ref_type == 'tag'
  with:
    files: |
      dist/*.zip
      LICENSE
```

Needs `permissions: contents: write`. Add `discussions: write` if you set `discussion_category_name`. Inputs live in `action.yml`.

## Gotchas

<<<<<<< Updated upstream
- Reference a **published tag** (`@v1`). `dist/index.js` is only on release tags; `@main` will not work.
- New releases that upload `files` are created as **drafts**, assets go up, then the action publishes. Reusing an existing draft: set `draft: true` to keep it draft; **omit** `draft` to publish after upload. Prereleases **without** files publish immediately unless `draft: true`.
- Default `github.token` will **not** trigger other `on: release` workflows. Use a PAT when you need that chain.
- `files` is glob-based. Escape `[` / `]` in literal names. Windows accepts `\` and `/`. `working_directory` makes patterns relative to a subdirectory. GitHub may rewrite asset names that contain emoji or special characters.
- Unset `name` / `body` / `prerelease` on an existing release leave the old values. Prefer `body_concat_strategy` (`replace` / `append` / `prepend`) over `append_body`. `preserve_order` only serializes uploads; GitHub’s UI order is not controlled here.
- `token: ""` is treated as unset. Omit the input or use `${{ inputs.token || github.token }}` in a composite wrapper.
=======
```yaml
name: Main

on: push

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7
      - name: Generate Changelog
        run: echo "# Good things have arrived" > ${{ github.workspace }}-CHANGELOG.txt
      - name: Release
        uses: iShark5060/actions-gh-release@v1
        if: github.ref_type == 'tag'
        with:
          body_path: ${{ github.workspace }}-CHANGELOG.txt
          repository: my_gh_org/my_gh_repo
          # note you'll typically need to create a personal access token
          # with permissions to create releases in the other repo.
          # A non-empty explicit token overrides GITHUB_TOKEN.
          # Omit the input to use github.token; passing "" treats the token as unset.
          token: ${{ secrets.CUSTOM_GITHUB_TOKEN }}
```

When you use GitHub's built-in `generate_release_notes` support, you can optionally
pin the comparison base explicitly with `previous_tag`. This is useful when the default
comparison range does not match the release series you want to publish.

```yaml
- name: Release
  uses: iShark5060/actions-gh-release@v1
  with:
    tag_name: stage-2026-03-15
    target_commitish: ${{ github.sha }}
    previous_tag: prod-2026-03-01
    generate_release_notes: true
```

### 💅 Customizing

#### inputs

The following are optional as `step.with` keys

| Name                       | Type    | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| -------------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `body`                     | String  | Text communicating notable changes in this release                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `body_path`                | String  | Path to load text communicating notable changes in this release                                                                                                                                                                                                                                                                                                                                                                                                        |
| `draft`                    | Boolean | Keep the release as a draft. Defaults to false. When reusing an existing draft release, set this to true to keep it draft; omit it to publish after upload.                                                                                                                                                                                                                                                                                                            |
| `prerelease`               | Boolean | Indicator of whether or not is a prerelease                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `preserve_order`           | Boolean | Upload assets sequentially in the provided order. This controls the action's upload behavior, but it does not control the final asset ordering that GitHub may display on the release page or return from the Releases API.                                                                                                                                                                                                                                            |
| `files`                    | String  | Newline-delimited globs of paths to assets to upload for release. Escape glob metacharacters when you need to match a literal filename that contains them, such as `[` or `]`. `~/...` expands to the runner home directory. On Windows, both `\` and `/` separators are accepted. GitHub may normalize raw asset filenames that contain special characters; the action restores the asset label when possible, but the final download name remains GitHub-controlled. |
| `working_directory`        | String  | Base directory to resolve `files` globs against. Use this when release assets live under a subdirectory. If omitted, the action resolves `files` from `${{ github.workspace }}`.                                                                                                                                                                                                                                                                                       |
| `overwrite_files`          | Boolean | Indicator of whether files should be overwritten when they already exist. Defaults to true                                                                                                                                                                                                                                                                                                                                                                             |
| `name`                     | String  | Name of the release. defaults to tag name                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `tag_name`                 | String  | Name of a tag. defaults to `github.ref_name`. `refs/tags/<name>` values are normalized to `<name>`.                                                                                                                                                                                                                                                                                                                                                                    |
| `fail_on_unmatched_files`  | Boolean | Indicator of whether to fail if any of the `files` globs match nothing                                                                                                                                                                                                                                                                                                                                                                                                 |
| `repository`               | String  | Name of a target repository in `<owner>/<repo>` format. Defaults to GITHUB_REPOSITORY env variable                                                                                                                                                                                                                                                                                                                                                                     |
| `target_commitish`         | String  | Commitish value that determines where the Git tag is created from. Can be any branch or commit SHA. Defaults to repository default branch. When creating a new tag for an older commit, `github.token` may not have permission to create the ref; use a PAT or another token with sufficient contents permissions if you hit `403 Resource not accessible by integration`.                                                                                             |
| `token`                    | String  | Authorized GitHub token or PAT. Defaults to `${{ github.token }}` when omitted. A non-empty explicit token overrides `GITHUB_TOKEN`. Passing `""` treats the token as explicitly unset, so omit the input entirely or use an expression such as `${{ inputs.token                                                                                                                                                                                                      |     | github.token }}` when wrapping this action in a composite action. |
| `discussion_category_name` | String  | If specified, a discussion of the specified category is created and linked to the release. The value must be a category that already exists in the repository. For more information, see ["Managing categories for discussions in your repository."](https://docs.github.com/en/discussions/managing-discussions-for-your-community/managing-categories-for-discussions-in-your-repository)                                                                            |
| `generate_release_notes`   | Boolean | Whether to automatically generate the name and body for this release. If name is specified, the specified name will be used; otherwise, a name will be automatically generated. If body is specified, the body will be pre-pended to the automatically generated notes. See the [GitHub docs for this feature](https://docs.github.com/en/repositories/releasing-projects-on-github/automatically-generated-release-notes) for more information                        |
| `previous_tag`             | String  | Optional. When `generate_release_notes` is enabled, use this tag as GitHub's `previous_tag_name` comparison base. If omitted, GitHub chooses the comparison base automatically.                                                                                                                                                                                                                                                                                        |
| `append_body`              | Boolean | Append to existing body instead of overwriting it. Prefer `body_concat_strategy` for prepend/replace control.                                                                                                                                                                                                                                                                                                                                                          |
| `body_concat_strategy`     | String  | How to combine workflow body with an existing release body on update: `replace` (default), `append`, or `prepend`. When unset, `append_body: true` acts as `append`.                                                                                                                                                                                                                                                                                                   |
| `upload_checksums`         | Boolean | When true and `files` are uploaded, generate a `SHA256SUMS` file from the matched assets and upload it alongside them.                                                                                                                                                                                                                                                                                                                                                 |
| `make_latest`              | String  | Specifies whether this release should be set as the latest release for the repository. Drafts and prereleases cannot be set as latest. Can be `true`, `false`, or `legacy`. Uses GitHub api defaults if not provided                                                                                                                                                                                                                                                   |

💡 When providing a `body` and `body_path` at the same time, `body_path` will be
attempted first, then falling back on `body` if the path can not be read from.

💡 When the release info keys (such as `name`, `body`, `prerelease`, etc.) are not
explicitly set and there is already an existing release for the tag, the release
will retain its original info.

💡 Draft status is handled separately during finalization. If the action reuses an
existing draft release, set `draft: true` to keep it draft; if `draft` is omitted,
the action will publish that draft after uploading assets.

💡 GitHub immutable releases lock assets after publication. This action creates
standard releases as drafts, uploads assets, then publishes. Prereleases that
include `files` follow the same draft-first path. Prereleases **without** files
still publish immediately (so `release.prereleased` keeps firing) unless
`draft: true` is set.

💡 `files` is glob-based, so literal filenames that contain glob metacharacters such as
`[` or `]` must be escaped in the pattern.

💡 GitHub may normalize or rewrite uploaded asset filenames that contain special or
non-ASCII characters. This action uploads the requested file, but it cannot force the
final asset name that GitHub stores or returns from the Releases API. In particular,
4-byte Unicode characters such as emoji cannot currently be restored via asset labels.

#### outputs

The following outputs can be accessed via `${{ steps.<step-id>.outputs }}` from this action

| Name             | Type   | Description                                                                                                                      |
| ---------------- | ------ | -------------------------------------------------------------------------------------------------------------------------------- |
| `url`            | String | Github.com URL for the release                                                                                                   |
| `id`             | String | Release ID                                                                                                                       |
| `upload_url`     | String | URL for uploading assets to the release                                                                                          |
| `assets`         | String | JSON array of updated assets (GitHub API shape minus `uploader`). May include `digest` when GitHub returns it (e.g. `sha256:…`). |
| `tag_name`       | String | Tag name associated with the release                                                                                             |
| `created`        | String | `true` if this run created a new release; `false` if an existing release was updated                                             |
| `discussion_url` | String | Discussion URL linked to the release, when present                                                                               |

As an example, you can use `${{ fromJSON(steps.<step-id>.outputs.assets)[0].browser_download_url }}` to get the download URL of the first asset, or `.digest` when GitHub provides a content digest.

#### environment variables

The following `step.env` keys are allowed as a fallback but deprecated in favor of using inputs.

| Name                | Description                                                                                |
| ------------------- | ------------------------------------------------------------------------------------------ |
| `GITHUB_TOKEN`      | GITHUB_TOKEN as provided by `secrets`                                                      |
| `GITHUB_REPOSITORY` | Name of a target repository in `<owner>/<repo>` format. defaults to the current repository |

> **⚠️ Note:** This action was previously implemented as a Docker container, limiting its use to GitHub Actions Linux virtual environments only. With recent releases, we now support cross platform usage. You'll need to remove the `docker://` prefix in these versions

### Permissions

This Action requires the following permissions on the GitHub integration token:

```yaml
permissions:
  contents: write
```

When used with `discussion_category_name`, additional permission is needed:

```yaml
permissions:
  contents: write
  discussions: write
```

[GitHub token permissions](https://docs.github.com/en/actions/security-guides/automatic-token-authentication#permissions-for-the-github_token) can be set for an individual job, workflow, or for Actions as a whole.

Note that if you intend to run workflows on the release event (`on: { release: { types: [published] } }`), you need to use
a personal access token for this action, as the [default `secrets.GITHUB_TOKEN` does not trigger another workflow](https://github.com/actions/create-release/issues/71).
>>>>>>> Stashed changes

## License

MIT. See [LICENSE](LICENSE). Originally by Doug Tangren (softprops).
