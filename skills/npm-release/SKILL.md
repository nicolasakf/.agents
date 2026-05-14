---
name: npm-release
description: Bumps package semver, commits release changes with conventional commits, tags the release, and publishes to npm. Use when cutting a release, running npm publish, bumping package.json version, or shipping a new npm package version.
disable-model-invocation: true
---

# npm release (version, commit, tag, publish)

End-to-end release for npm packages. **Differs from the standalone git-commit skill**: this workflow **must Use** `git tag`, `npm version` / `npm publish`, and usually `git push` to publish tags and commits—those commands are required here when the user wants a full release.

For **commit message quality, staging discipline, and atomic grouping**, align with [git-commit](../git-commit/SKILL.md): inspect `git status` / `git diff`, stage explicit paths, conventional-commit style, imperative subject, body only when useful.

## Preconditions

- **Branch**: Prefer the repo’s release branch (often `main` / `master`). Confirm with the user if the repo uses something else.
- **Clean intent**: No accidental unrelated edits in the release commit. If the working tree mixes release work with other changes, split commits (atomic rule from git-commit) before tagging.
- **CI green**: Run project checks if they exist (`npm test`, `npm run typecheck`, etc.). Run **`npm run build`** when `files` or publish rely on `dist/` (or similar).
- **Auth**: `npm whoami` should succeed; scoped public packages need `"publishConfig": { "access": "public" }` in `package.json`.

## Version bump

Choose **one** path:

### A. `npm version` (recommended when defaults fit)

Runs lifecycle scripts and updates `package.json` and lockfile (when present).

```bash
npm version patch   # or minor | major
```

By default this **commits** with message like `0.3.0` and creates an **annotated tag** `v0.3.0`. Skip the separate “commit + tag” steps below if this single step already did both.

Use **`npm version <type> --no-git-tag-version`** when you need to commit or tag manually (e.g. monorepo policy, custom message, or splitting steps).

### B. Manual bump

Edit `version` in `package.json` (and lockfile if the team keeps it in sync), then follow **Commit** and **Tag** below.

**After any bump**, run install or lock refresh only if the repo requires it (e.g. `npm install` when lock semantics demand it).

## Commit (when `npm version` did not commit)

If you used `--no-git-tag-version` or bumped manually:

1. Inspect changes: `git status`, `git diff`.
2. Stage only release-related files (typically `package.json`, `package-lock.json` / `npm-shrinkwrap.json`).
3. Message: conventional commits, e.g. `chore(release): 0.3.0` or `chore(release): bump version to 0.3.0`. Match **complexity** to the change (see git-commit).

```bash
git add package.json package-lock.json   # adjust paths
git commit -m "chore(release): 0.3.0"
```

## Tag (when not already created by `npm version`)

Tag **must** match the version in `package.json`. npm’s default tag name is `v` + semver (e.g. `v0.3.0`).

```bash
git tag -a "v0.3.0" -m "v0.3.0"
```

Use lightweight tags only if the project standard requires it: `git tag "v0.3.0"`.

## Publish

From package root, after build if needed:

```bash
npm publish
```

- **One-time / CI**: `npm publish --dry-run` to verify contents.
- **2FA on npm**: pass OTP when prompted, or `npm publish --otp=<code>` if the environment requires it.

## Push commits and tags (almost always required)

So the remote has the same version and tag:

```bash
git push
git push --tags
```

Use the branch name explicitly if needed (e.g. `git push origin main`).

## Checklist

- [ ] Version in `package.json` matches intended release; lockfile updated if applicable
- [ ] Tests / typecheck / build run per project norms
- [ ] Commit exists with appropriate message; no unrelated files in that commit
- [ ] Tag points at that commit and matches version (`v` prefix per convention)
- [ ] `npm publish` succeeds; `npm whoami` verified beforehand
- [ ] Pushes done so remote has commit + tag

## Edge cases

- **Pre-release**: `npm version prerelease --preid=beta` (or `npm publish --tag beta`)—confirm tag/dist-tag strategy with the user.
- **`npm version` failed on git**: Often due to dirty tree or hooks; clean or use `--no-git-tag-version` and commit/tag manually.
- **Monorepos**: Prefer the workspace’s tool (`changeset`, `lerna`, `nx release`, etc.) if present; this skill is for single-package or manual flows.
