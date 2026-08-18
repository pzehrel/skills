---
name: parallel-agent-dispatch
description: 并行派发独立开发任务。当用户下达开发、修复、重构等代码任务，且需要多 session 并行推进或不想受当前 checkout 状态影响时使用；自动在隔离的分支和 worktree 命名空间中完成开发并收束成果，任务值得并行时才扇出多 Agent，各任务互不冲突。合入主分支需用户明确下令。
license: MIT
metadata:
  repository: https://github.com/pzehrel/skills
---

# 并行 Agent 任务派发编排

把当前 Agent 作为一个独立的主 Agent。用户只管发布任务，无需关心当前代码状态；可以顺序打开多个 session 派发不同任务。接收任务后，主 Agent 自主完成任务拆解、串并行调度、Git 隔离、成果集成和资源回收；**Git 隔离规范（命名空间、worktree、移交、清理、合主纪律）是必选骨架，扇出 subagent 是按需增强**——任务值得并行时才扇出，否则主 Agent 直接在父 worktree 中开发。多个主 Agent 可以同时在同一仓库工作；每个主 Agent 只能管理自己的命名空间，命名空间完全隔离、互不冲突。

## 总流程

1. 只读检查仓库，保护共享 checkout；
2. 确认主分支并记录精确 base；
3. 建立 `task-slug` 命名空间，创建父任务分支和父 worktree；
4. 拆解任务：值得并行时按波次创建并派发 subagent，否则主 Agent 直接在父 worktree 中开发；
5. （有 subagent 时）逐波次集成子分支到父任务分支，清理子资源；
6. 移交：验证子资源清零，detached 父 worktree 释放集成分支，汇报；
7. （仅用户再次显式命令时）合入主分支并最终回收，见 `references/merge-to-main.md`。

## 授权边界

除非用户明确要求仅规划，本 skill 被调用（显式或自动触发）即授权当前主 Agent 在用户指定任务范围内端到端执行以下本地动作：确认主分支并记录 base commit/tree；创建一个父任务分支和父 worktree；创建 subagent 子分支和子 worktree；派发、跟进和等待 subagent；创建任务所需的本地 commit；将 subagent 子分支合入父任务分支；运行范围内检查；detached 父 worktree 的 HEAD；删除干净且已合并的 subagent worktree 和本地子分支。

不得据此执行 push、force-push、创建 PR、合入主分支、删除远端分支、删除未合并分支，或丢弃任何未提交内容。不得把父任务分支自动合回用户原分支或主分支。

用户在本次任务中之后明确下令把父任务分支合入主分支时，才授权进入"合入主分支并最终回收"阶段（流程见 `references/merge-to-main.md`）。该命令仍不授权 push、删除远端分支、强制删除或丢弃内容。

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

把用户最初打开的仓库 checkout 视为共享控制入口：不在其中切换分支，也不把它作为 subagent 开发目录。保留共享 checkout 中已有的全部修改和未跟踪文件；不得自动 stash、reset、clean、restore、checkout 路径、提交、移动或复制这些内容。父任务分支只从已确认主分支的已提交 SHA 创建；任务依赖共享 checkout 中的未提交内容时，停止并询问用户。

## 确认主分支

每个主 Agent 都必须先确认主分支，不得根据当前 checkout 或常见名称猜测。按以下优先级确认：

1. 用户在当前任务中明确指定的主分支；
2. 仓库只有一个 remote，其 symbolic HEAD 唯一指向一个默认分支，且对应本地分支存在；
3. 多个 remote 的 symbolic HEAD 全部一致指向同一个默认分支，且对应本地分支存在。

辅助判断（只读）：

```text
git remote
git symbolic-ref --quiet "refs/remotes/<remote>/HEAD"
git for-each-ref --format='%(refname)' refs/heads refs/remotes
```

以下信息不能单独证明主分支：当前分支、名为 `main`/`master`/`develop` 的分支、提交最多的分支、另一个 Agent 正在使用的分支。唯一 remote 没有 symbolic HEAD、多个 remote 的 symbolic HEAD 缺失或不一致、本地默认分支缺失，或仍存在多个合理候选时，停止并询问用户，不得自行选择。detached HEAD 本身不能证明或否定主分支，仍按 remote 证据判断。

