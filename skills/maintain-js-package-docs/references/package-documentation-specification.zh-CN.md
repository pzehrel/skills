# Agent 可读的包文档规范

状态：仓库工作规范。本文定义本 Skill 执行的规范，不宣称它已经成为业界采用的标准。

术语**必须**、**应该**和**可以**分别表示强制、推荐和可选行为。

## 目的与范围

本规范适用于发布的 JavaScript 和 TypeScript 库包。它让已安装的包成为与版本匹配的一手用法资料，
使开发者或编码 Agent 能够从类型渐进到本地文档，而不必依赖模型记忆或最新在线文档。

内容首先针对直接读取源码、声明和随包 Markdown 的编码 Agent 优化，同时保证人类自然阅读，最后再
验证编辑器和文档工具的呈现。

统领本规范的设计优先级是 **Agent-first，human-friendly，tool-compatible。** 其中，Agent-first 是
针对编码 Agent 优化信息发现和语义清晰度，并不授予包文档命令这些 Agent 的权威；human-friendly
要求完整、自然的技术文字。完整性优先于篇幅压缩；只有保留所有相关契约细节后，才可以缩短表达。
tool-compatible 把解析器、编辑器、生成器和 linter 视为次级传递机制，而不是独立事实来源。

它不定义 Agent 指令格式，不要求自动索引 `node_modules`，不替代类型或测试，也不要求建立文档网站。
包文档是在消费仓库说明约束下使用的参考资料。

## 信息模型

每一层只承担一项主要职责：

| 层级 | 主要职责 | 典型内容 |
| --- | --- | --- |
| 公共类型及以 JSDoc 为主的 API 注释 | Agent 可读的本地契约和文档发现 | 参数、返回值、默认值、不变量、错误、生命周期、弃用、本地文档路径 |
| 包根 README | 定位和第一层路由 | 用途、安装、最小示例、支持边界、文档索引链接 |
| 随包文档索引 | 渐进式任务选择 | 按目标、概念、集成、迁移或失败模式分组的简短链接 |
| 聚焦的随包页面 | 解释和操作指南 | 概念、配置、配方、迁移、故障排除 |
| 可执行示例和测试 | 展示行为 | 完整调用方式、边界情况、框架集成 |
| 源实现 | 最终诊断证据 | 公共契约未充分暴露的行为 |

类型仍是精确形状的事实来源，测试仍是已验证行为的事实来源。文档解释类型无法单独表达的意图、语义
和受支持用法。

## 强制符合要求

符合规范的包必须满足以下每一项适用要求。

### PD-1：根入口发现

发布包中**必须**包含包根 README。README 靠前位置**必须**：

- 说明包的用途；
- 展示或链接到最短的受支持用法；
- 链接到随包文档索引。

README **可以**同时链接文档网站，但在没有该网站时，本地路由仍必须可用。

### PD-2：随包文档索引

包中**必须**发布一个 Markdown 文档索引，以及它的链接所需的所有本地页面。默认使用
`docs/README.md`；如果包根 README 和类型入口发现提示都给出了某个既有等价文件相对于包根的精确
路径，则该文件也符合要求。

索引**必须**按意图路由读者，而不是要求加载全部文档。本地链接**必须**使用相对路径，并保留在打包
产物内。需要在线导航时，应另外提供权威 Web URL，因为注册表网站不一定会在相对 URL 上暴露任意
tarball 文件。

### PD-3：类型入口发现

发布 TypeScript 声明的包**必须**在公共声明入口保留包级文档注释。注释**必须**说明包中包含与版本
匹配的本地文档，并提供索引相对于包根的路径。

基线注释**必须**能够作为普通 JSDoc 风格正文被理解。已经采用 TSDoc 感知工具链的项目，在工具链
要求时还**必须**使用 `@packageDocumentation`。不发布声明的纯 JavaScript 包**应该**在公共源码入口
提供等价的 JSDoc 提示。

示例：

