# Claude Code 项目分析文档

## 1. 项目概述

Claude Code 是 Anthropic 官方推出的 CLI 工具，用于在终端中与 Claude 进行交互，完成软件工程任务如编辑文件、运行命令、搜索代码库等。

| 属性 | 值 |
|------|-----|
| **定位** | AI 辅助编程 CLI 工具 |
| **语言** | TypeScript (strict) |
| **运行时** | Bun |
| **终端 UI** | React + Ink |
| **代码规模** | ~1,900 文件，512,000+ 行 |
| **公开暴露日期** | 2026-03-31 (通过 npm 的 .map 文件泄露) |

---

## 2. 技术栈

| 层级 | 技术选型 |
|------|---------|
| **运行时** | Bun |
| **语言** | TypeScript (strict mode) |
| **终端 UI 框架** | React + Ink |
| **CLI 参数解析** | Commander.js (@commander-js/extra-typings) |
| **Schema 验证** | Zod v4 |
| **代码搜索** | ripgrep |
| **协议** | MCP SDK, LSP (Language Server Protocol) |
| **AI API** | Anthropic SDK |
| **遥测/链路追踪** | OpenTelemetry + gRPC |
| **特性开关** | GrowthBook |
| **认证** | OAuth 2.0, JWT, macOS Keychain |

---

## 3. 核心架构