确认后解析并记录 `mainBranch`、`mainBranchRef`、`baseCommit`、`baseTree`、`remoteFreshness`。默认使用本地主分支当前 commit，不自动 fetch，并明确记录未校验远端新鲜度；只有用户要求基于远端最新主分支时才 fetch。

## 建立主 Agent 的独立命名空间

从任务内容提炼一个简短 `task-slug` 作为本次任务的唯一命名空间标识（如 `fix-login`）。`task-slug` 和 `subtask-id` 规范化为小写 ASCII，仅允许 `[a-z0-9-]`；连续连字符折叠为一个，去掉首尾连字符，`task-slug` 限制在 20 字符以内。不得把用户原文、分支名或路径直接拼接进 shell 源码；优先使用 argv 数组调用 Git，必须经过 shell 时把动态值绑定为变量并逐项可靠引用，禁止 `eval`、命令替换和未引用的变量展开。`git check-ref-format` 只验证 ref 格式，不证明参数可以安全拼入 shell。

创建前检查现有 refs 和 worktree：若 `agent/<task-slug>/` 已被另一个进行中的任务占用，依次尝试 `<task-slug>-2`、`<task-slug>-3`；仍冲突时退化为 `<task-slug>-<rand4>`（4 位随机小写字母数字）。多个主 Agent 同时收到同名任务也不得共享命名空间。

命名空间结构（前缀统一为 `agent/`）：

```text
父任务分支：agent/<task-slug>/integration
子任务分支：agent/<task-slug>/<subtask-id>
worktree 根：<repo-parent>/<repo-name>-worktrees/<task-slug>/
父 worktree：<worktree-root>/integration
子 worktree：<worktree-root>/<subtask-id>
候选 worktree（仅合主阶段）：<worktree-root>/candidate
临时主分支合并 worktree（仅合主阶段）：<worktree-root>/main-merge
```

不要创建 `agent/<task-slug>` 分支后再创建 `agent/<task-slug>/<subtask-id>`，避免 Git ref 文件/目录前缀冲突。

在任何创建动作前，把路径解析为显式绝对路径，并验证：目标分支和目标路径都不存在；`git check-ref-format --branch "<branch>"` 成功；worktree 根不是系统临时目录；目标不属于另一个主 Agent 的命名空间。

多个主 Agent 可能同时写入共享 Git 元数据。遇到 `index.lock`、`packed-refs.lock`、`config.lock` 等锁竞争时，重新读取状态并做有限重试；不得删除来源不明的 Git lock 文件。

## 创建父任务分支和父 worktree

从已确认主分支的完整 `baseCommit` 直接创建：

```text
git worktree add \
  -b "agent/<task-slug>/integration" \
  -- "<absolute-parent-worktree>" \
  "<full-base-commit>"
```

立即按 `references/safety-inventory.md` 的"创建回读"规程回读 branch、HEAD、tree 和干净状态，全部匹配才继续。父任务分支创建后，即使主分支被推进，也不得自动 rebase 或移动 base；最终汇报原始 base 和当前差异。

## 自主拆解串并行任务

只在父 worktree 中分析代码并建立子任务依赖图，按依赖关系组成一个或多个执行波次。满足以下全部条件才并行：

- 子任务之间没有前置依赖；
- 预计写入路径不重叠；
- 不共同修改 lockfile、schema、migration、入口文件或生成物；
- 一个子任务不依赖另一个尚未产出的接口或类型；
- 每个子任务都可以独立检查和提交。

存在依赖、共享写入、顺序敏感或边界不确定时改为串行。共享 manifest 与 lockfile 交给同一个子任务；共享接口和集成胶水交给单一 owner，或安排在后续波次。不要为了并行而过度拆分。

**扇出是可选的，Git 隔离不是。** 任务规模小或无法有效并行时，不创建任何 subagent：主 Agent 直接在父 worktree 中开发并提交，随后照常进入移交阶段——移交 detach、子资源清零验证（零扇出时自然通过，仍须执行并纳入报告）和汇报规范全部不变。扇出时主 Agent 负责编排与集成，不把整体任务再交给另一个主 Agent，也不允许 subagent 递归扇出。

