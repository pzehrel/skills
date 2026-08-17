---
name: multiple-agent-develop
description: 在 Git 仓库中自主编排多 Agent 开发：确认主分支，为当前主 Agent 从主分支创建独立父任务分支和父 worktree，自主判断任务的串并行关系，为每个 subagent 创建独立子分支和子 worktree，完成开发后将成果收束到父任务分支并清理子资源；用户之后再次显式调用 `$multiple-agent-develop` 并命令合入主分支时，完成合并并清理该主 Agent 的父子分支和 worktree。仅当用户显式调用 `$multiple-agent-develop` 时使用；不得为普通开发或一般 Git 请求隐式触发。
---

# 多 Agent 开发编排

把当前 Agent 作为一个独立的主 Agent。接收用户的完整开发任务后，自主完成任务拆解、串并行调度、subagent 扇出、Git 隔离、成果集成和子资源回收。允许多个主 Agent 同时在同一仓库工作；每个主 Agent 只能管理自己的命名空间。

## 授权边界

除非用户明确要求仅规划，显式调用本 skill 即授权当前主 Agent 在用户指定任务范围内端到端执行以下本地动作：

- 确认主分支并记录精确 base commit/tree；
- 创建一个父任务分支和父 worktree；
- 创建 subagent 子分支和子 worktree；
- 派发、跟进和等待 subagent；
- 创建任务所需的本地 commit；
- 将 subagent 子分支合入父任务分支；
- 运行范围内检查；
- 删除干净且已合并的 subagent worktree 和本地子分支。

不得据此执行 push、force-push、创建 PR、合入主分支、删除远端分支、删除未合并分支，或丢弃任何未提交内容。不得把父任务分支自动合回用户原分支或主分支。

用户在本次任务中之后再次显式调用 `$multiple-agent-develop` 并明确下令把父任务分支合入主分支时，该命令才授权进入“合入主分支并最终回收”阶段：把当前 `run-id` 的父任务分支合入已确认的主分支，并在成功后安全删除当前主 Agent 创建的父子本地分支和 worktree。该命令仍不授权 push、删除远端分支、强制删除或丢弃内容。

读取仓库根及相关路径下的 `AGENTS.md` 等本地指令并遵守它们。本 skill 只负责任务编排和本地 Git 生命周期，不增加项目专属的准入、角色或台账要求。

## 保护共享仓库

先只读检查：

```text
git rev-parse --show-toplevel
git status --short --branch
git branch --show-current
git worktree list --porcelain
git remote
git branch --list
```

把用户最初打开的仓库 checkout 视为共享控制入口，不在其中切换到父任务分支，也不把它作为 subagent 开发目录。这样多个主 Agent 可以同时从同一仓库开始任务，而不会互相改变当前分支。

保留共享 checkout 中已有的全部修改和未跟踪文件。不得自动 stash、reset、clean、restore、checkout 路径、提交、移动或复制这些内容。父任务分支只从已确认主分支的已提交 SHA 创建；如果任务依赖共享 checkout 中的未提交内容，停止并询问用户如何处理。

## 确认主分支

每个主 Agent 都必须先确认主分支，不得根据当前 checkout 或常见名称猜测。

按以下优先级确认：

1. 用户在当前任务中明确指定的主分支；
2. 仓库只有一个 remote，该 remote 的 symbolic HEAD 唯一指向一个默认分支，并且对应本地分支存在；
3. 仓库存在多个 remote，但每个 remote 都有 symbolic HEAD，且全部一致指向同一个默认分支，并且对应本地分支存在。

使用以下只读信息辅助判断：

```text
git remote
git symbolic-ref --quiet "refs/remotes/<remote>/HEAD"
git for-each-ref --format='%(refname)' refs/heads refs/remotes
```

以下信息不能单独证明主分支：当前分支、名为 `main`、`master` 或 `develop` 的分支、提交最多的分支，或另一个 Agent 正在使用的分支。

如果唯一 remote 没有 symbolic HEAD、存在多个 remote 且 symbolic HEAD 缺失或指向不一致、本地默认分支缺失，或仍存在多个合理候选，停止并询问用户主分支名称。不得自行选择。当前 checkout 处于 detached HEAD 本身不能证明或否定主分支，仍按上述 remote 证据判断。

