---
name: subagents-develop
description: Dispatch independent development tasks in parallel. Use for coding, fixes, or refactors that benefit from multiple sessions or isolation from the current checkout; develop in isolated branch and worktree namespaces, fan out to multiple agents only when tasks are safely parallel, and converge all results. Merging into the main branch requires an explicit user command.
license: MIT
metadata:
  repository: https://github.com/pzehrel/skills
---

# Subagent Development Orchestration

Treat the current agent as an independent lead agent. The user only needs to assign work and need not manage the current Git state; they may open multiple sessions sequentially and dispatch different tasks. After receiving a task, the lead agent owns decomposition, serial and parallel scheduling, Git isolation, integration, and resource cleanup. **Git isolation rules—namespace, worktrees, handoff, cleanup, and main-branch discipline—are mandatory; subagent fan-out is optional.** Fan out only when the task benefits from parallelism. Otherwise, develop directly in the parent worktree. Multiple lead agents may operate in the same repository concurrently, but each may manage only its own fully isolated namespace.

## Overall workflow

1. Inspect the repository read-only and protect the shared checkout.
2. Confirm the main branch and record the exact base.
3. Establish a `task-slug` namespace, parent task branch, and parent worktree.
4. Decompose the task. Create and dispatch subagents in waves when parallelism is safe; otherwise develop directly in the parent worktree.
5. When subagents are used, integrate each wave's child branches into the parent branch and clean up child resources.
6. Handoff: verify that child resources are zero, detach the parent worktree to release the integration branch, and report.
7. Only after a later explicit user command, merge into the main branch and perform final cleanup as defined in `references/merge-to-main.md`.

## Authorization boundary

Unless the user explicitly requests planning only, explicit or automatic invocation of this skill authorizes the lead agent to perform these local actions end to end within the user's stated task: confirm the main branch and record the base commit and tree; create one parent task branch and parent worktree; create subagent branches and worktrees; dispatch, follow up with, and wait for subagents; create local commits required by the task; merge subagent branches into the parent task branch; run in-scope checks; detach the parent worktree HEAD; and delete clean, merged subagent worktrees and local child branches.

This authorization does not include push, force-push, PR creation, merging into the main branch, deleting remote branches, deleting unmerged branches, or discarding uncommitted content. Never automatically merge the parent task branch into the user's original branch or the main branch.

Only a later explicit command from the user to merge the parent task branch into the main branch authorizes the "merge into main and final cleanup" phase described in `references/merge-to-main.md`. That command still does not authorize push, remote-branch deletion, force deletion, or discarding content.

Read and follow repository-root and relevant-path instructions such as `AGENTS.md`. This skill governs task orchestration and the local Git lifecycle only; it does not add project-specific admission rules, roles, or tracking requirements.

## Protect the shared repository

Begin with read-only checks:

```text
git rev-parse --show-toplevel
git status --short --branch
git branch --show-current
git worktree list --porcelain
git remote
git branch --list
```

Treat the repository checkout initially opened by the user as a shared control entry point. Do not switch branches there or use it as a subagent development directory. Preserve all existing modifications and untracked files in the shared checkout. Never automatically stash, reset, clean, restore, check out paths, commit, move, or copy them. Create the parent task branch only from a committed SHA on the confirmed main branch. If the task depends on uncommitted content in the shared checkout, stop and ask the user.

## Confirm the main branch

Every lead agent must confirm the main branch first and must not infer it from the current checkout or a conventional name. Use this priority:

1. A main branch explicitly named by the user for the current task.
2. The repository has exactly one remote, its symbolic HEAD uniquely identifies a default branch, and the corresponding local branch exists.
3. Every remote's symbolic HEAD resolves to the same default branch, and the corresponding local branch exists.

Read-only supporting checks:

```text
git remote
git symbolic-ref --quiet "refs/remotes/<remote>/HEAD"
git for-each-ref --format='%(refname)' refs/heads refs/remotes
```

