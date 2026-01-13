# OpenCode 架构与实现报告

**目标受众：** 企业架构师与高级工程师
**范围：** 内部架构、Agent 机制、扩展性与云集成

---

## 1. 执行摘要 (Executive Summary)

OpenCode 是一个专为扩展性和企业应用设计的 **客户端/服务端 (Client/Server)** 架构的 AI 编程 Agent。与单体 CLI 工具不同，OpenCode 将用户界面 (TUI/CLI) 与核心逻辑 (Server) 分离，从而支持灵活的部署模式（本地、远程或云托管）。它基于现代 TypeScript 技术栈构建 (Bun, Hono, SolidJS, SST)，并具备强大的插件系统、原生的模型上下文协议 (MCP) 支持，以及用于企业管理的 "Zen" 云后端。

## 2. 系统架构 (System Architecture)

该系统采用 **客户端-服务端** 模型，即使在本地运行也是如此。

### 2.1. 核心组件 (Core Components)
*   **服务端 (Server) (`packages/opencode/src/server/`)**：基于 Hono 的 HTTP/WebSocket 服务器，承载 Agent 逻辑、工具注册表和项目状态。它对外暴露 REST API 和 Server-Sent Events (SSE) 以实现实时更新。
*   **客户端 (Client) (CLI/TUI)**：连接到服务端以显示用户界面并捕获用户输入。它通过 API 和专用的 TUI 控制队列 (`callTui`) 进行通信。
*   **项目与实例管理 (Project & Instance Management) (`packages/opencode/src/project/`)**：
    *   **实例 (Instance)**：代表特定目录的运行环境。它使用 `Context` 提供者模式来注入依赖。
    *   **项目 (Project)**：工作树 (Worktree) 的逻辑分组，由 Git 根提交哈希标识（对于非 Git 文件夹则为 "global"）。
    *   **多租户 (Multi-tenancy)**：服务端设计为可同时处理多个实例，允许单个服务器进程管理跨不同仓库的多个 Agent 会话。

### 2.2. 目录结构
*   `packages/opencode`：核心 Agent 逻辑 (Server, CLI, Session, Tools)。
*   `packages/console`："OpenCode Zen" 后端 (Cloudflare Workers, PlanetScale)。
*   `packages/enterprise`：自托管企业仪表板 (SolidStart)。
*   `packages/plugin`：插件 SDK 和接口。

---

## 3. Agent 核心实现 (Agent Core Implementation)

OpenCode 的核心在于 `SessionProcessor`，它编排用户、LLM 和工具之间的交互。

### 3.1. Agent 循环 (The Agent Loop) (`SessionProcessor.ts`)
`SessionProcessor.create(...).process()` 方法运行一个 `while(true)` 循环来驱动 Agent：
1.  **流式传输 (Stream)**：调用 `LLM.stream()` 生成 AI 的响应。
2.  **事件处理 (Event Handling)**：处理流事件（`reasoning-start` 推理开始, `tool-call` 工具调用, `text-delta` 文本增量）。
3.  **工具执行 (Tool Execution)**：
    *   收到 `tool-call` 事件时执行工具。
    *   **死循环检测 (Doom Loop Detection)**：跟踪最近 3 次工具调用。如果同一工具以相同输入被调用 3 次，将触发 `PermissionNext.ask` 进行干预。
4.  **状态更新 (State Updates)**：实时更新 `MessageV2` 结构。
5.  **快照 (Snapshotting)**：在步骤开始/结束时跟踪文件系统变更 (`Snapshot.track()`)，以支持 "撤销 (Revert)" 功能。
6.  **压缩 (Compaction)**：如果上下文窗口溢出，触发 `SessionCompaction`，总结过去的消息以节省 Token。

### 3.2. 上下文与记忆 (Context & Memory) (`MessageV2.ts`)
消息结构化为 **Parts (部分)** (`MessageV2.Part`) 列表，支持丰富的内容：
*   **类型**：`text` (文本), `tool` (调用/结果/错误), `reasoning` (推理), `patch` (diffs), `file` (附件)。
*   **协议**：内部 `MessageV2` 格式通过 `toModelMessage` 转换为 Vercel AI SDK 格式。

### 3.3. 提示词工程 (Prompt Engineering) (`SystemPrompt.ts`)
*   **动态构建**：系统提示词根据提供商（Anthropic vs. OpenAI）、环境（操作系统、Git 状态）和用户自定义指令 (`AGENTS.md`) 动态构建。
*   **欺骗 (Spoofing)**：实施 "Prompt Spoofing"（例如 `PROMPT_ANTHROPIC_SPOOF`）以优化特定模型（如 Claude）的性能。