确认后解析并记录：

```text
mainBranch
mainBranchRef
baseCommit
baseTree
remoteFreshness
```

默认使用本地主分支当前 commit，不自动 fetch，并明确记录未校验远端新鲜度。只有用户要求基于远端最新主分支时才 fetch。

## 建立主 Agent 的独立命名空间

为当前任务生成唯一 `run-id`，格式使用 `<YYYYMMDD-HHMMSS>-<task-slug>-<agent-or-random8>`。优先使用可用的 Agent/thread/task 短标识；没有稳定标识时生成随机 8 位后缀。创建前检查现有 refs 和 worktree；原子创建仍发生冲突时重新生成整个 `run-id`。即使多个主 Agent 同时收到同名任务，也不能共享命名空间。

把 `task-slug`、Agent 标识和 `subtask-id` 规范化为小写 ASCII，仅允许 `[a-z0-9-]`；连续连字符折叠为一个，去掉首尾连字符，并限制为合理长度。不得把用户原文、分支名或路径直接拼接进 shell 源码。优先使用 argv 数组调用 Git；必须经过 shell 时，把动态值绑定为变量并逐项可靠引用，禁止 `eval`、命令替换和未引用的变量展开。`git check-ref-format` 只验证 ref 格式，不证明参数可以安全拼入 shell。

使用以下结构：

```text
父任务分支：codex/<run-id>/integration
子任务分支：codex/<run-id>/<subtask-id>
worktree 根：<repo-parent>/<repo-name>-worktrees/<run-id>/
父 worktree：<worktree-root>/integration
子 worktree：<worktree-root>/<subtask-id>
```

不要创建 `codex/<run-id>` 分支后再创建 `codex/<run-id>/<subtask-id>`，避免 Git ref 文件/目录前缀冲突。

在任何创建动作前，把路径解析为显式绝对路径，并验证：

- 父分支和目标路径都不存在；
- `git check-ref-format --branch "<branch>"` 成功；
- worktree 根不是系统临时目录；
- 目标不属于另一个主 Agent 的命名空间。

多个主 Agent 可能同时写入共享 Git 元数据。遇到 `index.lock`、`packed-refs.lock`、`config.lock` 或其他锁竞争时，重新读取状态并做有限重试；不得删除来源不明的 Git lock 文件。

## 创建父任务分支和父 worktree

从已确认主分支的完整 `baseCommit` 直接创建：

```text
git worktree add \
  -b "codex/<run-id>/integration" \
  -- "<absolute-parent-worktree>" \
  "<full-base-commit>"
```

立即回读：

```text
git worktree list --porcelain
git -C "<absolute-parent-worktree>" branch --show-current
git -C "<absolute-parent-worktree>" rev-parse HEAD
git -C "<absolute-parent-worktree>" rev-parse 'HEAD^{tree}'
git -C "<absolute-parent-worktree>" status --short
```

只有父分支、base commit/tree、路径和干净状态全部匹配时才继续。父任务分支创建后，即使主分支被其他人推进，也不得自动 rebase 或移动 base；最终汇报原始 base 和当前差异。

## 自主拆解串并行任务

只在父 worktree 中分析代码并建立子任务依赖图。按依赖关系组成一个或多个执行波次。

满足以下条件时才并行：

- 子任务之间没有前置依赖；
- 预计写入路径不重叠；
- 不共同修改 lockfile、schema、migration、入口文件或生成物；
- 一个子任务不依赖另一个尚未产出的接口或类型；
- 每个子任务都可以独立检查和提交。

存在依赖、共享写入、顺序敏感或边界不确定时改为串行。共享 manifest 与 lockfile 交给同一个子任务；共享接口和集成胶水交给单一 owner，或安排在后续波次。

不要为了并行而过度拆分。任务无法有效并行时，可以只派发一个 subagent。主 Agent 负责编排与集成，不把整体任务再交给另一个主 Agent，也不允许 subagent 继续递归扇出 subagent。

## 按波次创建和派发 subagent