None of the following proves the main branch on its own: the current branch; a branch named `main`, `master`, or `develop`; the branch with the most commits; or a branch used by another agent. Stop and ask the user when the only remote lacks a symbolic HEAD, multiple remotes have missing or conflicting symbolic HEADs, the local default branch is absent, or multiple reasonable candidates remain. A detached HEAD neither proves nor disproves the main branch; continue to use remote evidence.

After confirmation, resolve and record `mainBranch`, `mainBranchRef`, `baseCommit`, `baseTree`, and `remoteFreshness`. By default, use the current local main-branch commit, do not fetch automatically, and explicitly record that remote freshness was not verified. Fetch only when the user asks to base the task on the latest remote main branch.

## Establish an isolated lead-agent namespace

Derive a short `task-slug` from the task as its unique namespace identifier, such as `fix-login`. Normalize `task-slug` and `subtask-id` to lowercase ASCII containing only `[a-z0-9-]`; collapse consecutive hyphens, trim leading and trailing hyphens, and limit `task-slug` to 20 characters. Never interpolate raw user text, branch names, or paths directly into shell source. Prefer argv-based Git calls. When a shell is unavoidable, bind dynamic values to variables and quote each expansion reliably. Prohibit `eval`, command substitution, and unquoted variable expansion. `git check-ref-format` validates only ref syntax; it does not make a value safe to interpolate into shell source.

Before creation, inspect existing refs and worktrees. If another in-progress task already owns `agent/<task-slug>/`, try `<task-slug>-2`, then `<task-slug>-3`; if conflicts remain, use `<task-slug>-<rand4>`, where `<rand4>` is four random lowercase alphanumeric characters. Lead agents receiving identical tasks concurrently must never share a namespace.

Use this namespace layout with a consistent `agent/` prefix:

```text
Parent task branch: agent/<task-slug>/integration
Child task branch: agent/<task-slug>/<subtask-id>
Worktree root: <repo-parent>/<repo-name>-worktrees/<task-slug>/
Parent worktree: <worktree-root>/integration
Child worktree: <worktree-root>/<subtask-id>
Candidate worktree, main-merge phase only: <worktree-root>/candidate
Temporary main-branch worktree, main-merge phase only: <worktree-root>/main-merge
```

Do not create an `agent/<task-slug>` branch before creating `agent/<task-slug>/<subtask-id>` branches; Git ref file/directory prefixes would conflict.

Before any creation, resolve paths to explicit absolute paths and verify that the target branch and path do not exist, `git check-ref-format --branch "<branch>"` succeeds, the worktree root is not a system temporary directory, and the target does not belong to another lead agent's namespace.

Multiple lead agents may write shared Git metadata concurrently. On `index.lock`, `packed-refs.lock`, `config.lock`, or similar lock contention, re-read state and retry a limited number of times. Never delete a Git lock file whose origin is unknown.

## Create the parent task branch and worktree

Create them directly from the confirmed main branch's full `baseCommit`:

```text
git worktree add \
  -b "agent/<task-slug>/integration" \
  -- "<absolute-parent-worktree>" \
  "<full-base-commit>"
```

Immediately perform the post-creation readback defined in `references/safety-inventory.md`, confirming branch, HEAD, tree, and clean state before continuing. Once the parent task branch exists, do not automatically rebase it or move its base if the main branch advances. Report the original base and the current divergence at handoff.

## Decompose serial and parallel work autonomously

Analyze the code only in the parent worktree and build a dependency graph of subtasks, grouped into one or more execution waves. Parallelize only when every condition below holds:

- The subtasks have no prerequisite relationship.
- Their expected write paths do not overlap.
- They do not jointly modify a lockfile, schema, migration, entry point, or generated artifact.
- No subtask depends on an interface or type another unfinished subtask must produce.
- Each subtask can be checked and committed independently.

Use serial execution when dependencies, shared writes, ordering sensitivity, or uncertain boundaries exist. Assign a shared manifest and lockfile to the same subtask. Give shared interfaces and integration glue a single owner or schedule them in a later wave. Do not over-decompose merely to create parallelism.

