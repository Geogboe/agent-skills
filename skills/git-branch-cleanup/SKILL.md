---
name: git-branch-cleanup
description: >
  Safely inspects and cleans up Git branches, remote-tracking references, and stashes.
  Use this skill when the user asks to: delete stale branches, clean up local or remote branches,
  remove merged branches, prune old remote branches, figure out which branches are safe to delete,
  clean up git refs after PRs merged, perform general branch hygiene, clean up stashes, remove old
  or outdated stashes, or review what stashes exist. Trigger conservatively —
  always classify branches and stashes before deleting anything, and never delete without understanding whether
  a branch has unique commits, a stash has unsaved work, or a branch is automation-managed.
---

# Git Branch Cleanup Skill

Safe, conservative branch and stash cleanup. The skill's job is to do the hard research work and deliver a clear, reasoned recommendation — **never to delete anything on its own**. No branch, remote ref, or stash is removed until the user has reviewed the full findings and given explicit approval.

Scope: local branches, remote-tracking refs, and stashes. Do not touch the working tree or tags unless explicitly asked.

---

## Step 1: Confirm safe starting state and sync with remote

Before anything else, fetch the latest state from the remote so all subsequent inspection reflects current reality:

```
git fetch --prune origin
```

This also prunes stale remote-tracking refs in one step. If the user prefers a full pull on the current branch first, run:

```
git pull
```

Do not skip this step. Classification based on stale remote data will produce incorrect recommendations.

Then establish local context:

```
git status --short
git branch --show-current
git remote -v
```

- If the working tree has uncommitted changes, note it. Do not mix branch cleanup with stash or commit operations unless asked.
- Record the **current branch** — it must never be deleted.
- Record the **default branch** (usually `main` or `master`) — it must never be deleted.

To confirm the default branch:

```
git remote show origin | grep "HEAD branch"
```

---

## Step 2: Inspect branches and stashes in parallel

Use **two subagents running concurrently** to gather all inspection data efficiently. Spawn them at the same time and wait for both before classifying anything.

**Subagent A — Branch inspection:**

Run this inspection set (fetch was already done in Step 1). Read `references/git-branch-analysis.md` for a full explanation of what each command means.

```
git branch -vv
git branch --merged <default>
git branch --no-merged <default>
git branch -r
git branch -r --merged origin/<default>
git branch -r --no-merged origin/<default>
```

Key things to look for in `git branch -vv` output:
- `[origin/branch: gone]` — upstream was deleted; local ref is now orphaned
- `ahead N` / `behind N` — how far the branch has diverged
- No upstream shown — branch has never been pushed or tracking was removed

**Subagent B — Stash inspection:**

```
git stash list
```

For each stash entry, record:
- Stash index (`stash@{N}`)
- The branch it was created on
- The commit message/description
- How old it is (date shown in stash list output)

If the stash list is non-empty, also run per-stash diff summaries (these can be parallelised across stashes as additional subagents or chained commands):

```
git stash show stash@{N}
```

Collect all subagent output before moving to classification.

---

## Step 3: Classify branches and stashes

**Branches** — assign each to one of three buckets:

