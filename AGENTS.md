# actions-gh-release

## Org standards

CI/README/validate conventions live in AppBase `docs/org-standards/` with personal-repo overrides (`personal-repos.md`). GitHub-hosted runners, not Blacksmith. Action-publish track: `release` event → build-and-tag. Quality gate: `pnpm run validate`.

## Overview

Creates/updates GitHub Releases and uploads assets (glob `files`, optional checksums). Maintained fork of softprops/action-gh-release. This fork’s published major is `@v1` (Node 24); do not confuse with upstream softprops `v2` / `v3` tags.

## Behavior

Prefer narrow fixes. Keep release/upload/race logic in `src/github.ts` and parsing/paths in `src/util.ts`. Do not grow ad-hoc branches in `src/index.ts`. Bundled `dist/index.js` is only on release tags. CI builds and verifies the bundle; committing `dist/` is not required for PR validate.

New releases that will upload assets are created as **drafts**, assets upload, then finalize publishes. Prereleases **without** files publish immediately unless `draft: true` (so `release.prereleased` still fires). When reusing an existing draft: set `draft: true` to keep it draft; **omit** `draft` to publish after uploads.

`preserve_order` only serializes uploads; GitHub’s release UI/API order is not controlled by this action. Asset filenames with special/emoji characters may be rewritten by GitHub. Unset `name` / `body` / `prerelease` on update leave the existing release values alone. Prefer `body_concat_strategy` (`replace` / `append` / `prepend`) over `append_body`.

`token: ""` clears the input override and falls back to env `GITHUB_TOKEN`. Composite wrappers should omit the input or use `${{ inputs.token || github.token }}`. Default `github.token` will not trigger other workflows on `release` events; use a PAT when that chaining is required. Be careful with `files` glob parsing (brace-aware), Windows `\`/`/`, and literal glob metacharacters.
