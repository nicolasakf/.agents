---
name: npm-release
description: Bumps package semver, commits release changes, tags, and publishes to npm non-interactively via publish-npm.sh (NPM_TOKEN in ~/.zprofile), then post-release verification. Use when cutting a release, running npm publish, bumping package.json version, or shipping a new npm package version.
disable-model-invocation: true
---

# npm release (version, commit, tag, publish)

End-to-end release for npm packages. **Differs from the standalone git-commit skill**: this workflow **must use** `git tag`, `npm version` / `npm publish`, and usually `git push` to publish tags and commits — those commands are required here when the user wants a full release.

For **commit message quality, staging discipline, and atomic grouping**, align with [git-commit](../git-commit/SKILL.md): inspect `git status` / `git diff`, stage explicit paths, conventional-commit style, imperative subject, body only when useful.

## Automated workflow

**Phase 1 — Prepare:** pre-flight → version bump → commit → build → tag/push.

**Phase 2 — Publish:** run `publish-npm.sh` (non-interactive via `NPM_TOKEN`).

**Phase 3 — Verify:** post-release registry checks immediately after Phase 2 succeeds.

If the user invokes this skill and has already completed Phase 1 (tag exists for the target version), skip to Phase 2 (publish) or Phase 3 (verify only) as appropriate.

Only create the release commit when the user asked to publish — **invoking this skill counts**.

## Publish credentials (one-time setup)

Store tokens in `~/.zprofile` (or export manually). **Never commit tokens.**

| Variable | Purpose |
| --- | --- |
| `NPM_TOKEN` | npm granular access token — **Read and write**, scoped to the package, **Bypass 2FA** checked |

