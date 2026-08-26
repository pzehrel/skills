# AGENTS.md

本文件面向 AI 编码助手（Claude Code、Cursor、Codex 等），说明本仓库的结构与工作约定。

## 仓库概述

这是一个 **Agent Skills 集合**仓库。每个技能是 `skills/` 下的一个独立目录，包含 `SKILL.md` 指令文件及可选的辅助资源（脚本、参考文档、模板等）。

```
skills/<skill-name>/
├── SKILL.md           # 英文技能定义（必填）：YAML frontmatter + 指令正文
├── SKILL.zh-CN.md     # 中文技能定义（必填）：独立文件，与英文版语义一致
├── agents/
│   └── openai.yaml    # 可选：英文配置值，并以相邻注释提供中文翻译
├── scripts/           # 可选：可执行脚本
├── references/        # 可选：参考文档；英文与中文文件成对维护
│   ├── guide.md       # 英文参考文档
│   └── guide.zh-CN.md # 中文参考文档
└── assets/            # 可选：模板、示例等资源；其中的文本内容也须双语
```

技能必须放在 `skills/` 下：这是 skills CLI（`npx skills`）等工具的标准发现位置，支持 `skills/<name>/SKILL.md` 及最多两层分类（`skills/<category>/<name>/SKILL.md`）。不要在仓库根目录直接放置技能目录。

## 核心规范

### SKILL.md 格式

每个技能必须包含合法的 YAML frontmatter：

```markdown
---
name: skill-name          # 与目录名一致，kebab-case
description: A concise sentence describing what the skill does and when to use it
---

# Skill Name

Concrete task instructions...
```

### 跨 Agent 兼容

