# Services

## Services Directory Map

```mermaid
graph TB
    subgraph "src/services/"
        API["api/<br/>22 files"]
        MCP["mcp/<br/>25 files"]
        OAuth["oauth/<br/>8 files"]
        LSP["lsp/<br/>10 files"]
        Compact["compact/<br/>14 files"]
        Analytics["analytics/<br/>9 files"]
        Tools["tools/<br/>6 files"]
        SessionMem["SessionMemory/<br/>5 files"]
        PolicyLimits["policyLimits/<br/>2 files"]
        RemoteSettings["remoteManagedSettings/<br/>7 files"]
        ExtractMem["extractMemories/<br/>2 files"]
        SkillSearch["skillSearch/<br/>9 files"]
        Plugins["plugins/<br/>3 files"]
        TeamMemSync["teamMemorySync/<br/>7 files"]
        ContextCollapse["contextCollapse/<br/>3 files"]
        ToolUseSummary["toolUseSummary/"]
        Tips["tips/"]
        AgentSummary["AgentSummary/"]
    end

    API --- MCP
    OAuth --- API
    LSP --- Tools
    Compact --- API
    Analytics --- API
```

## MCP (Model Context Protocol)

MCP is the largest service subsystem (25 files, 12,238 LOC), managing integration with external tool servers.

### MCP Architecture

```mermaid
flowchart TD
    subgraph "Configuration Discovery"
        Local[".mcp.json<br/>(cwd)"]
        User["~/.mcp.json<br/>(global)"]
        Project["Project settings"]
        Enterprise["Managed settings"]
        ClaudeAI["Claude.ai connectors"]
        Dynamic["Runtime additions"]
    end

    subgraph "Config Processing"
        Merge["Merge with precedence"]
        Dedup["Signature-based dedup"]
        Filter["Policy allow/deny list"]
        Enable["Enable/disable state"]
    end

    subgraph "Connection Manager"
        Init["Initialize connections"]
        Transport["Create transport"]
        Auth["Handle authentication"]
        Reconnect["Reconnection logic<br/>(backoff: 1s→30s, max 5)"]
    end

    subgraph "Transport Types"
        Stdio["stdio<br/>(local command)"]
        SSE["SSE<br/>(server-sent events)"]
        HTTP["HTTP<br/>(REST-based)"]
        WS["WebSocket"]
        SDK["SDK<br/>(in-process)"]
        Proxy["claudeai-proxy<br/>(URL rewriting)"]
    end

    subgraph "Tool Exposure"
        ListTools["listTools() RPC"]
        Normalize["Normalize names"]
        Cap["Cap descriptions<br/>(2048 chars)"]
        Wrap["Wrap as MCPTool"]
    end

    Local & User & Project & Enterprise & ClaudeAI & Dynamic --> Merge
    Merge --> Dedup --> Filter --> Enable

    Enable --> Init --> Transport
    Transport --> Stdio & SSE & HTTP & WS & SDK & Proxy

    Init --> Auth
    Init --> Reconnect

    Transport --> ListTools --> Normalize --> Cap --> Wrap
```

### MCP Server Configuration Schema

```mermaid
classDiagram
    class MCPServerConfig {
        <<union>>
    }

    class StdioConfig {
        +type: "stdio"
        +string command
        +string[] args
        +Record env
    }

    class SSEConfig {
        +type: "sse"
        +string url
        +Record headers
        +AuthConfig auth
    }

    class HTTPConfig {
        +type: "http"
        +string url
        +Record headers
        +string headersHelper
    }

    class WebSocketConfig {
        +type: "ws"
        +string url
        +Record headers
    }

    class SDKConfig {
        +type: "sdk"
        +string name
    }

    class ProxyConfig {
        +type: "claudeai-proxy"
        +string url
    }

    MCPServerConfig <|-- StdioConfig
    MCPServerConfig <|-- SSEConfig
    MCPServerConfig <|-- HTTPConfig
    MCPServerConfig <|-- WebSocketConfig
    MCPServerConfig <|-- SDKConfig
    MCPServerConfig <|-- ProxyConfig
```

### MCP Connection Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Discovered : Config loaded

    Discovered --> Connecting : initializeConnection()
    Connecting --> Connected : Transport established
    Connecting --> NeedsAuth : Auth required (OAuth)
    Connecting --> Failed : Connection error

    NeedsAuth --> Authenticating : User approves
    Authenticating --> Connected : Auth success
    Authenticating --> Failed : Auth failure

    Connected --> ListingTools : listTools() RPC
    ListingTools --> Ready : Tools loaded
    ListingTools --> Failed : RPC error

    Ready --> Disconnected : Connection lost
    Disconnected --> Reconnecting : Auto-reconnect

    Reconnecting --> Connected : Reconnected
    Reconnecting --> Failed : Max retries (5)

    Ready --> Disabled : User disables
    Disabled --> Connecting : User re-enables

    Failed --> Reconnecting : Retry eligible
    Failed --> Disabled : Max retries exceeded

    note right of Reconnecting : Exponential backoff<br/>1s → 2s → 4s → ... → 30s max