```
┌──────────────────────────────────────────────────────────────┐
│                      main.tsx (入口)                         │
│   - Commander.js CLI 参数解析                                 │
│   - 并行预取: MDM配置 / Keychain / API预连接                  │
└────────────────────────┬─────────────────────────────────────┘
                         │
                         ▼
┌──────────────────────────────────────────────────────────────┐
│                    QueryEngine (查询引擎)                     │
│   - 流式 API 调用                                            │
│   - Tool Call 循环                                           │
│   - Thinking 模式                                           │
│   - 重试逻辑                                                │
│   - Token 计数                                              │
└────────┬────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────┐
│                    Tool System (工具系统)                     │
│   ~40 种工具, 定义在 src/tools/                             │
└────────┬────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────┐
│              Permission System (权限系统)                    │
│   - default / plan / auto / bypassPermissions / dontAsk    │
│   - 交互式权限确认                                           │
│   - 策略限制 (PolicyLimits)                                  │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. 目录结构

```
src/
├── main.tsx                  # CLI 入口, Commander.js + Ink 初始化
├── QueryEngine.ts            # LLM 查询引擎 (~46K 行)
├── Tool.ts                   # 工具基类与类型定义 (~29K 行)
├── commands.ts               # 命令注册表 (~25K 行)
├── tools.ts                  # 工具注册表 (~40 种工具)
├── context.ts                # 上下文收集
├── cost-tracker.ts           # Token 费用跟踪
│
├── commands/                 # 斜杠命令实现 (~50 种)
│   ├── commit.ts            # Git 提交
│   ├── review.ts            # 代码审查
│   ├── compact.ts           # 上下文压缩
│   ├── mcp.ts               # MCP 服务器管理
│   ├── config.tsx           # 设置管理
│   ├── doctor.tsx           # 环境诊断
│   ├── memory/              # 持久化记忆管理
│   ├── skills/              # 技能管理
│   ├── tasks/               # 任务管理
│   └── ...
│
├── tools/                    # 代理工具实现 (~40 种)
│   ├── BashTool/            # Shell 命令执行
│   ├── FileReadTool/        # 文件读取 (图片, PDF, Notebook)
│   ├── FileWriteTool/       # 文件写入
│   ├── FileEditTool/        # 文件修改 (字符串替换)
│   ├── GlobTool/            # 文件模式匹配
│   ├── GrepTool/            # ripgrep 内容搜索
│   ├── WebFetchTool/        # URL 内容获取
│   ├── WebSearchTool/        # 网页搜索
│   ├── AgentTool/            # 子代理生成
│   ├── SkillTool/            # 技能执行
│   ├── MCPTool/              # MCP 服务器工具调用
│   ├── LSPTool/              # 语言服务器协议集成
│   ├── TaskCreateTool/       # 任务创建
│   ├── TeamCreateTool/       # 团队代理管理
│   └── ...
│
├── services/                 # 外部服务集成
│   ├── api/                  # Anthropic API 客户端
│   ├── mcp/                  # Model Context Protocol
│   ├── oauth/                # OAuth 2.0 认证
│   ├── lsp/                  # LSP 管理器
│   ├── analytics/            # GrowthBook 特性开关 + 分析
│   ├── compact/               # 对话上下文压缩
│   └── plugins/              # 插件加载器
│
├── bridge/                   # IDE 桥接 (VS Code / JetBrains)
│   ├── bridgeMain.ts         # 桥接主循环
│   ├── bridgeMessaging.ts    # 消息协议
│   ├── replBridge.ts         # REPL 会话桥接
│   └── jwtUtils.ts           # JWT 认证
│
├── components/               # Ink UI 组件 (~140 个)
│   ├── App.tsx               # 主应用组件
│   ├── Messages.tsx          # 消息列表
│   ├── MessageRow.tsx        # 单条消息
│   └── ...
│
├── hooks/                    # React Hooks
│   ├── toolPermission/       # 工具权限处理
│   ├── useMainLoopModel.ts  # 主循环模型
│   └── ...
│
├── screens/                  # 全屏 UI
│   ├── REPL.tsx             # 主 REPL 界面
│   ├── Doctor.tsx           # 诊断界面
│   └── ResumeConversation.tsx
│
├── ink/                      # Ink 渲染器封装
│   ├── reconciler.ts        # React Reconciler
│   ├── render-to-screen.ts  # 屏幕渲染
│   └── termio/              # 终端 I/O
│
├── state/                    # 状态管理 (Zustand)
│   ├── AppState.tsx
│   └── store.ts
│
├── skills/                   # 技能系统
│   └── bundled/             # 内置技能
│
├── plugins/                  # 插件系统
│   └── bundled/             # 内置插件
│
├── memdir/                   # 持久化记忆目录
├── tasks/                    # 任务管理
├── keybindings/             # 键位配置
├── vim/                      # Vim 模式
├── voice/                    # 语音输入
├── server/                   # 服务端模式
├── entrypoints/              # 入口点初始化
│   ├── cli.tsx              # CLI 入口
│   └── mcp.ts               # MCP 入口
└── utils/                    # 工具函数
```

---

## 5. 核心流程图

### 5.1 启动流程

```
┌─────────────────────────────────────────────────────────────┐
│                      CLI 启动 (cli.tsx)                      │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  解析 CLI 参数 (Commander.js)                                │
│  - --version / --dump-system-prompt 等快速路径              │
│  - 正常启动加载完整模块                                      │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  并行预取 (main.tsx 副作用导入)                              │
│  - startMdmRawRead()     :  MDM 配置 (macOS)                │
│  - startKeychainPrefetch():  Keychain OAuth/API Key          │
│  - fetchBootstrapData() :  引导数据                        │
│  - prefetchPassesEligibility(): 推荐积分                     │
│  - prefetchOfficialMcpUrls():  官方 MCP 注册表              │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  初始化 (entrypoints/init.ts)                                │
│  - initializeTelemetryAfterTrust()  :  遥测初始化            │
│  - initializeGrowthBook()          :  GrowthBook 特性开关    │
│  - initBundledSkills()             :  内置技能加载          │
│  - initBuiltinPlugins()            :  内置插件加载          │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  启动 REPL (replLauncher.tsx)                                │
│  - launchRepl() → screens/REPL.tsx                          │
│  - 渲染 Ink React 界面                                       │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 主交互循环 (Query Loop)

