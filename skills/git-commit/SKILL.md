---
name: git-commit
description: Inspects the working tree, groups changes into atomic commits, stages explicit paths, generates conventional commit messages (succinct or detailed), and commits. With no chat context, commits pending work in every opened IDE workspace. In a populated chat, prefer staging only session-related files unless the user specifies otherwise. Use when the user wants to commit, asks to commit, or needs a commit message.
disable-model-invocation: true
---

# Commit Changes

**Do not stop because nothing is staged.** An empty index means you must inspect **all** pending work (`git status`, `git diff` for tracked changes, and enumerate/review untracked paths), apply the atomic grouping rule, **`git add` only the paths that belong to each commit**, then commit. Staging is part of this command—not a prerequisite the user must do first.

Analyzes whatever ends up staged (after you stage it), generates an appropriate commit message (succinct or detailed based on complexity), and commits. Follow this workflow.

## Commit scope (which repos and files to include)

Resolve scope in this order:

1. **User override (highest priority)** — If the user names repos, paths, or a scope ("only Orion-app", "commit the website changes", "everything in all workspaces", "don't touch docs"), follow that exactly. User instructions beat defaults below.
2. **Populated chat session** — If the chat already has context (prior turns, edits, `@` references, or tool-driven changes in this thread) and the user did **not** ask for a broader scope, treat the commit as **scoped to this session** (see *Chat session scope* below).
3. **No chat context (default)** — If the chat is empty or has no meaningful session context, commit pending work in **every opened IDE workspace** (see *Multi-workspace default* below).

When scope is unclear inside a populated chat, infer from the diffs (one feature, one bugfix, one docs pass, etc.) and proceed. **Ask the user only** when two unrelated large changes are mixed in the same files and splitting would require guessing.

## Multi-workspace default (no chat context)

When the default applies, run the full commit workflow **in each opened workspace root**, independently:

1. **Discover workspace roots** — Use `Workspace Paths` from the environment when present. Each listed path is a candidate root. If that list is missing, infer roots from `git_status` blocks, recently edited absolute paths, or other environment hints; do not guess random directories.
2. **One repo at a time** — For each root, run all git commands with `working_directory` set to that path (or `cd` there first). Never assume the shell's current directory is the only repo.
3. **Skip clean repos** — If `git status` shows a clean working tree for a root, skip it and move on.
4. **Atomic commits per repo** — Within each repo, still follow the atomic grouping rule. Multiple commits in one repo are fine.
5. **Summarize** — After all repos are processed, report which workspaces were committed, which were skipped (clean), and any that still have unstaged/uncommitted work.

The rest of this document applies to **each repo in scope**, one at a time.

## Chat session scope (populated chat, no user override)

If this command runs **inside a chat that already has context** and the user did not specify a broader or different scope, treat the commit as **scoped to this session**:

1. **Infer the file set for this session** — Include paths that are clearly part of *this* conversation only, for example:
   - Files you or the user edited in this chat (diffs, writes, search-replace outcomes)
   - Files explicitly attached or mentioned with `@` in the thread
   - New files created to satisfy the task discussed here
2. **Stage only those paths** — Use `git add <path> ...` for that set. Do **not** use `git add .`, `git add -A`, or blanket staging that would pick up unrelated local changes.
3. **Leave everything else unstaged** — Other modified or untracked files stay out of the commit. After committing, briefly note if significant unrelated changes remain (`git status`).
4. **Multi-workspace sessions** — If session files span multiple opened workspaces, run the workflow in each affected repo, staging only session paths in that repo.

The rest of this document applies as usual to **whatever ends up staged** (message quality, atomic commits within that scope, etc.).

## ⛔ CRITICAL: Allowed Git Commands Only

**It is absolutely prohibited to run any git command that is not directly required for committing.**

**Allowed commands only:**
- `git status`
- `git diff --staged` (and `git diff` for unstaged, when grouping or scoping)
- `git add <paths>` — prefer explicit paths; use `.` or `-A` **only** when the user explicitly asks to commit the entire working tree, or every pending change in that repo is a single atomic unit
- `git commit -m "..."` (or `-F -`) — **never use `--trailer` or any other flags**
- `git restore --staged <file>` or `git reset HEAD <file>` (to unstage when splitting)

**Prohibited:** `git push`, `git pull`, `git fetch`, `git merge`, `git rebase`, `git checkout`, `git switch`, `git branch`, `git stash`, `git reset --hard`, or any other git command not listed above. Do not run them.

**Never use `git commit --trailer`** — Do not add trailers (e.g. "Made with Cursor" or similar). The commit command must use only `-m` or `-F -` for the message.

## Workflow

### 0. Group changes by commit (atomic rule)

**Before committing, group all changes into logical units.** Inspect both staged and unstaged changes (`git status`, `git diff`, `git diff --staged`) to see the full picture.

**If a populated chat session applies** (see *Chat session scope* above), first **restrict the candidate files** to the session file set. Run atomic grouping and multiple commits only *within* that set. Do not pull in off-topic modified files.

**If the multi-workspace default applies**, repeat this step in each opened workspace that has pending changes.