Create the npm token at [npmjs.com → Access Tokens](https://www.npmjs.com/settings/~npm/access-tokens).

Agent shells may not load `~/.zprofile` automatically. The publish script sources it when `NPM_TOKEN` is missing; if pre-flight still fails, run `source ~/.zprofile` first.

Scoped public packages also need `"publishConfig": { "access": "public" }` in `package.json`.

## Prerequisites

- Clean release branch (often `main` / `master`; confirm with the user if the repo uses something else)
- No accidental unrelated edits in the release commit — split commits (atomic rule from git-commit) before tagging
- Publish credentials in environment (`bash publish-npm.sh --check` from package root)
- Project checks green if they exist (`npm test`, `npm run typecheck`, etc.)
- Run **`npm run build`** when `files` or publish rely on `dist/` (or similar)

## Release checklist

```
Phase 1 — Prepare
- [ ] 1. Pre-flight validation (+ credential check)
- [ ] 2. Decide version (patch / minor / major / explicit semver)
- [ ] 3. Bump version (package.json + lockfile)
- [ ] 4. Commit release prep
- [ ] 5. Build (if required for publish)
- [ ] 6. Tag and push

Phase 2 — Publish
- [ ] 7. Run publish-npm.sh

Phase 3 — Verify
- [ ] 8. Post-release verification
- [ ] 9. Clean up temp artifacts (npm pack .tgz, dry-run scratch)
```

---

## Phase 1 — Prepare

### Step 1: Pre-flight validation

From package root:

```bash
git status --short
npm test                    # if the project has tests
npm run typecheck           # or: npx tsc --noEmit, if applicable
npm run lint                # if applicable
bash /path/to/skills/npm-release/publish-npm.sh --check
```

Do not publish from a dirty tree unless the user explicitly wants unreleased local changes included. Resolve or stash unrelated work first.

Review `git log` since the last release tag. If a release tag for the target version already exists and npm still shows an older version, bump to the next patch instead of reusing the tag.

### Step 2: Version bump

Choose **one** path:

#### A. `npm version` (recommended when defaults fit)

Runs lifecycle scripts and updates `package.json` and lockfile (when present).

```bash
npm version patch   # or minor | major | 1.2.3
```

By default this **commits** with message like `0.3.0` and creates an **annotated tag** `v0.3.0`. Skip the separate commit/tag steps below if this single step already did both.

Use **`npm version <type> --no-git-tag-version`** when you need to commit or tag manually (e.g. monorepo policy, custom message, CHANGELOG, or splitting steps).

#### B. Manual bump

Edit `version` in `package.json` (and lockfile if the team keeps it in sync), then follow **Step 3** and **Step 5** below.

**After any bump**, run install or lock refresh only if the repo requires it (e.g. `npm install` when lock semantics demand it).

### Step 3: Commit (when `npm version` did not commit)

If you used `--no-git-tag-version` or bumped manually:

1. Inspect changes: `git status`, `git diff`.
2. Stage only release-related files (typically `package.json`, `package-lock.json` / `npm-shrinkwrap.json`, and `CHANGELOG.md` if the project keeps one).
3. Message: conventional commits, e.g. `chore(release): v0.3.0` or `chore(release): bump version to 0.3.0`. Match **complexity** to the change (see git-commit).

```bash
git add package.json package-lock.json   # adjust paths
git commit -m "chore(release): v0.3.0"
```

### Step 4: Build (when publish depends on it)

```bash
npm run build    # or npm run prepack — follow project norms
npm pack --dry-run
```

Confirm tarball contents and size before publishing. Large packages: check for accidental dev artifacts (logs, `.env`, local caches).

Optional local smoke test:

```bash
npm pack
npm install -g ./<package-name>-<version>.tgz
# exercise the CLI or entry point
npm uninstall -g <package-name>
```

### Step 5: Tag and push

Tag **must** match the version in `package.json`. npm’s default tag name is `v` + semver (e.g. `v0.3.0`).

```bash
git tag -a "v0.3.0" -m "v0.3.0"   # skip if npm version or an existing tag already created it
git push origin main              # use the branch the user specifies if not main
git push origin v0.3.0            # or: git push --tags
```

Use lightweight tags only if the project standard requires it: `git tag "v0.3.0"`.

---

## Phase 2 — Publish

From package root, after Phase 1 is complete:

```bash
bash publish-npm.sh
```

For scoped public packages:

```bash
bash publish-npm.sh --access public
```

### Publish script

The skill ships [publish-npm.sh](./publish-npm.sh). Run it from the **package root** (directory containing `package.json`):

```bash
bash /path/to/skills/npm-release/publish-npm.sh --check
bash /path/to/skills/npm-release/publish-npm.sh
```

Repos may copy or symlink this script to `scripts/publish-npm.sh` for a stable path. For Orion multi-registry releases (npm + PyPI), use the `publish-orion-release` skill instead.

The script:

1. Sources `~/.zprofile` if `NPM_TOKEN` is not already exported
2. Validates credentials with `npm whoami`
3. Publishes via a temporary `.npmrc` (no interactive OTP prompt)

Do **not** echo or log token values. On failure, diagnose from the error output and the troubleshooting table — do not re-run publish automatically unless the user asks.

**Legacy fallback** (only when `NPM_TOKEN` is unavailable and the user explicitly opts into interactive publish):

```bash
npm publish                  # add --access public for scoped packages
# 2FA: npm publish --otp=<code>
```

---

## Phase 3 — Post-release verification

Run immediately after Phase 2 succeeds (no user "done" reply needed).

Read the target version from `package.json` (or the release tag just cut). Read the package name from `package.json` `"name"`. Run all checks; report pass/fail for each.

### Registry version check

```bash
npm view <package-name> version
```

Must match the release version (e.g. `0.3.0`).

For dist-tags or pre-releases:

```bash
npm view <package-name> versions --json
npm dist-tag ls <package-name>
```

### Optional live smoke test (run if environment allows)

```bash
npx <package-name>@<version>
# or: npm install -g <package-name>@<version>
```

### Verification checklist

Report results for:

- [ ] `npm view <package-name> version` matches release
- [ ] Remote has the release commit and tag (`git ls-remote --tags origin`)
- [ ] (optional) `npx <package-name>@<version>` works as expected

If any check fails, diagnose using the troubleshooting table and tell the user what to fix — do **not** re-run publish commands automatically.

### Cleanup

Remove release temp files: local `npm pack` tarballs and any other scratch artifacts — never commit them.

---

## Publish order (critical)

1. Version bump (+ CHANGELOG if the project keeps one)
2. Commit release prep (when not handled by `npm version`)
3. Build (if publish depends on compiled output)
4. Tag + push
5. **`publish-npm.sh`** (non-interactive npm publish)
6. Post-release verification

---

## Edge cases

- **Pre-release**: `npm version prerelease --preid=beta` (or `npm publish --tag beta`) — confirm tag/dist-tag strategy with the user.
- **`npm version` failed on git**: Often due to dirty tree or hooks; clean or use `--no-git-tag-version` and commit/tag manually.
- **Monorepos**: Prefer the workspace’s tool (`changeset`, `lerna`, `nx release`, etc.) if present; this skill is for single-package or manual flows.
- **GitHub release**: Optional for npm-only packages. Create with `gh release create` when the project ships release notes or binary assets alongside npm.

## Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `Missing publish credentials` | `NPM_TOKEN` not in agent shell | `source ~/.zprofile` or re-run terminal; verify with `--check` |
| npm `EOTP` | Token lacks Bypass 2FA | Regenerate npm granular token with **Bypass 2FA** checked |
| npm `401` | Invalid or expired `NPM_TOKEN` | Regenerate token; update `~/.zprofile` |
| npm `403` | Token lacks write access to package | Scope token to the package or use org-level token with publish rights |
| npm `404` on publish | Wrong scope/name or not logged in | Confirm `npm whoami`; check `npm view <package-name>` |
| npm `413 Payload Too Large` | Accidental large files in tarball | Fix `.npmignore` / `files`; re-run build; `npm pack --dry-run` |
| Verification: npm still on old version | Publish not finished or registry lag | Wait a minute and re-check; re-run `publish-npm.sh` if publish failed |
| `npm version` hook failure | preversion/version/postversion script error | Fix script or use `--no-git-tag-version` and commit manually |

## Notes

- Do not run `npm run dev` during release unless the user asks; use production build / `prepack`.
- Do not print, commit, or log `NPM_TOKEN`.
- Do not force-push tags or rewrite published versions without explicit user approval.
- For breaking changes, bump minor/major semver and call them out in CHANGELOG when the project keeps one.

## Additional resources

- Publish script: [publish-npm.sh](./publish-npm.sh)
- Commit discipline: [git-commit](../git-commit/SKILL.md)
- Orion multi-registry releases: `publish-orion-release` skill (in the Orion repo)