```
┌─────────────────────────────────────────────────────────────┐
│                     用户输入 (REPL.tsx)                       │
│  - useTextInput() 处理键盘输入                               │
│  - useInputBuffer() 管理输入缓冲                             │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  processUserInput() (utils/processUserInput.ts)              │
│  - 解析斜杠命令 (/commit, /review, /mcp 等)                 │
│  - 提取 @mention 代理提及                                    │
│  - 追加用户上下文                                            │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  QueryEngine.query() (QueryEngine.ts)                        │
│  - 构建 API 请求 (system prompt + messages)                  │
│  - applyConfigEnvironmentVariables() 处理环境变量            │
│  - prependUserContext() / appendSystemContext()              │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  流式 API 调用 (Anthropic SDK)                               │
│  - calculateTokenWarningState() 监控 token 警告              │
│  - isAutoCompactEnabled() 检测自动压缩                       │
└───────────────┬─────────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────────────────────────┐
│  处理响应消息                                                │
│  - AssistantMessage / UserMessage / SystemMessage            │
│  - createToolUseSummaryMessage() 工具使用摘要               │
│  - stripSignatureBlocks() 移除签名块                        │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  工具调用循环 (Tool Call Loop)                               │
│  ┌─────────────────────────────────────────────────────┐   │
│  │ while (hasToolCalls(response)) {                     │   │
│  │   for (toolCall in toolCalls) {                      │   │
│  │     - 权限检查 (toolPermission/)                     │   │
│  │     - 执行工具 (services/tools/toolExecution.ts)      │   │
│  │     - 工具结果 → messages                             │   │
│  │   }                                                   │   │
│  │   - 继续 API 调用 (fetch next response)               │   │
│  │ }                                                      │   │
│  └─────────────────────────────────────────────────────┘   │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  响应渲染 (Messages.tsx / MessageRow.tsx)                    │
│  - Ink <Box> / <Text> 组件渲染                              │
│  - useVirtualScroll() 虚拟滚动                              │
│  - 工具执行结果格式化输出                                    │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 工具执行流程 (Tool Execution)

```
┌─────────────────────────────────────────────────────────────┐
│  发现工具调用 (ToolUseBlock)                                 │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  权限检查 (useCanUseTool.tsx / toolPermission/)             │
│  - PermissionMode: default / plan / auto / bypass           │
│  - ask(): 弹出交互确认 (useCommandQueue.ts)                 │
│  - allow(): 自动放行                                         │
│  - deny(): 记录拒绝, 返回错误消息                            │
└────────────────────────────┬────────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              │         权限结果            │
              ▼                             ▼
┌─────────────────────────┐   ┌─────────────────────────────┐
│      放行继续执行        │   │          拒绝                │
└───────────┬─────────────┘   └──────────────┬──────────────┘
            │                                │
            ▼                                ▼
┌─────────────────────────────────────────────────────────────┐
│  工具执行 (services/tools/toolExecution.ts)                  │
│  - findToolByName() 查找工具实现                            │
│  - StreamingToolExecutor 处理流式工具                      │
│  - toolHooks 执行前置/后置钩子                               │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  工具特定实现 (src/tools/<ToolName>/)                        │
│  - BashTool:    spawn shell, 捕获 stdout/stderr             │
│  - FileReadTool: readFile, 图片/PDF/Notebook 处理           │
│  - FileEditTool: 字符串替换编辑                             │
│  - GrepTool:    ripgrep 搜索                                │
│  - MCPTool:     MCP 协议调用                                │
│  - LSPTool:     语言服务器协议                              │
│  - AgentTool:   启动子代理 (forkedAgent.ts)                │
│  - SkillTool:   技能工作流执行                              │
│  - WebFetchTool: HTTP 请求                                  │
│  - ...                                                      │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  结果转换 (ToolResultBlock → AssistantMessage)              │
│  - generateToolUseSummary() 生成工具使用摘要                │
│  - 添加到 messages 数组                                      │
│  - 继续循环或返回最终响应                                    │
└─────────────────────────────────────────────────────────────┘
```

### 5.4 权限系统流程

```
┌─────────────────────────────────────────────────────────────┐
│  工具调用请求                                                │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  获取 PermissionMode (settings.json / CLI --permission-mode)│
│  - default:  按工具类型默认行为                             │
│  - plan:     Plan 模式下仅询问敏感操作                     │
│  - auto:     自动模式 (ANT-ONLY)                           │
│  - bypassPermissions: 完全跳过确认                         │
│  - dontAsk:  静默拒绝写操作                                 │
└────────────────────────────┬────────────────────────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  auto / bypass  │  │     plan       │  │    default      │
│   直接放行      │  │   区分工具类型   │  │  工具类型规则   │
└────────┬────────┘  └────────┬────────┘  └────────┬────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│  判断是否需要询问 (perToolPolicy)                           │
│  - 高危工具 (Bash / Write / Edit): 始终询问                │
│  - 只读工具 (Read / Grep / Glob): 通常放行                  │
└────────────────────────────┬────────────────────────────────┘
                             │
              ┌──────────────┴──────────────┐
              │         需要确认?            │
              ▼                             ▼
