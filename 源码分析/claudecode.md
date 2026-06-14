# Claude Code 源码功能与设计目的解析

> 说明：本文基于 `collection-claude-code-source-code` 这个逆向整理仓库中的 TypeScript 原始源码、Python 重写版和附带文档进行归纳，不等同于 Anthropic 官方完整内部实现源码。它适合做“结构理解”和“机制学习”，不适合当作官方实现的逐行等价证明。

## 1. 这个项目本质上是什么

Claude Code 不是一个普通的命令行聊天框，而是一个**以代码工作流为中心的代理式编程 CLI**。它的目标不是“回答问题”，而是“在受控权限下持续完成开发任务”。

从源码结构看，它包含四层核心能力：

- **对话层**：维护会话、历史、上下文压缩、流式输出
- **代理层**：让模型自行规划、调用工具、恢复、继续、分工
- **工具层**：文件、Shell、Git、Web、MCP、任务、子代理等
- **约束层**：权限控制、危险操作拦截、自动审批、审计和回收

换句话说，Claude Code 的核心不是“一个大模型”，而是**模型 + 工具编排 + 上下文治理 + 安全约束**的组合系统。

## 2. 总体架构

从 `original-source-code/src` 可以看出，主入口大致分成这些区域：

- `main.tsx` / `cli/*`：命令行入口、参数解析、交互 UI
- `query.ts` / `QueryEngine.ts`：主循环、消息流、工具执行、上下文压缩
- `constants/prompts.ts`：系统提示词组装
- `Tool.ts` / `tools.ts`：工具定义、权限上下文、工具注册
- `services/tools/*`：工具调度与并行执行
- `services/mcp/*`：MCP 连接、认证、资源读取、权限和通知
- `tools/AgentTool/*`：子代理体系
- `tools/SkillTool/*`、`skills/*`：技能发现、加载、监听和执行
- `utils/permissions/*`：权限决策、规则、分类器、拒绝跟踪
- `services/compact/*`：自动压缩和长会话管理

这套结构说明它并不是“一个 prompt 调一次 API”，而是一个**可持续运行的代理操作系统**。

## 3. 核心运行流程

主循环大致是：

1. 用户输入命令或自然语言
2. CLI 解析 `/slash` 命令、参数和会话状态
3. 构造 system prompt、user context、system context
4. 调用模型生成 assistant 消息
5. 如果模型发起 tool use，就进入工具执行器
6. 工具返回结果后继续喂回模型
7. 如果上下文太长，触发 auto compact
8. 最终以流式方式输出给终端

这和普通聊天应用不同的地方在于：

- 模型会被允许“连续行动”
- 工具结果会回灌给模型形成闭环
- 会话不会简单结束，而是通过压缩和恢复继续运行
- 子代理、MCP、技能、任务都可以嵌入同一回路

## 4. 提示词工程：Claude Code 的真正核心

### 4.1 system prompt 不是单段文本，而是分段拼装

`constants/prompts.ts` 和 `utils/queryContext.ts` 说明系统提示词不是硬编码成一个长字符串，而是按模块动态组装。常见来源包括：

- 主系统规则
- 当前工具列表说明
- 当前模型能力说明
- 工作目录/仓库状态
- MCP server 说明
- 技能相关说明
- 输出风格或模式切换说明
- 记忆、任务、计划等补充块

这种分段设计的目的有三个：

- **可组合**：不同模式下只拼装需要的段落
- **可缓存**：稳定部分可作为 cache key 前缀
- **可注入**：不同命令、代理、技能、MCP 状态会影响 prompt

### 4.2 提示词的设计目标不是“描述能力”，而是“约束行为”

Claude Code 的提示词主要在做这些事：

- 告诉模型哪些工具可用
- 告诉模型每个工具何时应该用、何时不该用
- 强制模型在必要时先询问用户
- 限制高风险操作的默认行为
- 引导模型先规划、后执行、再校验
- 引导模型维护最小改动、局部验证、失败回退

也就是说，prompt 的重点不是“让模型更聪明”，而是**让模型更像一个受控的工程代理**。

### 4.3 prompt 中的“工程习惯”非常明显

源码里能看出它明显偏向这些工程习惯：

- 先理解上下文，再改代码
- 先搜索，再编辑
- 尽量使用工具而不是臆测
- 在危险操作前要求权限
- 长会话自动压缩，避免上下文失控
- 子代理分工，减少主线程负担

这就是 Claude Code 的提示词工程重点：**用结构化行为规则压过自由聊天倾向**。