## 按波次创建和派发 subagent

本节及随后"收集并合入父任务分支"一节仅在决定扇出时执行；零扇出时主 Agent 直接在父 worktree 中完成开发并提交，然后进入移交阶段。"清理 subagent 资源"一节中的子资源删除规程在零扇出时不适用，但其中的**子资源清零验证在零扇出时仍须执行**并纳入报告（见"移交"一节）。

每个波次开始时，固定父任务分支当前完整 SHA 为 `waveBase`。同一波次的所有子分支必须从同一个 `waveBase` 创建：

```text
git worktree add \
  -b "agent/<task-slug>/<subtask-id>" \
  -- "<absolute-child-worktree>" \
  "<full-wave-base>"
```

逐个创建并按 `references/safety-inventory.md` 回读确认后再派发。并发 subagent 数量不得超过当前可用 agent slots。

给每个 subagent 的任务必须包含：精确任务目标和完成条件；子分支、`waveBase` 和绝对 worktree 路径；允许修改的路径以及禁止触碰的共享文件；已知接口、依赖和其他子任务边界；需要运行的范围内检查；必须提交全部预期改动并保持 worktree 干净；返回 commit SHA、变更文件、检查结果和剩余风险；禁止创建更多 subagent、切换分支、merge、rebase、push、删除分支或移除 worktree。

主 Agent 持续跟进 subagent。subagent 提前结束但未满足交付条件时，发送明确的 follow-up；不要由主 Agent 猜测或重写其未交接内容。

## 收集并合入父任务分支

等待当前波次的 subagent 全部完成，再逐个检查：subagent 已停止运行；子 worktree 干净；子分支 HEAD 包含至少一个预期任务 commit 且从 `waveBase` 派生；实际变更没有越过任务范围；范围内检查结果可回读。

子 worktree 仍有未提交内容时，优先要求原 subagent 完成提交；无法完成时保留分支和 worktree，把该子任务标记为 blocked，不得清理。

在干净的父 worktree 中按依赖拓扑顺序集成，同一波次内使用稳定的 subtask ID 顺序：

```text
git merge --no-ff \
  -m "merge: integrate <subtask-id> into <task-slug>" \
  -- "agent/<task-slug>/<subtask-id>"
```

冲突只能在父 worktree 中处理。能安全解决时由主 Agent 完成并记录；无法判断时中止当前集成，保留相关子资源并报告 blocker。不得通过 `ours` merge 或丢弃子分支内容制造"已合并"状态。

每个波次集成后运行组合检查。需要修复时创建新的修复子任务和新一轮子分支/worktree，而不是让已完成的 subagent 在已交接目录中继续漂移。下一波次必须从更新后的父任务分支 HEAD 创建。

## 清理 subagent 资源（完成定义的一部分）

只有子任务已合入父任务分支、父分支组合检查没有要求回退该成果、子 worktree 干净且 subagent 已结束时，才执行清理。清理的安全盘点与删除规程见 `references/safety-inventory.md`，核心规则：

- 所有盘点与删除命令逐条独立执行并检查退出码，任何一项非零立即停止；
- 存在特殊 index flag（`assume-unchanged`/`skip-worktree`）、任何改动、普通 untracked 或 ignored 文件时，不得移除 worktree；保留资源、列出路径并询问用户；
- 不得使用 `git branch -D`；不得移除脏 worktree、未合并分支或其他主 Agent 的资源。

**清理是完成定义的一部分，不是可选项。** 每个波次成功后立即清理该波次子资源。进入移交阶段前，必须执行"子资源清零验证"（枚举命令见 `references/safety-inventory.md`）：当前 `task-slug` 前缀下除 `integration` 外不得残留任何本地分支、worktree 或 worktree 根下的磁盘项；验证输出必须纳入最终报告。清零验证未通过时不得声称开发完成，继续清理或明确报告保留原因。

## 移交：释放集成分支并汇报

全部波次集成完成、组合检查通过且子资源清零验证通过后：

