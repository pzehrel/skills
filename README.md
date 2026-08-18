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

## 使用方式

### 通过 skills CLI 安装（推荐）

```bash
# 列出本仓库的技能
npx skills add pzehrel/skills --list

# 安装指定技能
npx skills add pzehrel/skills --skill <skill-name>
```

### 通过 gh CLI 安装

GitHub CLI（v2.97+）内置了 `gh skill` 命令（preview 阶段）：

```bash
# 预览技能内容
gh skill preview pzehrel/skills <skill-name>

# 安装指定技能
gh skill install pzehrel/skills <skill-name>

# 列出已安装的技能
gh skill list
```

### 手动链接

将技能目录复制或链接到 Agent 的技能加载路径（如 Claude Code 的 `~/.claude/skills/` 或项目的 `.claude/skills/`）：

```bash
# 示例：链接单个技能
ln -s "$(pwd)/skills/<skill-name>" ~/.claude/skills/<skill-name>
```

## 许可证

[MIT](LICENSE)
