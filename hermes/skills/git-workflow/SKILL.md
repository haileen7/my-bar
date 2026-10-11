---
name: git-workflow
description: "Git pitfalls: nested repos, staging, identity."
version: 1.0.0
author: Sydney
license: MIT
metadata:
  hermes:
    tags: [git, workflow, pitfalls, staging, nested-repos, commit]
    category: software-development
---

# Git Workflow

Pitfalls and procedures for everyday git operations that fall outside standard
`gh` CLI workflows (for gh-specific flows see the `github` skill).

## Standing Rules

- Always check `git status` and `git remote -v` before staging or pushing.
- Configure per-repo identity before first commit if global config is absent.
- Never assume a branch name — read it from `git branch --show-current`.

## Pitfalls

### Syncing a fork branch against a canonical upstream

When the local remote is a fork and the real base is elsewhere, add the
canonical repo as a **named remote-tracking ref** rather than a permanent remote,
so no existing remote config is modified:

```bash
git fetch https://github.com/<owner>/<repo>.git <base-branch>:refs/remotes/canonical/<base-branch>
git rev-list --left-right --count HEAD...canonical/<base-branch>   # ahead<TAB>behind
git log --oneline HEAD..canonical/<base-branch>                    # incoming
git merge --ff-only canonical/<base-branch>
```

Fetch the URL separately (`git ls-remote <url> <branch>`) to confirm the
upstream SHA before merging — a fork's own tracking branch can be stale or
diverged, and comparing against it answers the wrong question.

User preference: the fork's `beta` must be EXACTLY the canonical upstream `beta` — use `--force` when the fork has diverged, not a ff merge:

```bash
git push origin refs/remotes/canonical/<base>:refs/heads/<base> --force
git fetch origin <base> && git merge --ff-only origin/<base>   # working branch catches up
```

Pushing a working branch to the fork by the same refspec creates it on the
remote when missing — `git push origin <server-branch>` is enough; no separate
`git branch --track` / `git push -u` dance needed:

```bash
git push origin <server-branch>   # creates origin/<server-branch> if absent
```

After fast-forwarding the working branch to canonical, also advance the
fork's own base branch so it stops lagging canonical — otherwise the fork's
`beta` and the server branch drift apart again on the next session:

```bash
git push origin refs/remotes/canonical/<base>:refs/heads/<base>   # fork's base catches up
git merge --ff-only refs/remotes/canonical/<base>                # working branch advances
git push origin HEAD:refs/heads/<server-branch>
```

**Two different local-state blockers abort a `--ff-only` merge.** Diagnose which
one you hit by reading the exact error line before acting.

1. `untracked working tree files would be overwritten by merge` — a boot-time
   file (`AGENTS.md`, `.hermes.md`, editor settings) was written locally every
   session start and later committed upstream. Delete the local copy, merge, let
   the tracked version land, then diff the two to confirm nothing local was lost.
   Do not resolve it by adding the path to `.git/info/exclude` — that guarantees a
   permanent divergence from upstream.
2. `Your local changes to the following files would be overwritten by merge` — an
   already-TRACKED file carries a local diff and upstream also changed it, so the
   fast-forward cannot apply cleanly. Do NOT blind `git checkout -- <file>`: that
   silently discards local work. Diff first, then branch on the answer:
   ```bash
   git diff <file>                  # what local change exists
   git diff HEAD canonical/<base> -- <file>   # what upstream changed
   ```
   - If the two diffs are **identical**, the local edit is a duplicate of upstream's
     (common after a manual dependency bump on one server) — `git checkout -- <file>`
     then merge; the incoming change lands the same content anyway.
   - If they **differ**, `git stash push -- <file>`, merge, then `git stash pop` and
     resolve the conflict deliberately.

**One long-lived working branch cannot carry a second open PR.** GitHub allows only one open PR per (head, base) pair, so a server branch that accumulates several unrelated features across sessions can never open a clean, single-topic PR — `gh pr create` fails with *"a pull request for branch X into branch Y already exists"*. Do not merge the new work into the stale PR and do not close it without asking; the previous PR may be someone's reviewed work.

Build a throwaway feature branch from the canonical base instead, and move only the new commits onto it:

```bash
git fetch https://github.com/<owner>/<repo>.git <base>:refs/remotes/canonical/<base>
git branch <topic-slug> canonical/<base>
git checkout <topic-slug>
git cherry-pick <new-commit> [<new-commit> ...]   # NOT the older ones already in the open PR
# re-verify the gates on the new base, then:
git push -u origin <topic-slug>
gh pr create --repo <owner>/<repo> --base <base> --head <fork>:<topic-slug> --title "..." --body-file pr-body.md
git checkout <server-branch>                        # leave the server branch as it was
```

