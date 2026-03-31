# Bridge System, Coordinator, and Skills

## Bridge System (IDE Communication)

The bridge system (`src/bridge/`) enables bidirectional communication between the CLI and IDEs (VS Code, JetBrains) or the claude.ai web interface.

### Bridge Architecture

```mermaid
flowchart TB
    subgraph "Claude Code CLI"
        BridgeMain["bridgeMain()"]
        BridgeAPI["bridgeApi.ts<br/>HTTP Client"]
        Messaging["bridgeMessaging.ts<br/>Message Protocol"]
        JWT["jwtUtils.ts<br/>Token Refresh"]
        Transport["replBridgeTransport.ts<br/>V1/V2 Transport"]
    end

    subgraph "Communication Channels"
        Poll["Polling Channel<br/>(SSE-like long-poll)"]
        WS["WebSocket Channel<br/>(session ingress)"]
    end

    subgraph "External Systems"
        EnvAPI["Environments API<br/>/v1/environments/bridge"]
        ClaudeAI["claude.ai Web App"]
        VSCode["VS Code Extension"]
        JetBrains["JetBrains Plugin"]
    end

    BridgeMain --> BridgeAPI --> Poll --> EnvAPI
    BridgeMain --> Transport --> WS
    Messaging --> WS
    JWT --> WS

    EnvAPI --> ClaudeAI
    WS --> ClaudeAI
    WS --> VSCode
    WS --> JetBrains
```

### Bridge Connection Flow

```mermaid
sequenceDiagram
    participant CLI as Claude Code CLI
    participant API as Environments API
    participant WS as WebSocket
    participant Web as claude.ai / IDE

    CLI->>API: POST /v1/environments/bridge<br/>(register environment)
    API-->>CLI: {environment_id, environment_secret}

    loop Polling for Work
        CLI->>API: POST /v1/environments/{id}/work<br/>(long-poll, 30-60s timeout)

        alt Work available
            API-->>CLI: WorkResponse {type: 'session', secret: base64(JWT)}
            CLI->>CLI: Decode secret → session_ingress_token
            CLI->>API: PATCH /v1/environments/{id}/work/{workId}<br/>(acknowledge)
            CLI->>WS: Connect with JWT Bearer token
            WS-->>CLI: Connected

            par Session Active
                Web->>WS: User prompt (SDKMessage)
                WS->>CLI: Inbound message
                CLI->>CLI: Process + run tools
                CLI->>WS: Assistant response
                WS->>Web: Display response
            end

        else No work (timeout)
            API-->>CLI: null
            CLI->>CLI: Backoff (2s→2m→10m)
        end
    end
```

### Bridge Message Protocol

```mermaid
classDiagram
    class SDKMessage {
        +type: "user"|"assistant"|"system"
        +uuid: string
        +content: ContentBlockParam[]
        +metadata: object
    }

    class SDKControlRequest {
        +type: "control_request"
        +request_id: string
        +request: ControlRequestBody
    }

    class ControlRequestBody {
        <<union>>
        +subtype: "initialize"
        +subtype: "set_model"
        +subtype: "can_use_tool"
        +subtype: "set_permission_mode"
    }

    class SDKControlResponse {
        +type: "control_response"
        +response.subtype: "success"
        +response.request_id: string
        +response.response: object
    }

    class WorkResponse {
        +id: string
        +type: "work"
        +environment_id: string
        +state: string
        +data.type: "session"|"healthcheck"
        +data.id: string
        +secret: string (base64 JWT)
        +created_at: ISO8601
    }

    SDKMessage <-- SDKControlRequest : extends
    SDKMessage <-- SDKControlResponse : extends
```

### JWT Token Refresh

