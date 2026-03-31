# System Architecture Overview

> Reverse-engineered from Claude Code CLI source snapshot (2026-03-31)

## High-Level Architecture

Claude Code is a terminal-based AI coding assistant built with **Bun + TypeScript**, **React 19 + Ink** for terminal UI, and **Commander.js** for CLI parsing. It operates as an agentic loop — accepting user input, querying the Claude API, executing tools, and iterating until the task is complete.

```mermaid
graph TB
    subgraph "Entry Layer"
        CLI["cli.tsx<br/>CLI Entry Point"]
        SDK["SDK Entry<br/>(Programmatic)"]
        Bridge["Bridge Entry<br/>(IDE Integration)"]
    end

    subgraph "Bootstrap Layer"
        Main["main.tsx<br/>Full CLI Setup"]
        State["bootstrap/state.ts<br/>Global Session State"]
        Init["init.ts<br/>Initialization"]
    end

    subgraph "Core Engine"
        QE["QueryEngine.ts<br/>LLM Orchestrator"]
        Query["query.ts<br/>Agentic Loop"]
        Stream["claude.ts<br/>Streaming API"]
    end

    subgraph "Tool System"
        ToolReg["tools.ts<br/>Tool Registry"]
        ToolBase["Tool.ts<br/>Base Types"]
        ToolExec["toolOrchestration.ts<br/>Execution Engine"]
        ToolPerm["toolPermission/<br/>Permission System"]
    end

    subgraph "Services"
        API["services/api/<br/>API Client"]
        MCP["services/mcp/<br/>MCP Protocol"]
        OAuth["services/oauth/<br/>Authentication"]
        LSP["services/lsp/<br/>Language Server"]
        Compact["services/compact/<br/>Context Compaction"]
        Analytics["services/analytics/<br/>Telemetry"]
    end

    subgraph "UI Layer"
        REPL["screens/REPL.tsx<br/>Main Interface"]
        Components["components/<br/>140+ Components"]
        Ink["ink/<br/>Terminal Renderer"]
        AppState["state/<br/>Zustand Store"]
    end

    subgraph "Integration Layer"
        BridgeSys["bridge/<br/>IDE Communication"]
        Coordinator["coordinator/<br/>Multi-Agent"]
        Skills["skills/<br/>Workflow System"]
    end

    subgraph "External"
        ClaudeAPI["Claude API"]
        IDEs["VS Code / JetBrains"]
        MCPServers["MCP Servers"]
        LSPServers["LSP Servers"]
    end

    CLI --> Main
    SDK --> QE
    Bridge --> BridgeSys

    Main --> Init --> State
    Main --> REPL

    REPL --> QE
    QE --> Query
    Query --> Stream --> API --> ClaudeAPI
    Query --> ToolExec
    ToolExec --> ToolPerm
    ToolExec --> ToolReg --> ToolBase

    REPL --> Components --> Ink
    REPL --> AppState

    BridgeSys --> IDEs
    MCP --> MCPServers
    LSP --> LSPServers
    Coordinator --> QE

    Query --> Compact
    Main --> Analytics
    Main --> OAuth
```

## Module Dependency Graph

```mermaid
graph LR
    subgraph "Entrypoints"
        E1["cli.tsx"]
        E2["main.tsx"]
        E3["sdk/"]
    end

    subgraph "Core"
        C1["QueryEngine.ts"]
        C2["query.ts"]
        C3["Tool.ts"]
        C4["tools.ts"]
        C5["commands.ts"]
    end

    subgraph "Services"
        S1["api/client.ts"]
        S2["api/claude.ts"]
        S3["api/withRetry.ts"]
        S4["mcp/client.ts"]
        S5["mcp/config.ts"]
        S6["compact/compact.ts"]
        S7["oauth/"]
        S8["lsp/"]
        S9["analytics/"]
    end

    subgraph "Hooks"
        H1["useCanUseTool.tsx"]
        H2["useReplBridge.tsx"]
        H3["useManageMCPConnections.ts"]
    end

    subgraph "State"
        ST1["bootstrap/state.ts"]
        ST2["state/AppStateStore.ts"]
    end

    subgraph "Utils"
        U1["systemPrompt.ts"]
        U2["config.ts"]
        U3["claudemd.ts"]
        U4["analyzeContext.ts"]
        U5["permissions/"]
    end

    E1 --> E2 --> C1
    E3 --> C1
    C1 --> C2 --> S2 --> S1
    C2 --> S3
    C2 --> C4 --> C3
    C2 --> S6
    C1 --> C5
    C1 --> U1 --> U3
    C4 --> S4
    H1 --> U5
    H3 --> S5 --> S4
    E2 --> ST1
    H2 --> ST2
    S1 --> S7
    E2 --> S9
```

## Subsystem Overview