┌─────────────────────────┐   ┌─────────────────────────────┐
│    interactiveHandler   │   │      直接放行 / 拒绝        │
│    (hooks/toolPermission/)│   └─────────────────────────────┘
│    - 展示确认对话框      │
│    - 用户选择 Allow/Deny │
│    - 记住选择?           │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────────┐
│  权限结果 → 工具执行或拒绝消息                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.5 MCP (Model Context Protocol) 集成

```
┌─────────────────────────────────────────────────────────────┐
│  MCP 服务器配置 (settings.json / /mcp 命令)                  │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  MCPConnectionManager (services/mcp/)                        │
│  - SdkControlTransport:  SDK 控制传输                       │
│  - InProcessTransport:   进程内传输                         │
│  - OAuth 认证流程 (oauthPort.ts)                           │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  MCPTool (tools/MCPTool/)                                   │
│  - 列出可用工具 (list_mcp_resources)                        │
│  - 调用 MCP 工具 (call_mcp_tool)                            │
│  - 资源读写 (ReadMcpResourceTool)                          │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  MCP 协议消息 (services/mcp/types.ts)                       │
│  - JSON-RPC 2.0 格式                                        │
│  - ToolCallRequest / ToolCallResult                         │
│  - ResourceTemplate / Resource                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.6 IDE 桥接 (Bridge System)

```
┌─────────────────────────────────────────────────────────────┐
│  Claude Code CLI (bridge/bridgeMain.ts)                      │
│  - bridgeMessaging.ts 消息协议                              │
│  - jwtUtils.ts JWT 认证                                     │
└────────────────────────────┬────────────────────────────────┘
                             │  WebSocket / stdio
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  IDE 扩展 (VS Code / JetBrains)                             │
│  - bridgeUI.ts 显示 IDE 内 UI                              │
│  - bridgePermissionCallbacks.ts 权限回调                   │
│  - ideIntegration  IDE 交互 (useIdeConnectionStatus.ts)   │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  会话同步                                                    │
│  - codeSessionApi.ts 会话 API                               │
│  - sessionRunner.ts 会话运行管理                            │
│  - createSession.ts 创建新会话                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.7 上下文压缩 (Compact) 流程

```
┌─────────────────────────────────────────────────────────────┐
│  消息累积 → Token 接近限制                                  │
│  (calculateTokenWarningState)                              │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  触发压缩 (services/compact/compact.ts)                      │
│  - isAutoCompactEnabled() 检查开关                         │
│  - buildPostCompactMessages() 构建压缩后消息               │
│  - autoCompact.ts 自动压缩逻辑                              │
│  - microCompact.ts 微压缩                                   │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  摘要生成 (extractMemories/extractMemories.ts)               │
│  - 生成对话摘要消息                                          │
│  - 替换原始长消息序列                                        │
│  - 保留关键上下文                                           │
└────────────────────────────┬────────────────────────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  继续对话 (compactWarningState)                            │
│  - createMicrocompactBoundaryMessage() 标记压缩边界         │
│  - 通知用户压缩发生                                          │
└─────────────────────────────────────────────────────────────┘
```

---

## 6. 主要命令系统

