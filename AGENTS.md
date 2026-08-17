# AGENTS.md

本文件面向 AI 编码助手（Claude Code、Cursor、Codex 等），说明本仓库的结构与工作约定。

## 仓库概述

这是一个 **Agent Skills 集合**仓库。每个技能是 `skills/` 下的一个独立目录，包含 `SKILL.md` 指令文件及可选的辅助资源（脚本、参考文档、模板等）。

```
skills/<skill-name>/
├── SKILL.md        # 技能定义（必填）：YAML frontmatter + 指令正文
├── scripts/        # 可选：可执行脚本
├── references/     # 可选：参考文档（按需加载，避免占用上下文）
└── assets/         # 可选：模板、示例等资源
```

技能必须放在 `skills/` 下：这是 skills CLI（`npx skills`）等工具的标准发现位置，支持 `skills/<name>/SKILL.md` 及最多两层分类（`skills/<category>/<name>/SKILL.md`）。不要在仓库根目录直接放置技能目录。

## 核心规范

### SKILL.md 格式

每个技能必须包含合法的 YAML frontmatter：

```markdown
---
name: skill-name          # 与目录名一致，kebab-case
description: 一句话描述技能的用途和触发场景
---

# Skill Name

具体的任务指令……
```

### 编写原则

- **name**：小写字母、数字、连字符，必须与目录名一致。
- **description**：说明"做什么"和"什么时候用"，这是 Agent 选择技能的主要依据；控制在 1～2 句话。
- **正文精简**：SKILL.md 会被加载进上下文，保持简洁；长篇参考材料拆到 `references/`，正文注明按需读取。
- **指令可执行**：给出具体步骤、命令或检查清单，避免空泛建议。
- **脚本注明依赖**：`scripts/` 中的脚本需说明运行方式和依赖环境。

## 添加新技能的流程

1. 在 `skills/` 下创建 kebab-case 目录 `skills/<skill-name>/`（分类布局用 `skills/<category>/<skill-name>/`）
2. 编写 `SKILL.md`（frontmatter + 指令正文）
3. 按需添加 `scripts/`、`references/`、`assets/`
4. 自检清单：
   - [ ] `name` 与目录名一致
   - [ ] `description` 清晰说明用途与触发场景
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