```

### MCP Authentication

```mermaid
sequenceDiagram
    participant Client as MCP Client
    participant Server as MCP Server
    participant Browser as User Browser
    participant Listener as Localhost Listener
    participant Keychain as Secure Storage

    Client->>Server: Initial connection
    Server-->>Client: 401 Unauthorized

    Client->>Server: Discover OAuth metadata
    Server-->>Client: Authorization server info

    Client->>Client: Generate PKCE (code_verifier + challenge)
    Client->>Browser: Open authorization URL

    Browser->>Server: User authenticates
    Server->>Listener: Redirect with auth code

    Listener->>Client: Authorization code

    Client->>Server: Exchange code for tokens
    Server-->>Client: access_token + refresh_token

    Client->>Keychain: Store tokens securely

    Note over Client,Server: Subsequent requests use Bearer token

    Client->>Server: API call with Bearer token
    Server-->>Client: 401 (token expired)
    Client->>Server: Refresh token exchange
    Server-->>Client: New access_token
```

## OAuth Service

```mermaid
flowchart TD
    subgraph "OAuth Flow (src/services/oauth/)"
        Start["Login requested"] --> Method{"Auth method?"}

        Method -->|Automatic| AutoFlow["Open browser<br/>Start localhost listener"]
        Method -->|Manual| ManualFlow["Display URL<br/>User copies code"]

        AutoFlow --> Browser["Browser auth page"]
        ManualFlow --> Browser

        Browser --> Consent["User grants consent"]
        Consent --> Redirect{"Redirect?"}

        Redirect -->|Automatic| Listener["Localhost captures code"]
        Redirect -->|Manual| Paste["User pastes code"]

        Listener --> Exchange["Exchange code for tokens<br/>(PKCE verified)"]
        Paste --> Exchange

        Exchange --> Tokens["Access + Refresh tokens"]
        Tokens --> Store["Store in secure keychain"]
        Store --> PopulateInfo["Populate account info<br/>(name, email, subscription)"]
    end
```

## LSP Integration

```mermaid
classDiagram
    class LSPServerManager {
        +initialize()
        +shutdown()
        +getServerForExtension(ext) LSPServerInstance
        +getDiagnostics(file) Diagnostic[]
    }

    class LSPServerInstance {
        -Process process
        -Map openedFiles
        +initialize(rootPath)
        +textDocumentDidOpen(uri, content)
        +textDocumentDidChange(uri, changes)
        +textDocumentDidSave(uri)
        +textDocumentDidClose(uri)
        +shutdown()
    }

    class LSPClient {
        +hover(uri, position)
        +definition(uri, position)
        +references(uri, position)
        +diagnostics(uri)
    }

    class LSPDiagnosticRegistry {
        -Map diagnosticsCache
        +collect(server, file) Diagnostic[]
        +aggregate() Diagnostic[]
    }

    class LSPConfig {
        +loadFromPlugins() ServerConfig[]
        +mapExtensionToServer(ext) ServerConfig
    }

    LSPServerManager --> LSPServerInstance : manages multiple
    LSPServerInstance --> LSPClient : uses
    LSPServerManager --> LSPDiagnosticRegistry : aggregates
    LSPServerManager --> LSPConfig : configured by
```

### LSP Communication

```mermaid
sequenceDiagram
    participant Tool as LSP Tool
    participant Manager as LSPServerManager
    participant Instance as LSPServerInstance
    participant Process as LSP Server Process

    Tool->>Manager: getDiagnostics("file.ts")
    Manager->>Manager: Route by extension → TypeScript server

    Manager->>Instance: textDocumentDidOpen(uri, content)
    Instance->>Process: JSON-RPC: textDocument/didOpen

    Process-->>Instance: JSON-RPC: textDocument/publishDiagnostics
    Instance-->>Manager: Diagnostic[]

    Manager-->>Tool: Aggregated diagnostics
```

## Context Compaction

```mermaid
flowchart TD
    subgraph "Auto-Compact Trigger"
        Monitor["Token usage monitor<br/>(per API round)"]
        Threshold{"usage ≥<br/>context_window<br/>- reserved_output<br/>- 13K buffer?"}
        Monitor --> Threshold
    end

    subgraph "Circuit Breaker"
        Threshold -->|yes| Failures{"3 consecutive<br/>failures?"}
        Failures -->|yes| Stop["Disable auto-compact"]
        Failures -->|no| Compact["Run compaction"]
    end

    subgraph "Compaction Strategies"
        Compact --> Strategy{"Strategy?"}
        Strategy -->|"token threshold"| Auto["autoCompact<br/>(standard summarize)"]
        Strategy -->|"prompt too long"| Reactive["reactiveCompact<br/>(emergency recovery)"]
        Strategy -->|"token-conservative"| Micro["microCompact<br/>(minimal summary)"]
        Strategy -->|"snippet-based"| Snip["snipCompact<br/>(targeted segments)"]
        Strategy -->|"memory-backed"| Memory["sessionMemoryCompact<br/>(session memory)"]
    end

    subgraph "Compaction Process"
        Auto & Reactive & Micro & Snip & Memory --> Group["Group messages<br/>by API round"]
        Group --> Prompt["Build compaction prompt"]
        Prompt --> API["Call Claude API<br/>for summary"]
        API --> Boundary["Insert compact_boundary<br/>message"]
        Boundary --> Discard["Discard messages<br/>before boundary"]
        Discard --> Cleanup["postCompactCleanup()"]
    end

    Threshold -->|no| Continue["Continue normally"]
