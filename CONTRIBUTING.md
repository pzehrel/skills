# 贡献指南

感谢你的贡献！本仓库是一个 Agent Skills 集合，以下是添加或改进技能的约定。

## 添加新技能

1. 在仓库根目录创建技能目录，使用 kebab-case 命名：`<skill-name>/`
2. 目录内必须包含 `SKILL.md`，且带有合法的 YAML frontmatter：

   ```markdown
   ---
   name: skill-name
   description: 一句话描述技能的用途和触发场景
   ---
   ```

3. 正文编写要求：
   - 指令具体、可执行，避免空泛的建议
   - 保持精简；长篇参考材料放入 `references/`，正文中注明按需加载
   - 可执行脚本放入 `scripts/`，注明运行方式与依赖
   - 模板、示例放入 `assets/`

4. 提交前自检：
   - [ ] `name` 与目录名一致
   - [ ] `description` 说明了"做什么"和"什么时候用"
   - [ ] 技能已在真实任务中验证可用

## 提交规范

- 使用清晰简洁的 commit message，推荐格式：`<skill>: <变更说明>`
  - 例：`pdf: 添加表单填写支持`、`readme: 修正目录结构说明`
- 一个 PR 聚焦一个技能或一个改动点

## 提问与讨论

有问题或想法，欢迎通过 Issue 交流。
