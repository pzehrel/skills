# Skills

可复用的 Agent Skills 集合。每个技能是一个独立目录，包含 `SKILL.md` 指令文件及其辅助资源（脚本、模板、参考文档等）。

## 目录结构

```
skills/
├── <skill-name>/
│   ├── SKILL.md        # 技能定义（必填）：YAML frontmatter + 指令正文
│   ├── scripts/        # 可选：可执行脚本
│   ├── references/     # 可选：参考文档
│   └── assets/         # 可选：模板、示例等资源
├── CONTRIBUTING.md
├── AGENTS.md         # 面向 AI 助手的仓库工作约定
└── README.md
```

## 技能规范

每个技能目录下的 `SKILL.md` 必须包含 YAML frontmatter：

```markdown
---
name: skill-name          # 与目录名一致，kebab-case
description: 一句话描述技能的用途和触发场景
---

# Skill Name

具体的任务指令、工作流程、注意事项……
```

编写要点：

- **name**：小写字母、数字、连字符，与目录名一致。
- **description**：清晰说明"做什么"和"什么时候用"，这是 Agent 选择技能的主要依据。
- **正文**：给出具体、可执行的操作步骤；保持精简，把长篇参考材料拆到 `references/` 中按需加载。

## 使用方式

将技能目录复制或链接到 Agent 的技能加载路径（如 Claude Code 的 `~/.claude/skills/` 或项目的 `.claude/skills/`）：

```bash
# 示例：链接单个技能
ln -s "$(pwd)/<skill-name>" ~/.claude/skills/<skill-name>
```

## 贡献

欢迎提交新技能或改进现有技能，请阅读 [CONTRIBUTING.md](CONTRIBUTING.md)。

## 许可证

[MIT](LICENSE)