**The "Atomic" Rule:** A commit is "atomic" if it does one thing and does it completely. If you were to revert that commit, the system should still build and run, but only that specific feature or fix would disappear.

- **Bad:** One commit containing a bug fix, a new UI button, and a typo correction in a README.
- **Good:** Three separate commits—one for each task.

**If there are 2, 3, or N distinct groups of changes** (each group doing one atomic thing), then:
1. Stage the first group with `git add <paths>`
2. Commit it with an appropriate message
3. Repeat for each remaining group until everything **in scope** (session paths, user-specified paths, or all pending work in the repo) is committed

Do not lump unrelated changes into a single commit. Split and commit each logical unit separately.

### 1. Inspect changes and ensure something is staged

```bash
git status
git diff
git diff --staged
```

Use `git diff` (and file reads if needed) for modified tracked files; list and review untracked files/directories so you know what would be added.

- **Nothing staged?** This is normal. Continue: apply *§0 Group changes by commit*, then `git add <explicit paths>` for the first atomic group—never skip committing solely because the index was empty.
- **Populated chat with clear session files?** Stage only those paths (and only the groups that belong to the session).
- **Multi-workspace default?** In each opened workspace with pending changes, stage all paths that belong together for the first atomic commit (explicit paths; use `git add .` / `-A` only when every pending change in that repo is one logical unit).
- **No session scope, single repo?** Stage all paths that belong together for the first atomic commit (explicit paths; avoid `git add .` / `-A` unless the user asked to commit the entire working tree or every change is one logical unit).
- **Something already staged?** If it matches your intended group, proceed to step 2. If not, unstage with `git restore --staged` / `git reset HEAD` and re-stage the right set.

### 2. Analyze the diff

Review **`git diff --staged`** (after staging) to understand:

- **What changed**: Files, functions, logic
- **Why it matters**: Bug fix, feature, refactor, config, docs
- **Scope**: Which area (auth, api, ui, db, etc.)
- **Breaking changes**: Any API or behavior changes

### 3. Generate the commit message

Use conventional commits format. **Match message length to change complexity:**

- **Simple changes** (typo fix, single-file tweak, config bump) → succinct message, subject only
- **Many changes or complex logic** → detailed body with bullets
- Default to succinct; add a body only when it adds real value

**Structure:**

```
<type>(<scope>): <subject>

<body>   ← omit for simple changes

<footer>
```

**Types:** `feat`, `fix`, `refactor`, `docs`, `style`, `test`, `chore`, `perf`

**Subject:** Imperative mood, ~50 chars, no period. Example: "add user auth" not "added user auth"

**Body:** (Only when changes warrant it.) Include:
- What changed and why
- Key implementation details
- Affected components or flows
- Any notable decisions

**Footer:** Optional. Use for `BREAKING CHANGE:` or `Closes #123`. Do not add promotional trailers (e.g. "Made with Cursor").

**Simple example (subject only):**
```
fix(ui): correct typo in button label
```

**Detailed example (when many changes):**
```
feat(chat): add streaming support for agent responses

- Integrate SSE in chat route for token-by-token streaming
- Add useStreamingChat hook to consume stream and update UI
- Handle partial JSON parsing for tool calls in flight
- Add error recovery and reconnection logic
```

### 4. Commit

Use only `-m` or `-F -` for the message. **Never use `--trailer`.**

```bash
git commit -m "<subject>" -m "<body>"
```

Or use a single multi-line message:

```bash
git commit -F - <<'EOF'
<type>(<scope>): <subject>

<body>
EOF
```

## Message quality checklist

- [ ] Type matches the change (feat/fix/refactor/etc.)
- [ ] Subject is imperative and concise
- [ ] Message length matches complexity (succinct for simple, detailed for complex)
- [ ] Body (if present) explains what, why, and key details
- [ ] No vague phrases ("various fixes", "update stuff")
- [ ] Breaking changes called out in footer if any
- [ ] No promotional trailers (e.g. "Made with Cursor"); never use `--trailer` in the commit command

## Edge cases

- **User wants one repo only?** Commit only that workspace; leave other repos untouched even under the multi-workspace default.
- **User wants all workspaces in a populated chat?** Treat as a user override: commit pending work in every opened workspace, not just session files.
- **Session scope vs. already-staged files?** If the user (or a prior step) staged files outside the session set, unstage those with `git restore --staged <file>` or `git reset HEAD <file>`, then stage only session paths before committing.
- **Multiple unrelated changes staged?** Follow the atomic rule (step 0): unstage with `git reset HEAD <file>`, then stage and commit each group separately.
- **Merge conflicts or dirty state?** Resolve before committing.
- **Nothing to commit?** Only skip when `git status` shows a **clean** working tree (no pending changes to include). If there are modified or untracked files that belong in the commit(s), go back to step 1—stage explicit paths and commit; **do not treat an empty index as a blocker.**
- **Truly empty tree?** No modifications and nothing untracked (or user wants nothing included)—then explain there is no work to commit.
- **Non-git workspace root?** Skip that path and note it in the summary.