每个波次开始时，固定父任务分支当前完整 SHA 为 `waveBase`。同一波次的所有子分支必须从同一个 `waveBase` 创建：

```text
git worktree add \
  -b "codex/<run-id>/<subtask-id>" \
  -- "<absolute-child-worktree>" \
  "<full-wave-base>"
```

逐个创建并回读 branch、HEAD、tree 和干净状态，确认后再派发。并发 subagent 数量不得超过当前可用 agent slots。

给每个 subagent 的任务必须包含：

- 精确任务目标和完成条件；
- 子分支、`waveBase` 和绝对 worktree 路径；
- 允许修改的路径以及禁止触碰的共享文件；
- 已知接口、依赖和其他子任务边界；
- 需要运行的范围内检查；
- 必须提交全部预期改动并保持 worktree 干净；
- 返回 commit SHA、变更文件、检查结果和剩余风险；
- 禁止创建更多 subagent、切换分支、merge、rebase、push、删除分支或移除 worktree。

主 Agent 持续跟进 subagent。subagent 提前结束但未满足交付条件时，发送明确的 follow-up；不要由主 Agent 猜测或重写其未交接内容。

## 收集并合入父任务分支

等待当前波次的 subagent 全部完成，再逐个检查：

- subagent 已停止运行；
- 子 worktree 干净；
- 子分支 HEAD 包含至少一个预期任务 commit；
- commit 从 `waveBase` 派生；
- 实际变更没有越过任务范围；
- 范围内检查结果可回读。

子 worktree 仍有未提交内容时，优先要求原 subagent 完成提交；无法完成时保留分支和 worktree，并把该子任务标记为 blocked，不得清理。

在干净的父 worktree 中按依赖拓扑顺序集成；同一波次内使用稳定的 subtask ID 顺序：

```text
git merge --no-ff \
  -m "merge: integrate <subtask-id> into <run-id>" \
  -- "codex/<run-id>/<subtask-id>"
```

冲突只能在父 worktree 中处理。能够安全解决时由主 Agent完成并记录；无法判断时中止当前集成，保留相关子资源并报告 blocker。不得通过 `ours` merge 或丢弃子分支内容来制造“已合并”状态。

每个波次集成后运行组合检查。若发现需要修复，创建新的修复子任务和新一轮子分支/worktree，而不是让已经完成的 subagent 在已交接目录中继续漂移。下一波次必须从更新后的父任务分支 HEAD 创建。

## 清理 subagent 资源

只有子任务已经合入父任务分支、父分支组合检查没有要求回退该成果、子 worktree 干净且 subagent 已结束时，才执行清理：

```text
git -C "<absolute-child-worktree>" ls-files -v -z
git -C "<absolute-child-worktree>" ls-files -t -z
git -C "<absolute-child-worktree>" status --porcelain=v1 -z --untracked-files=all
git -C "<absolute-child-worktree>" ls-files -z --others --ignored --exclude-standard
git merge-base --is-ancestor "<child-branch>" "<parent-branch>"
git worktree remove -- "<absolute-child-worktree>"
git -C "<absolute-parent-worktree>" branch -d -- "<child-branch>"
git worktree list --porcelain
```

以上命令必须逐条独立执行并检查退出码，任何一项非零都立即停止；不得用未启用 `pipefail` 的管道掩盖 Git 失败。前四项使用 NUL 分隔记录并由调用方解析：第一项不得出现小写 tag（`assume-unchanged`），第二项不得出现 `S` tag（`skip-worktree`），第三、四项必须是零条记录。第三项强制显示全部普通 untracked 和可见 tracked 变更，不受 `status.showUntrackedFiles` 配置影响；第四项单独显示 ignored 文件。存在特殊 index flag、任何改动或额外文件时，不得移除 worktree；应保留资源、列出路径并询问用户如何处理。显式调用本 skill 本身不授权删除这些可能被 Git 常规 clean 检查隐藏的内容。

不得使用 `git branch -D` 自动清理。不得移除脏 worktree、未合并分支或其他主 Agent 的资源。删除远端分支不在本 skill 范围内。

