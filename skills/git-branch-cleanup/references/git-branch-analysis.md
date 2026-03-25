# Git Branch Analysis Reference

Detailed explanation of the inspection commands used in the `git-branch-cleanup` skill and how to interpret their output.

---

## `git fetch --prune origin`

Fetches latest state from the remote and removes any local remote-tracking refs that no longer exist on the remote (e.g., `origin/feature/login` when `feature/login` was deleted on origin).

This is safe to run at the start of cleanup. It does not delete local branches — only the local bookmarks that tracked remote branches.

Run this first so subsequent commands reflect the current remote state.

---

## `git branch -vv`

Shows all local branches with their upstream tracking information and divergence status.

Example output:
```
  feature/login-refactor   a3c1e2f [origin/feature/login-refactor: gone] Merge branch
  fix/null-pointer          b91f3a1 [origin/fix/null-pointer] Fix null pointer in parser
* main                      d42c8b0 [origin/main] Release v1.4.0
  experiment/new-parser     e7a12d9 Refactor tokenizer
```

What each part means:

| Indicator | Meaning |
|---|---|
| `*` prefix | Current branch — never delete |
| `[origin/branch]` | Has a tracked upstream |
| `[origin/branch: gone]` | Upstream was deleted; local ref is orphaned |
| `[origin/branch: ahead N]` | Has N commits not yet pushed |
| `[origin/branch: behind N]` | Remote has N commits not yet pulled |
| No upstream shown | Branch was never pushed or tracking was removed |

A branch showing `gone` with no local-only commits is typically safe to delete. A branch showing `ahead N` has unpushed work.

---

## `git branch --merged <default>`

Lists local branches whose tip commit is reachable from `<default>`. These branches have been fully incorporated into the default branch.

```
git branch --merged main
```

Branches in this list (excluding `main` itself and the current branch) are generally safe to delete locally with `git branch -d`.

**Caution:** A branch can be "merged" in the sense that all its commits exist in the default branch history, even if it was squash-merged or cherry-picked rather than merged with a merge commit. Verify with `git log` if uncertain.

---

## `git branch --no-merged <default>`

Lists local branches that have at least one commit NOT reachable from `<default>`. These branches have unique work.

```
git branch --no-merged main
```

Every branch in this list should be treated as **uncertain** unless you have verified the unique commits are intentionally abandoned.

---

## `git log --oneline <default>..<branch>`

Shows commits that exist in `<branch>` but NOT in `<default>`. This is the exact set of work that would be lost if the branch were deleted.

```
git log --oneline main..experiment/new-parser
```

Example output:
```
e7a12d9 Refactor tokenizer
c3b22f1 Add streaming lexer
```

If this produces output, the branch has unique work. Classify as **uncertain** and show this output to the user before proposing deletion.

If this produces no output, the branch has no unique commits relative to the default (it may have been squash-merged or is a stale pointer).

---

## `git diff --stat <default>...<branch>`

Shows the file-level diff between the common ancestor and the branch tip. Useful for summarizing what a branch changed without listing individual commits.

```
git diff --stat main...experiment/new-parser
```

Use this to give the user a concise summary of what would be lost if an uncertain branch were deleted.

---

## `git branch -r`

Lists all remote-tracking refs (local copies of what the remote reports).

```
  origin/HEAD -> origin/main
  origin/feature/login-refactor
  origin/main
  origin/release-please--branches--main
```

After `git fetch --prune`, this list reflects the actual current state of the remote. Refs that were removed by `--prune` will no longer appear.

---

## `git branch -r --merged origin/<default>` and `--no-merged`

Same as the local equivalents, but for remote-tracking refs.

```
git branch -r --merged origin/main
git branch -r --no-merged origin/main
```

Branches in the `--merged` list have been incorporated into the remote default branch and are candidates for remote deletion (with user confirmation). Branches in `--no-merged` should not be proposed for remote deletion without explicit review.

---

## Distinguishing ref types

There are three distinct concepts that are easy to confuse:

| Concept | Example | What it is |
|---|---|---|
| **Local branch** | `feature/login` | A local ref pointing to a commit; lives in `.git/refs/heads/` |
| **Remote-tracking ref** | `origin/feature/login` | A local copy of what the remote reported; lives in `.git/refs/remotes/` |
| **Remote branch** | `feature/login` on `origin` | The actual branch on the remote server |

**Pruning** (`git remote prune origin` or `git fetch --prune`) removes stale remote-tracking refs from your local repo. It does **not** touch the remote.

**Deleting a remote branch** (`git push origin --delete feature/login`) removes the branch from the remote server. This also removes the remote-tracking ref locally.

**Deleting a local branch** (`git branch -d feature/login`) removes only the local ref. If a remote branch exists, it is unaffected.

All three are separate operations. Never conflate pruning tracking refs with deleting remote branches.