```ts
/**
 * Utilities for building protocol clients.
 *
 * Offline documentation matching this installed version starts at `docs/README.md`, relative to the
 * package root.
 */
```

启用 TSDoc 的项目可以在结束分隔符前添加 `@packageDocumentation` 作为工具标记；该标记不能替代
人类和 Agent 可读的发现正文。

包的构建流程**必须**在消费者和语言工具读取的产物中保留该提示。

### PD-4：完整公共声明与语义覆盖

实际包公共表面中的每个导出函数、类、构造函数、值、类型别名、接口和枚举，**必须**在权威位置具有
`/** ... */` API 注释。消费者会交互的每个公共属性、方法、调用签名和构造函数也都**必须**记录。

应根据包 export map、公共入口和生成声明确定该表面，而不是仅根据源码级 `export` 关键字判断。如果
原始声明注释会保留，barrel re-export 不需要重复注释。每个重载都**必须**在生成声明和编辑器帮助中
暴露适用注释；契约不同的重载**必须**明确记录这些差异。

JSDoc 正文和标签是 JavaScript 与 TypeScript 源码的权威作者基线。即使忽略所有 TSDoc 专属结构，
基线也**必须**让直接读取原始注释的编码 Agent 和人类能够理解。TypeScript 语法仍是类型的权威来源：
注释**不得**在参数或返回值标签的花括号中重复类型，也**不得**使用 JSDoc `@template` 重新声明
TypeScript 泛型。当注释承担包的类型信息时，JavaScript 源码**可以**使用 TypeScript 支持的 JSDoc
类型标签。

TSDoc 是补充性的互操作层。只有对明确采用的 TSDoc 感知文档生成器、API 审查工具或 linter 有价值
时，才**可以**使用 TSDoc 专属标签和结构。仓库**可以**通过自身配置要求这些扩展，但不能仅因为源码
语言是 TypeScript 就把它们纳入基线要求。必需契约信息**不得**只存在于基线 JSDoc 消费者可能忽略的
TSDoc 扩展中。

每个公开 callable 签名都**必须**包含所有适用的语义文档：

| 组成部分 | 要求 |
| --- | --- |
| 摘要 | 说明操作、可观察行为和关键契约。 |
| `@param` | 每个运行时参数（包括 rest 参数）对应一项；解释角色、可选行为和默认值，不复述类型。 |
| 返回值语义 | 当结果包含 API 名称和 TypeScript 返回类型不能直接说明的含义时使用 `@returns`，包括所有权、身份、可变性、单位、结果分支、缓存或结果值失败。对于 `void`、`never` 或真正不言自明的结果，可以省略。 |
| 泛型语义 | 当类型参数的角色、约束、关系或生命周期不直观时进行解释。可写在普通正文或相关 `@param`/`@returns` 中；对于已采用的 TSDoc 配置，可以另外添加 `@typeParam`。 |
| 失败语义 | 记录有意义的同步异常、异步拒绝、传播的回调失败和结果值失败。抛出的异常使用 JSDoc `@throws`；当某些消费者可能忽略该标签时，在普通正文中也要能理解关键细节。不得虚构失败行为。 |
| 重载 | 每个可见重载都有完整的适用注释；只有经验证工具链会保留精确契约时才使用继承。 |

本规则同样适用于函数、方法、构造函数、调用签名和函数值属性。README、主题页、接口、实现体或其他
重载中的文档，不能替代附在消费者可见签名上的注释；只有经验证的文档继承明确连接二者时除外。

主要 JSDoc 词汇包括摘要正文、`@param`、`@returns`、`@throws`、`@deprecated`、`@see`、`@example`
和 `{@link}`。除非已采用的仓库配置要求，否则 `@packageDocumentation`、`@typeParam`、`@remarks`、
`@defaultValue`、发布阶段修饰符和 TSDoc 声明引用等都是可选增强。泛型公开类、接口和类型别名
**必须**解释不直观的泛型语义，但基线符合要求不机械地要求每个已声明类型参数都有一项
`@typeParam`。