可以在每个波次成功后清理该波次，也可以在最终组合检查后统一清理；结束前必须确认全部成功子任务的子 worktree 和本地子分支已经删除。

## 合入主分支并最终回收

只有用户在当前任务中再次显式调用 `$multiple-agent-develop` 并明确命令合入主分支时才执行本阶段。继续使用任务开始时确认并记录的 `mainBranch`、父任务分支和 `run-id`；任一身份无法精确回读时停止询问，不得按名称猜测。

合并前先对当前 `run-id` 做第一次查漏：

```text
git worktree list --porcelain -z
git for-each-ref --format='%(refname)' "refs/heads/codex/<run-id>/"
find "<worktree-root>" -mindepth 1 -maxdepth 1 -print0
```

仅在已确认的 `<worktree-root>` 存在时执行第三项。解析全部 Git worktree 记录、该精确 ref 前缀下的全部本地分支，以及 worktree 根的直属磁盘项；不得把 Git 已注销但目录仍存在的残留漏掉。确认所有 subagent 已结束；对遗漏的子资源重新执行“清理 subagent 资源”的盘点与祖先检查。已合入父分支且安全的子资源立即清理；存在脏 worktree、隐藏文件、特殊 index flag、未登记目录或未合入成果时，保留并报告，阻止主分支合并，不得跳过。

记录父任务分支完整 `parentHead` 和主分支合并前完整 `preMergeMain`。父 worktree 在此时就必须通过与删除前相同的四项 NUL 数据盘点，父分支 HEAD 必须仍等于 `parentHead`；存在任何特殊 index flag、改动、普通 untracked 或 ignored 文件时停止。不得用之后可能移动的父分支名作为待合并对象。

从两个固定 SHA 生成候选 merge commit，不直接移动主分支：

```text
git merge-tree --write-tree "<preMergeMain>" "<parentHead>"
git commit-tree "<candidateTree>" \
  -p "<preMergeMain>" \
  -p "<parentHead>" \
  -m "merge: integrate codex/<run-id>/integration"
git rev-list --parents -n 1 "<candidateMerge>"
git rev-parse "<candidateMerge>^{tree}"
```

以上命令逐条检查退出码并从 stdout 读取 SHA，不使用命令替换。`merge-tree` 有冲突或候选 commit 的 tree、第一 parent、第二 parent 不分别精确等于 `candidateTree`、`preMergeMain`、`parentHead` 时停止。使用 `git worktree add --detach -- "<absolute-candidate-worktree>" "<candidateMerge>"` 创建候选 worktree，在其中运行任务组合检查和仓库要求的 fresh exact-diff 独立审查；检查及审查必须针对 `preMergeMain..candidateMerge`。更新主分支前，候选 worktree 必须通过完整四项 NUL 数据盘点，确保稍后可以无损移除。

候选通过后，主分支只能在它自己的安全 worktree 中更新：

- 主分支已经在某个 worktree 中 checkout：只在精确识别且安全盘点通过的该 worktree 中合并；若它含有改动或属于无法协调的其他 Agent，停止询问用户。
- 主分支没有在任何 worktree 中 checkout：在当前 `run-id` 下创建临时主分支合并 worktree，回读 branch、HEAD 和 `preMergeMain` 全部匹配后使用。
- 不得切换用户最初打开的共享 checkout，不得覆盖其中的任何内容。

既有主分支 worktree 的盘点必须逐条成功：不得存在 `assume-unchanged`、`skip-worktree`、可见 tracked 改动或普通 untracked 文件。ignored 文件可以保留，但更新命令必须阻止覆盖它们。临时主分支合并 worktree 必须完全干净。

执行更新前最后一次回读主分支 HEAD，必须仍等于 `preMergeMain`，然后只允许 fast-forward 到固定的候选 SHA：


```text
git -C "<absolute-main-worktree>" merge \
  --ff-only \
  --no-overwrite-ignore \
  -- "<candidateMerge>"
```