| Subsystem | Location | Purpose | Key Files |
|-----------|----------|---------|-----------|
| **Entry Points** | `src/entrypoints/` | CLI bootstrap, fast-paths, mode dispatch | `cli.tsx`, `sdk/` |
| **Core Engine** | `src/` | Query orchestration, tool loop | `QueryEngine.ts`, `query.ts` |
| **Tool System** | `src/tools/`, `src/Tool.ts` | 55+ tools with schema, permissions, execution | `tools.ts`, `Tool.ts` |
| **Command System** | `src/commands/` | 60+ slash commands | `commands.ts` |
| **Permission System** | `src/hooks/toolPermission/` | Multi-mode permission checking | `useCanUseTool.tsx` |
| **API Client** | `src/services/api/` | Multi-provider Claude API access | `client.ts`, `claude.ts` |
| **MCP** | `src/services/mcp/` | Model Context Protocol integration | `client.ts`, `config.ts` |
| **OAuth** | `src/services/oauth/` | Authentication flows | `index.ts`, `client.ts` |
| **LSP** | `src/services/lsp/` | Language server integration | `LSPServerManager.ts` |
| **Compaction** | `src/services/compact/` | Context window management | `compact.ts`, `autoCompact.ts` |
| **Analytics** | `src/services/analytics/` | Telemetry and event tracking | `index.ts`, `growthbook.ts` |
| **UI** | `src/components/`, `src/ink/` | Terminal UI via React + Ink | `REPL.tsx`, `App.tsx` |
| **Bridge** | `src/bridge/` | IDE communication (VS Code, JetBrains) | `bridgeApi.ts`, `bridgeMessaging.ts` |
| **Coordinator** | `src/coordinator/` | Multi-agent orchestration | `coordinatorMode.ts` |
| **Skills** | `src/skills/` | Reusable workflow system | `bundledSkills.ts` |
| **Configuration** | `src/utils/settings/` | Multi-source settings hierarchy | `settings.ts`, `types.ts` |
| **Bootstrap** | `src/bootstrap/` | Session state and initialization | `state.ts` |
| **Utilities** | `src/utils/` | 340+ utility modules | Various |
| **Constants** | `src/constants/` | System prompts, limits, product info | `prompts.ts`, `betas.ts` |

## Runtime Architecture

```mermaid
graph TB
    subgraph "Process Model"
        MainProc["Main Process<br/>(Bun Runtime)"]
        MDM["MDM Subprocess<br/>(plutil/reg query)"]
        LSPProc["LSP Server Processes<br/>(per language)"]
        MCPProc["MCP Server Processes<br/>(per server)"]
        AgentProc["Sub-Agent Processes<br/>(spawned workers)"]
    end

    subgraph "Main Process Internals"
        ReactTree["React Component Tree<br/>(Ink Renderer)"]
        QueryLoop["Query Loop<br/>(Agentic Iteration)"]
        ToolExec2["Tool Executor<br/>(Concurrent/Serial)"]
        EventQueue["Analytics Event Queue"]
        HookSys["Hook System<br/>(Pre/Post Tool Use)"]
    end

    MainProc --> MDM
    MainProc --> LSPProc
    MainProc --> MCPProc
    MainProc --> AgentProc

    MainProc --- ReactTree
    MainProc --- QueryLoop
    MainProc --- ToolExec2
    MainProc --- EventQueue
    MainProc --- HookSys

    QueryLoop --> ToolExec2
    ToolExec2 --> HookSys
    ToolExec2 --> MCPProc
    ReactTree --> QueryLoop
```

## Technology Stack

| Layer | Technology | Version |
|-------|-----------|---------|
| Runtime | Bun | Latest |
| Language | TypeScript | Strict (relaxed for stubs) |
| UI Framework | React | 19 |
| Terminal Rendering | Ink (custom fork) | Custom |
| Layout Engine | Yoga | Native |
| CLI Parsing | Commander.js | Extra typings |
| Schema Validation | Zod | v4 |
| State Management | Zustand | Latest |
| Build System | Bun bundler | Built-in |
| Protocol | MCP SDK | @modelcontextprotocol/sdk |
| Protocol | LSP | Custom client |

## Design Philosophy

1. **Fast-Path Optimization**: Zero-import paths for `--version`, `--help` return instantly without loading the 200+ module dependency tree.

2. **Parallel Prefetch**: MDM settings, keychain reads, and API preconnect fire at module load time, completing during the ~135ms import evaluation.

3. **Build-Time DCE**: Feature flags via `bun:bundle` enable dead-code elimination — entire subsystems (Bridge, Daemon, Voice) are compiled out of non-Anthropic builds.

4. **Lazy Requires**: Circular dependency breaks use `const getFoo = () => require(...)` pattern extensively.

5. **Safety by Default**: Every tool invocation passes through a multi-layer permission system with rules, hooks, classifiers, and user consent.

6. **Extensibility**: MCP servers, plugins, skills, hooks, and custom agents provide multiple extension points without modifying core code.