1. 在父 worktree 中执行 `git -C "<absolute-parent-worktree>" checkout --detach HEAD`，使父 worktree 进入 detached HEAD。**这一步必须执行**：它释放 `agent/<task-slug>/integration` 分支，让用户可以在任意 checkout 中切换到该分支查看成果，同时保留父 worktree 供检查。
2. 回读确认父 worktree 为 detached 且 HEAD 等于集成分支最新 commit，父分支 ref 未被移动。
3. 按"结束状态与汇报"要求向用户报告。

## 合入主分支并最终回收

只有用户在当前任务中明确命令把父任务分支合入主分支时才执行本阶段。继续使用任务开始时记录的 `mainBranch`、父任务分支和 `task-slug`；任一身份无法精确回读时停止询问，不得按名称猜测。

核心不变量：

- 先对当前 `task-slug` 做子资源查漏，未安全清理的子资源阻止主分支合并；
- 记录父任务分支完整 `parentHead` 和主分支合并前完整 `preMergeMain`，全程只用固定 SHA，不用可能移动的分支名；
- 用 `git merge-tree --write-tree` + `git commit-tree` 生成候选 merge commit，在 detached 候选 worktree 中完成组合检查和独立审查（针对 `preMergeMain..candidateMerge`）后才允许更新主分支；
- 主分支只能在它自己的安全 worktree 中以 `--ff-only --no-overwrite-ignore` 快进到固定候选 SHA；其他主 Agent 已推进主分支时必须重新生成候选，不得沿用旧结论；
- 更新失败不得 reset、改写主分支或清理任务资源；保留现场并报告；
- 更新成功后按规程顺序回收父 worktree、父分支、候选 worktree 和临时主分支 worktree，并做第三次枚举确认当前 `task-slug` 命名空间无残留。

完整的查漏、候选生成、主分支 worktree 选择和回收顺序见 `references/merge-to-main.md`，必须严格按该文档执行。

## 失败与部分完成

- 无法确认主分支：询问用户，不创建任何分支或 worktree。
- 父任务分支创建失败：停止，不派发 subagent。
- 某个 subagent 失败：保留它的分支和 worktree；只集成与它无依赖且验证有效的成果。
- 集成或组合检查失败：保留相关资源，继续安排可安全执行的修复波次；无法继续时报告 blocker。
- 候选生成、审查或主分支 fast-forward 失败：不得 reset 或改写主分支；保留父任务分支、父 worktree 和无法安全移除的候选现场并报告。
- 用户中断任务：停止新的派发，保留所有尚未安全集成的分支和 worktree。
- 发现其他主 Agent 的分支或 worktree：只读识别，不修改、不合并、不清理。

资源保留优先于形式上的"全部清理"。只有成功集成的子资源才允许自动删除。

## 结束状态与汇报

开发完成但用户尚未命令合入主分支时，当前主 Agent 只保留：

```text
agent/<task-slug>/integration        # 集成分支（未被任何 worktree checkout）
<worktree-root>/integration       # 父 worktree（detached HEAD，供检查）
```

所有 subagent 已结束，所有成功成果已收束到父任务分支，子资源清零验证已通过。用户之后明确命令合入主分支且合并成功时，主分支包含 `parentHead`，当前 `task-slug` 下的全部分支和 worktree 均已删除。任何资源因数据保护条件无法删除时，不得声称最终回收完成。

停止前向用户报告：

- 确认的主分支及父任务分支的 base commit/tree；
- 父任务分支、父 worktree（含 detached 状态）和最终 HEAD；
- 串并行波次及每个 subagent 的任务；
- 已合入的 commit 和组合检查结果；
- 子资源清零验证的枚举输出，以及已删除的子 worktree 和子分支；
- 合入主分支后已删除的父任务 worktree 和父任务分支；
- 合入主分支时的 `preMergeMain`、`parentHead`、合并后主分支 HEAD，以及父子资源最终查漏结果；
- 因失败而保留的资源及恢复方式；
- 与当前主分支的 ahead/behind 或漂移情况；
- 是否执行主分支合并，以及未执行的 push、PR 或远端清理。