**Fan-out is optional; Git isolation is mandatory.** For a small task or one that cannot be parallelized safely, create no subagents. Develop and commit directly in the parent worktree, then proceed to handoff with the same detach, zero-child-resource verification, and reporting requirements. When fan-out is used, the lead agent owns orchestration and integration, must not delegate the whole task to another lead agent, and must prohibit recursive subagent fan-out.

## Create and dispatch subagents by wave

This section and the following integration section apply only when fan-out is selected. With zero fan-out, complete development and commit in the parent worktree, then continue to handoff. Child-resource deletion procedures do not apply when there were no children, but zero-child-resource verification is still mandatory and must appear in the report.

At the beginning of each wave, pin the current full SHA of the parent task branch as `waveBase`. Create every child branch in the wave from the same `waveBase`:

```text
git worktree add \
  -b "agent/<task-slug>/<subtask-id>" \
  -- "<absolute-child-worktree>" \
  "<full-wave-base>"
```

Create each child worktree and complete the `references/safety-inventory.md` readback before dispatch. Do not exceed the currently available agent slots.

Each subagent assignment must include: the exact objective and completion conditions; child branch, `waveBase`, and absolute worktree path; allowed write paths and prohibited shared files; known interfaces, dependencies, and boundaries with other subtasks; required in-scope checks; a requirement to commit all expected changes and leave the worktree clean; a required return of commit SHA, changed files, check results, and remaining risks; and prohibitions against creating more subagents, switching branches, merging, rebasing, pushing, deleting branches, or removing worktrees.

Continue following up with subagents. If a subagent stops before satisfying its delivery conditions, send an explicit follow-up instead of guessing at or rewriting its unhanded-off work.

## Collect and integrate into the parent task branch

Wait for every subagent in the current wave to finish, then verify that each subagent has stopped, its worktree is clean, its branch HEAD contains at least one expected task commit derived from `waveBase`, its actual changes remain in scope, and its in-scope check results can be inspected.

If a child worktree still has uncommitted content, first ask the original subagent to finish and commit it. If that cannot be completed, preserve the branch and worktree, mark the subtask blocked, and do not clean it up.

Integrate in a clean parent worktree, following dependency-topological order and stable subtask-ID order within a wave:

```text
git merge --no-ff \
  -m "merge: integrate <subtask-id> into <task-slug>" \
  -- "agent/<task-slug>/<subtask-id>"
```

Resolve conflicts only in the parent worktree. Resolve them when the correct result is safe to determine and record the resolution. Otherwise, abort the current integration, preserve the related child resources, and report the blocker. Never manufacture an "integrated" state with an `ours` merge or by discarding child-branch content.

Run combined checks after integrating each wave. When repairs are needed, create a new repair subtask in a new wave rather than allowing a completed subagent to continue drifting in a handed-off directory. Create the next wave from the updated parent task branch HEAD.

## Clean up subagent resources as part of completion

Clean up a child only after its task is merged into the parent task branch, combined checks no longer require reverting it, its worktree is clean, and the subagent has ended. Follow the safety inventory and deletion procedure in `references/safety-inventory.md`. Core rules:

- Run every inventory and deletion command independently and check its exit code. Stop immediately on any nonzero result.
- Do not remove a worktree containing special index flags (`assume-unchanged` or `skip-worktree`), any modification, ordinary untracked file, or ignored file. Preserve the resource, list the paths, and ask the user.
- Never use `git branch -D`. Never remove a dirty worktree, unmerged branch, or another lead agent's resource.

**Cleanup is part of completion, not optional.** Clean each wave's child resources immediately after successful integration. Before handoff, run the zero-child-resource verification from `references/safety-inventory.md`. Under the current `task-slug` prefix, no local branch, worktree, or direct disk entry may remain except `integration`. Include verification output in the final report. If verification fails, do not claim development is complete; continue cleanup or explicitly report why resources were preserved.

## Handoff: release the integration branch and report

After all waves are integrated, combined checks pass, and zero-child-resource verification succeeds:

