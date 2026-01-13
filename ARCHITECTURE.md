# OpenCode Architecture & Implementation Report

**Target Audience:** Enterprise Architects & Senior Engineers
**Scope:** Internal Architecture, Agent Mechanics, Extensibility, and Cloud Integration

---

## 1. Executive Summary

OpenCode is a **client/server** AI coding agent designed for extensibility and enterprise adoption. Unlike monolithic CLI tools, OpenCode separates the user interface (TUI/CLI) from the core logic (Server), allowing for flexible deployment models (local, remote, or cloud-hosted). It is built on a modern TypeScript stack (Bun, Hono, SolidJS, SST) and features a robust plugin system, native Model Context Protocol (MCP) support, and a "Zen" cloud backend for enterprise management.

## 2. System Architecture

The system operates on a **Client-Server** model, even when running locally.

### 2.1. Core Components
*   **Server (`packages/opencode/src/server/`)**: A Hono-based HTTP/WebSocket server that hosts the agent logic, tool registry, and project state. It exposes a REST API and Server-Sent Events (SSE) for real-time updates.
*   **Client (CLI/TUI)**: Connects to the server to display the UI and capture user input. It communicates via the API and a specialized TUI control queue (`callTui`).
*   **Project & Instance Management (`packages/opencode/src/project/`)**:
    *   **Instance**: Represents a running environment for a specific directory. It uses a `Context` provider pattern to inject dependencies.
    *   **Project**: A logical grouping of worktrees, identified by the git root commit hash (or "global" for non-git folders).
    *   **Multi-tenancy**: The server is designed to handle multiple instances simultaneously, enabling a single server process to manage multiple agent sessions across different repositories.

### 2.2. Directory Structure
*   `packages/opencode`: Core agent logic (Server, CLI, Session, Tools).
*   `packages/console`: "OpenCode Zen" backend (Cloudflare Workers, PlanetScale).
*   `packages/enterprise`: Self-hosted enterprise dashboard (SolidStart).
*   `packages/plugin`: Plugin SDK and interfaces.

---

## 3. Agent Core Implementation

The heart of OpenCode lies in the `SessionProcessor`, which orchestrates the interaction between the user, the LLM, and the tools.

### 3.1. The Agent Loop (`SessionProcessor.ts`)
The `SessionProcessor.create(...).process()` method runs a `while(true)` loop that drives the agent:
1.  **Stream**: Calls `LLM.stream()` to generate a response from the AI.
2.  **Event Handling**: Processes stream events (`reasoning-start`, `tool-call`, `text-delta`).
3.  **Tool Execution**:
    *   Tools are executed when `tool-call` events are received.
    *   **Doom Loop Detection**: Tracks the last 3 tool calls. If the same tool is called with the same input 3 times, it triggers a `PermissionNext.ask` to intervene.
4.  **State Updates**: Updates the `MessageV2` structure in real-time.
5.  **Snapshotting**: Tracks file system changes (`Snapshot.track()`) at the start/finish of steps to support "Revert" functionality.
6.  **Compaction**: Triggers `SessionCompaction` if the context window overflows, summarizing past messages to save tokens.

### 3.2. Context & Memory (`MessageV2.ts`)
Messages are structured as a list of **Parts** (`MessageV2.Part`), allowing for rich content:
*   **Types**: `text`, `tool` (call/result/error), `reasoning`, `patch` (diffs), `file` (attachments).
*   **Protocol**: The internal `MessageV2` format is converted to the Vercel AI SDK format via `toModelMessage`.

### 3.3. Prompt Engineering (`SystemPrompt.ts`)
*   **Dynamic Construction**: System prompts are built dynamically based on the provider (Anthropic vs. OpenAI), environment (OS, Git status), and user custom instructions (`AGENTS.md`).
*   **Spoofing**: Implements "Prompt Spoofing" (e.g., `PROMPT_ANTHROPIC_SPOOF`) to optimize performance for specific models like Claude.

---

## 4. Extensibility & Plugins

OpenCode is designed to be extended by enterprise teams.

### 4.1. Tool Registry (`packages/opencode/src/tool/`)
*   Tools are defined using `Tool.define` and registered in `ToolRegistry`.
*   **Dynamic Loading**: The registry scans the `tool/` directory and loaded plugins to register tools at runtime.
*   **Permissions**: Tools have granular permissions (e.g., `external_directory`, `read`, `edit`) managed by `PermissionNext`.

### 4.2. Plugin System (`packages/plugin/`)
Plugins implement the `Hooks` interface to intercept and modify system behavior:
*   `chat.message`: Intercept and modify messages before they are stored.
*   `chat.params`: Adjust LLM parameters (temperature, model options) dynamically.
*   `tool`: Register custom tools.
*   `auth`: Add custom authentication providers (e.g., internal OAuth).

### 4.3. MCP Support (`mcp/`)
*   Native support for the **Model Context Protocol (MCP)**.
*   Allows OpenCode to connect to external MCP servers to gain access to new resources and tools without custom plugin code.

---

## 5. Enterprise & Cloud Architecture

### 5.1. OpenCode Zen (`packages/console`)
*   **Infrastructure**: Built on **SST (Serverless Stack)** deploying to **Cloudflare Workers**.
*   **Database**: **PlanetScale** (MySQL) for relational data.
*   **Storage**: **R2** (Cloudflare Object Storage) for large assets.
*   **Auth**: Custom auth service handling GitHub/Google OAuth.

### 5.2. Enterprise Edition (`packages/enterprise`)
*   A standalone **SolidStart** application designed for self-hosting.
*   Focuses on team management, shared sessions, and secure deployment within a VPC (if adapted).

### 5.3. Deployment (`sst.config.ts`)
*   **Multi-Stage**: Supports `dev`, `production`, and feature branch stages.
*   **Infrastructure-as-Code**: Entire stack is defined in TypeScript, making it easy to audit and replicate.

---

## 6. Recommendations for Contribution

1.  **Adding a Tool**: Create a new file in `packages/opencode/src/tool/` or create a standalone plugin. Use `Tool.define` and schemas for type safety.
2.  **Modifying Agent Behavior**: Look into `packages/opencode/src/agent/prompt/` for prompt engineering or `SessionProcessor.ts` to alter the execution loop.
3.  **Custom Integrations**: Use the Plugin API to hook into `chat.params` or `auth` for integrating with internal enterprise systems (Issue Trackers, CI/CD).
