---
name: resolve-merge-conflicts
description: Resolves Git merge conflicts safely by inspecting markers, choosing or combining changes, validating the result, and completing merge, rebase, or cherry-pick. Use when git reports conflicts, conflict markers appear in files, or the user asks to fix merge/rebase conflicts.
---

# Resolve merge conflicts

## Goals

- Leave the repo in a **consistent** state: no conflict markers, intended logic preserved, builds/tests pass when the user expects them to.
- Finish the **in-progress Git operation** (`merge`, `rebase`, or `cherry-pick`) correctly.

## 1. Identify the operation and scope

Run or infer:

- `git status` — lists *unmerged paths* and whether merge, rebase, or cherry-pick is in progress.
- Note whether the user was merging branches, rebasing, or cherry-picking; continuation commands differ (see section 6).

## 2. List conflicted files

- `git diff --name-only --diff-filter=U`
- Open each file and search for conflict markers: `<<<<<<<`, `=======`, `>>>>>>>`.

Resolve **every** conflicted file before staging.

## 3. Read conflict markers

Typical shape:

```text
<<<<<<< HEAD
(current branch / upstream commit version)
=======
(incoming: merged-in branch, replayed commit, or cherry-pick)
>>>>>>> branch-or-commit-label
```

- **HEAD** (or first side): what you had before bringing in the other change (exact meaning depends on merge vs rebase; treat as “current side” of the operation).
- **Second side**: incoming changes.

Some tools leave one side empty; still decide what the final line or block should be.

## 4. Resolve the content

Pick a strategy per hunk:

| Situation | Action |
|-----------|--------|
| One side is clearly wrong or obsolete | Keep the other side; delete markers and the discarded block. |
| Both changed the same logic | Merge behavior intentionally: combine imports, preserve both features, dedupe. |
| Formatting-only | Normalize to project style (formatters, project conventions). |
| Generated or lockfile conflicts | Prefer regenerating from source (`package-lock.json`, etc.) if project workflow says so; otherwise merge carefully. |

**Do not** leave `<<<<<<<`, `=======`, or `>>>>>>>` in committed text.

After editing:

- Ensure the file **parses** and **typechecks** where applicable (syntax errors in conflict resolution are common).

## 5. Mark conflicts resolved

For each fixed file:

```bash
git add <path>
```

Use `git add -u` only when appropriate; prefer explicit paths for large conflict sets.

Verify:

```bash
git diff --check
git status
```

There should be **no** unmerged paths before continuing.

## 6. Complete the Git operation

- **Merge**: `git merge --continue` if a merge paused with conflicts, or simply commit if Git created a merge commit stage (follow `git status` guidance). If merging and conflicts are resolved: `git commit` completes the merge when Git expects a commit.
- **Rebase**: `git rebase --continue` after staging. To abort: `git rebase --abort`.
- **Cherry-pick**: `git cherry-pick --continue` after staging. To abort: `git cherry-pick --abort`.

Always prefer **`git status`** output: Git prints the exact next command.

## 7. Validate

After the operation completes:

- Run project checks the user cares about (e.g. lint, typecheck, targeted tests) when available.
- If `main` was updated during a long merge, consider whether a quick **re-test** is needed before push.

## 8. When stuck

- **Binary files**: Git may not show markers; choose ours/theirs or replace file manually, then `git add`. Example helpers: `git checkout --ours|--theirs -- path` (understand which side you want first).
- **`deleted by us/them`**: Decide whether the file should exist; `git add` the final path or `git rm` as appropriate.
- **Submodules or nested repos**: Resolve in the submodule first, commit there, then update the parent pointer.

## 9. Project conventions

If the repository documents workflow (e.g. `CONTRIBUTING.md`, `CLAUDE.md`), follow naming, imports, and conflict-resolution preferences (e.g. lockfile policy) documented there.
