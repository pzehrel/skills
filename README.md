# Skills

[![skills.sh](https://skills.sh/b/pzehrel/skills)](https://skills.sh/pzehrel/skills)

自用的 Agent Skills 集合。每个技能是 `skills/` 下的一个独立目录，包含 `SKILL.md` 指令文件及其辅助资源（脚本、模板、参考文档等）。

```
skills/
└── <skill-name>/
    ├── SKILL.md        # 技能定义：YAML frontmatter + 指令正文
    ├── scripts/        # 可选：可执行脚本
    ├── references/     # 可选：参考文档
    └── assets/         # 可选：模板、示例等资源
```

## 技能列表

| Skill | 作用 |
| --- | --- |
| [`easyeda-engineering`](skills/easyeda-engineering/) | 对 EasyEDA 原理图和 PCB 进行工程设计、重构与审查，并通过设计数据完成连接性、DRC 和生产前验证。 |
| [`write-code-docs`](skills/write-code-docs/) | 编写和审查准确、清晰的英语加本地化语言双语代码注释、API 文档、README、指南、示例及其他 Markdown 文档。 |
| [`maintain-agents-md`](skills/maintain-agents-md/) | 创建、更新、重组或审查仓库的 `AGENTS.md` 及其指令层级，使持久规则范围明确、精简且可验证。 |
| [`maintain-js-package-docs`](skills/maintain-js-package-docs/) | 在开发公共 JavaScript 或 TypeScript 包时，同步维护 README、随包文档、示例以及 JSDoc/TSDoc，使文档与发布代码和版本保持一致。 |
| [`subagents-develop`](skills/subagents-develop/) | 在隔离的 Git 分支和 worktree 中编排开发、修复或重构任务，并在适合时并行派发多个 Agent 后收束成果。 |

## 使用方式

### 通过 gh CLI 安装（推荐）

GitHub CLI（v2.97+）内置了 `gh skill` 命令（preview 阶段）：

```bash
# 预览技能内容
gh skill preview pzehrel/skills <skill-name>

# 安装指定技能
gh skill install pzehrel/skills <skill-name>

# 列出已安装的技能
gh skill list
```

### 通过 skills CLI 安装

```bash
# 列出本仓库的技能
npx skills add pzehrel/skills --list

# 安装指定技能
npx skills add pzehrel/skills --skill <skill-name>
```

### 手动链接

将技能目录复制或链接到 Agent 的技能加载路径（如 Claude Code 的 `~/.claude/skills/` 或项目的 `.claude/skills/`）：

```bash
# 示例：链接单个技能
ln -s "$(pwd)/skills/<skill-name>" ~/.claude/skills/<skill-name>
```

## 许可证

[MIT](LICENSE)