```mermaid
stateDiagram-v2
    [*] --> Scheduled : schedule sessionId jwt

    Scheduled --> Waiting : Set timer at exp minus 5min
    Waiting --> Refreshing : Timer fires

    Refreshing --> Delivered : getAccessToken succeeds
    Refreshing --> RetryWait : getAccessToken fails

    RetryWait --> Refreshing : Retry max 3
    RetryWait --> Failed : 3 consecutive failures

    Delivered --> Scheduled : onRefresh with new token

    Waiting --> Cancelled : cancel called
    Refreshing --> Cancelled : Generation mismatch

    note right of Cancelled : Generation counter invalidates\nin flight refreshes after cancel
```

### Bridge Transport Versions

```mermaid
flowchart TD
    Init["Initialize transport"] --> TryV2{"V2 available?"}

    TryV2 -->|yes| V2["V2 Transport<br/>(WebSocket)"]
    TryV2 -->|no| V1["V1 Transport<br/>(stdin/stdout, deprecated)"]

    V2 --> WSConn["WebSocket connection<br/>+ JWT auth header"]
    WSConn --> Heartbeat["Heartbeat every 30s"]
    WSConn --> UUIDDedup["UUID-based dedup<br/>(echo prevention)"]
    WSConn --> Reconnect["Auto-reconnect<br/>(exponential backoff)"]

    V1 --> StdioConn["Line-delimited JSON<br/>on child process stdio"]

    subgraph "Hybrid Strategy"
        Hybrid["Try V2 first<br/>Fall back to V1"]
    end
```

### Bridge Configuration

```mermaid
classDiagram
    class BridgeConfig {
        +string dir
        +string machineName
        +string branch
        +string gitRepoUrl
        +number maxSessions
        +SpawnMode spawnMode
        +boolean sandbox
        +UUID bridgeId
        +UUID environmentId
        +string reuseEnvironmentId
        +string apiBaseUrl
        +string sessionIngressUrl
        +WorkerType workerType
    }

    class SpawnMode {
        <<enumeration>>
        single-session
        worktree
        same-dir
    }

    class WorkerType {
        <<enumeration>>
        claude_code
        claude_code_assistant
    }

    BridgeConfig --> SpawnMode
    BridgeConfig --> WorkerType
```

## Coordinator (Multi-Agent Orchestration)

The coordinator (`src/coordinator/`) enables multi-agent workflows where a coordinator agent dispatches work to specialized workers.

### Coordinator Architecture

```mermaid
flowchart TD
    subgraph "Coordinator Agent"
        CoordPrompt["Coordinator System Prompt"]
        AgentTool["Agent Tool<br/>(spawn workers)"]
        SendMsg["SendMessage Tool<br/>(continue workers)"]
        TaskStop["TaskStop Tool<br/>(kill workers)"]
    end

    subgraph "Worker Pool"
        W1["Worker 1<br/>(research)"]
        W2["Worker 2<br/>(implementation)"]
        W3["Worker 3<br/>(verification)"]
    end

    subgraph "Worker Capabilities"
        WTools["Standard tools:<br/>Bash, FileRead, FileEdit"]
        WMCP["MCP tools from<br/>configured servers"]
        WSkills["Project skills<br/>via Skill tool"]
    end

    subgraph "Communication"
        Mailbox["Mailbox<br/>(inter-agent messaging)"]
        Notifications["Task notifications<br/>(XML protocol)"]
    end

    CoordPrompt --> AgentTool
    AgentTool --> W1 & W2 & W3
    SendMsg --> Mailbox --> W1 & W2 & W3
    W1 & W2 & W3 --> WTools & WMCP & WSkills
    W1 & W2 & W3 --> Notifications --> CoordPrompt
    TaskStop --> W1 & W2 & W3
```

### Coordinator Flow

