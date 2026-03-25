# Git Safe Deletion Rules Reference

Complete guardrail rules for the `git-branch-cleanup` skill. Read this when classifying branches, deciding whether to force-delete, handling remote branches, or determining when to escalate to the user.

---

## Hard guardrails — never delete without explicit confirmation

These branches must be classified as **keep** regardless of merge status:

| Rule | How to detect |
|---|---|
| Current branch | `git branch --show-current` — matches the branch in question |
| Default branch | Usually `main` or `master`; confirm with `git remote show origin \| grep "HEAD branch"` |
| Protected long-lived branches | Names matching `release/*`, `hotfix/*`, `develop`, `staging`, `production` |
| Automation-managed branches | See automation branch patterns below |

If the user explicitly asks to delete the current branch, explain that it is not possible without switching first, and ask which branch to switch to.

If the user explicitly asks to delete the default branch, refuse and explain why.

---

## Automation branch patterns

These branches are created and managed by external tools. Deleting them may break release pipelines or cause the tool to recreate them in an inconsistent state.

| Pattern | Tool | Action |
|---|---|---|
| `release-please--branches--*` | Release Please | Keep — automation-managed; may break release pipeline |
| `chore/release-*` | Release Please (older config) | Surface as uncertain; confirm purpose with user |
| `dependabot/*` | Dependabot | Keep — these have open PRs; let Dependabot manage lifecycle |
| `renovate/*` | Renovate | Keep — same as dependabot |
| `github-actions/*` | GitHub Actions | Surface as uncertain; confirm with user |

When in doubt, classify as **keep** and explain: "This appears to be an automation-managed branch. Recommend leaving it unless you know it is safe to remove."

---

## Force-delete rules (`git branch -D`)

`git branch -D` bypasses the merge safety check and can permanently destroy work. Only use it when all of the following are true:

1. The user has explicitly asked to delete this specific unmerged branch.
2. You have shown the user the unique commits (`git log --oneline <default>..<branch>`).
3. The user has confirmed they understand those commits will be lost.
4. The branch is not the current branch, default branch, or automation-managed.

Never use `-D` as a default or as a convenience to avoid merge errors. If `-d` fails with "not fully merged," treat that as a signal to escalate, not to retry with `-D`.

---

## Remote deletion policy

Remote branch deletion is strictly opt-in. The policy is more conservative than local deletion.

**Before proposing remote deletion:**
- Confirm the branch is merged into `origin/<default>` via `git branch -r --merged origin/<default>`
- Confirm no open pull requests reference this branch (if you have access to check)
- Present the branch to the user and ask for explicit confirmation

**Command to use:**
```
git push origin --delete <branch>
```

**Do not:**
- Batch-delete multiple remote branches without per-branch confirmation
- Delete a remote branch just because the local branch was deleted
- Delete a remote branch because it "looks old" or has not been updated recently
- Delete a remote branch that is unmerged without clear user instruction

**Separate from pruning:**
`git remote prune origin` removes stale *local* remote-tracking refs. It does not touch the remote. This is always safe to run. Do not confuse it with remote branch deletion.

---

## Handling "upstream gone" branches

When `git branch -vv` shows `[origin/branch: gone]`, the remote branch has been deleted (typically after a PR was merged). The local branch is now an orphaned tracking ref.

Recommended handling:
1. Check for unique commits: `git log --oneline <default>..<branch>`
2. If no unique commits → classify as **safe to delete** (the work is already merged)
3. If unique commits exist → classify as **uncertain** (work may not have been merged; show commits to user)

The "gone" state alone is not sufficient reason to force-delete. Always check for unique commits first.

---

## Uncertainty escalation

Stop and ask the user when:

- A branch is unmerged AND has unique commits relative to the default branch
- A branch name suggests it might still be relevant (e.g., `wip/`, `draft/`, `experiment/`) but has no clear PR or merge history
- A remote branch exists that has no corresponding local branch and is not in `--merged`
- A branch was last updated a long time ago but you cannot determine whether the work was abandoned or incorporated another way (squash merge, cherry-pick)
- The remote state is inconsistent (e.g., local shows merged but remote shows unmerged)

When escalating, show the user:
- The branch name
- The number of unique commits
- A short `git log --oneline` excerpt
- Your recommendation (delete, keep, or investigate further)

Do not guess. If uncertain, surface it and let the user decide.

---

## Summary decision table

| Condition | Classification | Deletion method |
|---|---|---|
| Merged into default; no unique commits | Safe to delete | `git branch -d` |
| Upstream gone; no unique commits | Safe to delete | `git branch -d` |
| Merged into default; is current branch | Keep | N/A |
| Merged into default; is default branch | Keep | N/A |
| Automation branch pattern | Keep | N/A |
| Unmerged; has unique commits | Uncertain — ask first | `git branch -D` only with explicit approval |
| Unmerged; no unique commits | Safe to delete (with note) | `git branch -d` |
| Remote branch; merged on remote | Propose for deletion | `git push origin --delete` with confirmation |
| Remote branch; not merged | Uncertain — ask first | Only with explicit per-branch approval |
| Stale remote-tracking ref only | Prune only | `git remote prune origin` |