以 JSDoc 为主的完整 TypeScript 签名示例：

```ts
/**
 * Creates a codec from reversible wire-format operations.
 *
 * The returned codec is immutable. `T` is the business value represented by the codec.
 *
 * @param definition - Serialization and parsing operations for `T`.
 * @returns A frozen codec carrying the supplied operations.
 * @throws When either required operation is missing.
 */
export function defineFieldCodec<T>(definition: FieldCodecDefinition<T>): FieldCodec<T>
```

公共 API 文档**必须**解释签名无法完整编码的适用语义：

- 心智模型、所有权、生命周期和资源清理；
- 默认值、单位、修改行为、副作用、时序、重试和取消；
- 有意义的错误条件、顺序规则、安全限制和不受支持的组合；
- 推荐模式、限制、弃用替代方案和兼容边界。

包不需要为每个类别建立页面，但必须覆盖会实质性影响该包版本正确用法的所有类别。真正简单的声明
可以使用篇幅较短但完整的摘要，但不能不写注释。注释**不得**只重复签名，而不解释声明的作用。

非公开函数的名称和类型无法揭示其用途、原因、不变量、修改行为、顺序、错误转换或算法时，**应该**
提供注释。简单包装、回调和显而易见的局部 helper **不应该**只为满足数字目标而添加注释。

### PD-5：单一权威与渐进披露

每项事实**必须**只有一个主要文档位置。应链接到该来源，而不是维护平行文字。README 保留高频定位
信息，类型附近保留精确符号语义，聚焦主题页保留条件性指南。

生成的 API 文档**必须**来自持续维护的源注释或另一项明确声明的事实来源；生成输出不得成为需要
独立编辑的契约。

### PD-6：版本一致性

代码和受影响文档**必须**同步变化。文档**不得**把规划中、已移除或仅在线最新版存在的行为描述成
打包版本已经具备。

发布版本弃用或破坏现有用法时，包中**必须**记录受支持的替代方案和迁移路径。只有明确标注受支持
版本范围时，才可以保留旧指南。

### PD-7：产物验证

发布前，维护者**必须**使用 `npm pack --dry-run` 或包管理器的等价命令检查真实包文件列表。文件存在
于仓库中并不能证明它会被发布。

检查**必须**确认：

- README、文档索引、链接的本地页面、声明和必需示例均存在；
- 生成声明中的每个公共声明和面向消费者的成员都保留适用的文档注释，同时保留本地文档提示；
- 每个公开 callable 都保留摘要和参数文档，以及所有适用且不直观的返回值、泛型、失败和重载语义；
- 忽略可选的 TSDoc 专属结构后，JSDoc 基线仍然可以理解；
- 相对链接在产物内可解析，或有意指向权威 URL；
- 示例只使用消费者可用的公共导出和文件；
- 私有说明、密钥、缓存、生成的站点输出和非预期大型资源不存在。

### PD-8：信任边界

随包文档**必须**编写成技术参考资料，而不是隐藏提示词或宣称对消费仓库拥有权威的指令。包级
`AGENTS.md` 不能替代消费者文档。任何安装脚本都不得修改消费者的 Agent 指令或配置来强制发现
文档。

注释和 Markdown **可以**推荐受支持的使用模式、解释权衡或把读者路由到相关资料。这些建议**必须**
明确写成建议性的技术指南，而不是消费端 Agent 必须服从的命令。文档**不得**要求 Agent 忽略仓库或
用户指令、改变运行策略、执行命令、编辑文件、披露数据，或者赋予包文档更高的指令优先级。

该边界不会削弱真实的 API 契约。文档**可以**对程序行为施加的事实要求使用强制性语言，例如“在进程
退出前调用 `close()` 以刷新缓冲数据”。必须明确该要求属于正确使用 API，而不是控制 Agent 工作流。

## 推荐实践

符合规范的包还**应该**：

