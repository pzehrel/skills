# Safety Inventory and Cleanup Procedure

This reference accompanies `subagents-develop` and defines the complete commands and decision rules for worktree creation readback, the four-part NUL-safe inventory, child-resource deletion, and zero-child-resource verification. Every instruction in `SKILL.md` to follow the safety-inventory procedure refers to this file.

## General execution rules

- Run every command independently and check its exit code. Stop all subsequent actions immediately when any command returns nonzero.
- Do not hide Git failures behind a pipeline that lacks `pipefail`.
- Parse NUL-delimited output from `-z` or `-print0` by record. Never guess at paths containing special characters by splitting on lines.
- Bind every dynamic value to a variable and quote it reliably. Prohibit `eval` and command substitution used to construct shell source.

## Post-creation readback for parent and child worktrees

Immediately after each `git worktree add`, run:

```text
git worktree list --porcelain
git -C "<absolute-worktree>" branch --show-current
git -C "<absolute-worktree>" rev-parse HEAD
git -C "<absolute-worktree>" rev-parse 'HEAD^{tree}'
git -C "<absolute-worktree>" status --short
```

Continue only when branch name, base commit and tree, path, and clean state all match expectations. Stop and investigate any mismatch; do not continue work in that worktree.

## Four-part NUL-safe inventory

Before removing any child, parent, candidate, or temporary main-branch worktree, run each command against that worktree:

```text
git -C "<absolute-worktree>" ls-files -v -z
git -C "<absolute-worktree>" ls-files -t -z
git -C "<absolute-worktree>" status --porcelain=v1 -z --untracked-files=all
git -C "<absolute-worktree>" ls-files -z --others --ignored --exclude-standard
```

Decision rules:

1. The first command, `ls-files -v`, must contain no lowercase tag; no `assume-unchanged` entry may exist.
2. The second command, `ls-files -t`, must contain no `S` tag; no `skip-worktree` entry may exist.
3. The third command, `status --porcelain -z --untracked-files=all`, must contain zero records. It forces all ordinary untracked files and visible tracked changes to appear regardless of `status.showUntrackedFiles`.
4. The fourth command, `ls-files --others --ignored`, must contain zero records. It exposes ignored files separately.

When special index flags, any changes, or additional files exist, do not remove the worktree. Preserve the resource, list the paths, and ask the user how to proceed. Invoking this skill, including automatic invocation, does not authorize deleting content hidden from ordinary Git clean checks.

Exception: an existing main-branch worktree not created by this task may retain ignored files, but the subsequent update command must include `--no-overwrite-ignore` to protect them. The other three inventory checks must still pass completely.

## Child-resource deletion order

After all four inventory checks pass and the ancestry check confirms integration, run in this order:

```text
git merge-base --is-ancestor "<child-branch>" "<parent-branch>"
git worktree remove -- "<absolute-child-worktree>"
git -C "<absolute-parent-worktree>" branch -d -- "<child-branch>"
git worktree list --porcelain
```

First use `merge-base --is-ancestor` to prove integration, then remove the worktree, delete the local child branch with `branch -d`, which rejects unmerged branches, and finally read back the worktree list. Stop immediately on any nonzero result and preserve the remaining state.

## Zero-child-resource verification before handoff

For the current `task-slug`, run:

```text
git worktree list --porcelain -z
git for-each-ref --format='%(refname)' "refs/heads/agent/<task-slug>/"
find "<worktree-root>" -mindepth 1 -maxdepth 1 -print0
```

Run the third command only when the confirmed `<worktree-root>` exists. Parse every Git worktree record, every local branch under the exact ref prefix, and every direct disk entry under the worktree root. Do not miss directories that remain on disk after Git registration has disappeared.

The verification passes only when nothing remains except the `agent/<task-slug>/integration` branch and `<worktree-root>/integration` parent worktree. Include the complete output of all three enumerations in the final report. When anything remains, return to the child-resource deletion order and inventory each item. If data-protection conditions prevent cleanup, identify the path and preservation reason explicitly and do not claim zero-resource verification succeeded.