Cherry-pick only the commits belonging to this change, then confirm the branch is clean relative to the base before pushing: `git log --oneline canonical/<base>..HEAD` and `git diff --stat canonical/<base>..HEAD`. A cherry-pick that silently drags in unrelated files (because an earlier commit touched them too) shows up immediately in that diff — check it, don't assume.

Note: the GitHub **MCP tools** may be authenticated to a different account (or not at all) than the `gh` CLI. If `mcp__github__create_pull_request` fails with `Requires authentication` while `gh auth status` is green, use `gh` instead — do not report the push/PR as blocked.

**A stale boot file that upstream has since IGNORED needs no merge action at all.**
Check `git log canonical/<base> -- <path>` before deleting anything: if the commit
trajectory is delete + add to `.gitignore`, the correct move is to let the merge
land and drop the local copy only if it still exists afterwards. Deleting eagerly
as a reflex removes a file the user may still want.

### Sync check when the canonical base has no named remote

Some servers have only their own fork configured (`git remote -v` shows one remote and no
canonical entry), so the familiar `git fetch <remote> <branch>` has nothing to fetch from
and a sync "completed" with zero output can mean the base was never actually contacted.
Establish the canonical ref by URL instead, then verify it is genuinely reachable:

```bash
git fetch https://github.com/<owner>/<repo>.git <base>:refs/remotes/canonical/<base>
git ls-remote https://github.com/<owner>/<repo>.git <base>   # independent SHAs must agree
git merge-base --is-ancestor refs/remotes/canonical/<base> HEAD && echo "not behind"
```

Read `[up to date]` as confirmation the ref was contacted and unchanged; an empty output
with no `[up to date]` line means nothing was fetched. Compare `refs/remotes/canonical/<base>`
against the FORK's own base branch too — they can differ, and a fetch that only proves the
fork is self-consistent answers nothing about the canonical base.

Report the sync as up to date only after the ancestry check, and state the branch's actual
commits ahead of base explicitly ("N commits ahead, 0 behind") rather than implying work was
done. When the branch is already canonical plus local commits and `origin/<branch>` matches
HEAD, there is nothing to commit or push for the sync itself — say so instead of
manufacturing an empty merge commit.

### A rejected push after syncing to canonical is usually a server-side commit

Fast-forwarding the working branch to `canonical/<base>` does **not** make it a
descendant of `origin/<branch>`. If anything landed on the server branch
directly — a dependency bot, a web edit, a colleague — your local branch now has
no fast-forward path left and the push is rejected as non-fast-forward.

```bash
git fetch origin <branch>
git rev-list --left-right --count HEAD...origin/<branch>
git log --oneline HEAD..origin/<branch>     # what you are "missing"
```

Read that log before deciding. When the incoming commits are mechanical
(`chore(deps)`, lockfile bumps) and touch files your change does not, **merge
them** — `git merge origin/<branch> --no-edit`. The merge resolves clean and
the push becomes fast-forward; nobody's work is discarded and no force is
needed.

Reserve `--force` for the case where you are certain the server branch's
history is disposable. Reaching for it here is how you silently delete a
dependency bump or a teammate's commit. If the incoming commits overlap the
files you changed, merge and resolve the conflict deliberately rather than
forcing.

### Nested `.git` directories when copying content

When `cp -r` (or similar) a directory that contains its own `.git` into another repo,
`git add` detects the nested `.git` and stages only a submodule reference (mode 160000),
not the actual files. Clones of the outer repo will not contain the copied content.

**Fix — remove nested `.git` before staging:**

```bash
cp -r /source/dir target/inside/repo
rm -rf target/inside/repo/.git
git add target/inside/repo/
```

If already staged as submodule:

```bash
git rm -r --cached target/inside/repo/
rm -rf target/inside/repo/.git
git add target/inside/repo/
git commit -m "Add dir contents (fix nested repo)"
```

### Missing git identity on fresh clones

New clones may lack both global and per-repo `user.name`/`user.email`.
Commits fail with `empty ident name`.

**Fix — set per-repo config before first commit:**

```bash
git config user.email "user@users.noreply.github.com"
git config user.name "username"
```

Detect from `gh auth status` or set manually.

## Verification

- `git status` shows no unexpected submodule entries.
- `git diff --cached --stat` shows actual file additions, not just mode changes.
- Commit and push succeed without warnings about embedded repos.