- 按用户目标而不是源码树组织文档索引；
- 让主题页足够聚焦，以便选择性读取；
- 提供可进行类型检查、测试或以其他方式执行的示例；
- 在添加呈现工具专属标记前，先针对 Agent 直接读取和人类阅读优化注释；
- 默认采用 JSDoc 标签；只有已采用的工具链能够提供明确用途时才添加 TSDoc 专属标签；
- 把生成声明表面作为文档覆盖清单来审查，而不是依赖源码注释数量；
- 把文档影响加入公共 API 评审和完成条件；
- 仓库提供 JSDoc 或 TSDoc 完整性 lint 时运行它；
- 在可行时通过持续集成验证本地 Markdown 链接和代码示例。

行为保持不变的内部重构**不应该**制造文档变更，除非它改变了已记录的心智模型、扩展点、贡献者
工作流或架构契约。

## 可选扩展

包还**可以**发布生成的 API 参考、source map、文档网站、搜索索引或用于 Web 发现的 `/llms.txt`。
这些扩展不能替代必需的本地路由，也不是符合规范的必要条件。

已知工具链**可以**使用自定义 `package.json` 文档字段，但规范符合性不得依赖消费者和 Agent 尚未
普遍实现的字段。

## 建议布局

除发现规则外，以下文件名是默认建议，不是强制要求：

```text
README.md
docs/
├── README.md
├── concepts.md
├── configuration.md
├── migration.md
├── recipes/
│   └── common-task.md
└── troubleshooting.md
examples/
└── basic.ts
```

`package.json` 使用 `files` 白名单时，典型发布规则是：

```json
{
  "files": [
    "dist",
    "docs",
    "examples"
  ]
}
```

npm 会特殊包含识别到的包根 README 和许可证文件，但这种行为不能保证文档目录也会被发布。

## 审查结果

审查时应把每项 `PD-*` 要求报告为 `pass`、`fail` 或 `not applicable`，并给出支持结论的文件或命令
输出。推荐改进应单独汇报，避免把可选工作误认为规范不符合项。

对于 PD-4，审查**必须**包含 callable 覆盖表，列为：API 签名、摘要、`@param`、不直观的返回值
语义、不直观的泛型语义、失败语义、重载覆盖、JSDoc 基线、TSDoc 增强及其理由、是否保留在生成
声明中、代表性工具呈现。只有名称和签名已经让相关行为足够明确时，语义单元格才可以标记为不适用。
只有仓库已经采用相应配置时，缺少 TSDoc 增强才属于失败。任何其他适用单元格缺失都属于不符合
规范。无法测试工具呈现时，应报告为未实测，不得声称兼容。

对于 PD-8，审查**必须**检查注释和随包 Markdown 中面向 Agent 的命令，并分别报告合理的 API 要求、
建议性的推荐以及禁止控制消费端 Agent 的尝试。

## 权威参考

- [JSDoc `@param`](https://jsdoc.app/tags-param)
- [JSDoc `@returns`](https://jsdoc.app/tags-returns)
- [JSDoc `@throws`](https://jsdoc.app/tags-throws)
- [JSDoc `@see`](https://jsdoc.app/tags-see)
- [JSDoc `{@link}`](https://jsdoc.app/tags-inline-link)
- [VS Code：编辑 TypeScript](https://code.visualstudio.com/docs/typescript/typescript-editing)
- [TypeScript JSDoc 参考](https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html)
- [TSDoc `@packageDocumentation`](https://tsdoc.org/pages/tags/packagedocumentation/)
- [TSDoc `@typeParam`](https://tsdoc.org/pages/tags/typeparam/)
- [TSDoc `@param`](https://tsdoc.org/pages/tags/param/)
- [TSDoc `@returns`](https://tsdoc.org/pages/tags/returns/)
- [TSDoc `@throws`](https://tsdoc.org/pages/tags/throws/)
- [npm `package.json` 文件包含规则](https://docs.npmjs.com/files/package.json/)
