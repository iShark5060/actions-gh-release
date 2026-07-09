# actions-gh-release

GitHub Action for creating and updating GitHub Releases with asset uploads.

## Architecture

- `src/index.ts` — orchestration: parse config, validate inputs, create/update release, upload assets, finalize, set outputs
- `src/github.ts` — release semantics, GitHub API interaction, race handling
- `src/util.ts` — parsing and path normalization
- `action.yml` — action metadata
- `dist/index.js` — published CJS bundle (checked in on `main` for CI freshness checks)

Keep behavior-specific logic in `src/github.ts` or `src/util.ts`; avoid growing `src/index.ts` with ad-hoc feature branches.

## Core rules

- Prefer narrow behavior fixes over structural churn.
- Reproduce current behavior on `main` before changing code.
- Treat GitHub platform behavior as distinct from action behavior.
- Be careful with parsing changes around `files`, path handling, and Windows compatibility.

## Contract sync

When behavior changes, update:

- `README.md`
- `action.yml`
- tests under `tests/`
- regenerate `dist/index.js` with `pnpm run build`

## Verification

```bash
pnpm run validate
pnpm run build
```

## Release

1. Create a pre-release from `main`
2. Verify the Release workflow completes
3. Promote to a full release when ready