| Classification | Criteria |
|---|---|
| **Safe to delete** | Merged into default branch; no unique commits; or upstream is already gone with no local-only work |
| **Uncertain — ask first** | Unmerged; has unique commits relative to default; purpose is unclear from name; remote state is inconsistent |
| **Keep** | Current branch; default branch; automation-managed (release-please, dependabot); protected long-lived branches (release/*, hotfix/*) |

For unmerged branches, check unique commits before classifying:

```
git log --oneline <default>..<branch>
```

If this produces output, the branch has work not in the default branch. Classify as **uncertain** unless the user explicitly says it can go.

Read `references/git-safe-deletion-rules.md` for complete guardrail rules.

**Stashes** — assign each to one of three buckets:

| Classification | Criteria |
|---|---|
| **Likely outdated** | Created on a branch that has since been merged and deleted; stash is older than 30 days; stash diff shows changes already present in the current default branch |
| **Uncertain — review first** | Created on a branch that no longer exists locally but was never merged; stash contains changes not reflected anywhere in history |
| **Keep** | Recent stash (under 30 days); created on an active branch; or the user has indicated they still need it |

Age alone is not sufficient to drop a stash — always show the stash diff summary before proposing deletion.

---

## Step 4: Present full findings and recommendations

This is the primary deliverable. After completing inspection and classification, present everything to the user **before taking any action**.

**Branches:**

```
Branch                        Classification       Recommendation       Reason
-----------------------------  -------------------  -------------------  ----------------------------------
feature/login-refactor         safe to delete       recommend delete     merged into main
fix/null-pointer               safe to delete       recommend delete     merged into main; upstream gone
experiment/new-parser          uncertain            review needed        unmerged; 3 unique commits (see below)
release-please--branches--main keep                 do not delete        automation branch
main                           keep                 do not delete        default branch
```

For any branch classified as **uncertain**, include the unique commit list inline:

```
experiment/new-parser — 3 unique commits:
  e7a12d9 Refactor tokenizer
  c3b22f1 Add streaming lexer
  a901ff2 WIP: draft output format
```

**Stashes** (if any exist):

```
Stash          Created on            Age      Classification        Recommendation       Summary
-------------  --------------------  -------  --------------------  -------------------  -------------------------
stash@{0}      feature/login         2 days   keep                  do not drop          Partial auth changes
stash@{1}      fix/old-bug           45 days  likely outdated       recommend drop       1 file, 8 lines (see below)
stash@{2}      experiment/parser     60 days  uncertain             review needed        30 lines, branch deleted
```

For stashes recommended for dropping or flagged as uncertain, include the diff summary:

```
stash@{1} — git stash show:
  src/auth.js | 8 +++-----
```

End with a clear summary of proposed actions and a prompt for the user to approve:

```
Proposed actions (awaiting your approval):
  Delete branches:  feature/login-refactor, fix/null-pointer
  Drop stashes:     stash@{1}
  Skip / keep:      experiment/new-parser (uncertain), main, release-please--branches--main, stash@{0}, stash@{2}

Reply with approval to proceed, adjust the list, or ask about any specific item.
```

Do not execute anything until the user responds.

---

## Step 5: Execute — only after explicit user approval

Once the user approves (all proposed actions, a subset, or modified list), execute in this order:

**Branches:**

1. Local merged branches — use `-d`:
   ```
   git branch -d <branch>
   ```

2. Remote branch deletion — one at a time, only branches the user approved:
   ```
   git push origin --delete <branch>
   ```

3. Unmerged branches with `-D` — only if the user explicitly approved after seeing the unique commits:
   ```
   git branch -D <branch>
   ```

**Stashes:**

Drop stashes in reverse index order (highest index first) to avoid index shifting:
```
git stash drop stash@{N}
```

Never batch-drop multiple stashes without confirming the order won't shift indexes unexpectedly. If dropping more than one, drop highest index first.

---

## Step 6: Re-verify

After executing, re-run branch and stash lists to confirm the expected state:

```
git branch -vv
git branch -r
git stash list
```

Report what was deleted/dropped and what remains. If anything unexpected is missing, surface it immediately.

---

## Reference Files

| File | Read when |
|---|---|
| `references/git-branch-analysis.md` | Need full explanation of inspection commands, how to compare branches to default, how to interpret merged/no-merged, or how to distinguish ref types |
| `references/git-safe-deletion-rules.md` | Need complete guardrail rules, remote deletion policy, automation branch patterns, or escalation rules for uncertain branches |

---

## Common mistakes to avoid

- **Acting before presenting recommendations.** The skill's job ends at Step 4. Do not run any deletion or drop command until the user has reviewed the full findings table and explicitly approved. Even "obviously safe" deletions require approval.
- **Skipping the fetch.** Always run `git fetch --prune origin` as the very first action. Branch merge status and remote-tracking state are meaningless against stale data.
- **Using `git branch -D` as the default.** Force-delete bypasses the merge check. Use `-d` first; escalate to `-D` only with explicit user approval after reviewing unique commits.
- **Deleting remote branches because they "look old."** Age is not a reliable signal. Use merge status and unique commit count instead.
- **Confusing remote-tracking refs with remote branches.** `git remote prune origin` removes stale local tracking refs — it does not delete anything on the remote. These are different operations.
- **Mixing branch cleanup with working tree operations.** Branch cleanup is scoped to refs. Do not stash, commit, or reset as part of this workflow unless explicitly asked.
- **Deleting automation branches.** Branches created by release-please, dependabot, or similar tools are often recreated automatically and may break pipelines if deleted. Surface them as **keep** unless the user knows what they are doing.