```mermaid
sequenceDiagram
    participant User as User
    participant Coord as Coordinator
    participant Agent as Agent Tool
    participant W1 as Worker 1
    participant W2 as Worker 2

    User->>Coord: "Refactor the auth system"
    Coord->>Coord: Plan work distribution

    par Spawn Workers
        Coord->>Agent: Agent(description, subagent_type: "worker", prompt)
        Agent->>W1: Spawn research worker
        Coord->>Agent: Agent(description, subagent_type: "worker", prompt)
        Agent->>W2: Spawn implementation worker
    end

    W1->>W1: Research current auth code
    W1-->>Coord: <task-notification><br/>status: completed<br/>result: findings...

    Coord->>Coord: Review findings

    Coord->>W2: SendMessage("Implement based on findings...")
    W2->>W2: Implement changes

    W2-->>Coord: <task-notification><br/>status: completed<br/>result: changes made...

    Coord->>User: "Refactoring complete. Here's what changed..."
```

### Task Notification Protocol

```mermaid
classDiagram
    class TaskNotification {
        +string task-id
        +Status status
        +string summary
        +string result
        +Usage usage
    }

    class Status {
        <<enumeration>>
        completed
        failed
        killed
    }

    class Usage {
        +number total_tokens
        +number tool_uses
        +number duration_ms
    }

    TaskNotification --> Status
    TaskNotification --> Usage
```

### Coordinator Mode Activation

```mermaid
flowchart TD
    Check1{"feature('COORDINATOR_MODE')?"}
    Check2{"CLAUDE_CODE_COORDINATOR_MODE=1?"}
    Check3{"GrowthBook gate:<br/>tengu_ccr_bridge_multi_session?"}

    Check1 -->|yes| Activated
    Check2 -->|yes| Activated
    Check3 -->|yes| Activated
    Check1 -->|no| Check2
    Check2 -->|no| Check3
    Check3 -->|no| NormalMode["Normal Mode"]

    Activated["Coordinator Mode"] --> InjectPrompt["Inject coordinator<br/>system prompt"]
    InjectPrompt --> ConfigureTools["Configure tools:<br/>Agent, SendMessage, TaskStop"]
    ConfigureTools --> SetContext["Set worker context:<br/>tools, MCP servers, skills"]
```

## Skills System

The skills system (`src/skills/`) provides reusable workflows that can be invoked as slash commands or by the model.

### Skills Architecture

```mermaid
flowchart TD
    subgraph "Skill Sources"
        Bundled["Bundled Skills<br/>(built into CLI)"]
        Disk["Disk Skills<br/>(~/.claude/skills/)"]
        Plugin["Plugin Skills<br/>(installed plugins)"]
        MCP_Skills["MCP Skills<br/>(auto-wrapped from servers)"]
        Dynamic["Dynamic Skills<br/>(discovered during file ops)"]
    end

    subgraph "Skill Registry"
        Register["registerBundledSkill()"]
        Load["loadSkillsDir()"]
        PluginLoad["getPluginSkills()"]
        MCPBuild["mcpSkillBuilders.ts"]
    end

    subgraph "Skill Definition"
        Name["name: string"]
        Desc["description: string"]
        Aliases["aliases: string[]"]
        AllowedTools["allowedTools: string[]"]
        Model["model: string (override)"]
        Prompt["getPromptForCommand()"]
        Context["context: 'inline'|'fork'"]
        Agent["agent: string (spawn as worker)"]
    end

    subgraph "Invocation"
        SlashCmd["/skill-name args"]
        ModelInvoke["SkillTool(name, args)"]
    end

    Bundled --> Register
    Disk --> Load
    Plugin --> PluginLoad
    MCP_Skills --> MCPBuild
    Dynamic --> Register

    Register & Load & PluginLoad & MCPBuild --> Name & Desc & Aliases & AllowedTools & Model & Prompt & Context & Agent

    SlashCmd --> Prompt
    ModelInvoke --> Prompt

    Prompt --> Expand["Expand to ContentBlockParam[]"]
    Expand --> UserMsg["Send as user message"]
```

### Skill Loading Flow