## 5. Agent 机制：不是一个代理，而是一组代理

### 5.1 主代理

主代理负责主循环、工具编排、任务推进和用户交互。它是所有能力的协调中心。

### 5.2 子代理系统

`tools/AgentTool/*` 说明 Claude Code 支持把任务拆成多个 agent：

- 通用执行 agent
- 探索型 agent
- 计划型 agent
- 验证型 agent
- 以及更偏功能化的内置 agent

这种设计的意义是：

- 把“读代码、分析、验证、总结”分开
- 避免单个上下文过载
- 支持 fork / resume / inline 等多种执行方式
- 让不同代理专注不同职责

### 5.3 Agent 的限制约束

源码里对 agent 的限制，核心是三类：

- **权限限制**：哪些工具能调用、哪些要询问
- **上下文限制**：代理只拿到必要信息，避免全量污染
- **职责限制**：不同 agent 适合不同工作，不是随便互换

尤其是权限和上下文限制，决定了 agent 不只是“多开几个模型”，而是一个有边界的任务系统。

## 6. 工具系统：Claude Code 为什么像“操作系统”

### 6.1 工具的作用

`Tool.ts`、`tools.ts` 和 `services/tools/*` 显示，工具不是模型的附属玩具，而是主能力接口。模型通过工具去做真实工作：

- 读文件、改文件、写文件
- 全局搜索、grep、glob
- 执行 shell / powershell / repl / notebook
- Git / worktree 操作
- 调 Web、拉文档、查信息
- 触发任务、计划、睡眠、定时器
- 访问 MCP 资源
- 启动子代理

### 6.2 工具执行不是同步直连，而是带调度的

工具调用经过：

- 参数校验
- 权限判断
- 并发调度
- 进度流式反馈
- 结果回写

这意味着 Claude Code 的工具层重点不是“能调用”，而是**如何安全、并发、可恢复地调用**。

### 6.3 工具权限上下文

`ToolPermissionContext` 和 `utils/permissions/permissions.ts` 表明，Claude Code 把权限分成了非常细的规则系统，典型思想包括：

- 允许 / 拒绝 / 询问
- 全局规则与会话规则并存
- 某些工具或命令可自动放行
- 危险操作需要明确确认
- 可通过分类器或 hook 做自动判断

这套设计的目标是：**让模型拥有行动力，但不拥有无限行动权**。

## 7. 权限系统：最重要的安全边界

### 7.1 为什么权限系统是核心

Claude Code 不是纯读取型工具，它能改文件、跑命令、访问外部资源，所以权限系统决定了它能否安全进入真实开发环境。

### 7.2 权限系统做了什么

从 `utils/permissions/*` 和相关状态文件可以看出它做了：

- 工具级别的准入控制
- 命令级别的匹配规则
- 自动拒绝/自动允许/询问用户
- 失败次数与降级策略
- hook、分类器、规则叠加决策
- 记忆化的拒绝追踪

### 7.3 设计目的

这套权限机制的真正目的不是“多问一次用户”，而是：

- 降低误删、误改、误执行的概率
- 让代理可在受限环境下工作
- 为自动化和人工审批之间提供平衡
- 让高风险工具能和低风险工具共存

## 8. Skill 系统：把能力打包成可发现、可加载、可触发的单元

### 8.1 Skill 是什么

从 `skills/loadSkillsDir.ts`、`tools/SkillTool/*`、`utils/skills/*` 看，skill 本质上是一个**可被发现的能力包**。它通常包含：

- 名称
- 描述
- 触发条件或元数据
- 对应的 `SKILL.md` / 说明内容
- 可选的动态加载逻辑

### 8.2 Skill 的作用

Skill 不是一般意义上的插件，更像是：

- 针对某类任务的工作模板
- 可复用的提示词模块
- 可被模型主动调用的知识包
- 可被系统监听变化的本地能力扩展

### 8.3 SkillTool 的职责

`SkillTool` 相关代码表明，技能系统不仅能“列出技能”，还可以：

- 加载技能目录
- 检测技能变更
- 清理缓存
- 追踪哪些技能在会话中被用过
- 在压缩或恢复时保留技能状态

这说明 skill 不只是静态配置，而是**参与运行时状态管理**的对象。

### 8.4 Skill 设计目的

技能系统解决的是“模型泛化能力够强，但专门流程太多”的问题：

- 让复杂流程模板化
- 让任务指导更一致
- 让不同项目沉淀自己的工作方式
- 让模型知道何时应该切换到特定技能