如果其他主 Agent 已推进主分支，`--ff-only` 必须拒绝不属于候选历史的 HEAD；重新基于新 HEAD 生成、检查和审查新的候选，不能沿用旧结论。`--no-overwrite-ignore` 拒绝覆盖既有 ignored 文件。遇到共享 Git lock 竞争时只做有限重试，并在每次重试前重新读取主分支和 worktree 状态。

更新失败时不得 reset、改写主分支或清理任务资源；保留现场并报告。更新成功后，主分支 HEAD 必须精确等于 `candidateMerge`，且其两个 parent 仍精确等于 `preMergeMain` 和 `parentHead`，否则不得进入最终回收。

确认主分支已精确更新且 `parentHead` 是新 HEAD 的祖先后，按以下顺序最终回收：

1. 第二次列举该 `run-id` 下的全部 worktree 和本地分支；对任何漏清的 subagent 资源再次执行完整安全盘点，只有已合入主分支且安全时才删除。
2. 再次确认父分支 HEAD 精确等于 `parentHead`，并对父 worktree 执行与子 worktree 相同的四项数据盘点；全部命令成功且没有特殊 index flag、改动、普通 untracked 或 ignored 文件后，移除父 worktree。
3. 从主分支 worktree 执行 `git branch -d -- "codex/<run-id>/integration"`，不得使用 `-D`。
4. 对 detached 候选 worktree 再执行完整四项数据盘点，全部通过后移除。
5. 如果主 Agent 为本阶段创建了临时主分支合并 worktree，对它再执行完整四项数据盘点，全部通过后移除；不得移除任务开始前已经存在的主分支 worktree。
6. 第三次执行上述 Git worktree、ref 和磁盘直属项枚举。当前 `run-id` 前缀下必须没有本地分支，也不得残留父、子、候选或临时主分支 worktree；worktree 根为空时只用非递归方式移除该空目录。否则保留无法安全删除的资源并明确报告清理未完成。

清理只匹配当前主 Agent 的精确 `run-id` 命名空间。不得删除其他主 Agent 的同名任务资源，不得使用 `git worktree prune` 代替逐项核对。

## 失败与部分完成

- 无法确认主分支：询问用户，不创建任何分支或 worktree。
- 父任务分支创建失败：停止，不派发 subagent。
- 某个 subagent 失败：保留它的分支和 worktree；只集成与它无依赖且验证有效的成果。
- 集成或组合检查失败：保留相关资源，继续安排可安全执行的修复波次；无法继续时报告 blocker。
- 候选生成、审查或主分支 fast-forward 失败：不得 reset 或改写主分支；保留父任务分支、父 worktree 和无法安全移除的候选现场并报告。
- 用户中断任务：停止新的派发，保留所有尚未安全集成的分支和 worktree。
- 发现其他主 Agent 的分支或 worktree：只读识别，不修改、不合并、不清理。

资源保留优先于形式上的“全部清理”。只有成功集成的子资源才允许自动删除。

## 结束状态

开发完成但用户尚未命令合入主分支时，只保留当前主 Agent 的：

```text
codex/<run-id>/integration
<worktree-root>/integration
```

所有 subagent 都已结束，所有成功成果都已收束到父任务分支，所有子 worktree 和已合并的本地子分支都已删除。默认保留父 worktree 供用户检查。

用户之后明确命令合入主分支且合并成功时，主分支包含 `parentHead`，当前 `run-id` 下的父分支、子分支、父 worktree、子 worktree、候选 worktree 和临时主分支合并 worktree 均已删除，随后停止。若任何资源因数据保护条件无法删除，不得声称最终回收完成。

停止前向用户报告：

- 确认的主分支及父任务分支的 base commit/tree；
- 父任务分支、父 worktree 和最终 HEAD；
- 串并行波次及每个 subagent 的任务；
- 已合入的 commit 和组合检查结果；
- 已删除的子 worktree 和子分支；
- 合入主分支后已删除的父任务 worktree 和父任务分支；
- 合入主分支时的 `preMergeMain`、`parentHead`、合并后主分支 HEAD，以及父子资源最终查漏结果；
- 因失败而保留的资源及恢复方式；
- 与当前主分支的 ahead/behind 或漂移情况；
- 是否执行主分支合并，以及未执行的 push、PR 或远端清理。