技能必须以 [Agent Skills 开放规范](https://agentskills.io/specification) 为可移植基线，并至少兼容以下 Agent：Codex、Claude Code、Cursor、GitHub Copilot（VS Code/CLI/cloud agent）、Gemini CLI 和 DeepSeek Harness（DSH）。

#### 通用核心

- `SKILL.md` 是唯一的跨 Agent 权威执行说明；不得为不同 Agent 复制多份会独立演化的技能正文。
- frontmatter 必须包含开放规范要求的 `name` 和 `description`。可按需使用当前目标 Agent 均能通过校验的 `license` 和 `metadata`。
- Agent Skills 开放规范虽然定义了 `compatibility`，但当前 Codex `quick_validate` 尚不接受该字段；在兼容矩阵全部通过前，不在 frontmatter 使用它，环境要求改写在正文中。
- `allowed-tools` 在开放规范中仍属实验字段；只有确认目标 Agent 支持且技能确实需要时才可使用，并在 `compatibility` 中记录限制。
- 默认不添加厂商专属字段。确有必要时，只使用目标 Agent 官方文档明确支持的字段，并保证不支持该字段的 Agent 仍能加载和执行核心流程。
- 厂商专属元数据只放在该厂商官方约定的位置。禁止为了目录对称而发明 `agents/claude.yaml`、`agents/cursor.yaml` 等非标准文件。

#### 兼容矩阵

| Agent | 通用技能支持 | 可选专属配置或扩展 | 兼容要求 |
| --- | --- | --- | --- |
| Codex / OpenAI | `SKILL.md` | `agents/openai.yaml` | OpenAI UI、依赖和调用策略放在 `openai.yaml`，不得替代通用 frontmatter。 |
| Claude Code | `SKILL.md` | Claude Code 支持的 frontmatter 字段，如 `argument-hint`、`disable-model-invocation`、`user-invocable`、`allowed-tools`、`context` 和 `agent` | 不创建独立 Claude 元数据文件；扩展字段必须按需使用并验证降级行为。 |
| Cursor | `SKILL.md` | Cursor 支持的 `paths`、`disable-model-invocation`、`icon`、`color` | 专属 UI 或作用域字段不得成为其他 Agent 执行核心流程的前提。 |
| GitHub Copilot | `SKILL.md` | `argument-hint`；`context: fork` 目前属于实验能力 | 未启用实验能力时，技能仍须能以内联上下文完成核心任务。 |
| Gemini CLI | `SKILL.md` | 无必需的独立元数据文件 | 保持 `name`、`description` 和资源相对路径符合开放规范。 |
| DeepSeek Harness（DSH） | `SKILL.md` | DSH 支持的 `whenToUse`、`disable-model-invocation`、`user-invocable` | 不创建独立 DSH 元数据文件；安装时技能目录必须直接位于 `.agents/skills/` 或 `.dsh/skills/` 下，不能依赖递归发现。 |

官方参考：

- [Claude Code Skills](https://code.claude.com/docs/en/skills)
- [Cursor Agent Skills](https://cursor.com/docs/skills)
- [GitHub Copilot / VS Code Agent Skills](https://code.visualstudio.com/docs/agent-customization/agent-skills)
- [Gemini CLI Agent Skills](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/using-agent-skills.md)
- [DeepSeek Harness Skills](https://github.com/deepseek-ai/deepseek-harness/blob/master/docs/subsystems/skills.md)

仓库允许使用 `skills/<category>/<skill-name>/` 分类，但发布或安装到 DSH、Gemini CLI 等只发现单层目录的 Agent 时，必须复制或链接实际的 `<skill-name>/` 到目标技能根目录，不能连同分类层级直接安装。

#### 兼容性验证

- 每次新增或修改 frontmatter、厂商元数据、脚本入口或资源路径时，都要检查兼容矩阵中的 Agent。
- 至少验证开放规范 frontmatter、相对资源链接和厂商配置的语法；环境中已安装对应 Agent 时，进一步验证技能可发现、可加载和可执行。
- 某个 Agent 无法在当前环境实测时，必须在变更说明中标记“未实测”，不得声称已完成该 Agent 的运行验证。
- 厂商专属增强验证失败时，应移除或隔离该增强，不能牺牲通用 `SKILL.md` 的可移植性。

### 中英文双语

新增或修改技能时，必须同步维护英文和中文两个独立文件：

- `SKILL.md`：英文版，也是 skills CLI 和 Agent 默认发现、加载的主文件。
- `SKILL.zh-CN.md`：中文版，使用与英文版相同的完整文档结构。
- 两个文件都必须包含合法的 YAML frontmatter；`name` 必须相同并与技能目录名一致，`description` 使用对应文件的语言。
- 两个版本的能力范围、触发条件、操作步骤、授权边界和安全约束必须保持语义一致。
- 禁止在同一个 `SKILL.md` 中以中英文并排、交替段落等方式混排来代替独立文件。
- 修改任一语言版本时，必须在同一次改动中同步更新另一版本；翻译应保留原意，不得自行扩大或缩小技能能力。

技能目录内所有面向人或 Agent 阅读的文本内容都必须提供中英文双语，而不只限于 `SKILL.md`：

- `references/` 中每份英文 Markdown 文档 `<name>.md` 都必须有对应的中文文档 `<name>.zh-CN.md`，反之亦然。
- `assets/` 中的文本模板、示例、提示词或说明文档也使用相同的配对命名规则；图片、字体、二进制模板等不可翻译资源不需要复制。
- 英文文件是默认加载的权威版本，中文文件是语义一致的完整翻译。两者的章节、步骤、表格、示例、授权边界和安全约束必须对应。
- 英文 `SKILL.md` 及英文 reference 只链接英文资源；中文 `SKILL.zh-CN.md` 及中文 reference 在存在对应翻译时必须链接 `.zh-CN` 资源。
- 修改任一 reference、文本 asset 或其他说明性文本时，必须在同一次改动中同步更新其语言配对文件。
- 机器读取且不支持注释的结构化文件不复制本地化版本；其中面向用户的文字应在官方允许的位置提供双语，或在同目录的成对说明文档中解释。

`agents/openai.yaml` 也必须提供中英文双语内容，但不创建独立的本地化 YAML 文件：

- `display_name`、`short_description`、`default_prompt` 以及其他面向用户的字符串配置值使用英文。
- 每个面向用户的英文配置项必须在紧邻的上一行使用 YAML 注释提供语义一致的中文翻译。
- 中文只作为注释存在，不新增非标准本地化字段，不改变配置结构。
- 修改英文配置值时，必须同步更新对应的中文注释。

示例：

```yaml
interface:
  # 中文：子 Agent 开发
  display_name: "Subagents Develop"
  # 中文：在隔离的分支和 worktree 中编排开发任务
  short_description: "Orchestrate development in isolated branches and worktrees"
  # 中文：使用 $skill-name 在隔离环境中编排并完成开发任务。
  default_prompt: "Use $skill-name to orchestrate and complete development in isolation."
```

如果技能包含 `scripts/`，脚本中的说明性注释和文档字符串也必须使用中英文双语：

- 每组双语注释先写英文，紧接着写语义一致的中文。
- 行注释使用相邻的两行注释；多行注释或文档字符串先给出完整英文内容，再给出完整中文内容。
- 修改代码导致注释含义变化时，必须同步更新中英文内容。
- shebang、编码声明、lint/type-check 指令以及工具要求的特殊注释不属于需要翻译的说明性注释，保持其规定格式。

示例：

```python
# Validate the skill directory before packaging.
# 打包前验证技能目录。
validate_skill(skill_dir)
```

### 编写原则

- **name**：小写字母、数字、连字符，必须与目录名一致。
- **description**：说明"做什么"和"什么时候用"，这是 Agent 选择技能的主要依据；控制在 1～2 句话。
- **正文精简**：SKILL.md 会被加载进上下文，保持简洁；长篇参考材料拆到 `references/`，正文注明按需读取。
- **指令可执行**：给出具体步骤、命令或检查清单，避免空泛建议。
- **脚本注明依赖**：`scripts/` 中的脚本需说明运行方式和依赖环境。

## 添加新技能的流程

1. 在 `skills/` 下创建 kebab-case 目录 `skills/<skill-name>/`（分类布局用 `skills/<category>/<skill-name>/`）
2. 编写英文版 `SKILL.md`（frontmatter + 指令正文）
3. 编写独立中文版 `SKILL.zh-CN.md`，并确保与英文版语义一致
4. 按兼容矩阵添加确有必要的厂商元数据或扩展字段
5. 按需添加 `scripts/`、`references/`、`assets/`，并为所有文本 reference 和文本 asset 创建中英文配对文件
6. 自检清单：
   - [ ] `name` 与目录名一致
   - [ ] `description` 清晰说明用途与触发场景
   - [ ] `SKILL.md` 符合 Agent Skills 开放规范，且未把厂商专属增强作为核心流程前提
   - [ ] 已检查 Codex、Claude Code、Cursor、GitHub Copilot、Gemini CLI 和 DSH 的兼容性
   - [ ] 无法实测的 Agent 已在变更说明中明确标记
   - [ ] `SKILL.md` 与 `SKILL.zh-CN.md` 均存在且 frontmatter 合法
   - [ ] 中英文版本的能力、流程、边界和约束语义一致
   - [ ] `references/` 和 `assets/` 中所有可翻译文本均有 `.md` 与 `.zh-CN.md` 配对，且内容语义一致
   - [ ] 英文文件只链接英文资源，中文文件优先链接对应的 `.zh-CN` 资源
   - [ ] 若存在 `agents/openai.yaml`，面向用户的配置值使用英文且紧邻中文翻译注释
   - [ ] 若存在 `scripts/`，说明性注释和文档字符串均按英文在前、中文在后的顺序提供双语内容
   - [ ] 技能在真实任务中验证可用

## 提交规范

- Commit message 格式：`<skill>: <变更说明>`
  - 例：`pdf: 添加表单填写支持`、`readme: 修正目录结构说明`
- 一次提交聚焦一个技能或一个改动点
- 本仓库无构建和测试流程，改动以文档与脚本为主，提交前确认 SKILL.md 的 YAML frontmatter 语法合法

## 注意事项

- 不要在技能中硬编码绝对路径或机器相关的配置
- 不要提交密钥、令牌等敏感信息（`.env*` 已在 `.gitignore` 中）
- 涉及外部服务/API 的技能，在 `references/` 中说明认证方式与限流注意事项
