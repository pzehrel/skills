# 合入主分支并最终回收

本文件是 `subagents-develop` 的配套参考，定义"合入主分支并最终回收"阶段的完整流程。只有用户在当前任务中明确命令把父任务分支合入主分支时才执行本阶段。继续使用任务开始时确认并记录的 `mainBranch`、父任务分支和 `task-slug`；任一身份无法精确回读时停止询问，不得按名称猜测。

四项 NUL 安全盘点的命令与判定规则见 `references/safety-inventory.md`，本文件中所有"四项盘点"均指该规程。

## 1. 合并前查漏

对当前 `task-slug` 做第一次查漏：

```text
git worktree list --porcelain -z
git for-each-ref --format='%(refname)' "refs/heads/agent/<task-slug>/"
find "<worktree-root>" -mindepth 1 -maxdepth 1 -print0
```

仅在已确认的 `<worktree-root>` 存在时执行第三项。解析全部 Git worktree 记录、该精确 ref 前缀下的全部本地分支，以及 worktree 根的直属磁盘项；不得漏掉 Git 已注销但目录仍存在的残留。

确认所有 subagent 已结束。对遗漏的子资源重新执行安全盘点与祖先检查：已合入父分支且安全的立即清理；存在脏 worktree、隐藏文件、特殊 index flag、未登记目录或未合入成果时，保留并报告，**阻止主分支合并**，不得跳过。

## 2. 固定 SHA 与父侧盘点

记录父任务分支完整 `parentHead` 和主分支合并前完整 `preMergeMain`：

```text
git rev-parse "refs/heads/agent/<task-slug>/integration"
git rev-parse "<mainBranchRef>"
```

父任务分支在移交阶段已被 detach，此时以分支 ref 为准（不得用父 worktree 的 HEAD 代替）。父 worktree 必须通过四项盘点，且 `refs/heads/agent/<task-slug>/integration` 必须仍等于 `parentHead`；存在任何特殊 index flag、改动、普通 untracked 或 ignored 文件时停止，列出路径并询问用户处理（组合检查产生的 ignored 构建产物是常见原因，经用户确认删除后可重试盘点）。**如果父 worktree 已被用户自行删除**（Git worktree 记录中不存在），跳过父 worktree 盘点，仅以分支 ref 为准继续。全程不得用之后可能移动的分支名作为待合并对象，只用固定 SHA。

## 3. 生成并验证候选 merge commit

从两个固定 SHA 生成候选 merge commit，不直接移动主分支：

```text
git merge-tree --write-tree "<preMergeMain>" "<parentHead>"
git commit-tree "<candidateTree>" \
  -p "<preMergeMain>" \
  -p "<parentHead>" \
  -m "merge: integrate agent/<task-slug>/integration"
git rev-list --parents -n 1 "<candidateMerge>"
git rev-parse "<candidateMerge>^{tree}"
```

以上命令逐条检查退出码并从 stdout 读取 SHA，不使用命令替换。`merge-tree` 有冲突，或候选 commit 的 tree、第一 parent、第二 parent 不分别精确等于 `candidateTree`、`preMergeMain`、`parentHead` 时，停止并报告。

创建 detached 候选 worktree（路径为 `<worktree-root>/candidate`）：

```text
git worktree add --detach -- "<worktree-root>/candidate" "<candidateMerge>"
```

在其中运行任务组合检查和仓库要求的 fresh exact-diff 独立审查；检查及审查必须针对 `preMergeMain..candidateMerge`。更新主分支前，候选 worktree 必须通过四项盘点，确保稍后可以无损移除；检查命令产生的 ignored 构建产物导致盘点不过时，列出路径并询问用户，经确认删除后重试盘点。

## 4. 选择主分支更新位置

主分支只能在它自己的安全 worktree 中更新：

- 主分支已经在某个 worktree 中 checkout：只在精确识别且四项盘点通过的该 worktree 中合并。该 worktree 的 ignored 文件可以保留，但更新命令必须带 `--no-overwrite-ignore` 阻止覆盖它们；其余三项盘点必须完全通过。若它含有改动或属于无法协调的其他 Agent，停止询问用户。
- 主分支没有在任何 worktree 中 checkout：在当前 `task-slug` 命名空间下创建临时主分支合并 worktree（路径为 `<worktree-root>/main-merge`），回读 branch、HEAD 和 `preMergeMain` 全部匹配后使用：

  ```text
  git worktree add -- "<worktree-root>/main-merge" "<mainBranch>"
  ```

  临时主分支合并 worktree 必须完全干净（四项盘点全部零记录）。
- 不得切换用户最初打开的共享 checkout，不得覆盖其中的任何内容。

## 5. 快进到候选

执行更新前最后一次回读主分支 HEAD，必须仍等于 `preMergeMain`，然后只允许 fast-forward 到固定的候选 SHA：

```text
git -C "<absolute-main-worktree>" merge \
  --ff-only \
  --no-overwrite-ignore \
  -- "<candidateMerge>"
```

如果其他主 Agent 已推进主分支，`--ff-only` 必须拒绝不属于候选历史的 HEAD；此时重新基于新 HEAD 生成、检查和审查新的候选，不能沿用旧结论。遇到共享 Git lock 竞争时只做有限重试，每次重试前重新读取主分支和 worktree 状态。

更新失败时不得 reset、改写主分支或清理任务资源；保留现场并报告。更新成功后，主分支 HEAD 必须精确等于 `candidateMerge`，且其两个 parent 仍精确等于 `preMergeMain` 和 `parentHead`，否则不得进入最终回收。

## 6. 最终回收

确认主分支已精确更新且 `parentHead` 是新 HEAD 的祖先后，按以下顺序回收：

1. 第二次列举该 `task-slug` 下的全部 worktree 和本地分支；对任何漏清的 subagent 资源再次执行完整安全盘点，只有已合入主分支且安全时才删除。
2. 再次确认 `refs/heads/agent/<task-slug>/integration` 精确等于 `parentHead`，并对父 worktree 执行四项盘点；全部通过后移除父 worktree。
3. 从主分支 worktree 执行 `git branch -d -- "agent/<task-slug>/integration"`，不得使用 `-D`。如果该分支被用户在某个 checkout 中占用导致删除失败，保留该分支并明确报告，不得强制删除。
4. 对 detached 候选 worktree 执行四项盘点，全部通过后移除。
5. 如果主 Agent 为本阶段创建了临时主分支合并 worktree，对它执行四项盘点，全部通过后移除；不得移除任务开始前已经存在的主分支 worktree。
6. 第三次执行 Git worktree、ref 和磁盘直属项枚举。当前 `task-slug` 前缀下必须没有本地分支，也不得残留父、子、候选或临时主分支 worktree；worktree 根为空时只用非递归方式移除该空目录。否则保留无法安全删除的资源并明确报告清理未完成。

清理只匹配当前主 Agent 的精确 `task-slug` 命名空间。不得删除其他主 Agent 的同名任务资源，不得使用 `git worktree prune` 代替逐项核对。