---

## 4. 扩展性与插件 (Extensibility & Plugins)

OpenCode 专为企业团队扩展而设计。

### 4.1. 工具注册表 (Tool Registry) (`packages/opencode/src/tool/`)
*   工具使用 `Tool.define` 定义，并在 `ToolRegistry` 中注册。
*   **动态加载**：注册表扫描 `tool/` 目录和加载的插件，在运行时注册工具。
*   **权限**：工具拥有细粒度的权限（例如 `external_directory`, `read`, `edit`），由 `PermissionNext` 管理。

### 4.2. 插件系统 (Plugin System) (`packages/plugin/`)
插件实现 `Hooks` 接口以拦截和修改系统行为：
*   `chat.message`：在存储消息前拦截并修改。
*   `chat.params`：动态调整 LLM 参数（温度、模型选项）。
*   `tool`：注册自定义工具。
*   `auth`：添加自定义认证提供商（例如内部 OAuth）。

### 4.3. MCP 支持 (MCP Support) (`mcp/`)
*   原生支持 **模型上下文协议 (Model Context Protocol, MCP)**。
*   允许 OpenCode 连接到外部 MCP 服务器，无需编写自定义插件代码即可访问新资源和工具。

---

## 5. 企业与云架构 (Enterprise & Cloud Architecture)

### 5.1. OpenCode Zen (`packages/console`)
*   **基础设施**：基于 **SST (Serverless Stack)** 构建，部署到 **Cloudflare Workers**。
*   **数据库**：**PlanetScale** (MySQL) 用于关系型数据。
*   **存储**：**R2** (Cloudflare Object Storage) 用于大型资产。
*   **认证**：处理 GitHub/Google OAuth 的自定义认证服务。

### 5.2. 企业版 (Enterprise Edition) (`packages/enterprise`)
*   一个独立的 **SolidStart** 应用程序，专为自托管设计。
*   专注于团队管理、共享会话和 VPC 内的安全部署（如适配）。

### 5.3. 部署 (Deployment) (`sst.config.ts`)
*   **多阶段 (Multi-Stage)**：支持 `dev` (开发)、`production` (生产) 和功能分支阶段。
*   **基础设施即代码 (IaC)**：整个技术栈用 TypeScript 定义，易于审计和复制。

---

## 6. 与 Claude Code 的对比 (Comparison with Claude Code)

虽然 OpenCode 与 Claude Code 在功能目标上相似，但其架构设计在支持企业用例和开源贡献方面有显著差异。

### 6.1. 架构模型 (Architecture Model)
*   **OpenCode**：解耦的 **客户端/服务端 (Client/Server)** 架构。核心 Agent 逻辑在暴露 API 的服务端进程中运行。TUI 只是众多可能客户端中的一个（支持通过移动或 Web 客户端连接远程 Agent）。
*   **Claude Code**：主要是单体 CLI 工具，界面与逻辑紧密耦合在单个可执行文件中。

### 6.2. 扩展性 (Extensibility)
*   **OpenCode**：开源并拥有完整的 **插件系统 (Plugin System)** 和 **MCP** 支持。团队可以编写自定义 TypeScript 插件来修改 Agent 行为、添加内部工具或集成私有 API。
*   **Claude Code**：闭源二进制文件。扩展性通常仅限于 MCP 服务器。

### 6.3. LLM 支持 (LLM Support)
*   **OpenCode**：**供应商无关 (Provider Agnostic)**。基于 Vercel 的 `ai-sdk` 构建，支持 Anthropic, OpenAI, Google Gemini, 以及潜在的本地 LLM。
*   **Claude Code**：与 Anthropic 的模型 (Claude) 紧密耦合。

### 6.4. 基础设施 (Infrastructure)
*   **OpenCode**：包含可自托管的 "Zen" 后端和 "Enterprise" 仪表板，用于团队管理、计费和共享上下文。
*   **Claude Code**：由 Anthropic 提供的托管服务。

---

## 7. 贡献建议 (Recommendations for Contribution)

1.  **添加工具**：在 `packages/opencode/src/tool/` 中创建新文件或创建独立插件。使用 `Tool.define` 和 Schema 以确保类型安全。
2.  **修改 Agent 行为**：查看 `packages/opencode/src/agent/prompt/` 进行提示词工程，或查看 `SessionProcessor.ts` 以更改执行循环。
3.  **自定义集成**：使用插件 API 挂钩 `chat.params` 或 `auth`，以便与内部企业系统（问题跟踪器、CI/CD）集成。