| 命令 | 文件 | 功能 |
|------|------|------|
| `/commit` | commit.ts | Git 提交 |
| `/review` | review.ts | 代码审查 |
| `/compact` | compact/index.ts | 上下文压缩 |
| `/mcp` | mcp/index.ts | MCP 服务器管理 |
| `/config` | config/config.tsx | 设置管理 |
| `/doctor` | doctor/doctor.tsx | 环境诊断 |
| `/login` | login/index.ts | 登录认证 |
| `/logout` | logout/index.ts | 登出 |
| `/memory` | memory/index.ts | 持久化记忆 |
| `/skills` | skills/index.ts | 技能管理 |
| `/tasks` | tasks/tasks.tsx | 任务管理 |
| `/diff` | diff/diff.tsx | 查看变更 |
| `/cost` | cost/cost.ts | 费用统计 |
| `/theme` | theme/theme.tsx | 主题切换 |
| `/vim` | vim/index.ts | Vim 模式 |
| `/resume` | resume/index.ts | 恢复会话 |
| `/share` | share/index.js | 分享会话 |
| `/init` | init.ts | 初始化项目 |

---

## 7. 主要工具系统

| 工具 | 文件 | 功能 |
|------|------|------|
| `Bash` | BashTool/ | Shell 命令执行 |
| `FileRead` | FileReadTool/ | 文件/图片/PDF 读取 |
| `FileWrite` | FileWriteTool/ | 文件创建/覆盖 |
| `FileEdit` | FileEditTool/ | 字符串替换编辑 |
| `Glob` | GlobTool/ | 文件模式搜索 |
| `Grep` | GrepTool/ | ripgrep 内容搜索 |
| `WebFetch` | WebFetchTool/ | URL 内容获取 |
| `WebSearch` | WebSearchTool/ | 网页搜索 |
| `Agent` | AgentTool/ | 子代理生成 |
| `Skill` | SkillTool/ | 技能执行 |
| `MCP` | MCPTool/ | MCP 工具调用 |
| `LSP` | LSPTool/ | 语言服务器协议 |
| `TaskCreate` | TaskCreateTool/ | 任务创建 |
| `TaskStop` | TaskStopTool/ | 任务停止 |
| `EnterPlanMode` | EnterPlanModeTool/ | 进入 Plan 模式 |
| `ExitPlanMode` | ExitPlanModeTool/ | 退出 Plan 模式 |
| `EnterWorktree` | EnterWorktreeTool/ | 进入 Git Worktree |
| `ExitWorktree` | ExitWorktreeTool/ | 退出 Git Worktree |
| `NotebookEdit` | NotebookEditTool/ | Jupyter Notebook 编辑 |

---

## 8. 特性开关 (Feature Flags)

通过 `bun:bundle` 的 `feature()` 实现死码消除 (DCE):

| 特性 | 功能 |
|------|------|
| `PROACTIVE` | 主动模式 |
| `KAIROS` | 助手模式 |
| `BRIDGE_MODE` | IDE 桥接模式 |
| `DAEMON` | 守护进程模式 |
| `VOICE_MODE` | 语音输入 |
| `AGENT_TRIGGERS` | 定时任务触发器 |
| `MONITOR_TOOL` | 监控工具 |
| `COORDINATOR_MODE` | 协调器模式 |
| `CONTEXT_COLLAPSE` | 上下文折叠 |
| `REACTIVE_COMPACT` | 响应式压缩 |

---

## 9. 设计亮点

### 9.1 并行预取优化启动

```typescript
// main.tsx 副作用导入, 在其他模块加载前并行执行
startMdmRawRead();        // MDM 配置 (macOS)
startKeychainPrefetch();  // Keychain OAuth/API Key
```

### 9.2 惰性加载

重模块通过 `import()` 动态导入, 避免首屏加载:
- OpenTelemetry / gRPC
- Analytics
- 特性门控子系统

### 9.3 子代理与团队协作

- `AgentTool` 启动子代理
- `TeamCreateTool` 并行团队工作
- `coordinator/` 处理多代理协调

### 9.4 插件架构

- 内置插件 `plugins/bundled/`
- 第三方插件支持
- 技能系统 `skills/` 可扩展工作流

---

## 10. 研究 / 所有权声明

- 本仓库为 **教育与防御性安全研究** 存档
- 原始 Claude Code 源代码归 **Anthropic** 所有
- 本仓库 **不附属于、不代表** Anthropic
