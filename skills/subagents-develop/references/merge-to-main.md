# Merge into the Main Branch and Final Cleanup

This reference accompanies `subagents-develop` and defines the complete "merge into main and final cleanup" phase. Enter this phase only after the user explicitly commands, during the current task, that the parent task branch be merged into the main branch. Continue using the `mainBranch`, parent task branch, and `task-slug` confirmed and recorded at task start. If any identity cannot be read back exactly, stop and ask instead of guessing from a name.

See `references/safety-inventory.md` for the commands and decision rules of the four-part NUL-safe inventory. Every reference to the "four-part inventory" in this file means that procedure.

## 1. Pre-merge missed-resource audit

Perform the first audit for the current `task-slug`:

```text
git worktree list --porcelain -z
git for-each-ref --format='%(refname)' "refs/heads/agent/<task-slug>/"
find "<worktree-root>" -mindepth 1 -maxdepth 1 -print0
```

Run the third command only when the confirmed `<worktree-root>` exists. Parse every Git worktree record, every local branch under the exact ref prefix, and every direct disk entry under the worktree root. Do not miss directories that remain after Git registration has disappeared.

Confirm that every subagent has ended. Repeat the safety inventory and ancestry checks for any missed child resource. Clean up resources that are integrated into the parent branch and safe to remove. Preserve and report any dirty worktree, hidden file, special index flag, unregistered directory, or unmerged result; **it blocks the main-branch merge** and must not be bypassed.

## 2. Pin SHAs and inventory the parent side

Record the full parent task branch `parentHead` and the main branch's pre-merge `preMergeMain`:

```text
git rev-parse "refs/heads/agent/<task-slug>/integration"
git rev-parse "<mainBranchRef>"
```

The parent task branch was detached during handoff, so use its branch ref as authority rather than substituting the parent worktree HEAD. The parent worktree must pass the four-part inventory, and `refs/heads/agent/<task-slug>/integration` must still equal `parentHead`. Stop, list paths, and ask the user when any special index flag, modification, ordinary untracked file, or ignored file exists. Ignored build output from combined checks is a common cause; inventory again only after the user authorizes its removal. **If the user has removed the parent worktree themselves** and it no longer appears in Git's worktree records, skip the parent worktree inventory and proceed using only the branch ref. Never use a branch name that may move later as the merge target; use only pinned SHAs.

## 3. Generate and verify a candidate merge commit

Generate a candidate merge commit from the two pinned SHAs without moving the main branch:

```text
git merge-tree --write-tree "<preMergeMain>" "<parentHead>"
git commit-tree "<candidateTree>" \
  -p "<preMergeMain>" \
  -p "<parentHead>" \
  -m "merge: integrate agent/<task-slug>/integration"
git rev-list --parents -n 1 "<candidateMerge>"
git rev-parse "<candidateMerge>^{tree}"
```

Check each command's exit code and read the SHA from stdout without command substitution. Stop and report if `merge-tree` finds a conflict or if the candidate commit's tree, first parent, and second parent do not exactly equal `candidateTree`, `preMergeMain`, and `parentHead`, respectively.

Create a detached candidate worktree at `<worktree-root>/candidate`:

```text
git worktree add --detach -- "<worktree-root>/candidate" "<candidateMerge>"
```

Run combined task checks and an independent fresh exact-diff review there, both against `preMergeMain..candidateMerge`. Before updating the main branch, the candidate worktree must pass the four-part inventory so it can later be removed without loss. If ignored build output from checks prevents a clean inventory, list the paths, ask the user, and repeat the inventory only after authorized deletion.

## 4. Select the main-branch update location

Update the main branch only in its own safe worktree:

- If the main branch is already checked out in a worktree, merge only in that exactly identified worktree after it passes the four-part inventory. Ignored files may remain there, but the update command must include `--no-overwrite-ignore`; the other three inventory checks must pass completely. If the worktree has changes or belongs to another agent that cannot be coordinated with, stop and ask the user.
- If no worktree has the main branch checked out, create a temporary main-branch worktree in the current `task-slug` namespace at `<worktree-root>/main-merge`, then read back that its branch, HEAD, and `preMergeMain` all match:

  ```text
  git worktree add -- "<worktree-root>/main-merge" "<mainBranch>"
  ```

  This temporary main-branch worktree must be completely clean under all four inventory checks.
- Never switch the shared checkout initially opened by the user or overwrite any content there.

## 5. Fast-forward to the candidate

Immediately before the update, read the main-branch HEAD one final time and require it to still equal `preMergeMain`. Then permit only a fast-forward to the pinned candidate SHA:

```text
git -C "<absolute-main-worktree>" merge \
  --ff-only \
  --no-overwrite-ignore \
  -- "<candidateMerge>"
```

If another lead agent advanced the main branch, `--ff-only` must reject a HEAD outside the candidate's history. Regenerate, check, and review a new candidate from the new HEAD; never reuse the previous conclusion. On shared Git lock contention, retry only a limited number of times and re-read the main branch and worktree state before each retry.

On update failure, do not reset, rewrite the main branch, or clean up task resources. Preserve the state and report it. After success, the main-branch HEAD must equal `candidateMerge` exactly, and its two parents must still equal `preMergeMain` and `parentHead` exactly. Otherwise, do not enter final cleanup.

## 6. Final cleanup

After proving that the main branch was updated exactly and that `parentHead` is an ancestor of the new HEAD, clean up in this order:

1. Enumerate every worktree and local branch under the `task-slug` a second time. Run the complete safety inventory on any missed subagent resource and delete it only when it is integrated into the main branch and safe to remove.
2. Confirm again that `refs/heads/agent/<task-slug>/integration` exactly equals `parentHead`, then run the four-part inventory on the parent worktree. Remove it only after all checks pass.
3. From the main-branch worktree, run `git branch -d -- "agent/<task-slug>/integration"`; never use `-D`. If deletion fails because the user checked out the branch elsewhere, preserve it and report the condition rather than forcing deletion.
4. Run the four-part inventory on the detached candidate worktree and remove it only after all checks pass.
5. If the lead agent created a temporary main-branch worktree for this phase, run the four-part inventory there and remove it only after all checks pass. Never remove a main-branch worktree that existed before the task.
6. Enumerate Git worktrees, refs, and direct disk entries a third time. No local branch may remain under the current `task-slug`, and no parent, child, candidate, or temporary main-branch worktree may remain. If the worktree root is empty, remove that empty directory non-recursively. Otherwise, preserve any resource that cannot be removed safely and report that cleanup is incomplete.

Match cleanup only to the current lead agent's exact `task-slug` namespace. Never delete another lead agent's same-named task resources, and never substitute `git worktree prune` for item-by-item verification.