```mermaid
sequenceDiagram
    participant REPL as REPL
    participant Loader as Skill Loader
    participant Bundled as Bundled Skills
    participant Disk as Disk Scanner
    participant Plugins as Plugin System

    REPL->>Loader: getCommands(cwd)

    par Load All Sources
        Loader->>Bundled: getBundledSkills()
        Bundled-->>Loader: BundledSkillDefinition[]

        Loader->>Disk: getSkillDirCommands(cwd)
        Disk->>Disk: Scan ~/.claude/skills/
        Disk->>Disk: Parse frontmatter
        Disk-->>Loader: Command[]

        Loader->>Plugins: getPluginSkills()
        Plugins-->>Loader: Command[]
    end

    Loader->>Loader: getDynamicSkills()
    Loader->>Loader: Merge + deduplicate
    Loader->>Loader: Filter by availability
    Loader-->>REPL: Command[] (all skills)
```

### Skill Execution

```mermaid
flowchart TD
    Invoke["User: /skill-name args<br/>or Model: SkillTool()"]

    Invoke --> Find["Find skill by name/alias"]
    Find --> CheckEnabled{"isEnabled()?"}

    CheckEnabled -->|no| Unavailable["Skill unavailable"]
    CheckEnabled -->|yes| GetPrompt["getPromptForCommand(args, context)"]

    GetPrompt --> Expand["Expand to ContentBlockParam[]"]
    Expand --> ContextCheck{"context type?"}

    ContextCheck -->|inline| Inline["Insert as user message<br/>in current conversation"]
    ContextCheck -->|fork| Fork["Fork as separate<br/>sub-conversation"]

    Inline --> ToolCheck{"allowedTools<br/>specified?"}
    ToolCheck -->|yes| Restrict["Restrict available tools"]
    ToolCheck -->|no| AllTools["All tools available"]

    Fork --> AgentCheck{"agent specified?"}
    AgentCheck -->|yes| SpawnWorker["Spawn as worker agent"]
    AgentCheck -->|no| SubConv["Run in sub-conversation"]

    Restrict & AllTools --> ModelProcess["Model processes expanded prompt"]
    SpawnWorker --> ModelProcess
    SubConv --> ModelProcess
```

## Swarm System

For distributed agent work, the swarm system enables leader-worker collaboration:

```mermaid
flowchart TD
    subgraph "Leader Agent"
        Leader["Coordinator/Leader"]
        PermDecision["Permission decision handler"]
        MailboxIn["Incoming mailbox"]
    end

    subgraph "Worker Agents"
        W1_2["Worker 1"]
        W2_2["Worker 2"]
        W3_2["Worker 3"]
    end

    subgraph "Communication"
        Mail["Mailbox system<br/>(inter-process)"]
        PermSync["Permission sync<br/>(swarm/permissionSync.ts)"]
    end

    W1_2 -->|"Permission request"| Mail
    W2_2 -->|"Permission request"| Mail
    W3_2 -->|"Permission request"| Mail

    Mail --> MailboxIn --> PermDecision

    PermDecision -->|"Allow/Deny"| Mail

    Mail -->|"Response"| W1_2
    Mail -->|"Response"| W2_2
    Mail -->|"Response"| W3_2
```

### Swarm Permission Flow

```mermaid
sequenceDiagram
    participant Worker as Worker Agent
    participant Classifier as Bash Classifier
    participant Mailbox as Mailbox
    participant Leader as Leader Agent
    participant UI as Leader UI

    Worker->>Worker: Tool needs permission

    Worker->>Classifier: Try auto-approval
    alt Classifier approves
        Classifier-->>Worker: Allow
        Worker->>Worker: Execute tool
    else Classifier uncertain
        Worker->>Worker: Register response callback
        Worker->>Mailbox: Send permission request
        Worker->>Worker: Show "waiting for approval"

        Mailbox->>Leader: Deliver request
        Leader->>UI: Show permission dialog
        UI-->>Leader: User decision

        Leader->>Mailbox: Send response (allow/deny)
        Mailbox->>Worker: Deliver response

        alt Approved
            Worker->>Worker: Execute tool
        else Denied
            Worker->>Worker: Skip tool
        end
    end
```