## 9. MCP：把外部工具世界接进来

### 9.1 MCP 在 Claude Code 里的位置

`services/mcp/*` 和 `tools/MCPTool/*` 表明，MCP 是 Claude Code 接外部能力的标准协议层。它让 Claude Code 可以连接：

- 本地或远程 MCP server
- 资源（resources）
- 工具（tools）
- 认证与授权流程
- 服务器健康检查与配置管理

### 9.2 MCPTool 的作用

MCPTool 不是普通文件工具，而是**协议桥接工具**。它负责：

- 将模型意图映射到 MCP server 调用
- 处理 server 能力发现
- 处理资源读取
- 处理认证失败、连接错误、权限控制
- 将结果转回模型上下文

### 9.3 MCP 的设计目的

MCP 的价值在于让 Claude Code 不必把所有能力都写死在本地：

- 可对接第三方系统
- 可对接自定义企业服务
- 可对接 IDE、数据库、知识库、CI/CD
- 使工具生态标准化

换句话说，Claude Code 里的 MCP 是“外部能力总线”。

## 10. 上下文压缩与长会话治理

Claude Code 很重视长会话，原因很简单：代理任务通常很长。

`services/compact/*` 和 `query.ts` 说明它会做：

- 自动压缩上下文
- 保留关键历史
- 生成紧凑摘要
- 避免 prompt 过长
- 在必要时恢复必要上下文

这类机制的目的不是省 token 而已，而是：

- 防止代理“失忆”
- 防止工具历史淹没主任务
- 让长时间任务可持续推进

## 11. 这个项目的设计哲学

我认为 Claude Code 的设计哲学可以概括成六点：

- **任务优先**：围绕完成开发任务组织交互
- **工具优先**：让模型少猜，多查、多做
- **约束优先**：默认安全边界强于自由执行
- **模块优先**：prompt、tool、skill、agent 分层明确
- **可恢复优先**：支持 resume、compact、snapshot、history
- **可扩展优先**：MCP、skills、plugins、agents 都是扩展点

## 12. 你最该掌握的几个重点

如果你是为了学习源码，我建议优先理解这五件事：

1. **prompt 是怎么拼装的**：看 `constants/prompts.ts` 和 `utils/queryContext.ts`
2. **工具是怎么被调度的**：看 `Tool.ts`、`services/tools/*`
3. **权限是怎么兜底的**：看 `utils/permissions/*`
4. **技能是怎么加载和监听的**：看 `skills/loadSkillsDir.ts`、`tools/SkillTool/*`
5. **MCP 是怎么桥接外部系统的**：看 `services/mcp/*`、`tools/MCPTool/*`

## 13. 一句话总结

Claude Code 的本质是：**一个把大模型包装成“安全、可控、可恢复、可扩展”的编程代理操作系统**。

它最有价值的地方，不是会聊天，而是：

- 会规划
- 会调用工具
- 会管理权限
- 会压缩上下文
- 会加载技能
- 会接入 MCP
- 会拆分子代理

这也是它真正的工程意义。


## 13. 关键源码阅读索引

下面这些文件建议按顺序阅读，它们最能解释 Claude Code 的真实工作方式：

| 阅读顺序 | 文件 | 重点 |
|---|---|---|
| 1 | `original-source-code/src/utils/queryContext.ts` | system prompt、user context、system context 的拼装入口 |
| 2 | `original-source-code/src/constants/prompts.ts` | 主提示词、工具提示词、MCP、skill、agent 等 prompt 片段组合 |
| 3 | `original-source-code/src/QueryEngine.ts` | SDK/headless 查询生命周期和上下文准备 |
| 4 | `original-source-code/src/query.ts` | 主 agent loop、模型响应、工具调用、恢复逻辑 |
| 5 | `original-source-code/src/Tool.ts` | 工具接口、权限上下文、工具执行上下文类型 |
| 6 | `original-source-code/src/tools.ts` | 工具注册与默认工具集合 |
| 7 | `original-source-code/src/services/tools/StreamingToolExecutor.ts` | 工具流式执行与进度回传 |
| 8 | `original-source-code/src/services/tools/toolOrchestration.ts` | 多工具调度、执行顺序、结果处理 |
| 9 | `original-source-code/src/utils/permissions/permissions.ts` | 权限决策、规则、分类器和拒绝策略 |
| 10 | `original-source-code/src/tools/AgentTool/AgentTool.tsx` | 子代理创建、运行、恢复、展示 |
| 11 | `original-source-code/src/tools/AgentTool/prompt.ts` | 子代理相关提示词约束 |
| 12 | `original-source-code/src/skills/loadSkillsDir.ts` | skill 目录加载、解析、缓存 |
| 13 | `original-source-code/src/tools/SkillTool/SkillTool.ts` | skill 调用工具实现 |
| 14 | `original-source-code/src/utils/skills/skillChangeDetector.ts` | skill 文件变更监听与缓存刷新 |
| 15 | `original-source-code/src/services/mcp/config.ts` | MCP server 配置、scope、读取写入 |
| 16 | `original-source-code/src/services/mcp/client.ts` | MCP client、连接、请求、资源与工具调用 |
| 17 | `original-source-code/src/tools/MCPTool/MCPTool.ts` | MCP 工具桥接层 |
| 18 | `original-source-code/src/services/compact/autoCompact.ts` | 自动上下文压缩策略 |
| 19 | `original-source-code/src/services/compact/prompt.ts` | 压缩摘要提示词 |
| 20 | `original-source-code/src/commands/*` | `/commit`、`/review`、`/memory`、`/mcp`、`/skills` 等 slash 命令实现 |

