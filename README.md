# GH Release

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE)
[![CI](https://img.shields.io/github/actions/workflow/status/iShark5060/actions-gh-release/ci.yml?style=flat-square&label=CI)](https://github.com/iShark5060/actions-gh-release/actions/workflows/ci.yml)
![Node](https://img.shields.io/badge/Node-%3E%3D24-339933?logo=node.js&logoColor=white&style=flat-square)
![TypeScript](https://img.shields.io/badge/TypeScript-7.x-3178C6?logo=typescript&logoColor=white&style=flat-square)
[![Cursor](https://img.shields.io/badge/Cursor-IDE-141414?logo=cursor&logoColor=white&style=flat-square)](https://cursor.com)

Creates GitHub Releases and uploads assets. Maintained fork of [softprops/action-gh-release](https://github.com/softprops/action-gh-release). This fork’s published major is `@v1` (Node 24) — not upstream `v2` / `v3`.

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

- Reference a **published tag** (`@v1`). `dist/index.js` is only on release tags; `@main` will not work.
- New releases that upload `files` are created as **drafts**, assets go up, then the action publishes. Reusing an existing draft: set `draft: true` to keep it draft; **omit** `draft` to publish after upload. Prereleases **without** files publish immediately unless `draft: true`.
- Default `github.token` will **not** trigger other `on: release` workflows. Use a PAT when you need that chain.
- `files` is glob-based. Escape `[` / `]` in literal names. Windows accepts `\` and `/`. `working_directory` makes patterns relative to a subdirectory. GitHub may rewrite asset names that contain emoji or special characters.
- Unset `name` / `body` / `prerelease` on an existing release leave the old values. Prefer `body_concat_strategy` (`replace` / `append` / `prepend`) over `append_body`. `preserve_order` only serializes uploads; GitHub’s UI order is not controlled here.
- `token: ""` is treated as unset. Omit the input or use `${{ inputs.token || github.token }}` in a composite wrapper.

## License

MIT. See [LICENSE](LICENSE). Originally by Doug Tangren (softprops).