1. Run `git -C "<absolute-parent-worktree>" checkout --detach HEAD` in the parent worktree. **This step is mandatory.** It releases `agent/<task-slug>/integration` so the user can check out the branch in any worktree while preserving the parent worktree for inspection.
2. Read back that the parent worktree is detached, its HEAD equals the latest integration-branch commit, and the parent branch ref has not moved.
3. Report according to "End state and reporting."

## Merge into the main branch and perform final cleanup

Enter this phase only after an explicit user command in the current task to merge the parent task branch into the main branch. Continue using the `mainBranch`, parent task branch, and `task-slug` recorded at task start. If any identity cannot be read back exactly, stop and ask instead of guessing from a name.

Core invariants:

- First audit the current `task-slug` for missed child resources. Any child resource that cannot be cleaned safely blocks the main-branch merge.
- Record the full parent-task-branch `parentHead` and the main branch's pre-merge `preMergeMain`. Use only pinned SHAs throughout, not branch names that might move.
- Use `git merge-tree --write-tree` and `git commit-tree` to create a candidate merge commit. Run combined checks and an independent review of `preMergeMain..candidateMerge` in a detached candidate worktree before updating the main branch.
- Fast-forward the main branch to the pinned candidate SHA only in its own safe worktree, with `--ff-only --no-overwrite-ignore`. If another lead agent advances the main branch, regenerate the candidate; never reuse a stale conclusion.
- On update failure, do not reset, rewrite the main branch, or clean task resources. Preserve the state and report it.
- After a successful update, clean up the parent worktree, parent branch, candidate worktree, and temporary main-branch worktree in the prescribed order, then perform a third enumeration proving that no resource remains in the current `task-slug` namespace.

Follow `references/merge-to-main.md` exactly for the complete missed-resource audit, candidate construction, main-branch worktree selection, and cleanup order.

## Failure and partial completion

- If the main branch cannot be confirmed, ask the user and create no branch or worktree.
- If parent task branch creation fails, stop and dispatch no subagent.
- If a subagent fails, preserve its branch and worktree. Integrate only valid results that do not depend on it.
- If integration or combined checks fail, preserve related resources and arrange additional safe repair waves. Report a blocker when progress cannot continue safely.
- If candidate generation, review, or main-branch fast-forward fails, do not reset or rewrite the main branch. Preserve the parent task branch, parent worktree, and any candidate state that cannot be removed safely, then report.
- If the user interrupts the task, stop new dispatches and preserve every branch and worktree not yet integrated safely.
- If another lead agent's branch or worktree is found, identify it read-only; do not modify, merge, or clean it.

Preserving resources takes precedence over the appearance of complete cleanup. Automatically delete only child resources that were integrated successfully.

## End state and reporting

When development is complete but the user has not ordered a main-branch merge, retain only:

```text
agent/<task-slug>/integration        # Integration branch, not checked out by any worktree
<worktree-root>/integration          # Parent worktree with detached HEAD, available for inspection
```

All subagents have ended, all successful results have converged into the parent task branch, and zero-child-resource verification has passed. After the user explicitly orders a main-branch merge and it succeeds, the main branch contains `parentHead`, and every branch and worktree under the current `task-slug` has been deleted. If data-protection conditions require retaining any resource, do not claim final cleanup is complete.

Before stopping, report:

- Confirmed main branch and the parent task branch's base commit and tree.
- Parent task branch, parent worktree including detached state, and final HEAD.
- Serial and parallel waves and each subagent's task.
- Integrated commits and combined-check results.
- Enumeration output from zero-child-resource verification and deleted child worktrees and branches.
- Parent task worktree and parent task branch deleted after a main-branch merge.
- For a main-branch merge: `preMergeMain`, `parentHead`, post-merge main-branch HEAD, and final audit of parent and child resources.
- Resources preserved because of failure and how to recover them.
- Ahead/behind or drift relative to the current main branch.
- Whether the main-branch merge occurred, and that push, PR creation, or remote cleanup did not occur unless separately authorized.
