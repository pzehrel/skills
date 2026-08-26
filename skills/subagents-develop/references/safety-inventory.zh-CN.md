# 安全盘点与清理规程

本文件是 `subagents-develop` 的配套参考，定义 worktree 创建回读、四项 NUL 安全盘点、子资源删除和清零验证的完整命令与判定规则。`SKILL.zh-CN.md` 中所有"按 safety-inventory 规程执行"的引用均指向本文件。

## 通用执行规则

- 所有命令逐条独立执行并检查退出码，任何一项非零立即停止后续动作；
- 不得用未启用 `pipefail` 的管道掩盖 Git 失败；
- NUL 分隔（`-z` / `-print0`）输出由调用方按记录解析，不得按行猜测含特殊字符的路径；
- 动态值一律绑定为变量并可靠引用，禁止 `eval` 和命令替换拼入 shell。

## 创建回读（父/子 worktree 通用）

每次 `git worktree add` 后立即执行：

```text
git worktree list --porcelain
git -C "<absolute-worktree>" branch --show-current
git -C "<absolute-worktree>" rev-parse HEAD
git -C "<absolute-worktree>" rev-parse 'HEAD^{tree}'
git -C "<absolute-worktree>" status --short
```

只有分支名、base commit/tree、路径和干净状态全部与预期匹配时才继续；任一不匹配立即停止并排查，不得在该 worktree 中继续工作。

## 四项 NUL 安全盘点

移除任何 worktree（子、父、候选、临时主分支 worktree）之前，必须对该 worktree 逐项执行：

```text
git -C "<absolute-worktree>" ls-files -v -z
git -C "<absolute-worktree>" ls-files -t -z
git -C "<absolute-worktree>" status --porcelain=v1 -z --untracked-files=all
git -C "<absolute-worktree>" ls-files -z --others --ignored --exclude-standard
```

判定规则：

1. 第一项（`ls-files -v`）不得出现小写 tag，即不得存在 `assume-unchanged` 条目；
2. 第二项（`ls-files -t`）不得出现 `S` tag，即不得存在 `skip-worktree` 条目；
3. 第三项（`status --porcelain -z --untracked-files=all`）必须是零条记录——强制显示全部普通 untracked 和可见 tracked 变更，不受 `status.showUntrackedFiles` 配置影响；
4. 第四项（`ls-files --others --ignored`）必须是零条记录——单独暴露 ignored 文件。

存在特殊 index flag、任何改动或额外文件时，不得移除该 worktree：保留资源、列出路径并询问用户如何处理。调用本 skill（含自动触发）本身不授权删除这些可能被 Git 常规 clean 检查隐藏的内容。

例外：既有主分支 worktree（非本任务创建）允许保留 ignored 文件，但后续更新命令必须带 `--no-overwrite-ignore` 阻止覆盖它们；其余三项仍必须完全通过。

## 子资源删除顺序

四项盘点全部通过、祖先检查确认已合入后，按序执行：

```text
git merge-base --is-ancestor "<child-branch>" "<parent-branch>"
git worktree remove -- "<absolute-child-worktree>"
git -C "<absolute-parent-worktree>" branch -d -- "<child-branch>"
git worktree list --porcelain
```

先 `merge-base --is-ancestor` 验证已合入，再移除 worktree，最后用 `branch -d`（拒绝未合并分支）删除本地子分支，并回读 worktree 列表确认。任何一步非零立即停止，保留现场。

## 子资源清零验证（移交前必做）

进入移交阶段前，对当前 `task-slug` 执行：

```text
git worktree list --porcelain -z
git for-each-ref --format='%(refname)' "refs/heads/agent/<task-slug>/"
find "<worktree-root>" -mindepth 1 -maxdepth 1 -print0
```

仅在已确认的 `<worktree-root>` 存在时执行第三项。解析全部 Git worktree 记录、该精确 ref 前缀下的全部本地分支，以及 worktree 根的直属磁盘项；不得漏掉 Git 已注销但目录仍存在的残留。

通过标准：除 `agent/<task-slug>/integration` 分支和 `<worktree-root>/integration` 父 worktree 外，无任何子分支、子 worktree 或未登记磁盘项。验证输出（三项命令的完整结果）必须纳入最终报告。存在残留时回到"子资源删除顺序"逐项盘点清理；因数据保护条件无法清理的，明确列出路径和保留原因，不得声称清零完成。