```

## Analytics System

```mermaid
flowchart LR
    subgraph "Event Producers"
        Tools["Tool executions"]
        Perms["Permission decisions"]
        API["API calls"]
        Session["Session events"]
        Errors["Error events"]
    end

    subgraph "Event Queue (Decoupled)"
        LogEvent["logEvent(name, metadata)"]
        Queue["In-memory queue<br/>(persists until sink)"]
    end

    subgraph "Sink Processing"
        Attach["attachAnalyticsSink()"]
        Enrich["Event enrichment<br/>(sanitize, redact)"]
        Fanout["Fanout to backends"]
    end

    subgraph "Backends"
        FirstParty["1P Event Logger<br/>(OpenTelemetry SDK)"]
        Datadog["Datadog"]
        GrowthBook["GrowthBook<br/>(feature gates)"]
    end

    Tools & Perms & API & Session & Errors --> LogEvent --> Queue
    Queue --> Attach --> Enrich --> Fanout
    Fanout --> FirstParty & Datadog
    GrowthBook --> Fanout
```

### GrowthBook Feature Gates

```mermaid
flowchart TD
    Init["initializeGrowthBook()"] --> Fetch["Fetch features<br/>(ETag-based caching)"]
    Fetch --> Cache["Cache in memory"]
    Cache --> Poll["Background polling<br/>(periodic refresh)"]

    subgraph "Feature Evaluation"
        Check["Check feature gate<br/>(tengu_* prefix)"]
        Override["Dynamic config overrides"]
        Cached["Cached access<br/>(non-blocking startup)"]
    end

    Cache --> Check & Override & Cached
    Poll --> Cache
```

## Policy Limits

```mermaid
flowchart TD
    Eligible{"Eligible user?<br/>(Console API key or<br/>Team/Enterprise OAuth)"}

    Eligible -->|no| Skip["Skip policy checks<br/>(fail-open)"]
    Eligible -->|yes| Load["Load policy limits"]

    Load --> Sources{"Cache source?"}
    Sources -->|"HTTP (ETag)"| HTTP["Fetch from API"]
    Sources -->|"File cache"| File["Read cached JSON"]
    Sources -->|"Session cache"| Memory["In-memory cache"]

    HTTP --> Apply["Apply restrictions"]
    File --> Apply
    Memory --> Apply

    Apply --> Restrictions["Named boolean flags:<br/>allow_remote_sessions<br/>allow_product_feedback<br/>allow_remote_control<br/>..."]

    subgraph "Background Refresh"
        Poll["Hourly polling"]
        Poll --> HTTP
    end

    subgraph "Safety"
        Essential["Essential-traffic-only mode<br/>denies on cache miss"]
    end
```

## Tool Execution Orchestration

```mermaid
flowchart TD
    subgraph "src/services/tools/"
        Orchestration["toolOrchestration.ts<br/>Main execution coordinator"]
        Execution["toolExecution.ts<br/>Per-tool execution wrapper"]
        Streaming["StreamingToolExecutor<br/>(for long-running tools)"]
    end

    QueryLoop["query.ts loop"] --> Orchestration

    Orchestration --> Partition["Partition into batches"]
    Partition --> ReadBatch["Read-only batch<br/>(concurrent, max 10)"]
    Partition --> WriteBatch["Write batch<br/>(serial, one at a time)"]

    ReadBatch --> Execution
    WriteBatch --> Execution

    Execution --> Validate["validateInput()"]
    Validate --> Permission["Permission check"]
    Permission --> Call["tool.call()"]
    Call --> Progress["onProgress() callbacks"]
    Call --> Result["ToolResult"]
    Result --> ContextMod["Apply contextModifier"]
    ContextMod --> MapResult["Map to ToolResultBlockParam"]
```

## Service Initialization Order

```mermaid
gantt
    title Service Initialization Timeline
    dateFormat X
    axisFormat %L

    section Phase 1 - Module Load
    MDM subprocess (async)          : 0, 5
    Keychain prefetch (async)       : 0, 5

    section Phase 2 - init()
    enableConfigs()                 : 5, 10
    applySafeConfigEnvVars()        : 10, 12
    applyExtraCACerts()             : 12, 14
    setupGracefulShutdown()         : 14, 16
    configureGlobalMTLS()           : 16, 18
    configureGlobalAgents()         : 18, 20
    preconnectAnthropicApi()        : 20, 25
    attachAnalyticsSink()           : 25, 27

    section Phase 3 - Action Handler
    GrowthBook init                 : 30, 35
    AWS/GCP credential prefetch     : 35, 45
    MCP resource prefetch           : 35, 45
    Model capabilities refresh      : 35, 45
    LSP server init (async)         : 35, 50
    Remote settings load            : 35, 50
    Policy limits load              : 35, 50
```