## 14. Tool 功能分类细读

Claude Code 的工具可以分成这些功能族：

| 类型 | 代表目录 | 作用 |
|---|---|---|
| 文件工具 | `FileReadTool`、`FileEditTool`、`FileWriteTool` | 读、改、写文件，是代码修改能力核心 |
| 搜索工具 | `GlobTool`、`GrepTool` | 快速定位文件、符号、文本片段，避免模型瞎猜 |
| 命令工具 | `BashTool`、`PowerShellTool`、`REPLTool` | 执行系统命令、脚本、交互式语言环境 |
| 计划工具 | `TodoWriteTool`、`EnterPlanModeTool`、`ExitPlanModeTool` | 让 agent 显式规划、更新任务状态、进入/退出计划模式 |
| 子代理工具 | `AgentTool` | 创建、恢复、运行子代理，用于探索、计划、验证等分工 |
| MCP 工具 | `MCPTool`、`ListMcpResourcesTool`、`ReadMcpResourceTool`、`McpAuthTool` | 连接外部 MCP server，读取资源、调用外部工具、处理认证 |
| Web 工具 | `WebFetchTool`、`WebSearchTool` | 获取网页、搜索资料，用于补充外部信息 |
| 任务工具 | `TaskCreateTool`、`TaskUpdateTool`、`TaskGetTool`、`TaskListTool`、`TaskStopTool` | 管理长期或后台任务 |
| 环境工具 | `EnterWorktreeTool`、`ExitWorktreeTool`、`LSPTool`、`NotebookEditTool` | 支持 Git worktree、语言服务、Notebook 等开发环境 |
| 自动化工具 | `ScheduleCronTool`、`RemoteTriggerTool`、`SleepTool` | 定时、远程触发、等待等自动化流程 |
| 交互工具 | `AskUserQuestionTool`、`SendMessageTool` | 在不确定或高风险时回问用户或发送消息 |
| skill 工具 | `SkillTool` | 触发特定技能，让模型临时加载专项工作流 |

## 15. Slash Command 的作用

`commands/*` 目录实现了用户直接输入的 `/命令`。它们和工具不同：

- **工具**主要给模型调用
- **slash command**主要给用户调用
- 有些命令会改变会话状态，比如 `/model`、`/permissions`、`/vim`
- 有些命令会触发工程工作流，比如 `/commit`、`/review`
- 有些命令用于配置扩展能力，比如 `/mcp`、`/skills`、`/plugin`

这说明 Claude Code 同时有两套控制面：

- 用户控制面：slash commands
- 模型控制面：tools

## 16. 设计上最值得借鉴的点

如果你想把 Claude Code 的思想迁移到自己的 agent 项目，最值得借鉴的是：

1. **Prompt 分层**：不要写一个巨型 system prompt，而要按能力、工具、上下文、模式分段拼装。
2. **工具元数据化**：每个工具都要有名称、描述、参数 schema、权限规则、执行器。
3. **权限独立化**：权限判断不能散落在工具里，应该有统一权限层。
4. **长会话压缩**：agent 做复杂任务必须有 compact / summary / resume。
5. **技能化工作流**：把重复任务沉淀成 skill，而不是每次都靠用户重新描述。
6. **外部能力协议化**：用 MCP 这类协议桥接外部系统，而不是把所有能力硬编码。
7. **子代理分工**：探索、计划、执行、验证最好拆开，减少单 agent 上下文和职责混乱。