# Data Flow and State Machines

## End-to-End Data Flow

### Happy Path: User Prompt to Response

```mermaid
sequenceDiagram
    participant User as User (Terminal)
    participant Ink as Ink Renderer
    participant REPL as REPL Component
    participant QE as QueryEngine
    participant Query as query.ts Loop
    participant API as Claude API
    participant Tools as Tool Executor
    participant State as AppState

    User->>Ink: Keyboard input
    Ink->>REPL: Parsed keypress events
    REPL->>REPL: Buffer input in PromptInput

    User->>REPL: Press Enter
    REPL->>REPL: Parse input (command or prompt)

    alt Slash Command
        REPL->>REPL: findCommand(name)
        REPL->>REPL: Dispatch by type (local/jsx/prompt)
    else User Prompt
        REPL->>QE: submitMessage(prompt)
        QE->>QE: processUserInput(prompt)
        Note over QE: Handle attachments,<br/>CLAUDE.md, memory files

        QE->>State: Update messages[]
        QE-->>REPL: yield system_init

        QE->>Query: query(messages)

        loop Agentic Loop
            Query->>Query: Pre-process (compact, snip)
            Query->>API: queryModelWithStreaming()

            loop Streaming
                API-->>Query: content_block_delta (text)
                Query-->>QE: yield stream_event
                QE-->>REPL: yield stream_event
                REPL->>Ink: Update AssistantTextMessage
                Ink->>User: Render streamed text
            end

            API-->>Query: message_stop

            alt Tool calls present
                Query->>Tools: runTools(toolUseBlocks)

                loop Per Tool
                    Tools->>Tools: Schema validation
                    Tools->>Tools: Permission check
                    Tools->>Tools: tool.call()
                    Tools-->>REPL: Progress updates
                    REPL->>Ink: Update tool progress UI
                end

                Tools-->>Query: Tool results
                Query->>Query: Append results, continue loop
            else end_turn
                Query-->>QE: Terminal result
            end
        end

        QE-->>REPL: yield result message
        REPL->>State: Update usage, messages
        REPL->>Ink: Render final state
        Ink->>User: Display response
    end
```

### Error Recovery Flow

```mermaid
sequenceDiagram
    participant Query as query.ts
    participant Retry as withRetry
    participant API as Claude API
    participant Compact as Compaction
    participant User as User

    Query->>API: queryModelWithStreaming()
    API-->>Retry: 429 Too Many Requests

    alt Short retry-after (< 2s)
        Retry->>Retry: Sleep(retry-after)
        Retry->>API: Retry same model
    else Long retry-after
        Retry->>Retry: Enter fast-mode cooldown
        Retry->>API: Retry with fallback model
    end

    API-->>Retry: 529 Server Overloaded
    Retry-->>Query: yield SystemAPIErrorMessage
    Query-->>User: Show retry indicator

    Retry->>API: Retry attempt 2/10
    API-->>Query: Success (stream begins)

    Note over Query: Later in conversation...

    Query->>API: queryModelWithStreaming()
    API-->>Query: Error: prompt_too_long

    Query->>Compact: reactiveCompact()
    Compact->>Compact: Summarize old messages
    Compact-->>Query: Compacted conversation

    Query->>API: Retry with compacted context
    API-->>Query: Success
```

## Session Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> Initializing : CLI invoked

    state Initializing {
        [*] --> FastPathCheck
        FastPathCheck --> FastPathExit : Fast-path match
        FastPathCheck --> FullLoad : No fast-path
        FullLoad --> PreAction : Modules loaded
        PreAction --> Init : preAction hook
        Init --> SetupScreens : init() complete
        SetupScreens --> ToolLoad : Trust + auth done
        ToolLoad --> Ready : Tools + MCP loaded
    }

    FastPathExit --> [*] : Output and exit

    Ready --> Interactive : launchRepl()
    Ready --> Headless : -p flag

    state Interactive {
        [*] --> WaitingForInput

        WaitingForInput --> ProcessingInput : User submits
        ProcessingInput --> QueryLoop : Start query

        state QueryLoop {
            [*] --> PreProcess
            PreProcess --> APICall
            APICall --> Streaming
            Streaming --> ToolExecution : tool_use
            Streaming --> ResponseComplete : end_turn
            ToolExecution --> PreProcess : results appended
        }

        ResponseComplete --> WaitingForInput
        WaitingForInput --> SlashCommand : /command
        SlashCommand --> WaitingForInput : Command complete
    end

    state Headless {
        [*] --> SingleQuery
        SingleQuery --> StreamOutput
        StreamOutput --> Complete
    }

    Interactive --> Compacting : Auto-compact triggered
    Compacting --> Interactive : Compaction complete

    Interactive --> PlanMode : /plan
    PlanMode --> Interactive : ExitPlanMode

    Interactive --> Backgrounding : --bg / Ctrl+Z
    Backgrounding --> Interactive : attach

    Interactive --> GracefulShutdown : exit / Ctrl+C+D
    Headless --> GracefulShutdown : Output complete
    Complete --> GracefulShutdown

    GracefulShutdown --> [*] : Cleanup done

    state GracefulShutdown {
        [*] --> FlushTelemetry
        FlushTelemetry --> ShutdownLSP
        ShutdownLSP --> CleanupTeams
        CleanupTeams --> ResetCursor
        ResetCursor --> [*]
    }
```

## Tool Execution State Machine

```mermaid
stateDiagram-v2
    [*] --> Received : ToolUseBlock from API

    Received --> SchemaValidation : Parse input

    SchemaValidation --> SchemaFailed : Invalid input
    SchemaValidation --> InputValidation : Schema valid

    SchemaFailed --> ErrorResult : Return error

    InputValidation --> ValidationFailed : validateInput() fails
    InputValidation --> PermissionCheck : Input valid

    ValidationFailed --> ErrorResult : Return error

    PermissionCheck --> Checking : canUseTool()

    state Checking {
        [*] --> DenyRules
        DenyRules --> ToolSpecific : No deny match
        DenyRules --> Denied : Deny rule match
        ToolSpecific --> HookCheck : Tool allows
        ToolSpecific --> Denied : Tool denies
        HookCheck --> RuleCheck : Hooks pass
        HookCheck --> Denied : Hook denies
        RuleCheck --> ModeCheck : No rule match
        RuleCheck --> Allowed : Allow rule match
        RuleCheck --> Denied : Deny rule match
        ModeCheck --> UserPrompt : Default mode
        ModeCheck --> Allowed : Bypass mode
        ModeCheck --> ClassifierCheck : Auto mode
        ClassifierCheck --> Allowed : Safe
        ClassifierCheck --> UserPrompt : Uncertain
        UserPrompt --> Allowed : User approves
        UserPrompt --> Denied : User rejects
    end

    Denied --> RejectedResult : Return rejection

    Allowed --> Executing : tool.call()

    state Executing {
        [*] --> Running
        Running --> Progress : onProgress callback
        Progress --> Running
        Running --> Success : Completed
        Running --> Failed : Error
    }

    Success --> ProcessResult : Map to ToolResultBlockParam
    Failed --> ErrorResult

    ProcessResult --> ApplyBudget : Check result size
    ApplyBudget --> ApplyContextMod : Apply contextModifier
    ApplyContextMod --> [*] : Return result

    ErrorResult --> [*] : Return error
    RejectedResult --> [*] : Return rejection
```

## Permission Decision State Machine

```mermaid
stateDiagram-v2
    [*] --> Evaluating : Tool invocation

    Evaluating --> DenyRuleCheck

    state DenyRuleCheck {
        [*] --> ScanRules
        ScanRules --> RuleMatched : Deny rule matches
        ScanRules --> NoMatch : No deny rule
    }

    RuleMatched --> Denied

    NoMatch --> InputValidation2

    state InputValidation2 {
        [*] --> Validate
        Validate --> Valid
        Validate --> Invalid
    }

    Invalid --> Denied

    Valid --> ToolPermissionCheck

    state ToolPermissionCheck {
        [*] --> ToolCheck
        ToolCheck --> ToolAllows : behavior: allow
        ToolCheck --> ToolDenies : behavior: deny
        ToolCheck --> ToolAsks : behavior: ask
    }

    ToolDenies --> Denied
    ToolAllows --> Allowed

    ToolAsks --> HookExecution

    state HookExecution {
        [*] --> RunHooks
        RunHooks --> HookResolves : Hook decides
        RunHooks --> HookPassthrough : No hook match
    }

    HookResolves --> Allowed : Hook allows
    HookResolves --> Denied : Hook denies

    HookPassthrough --> RuleMatching

    state RuleMatching {
        [*] --> MatchRules
        MatchRules --> AllowRule : alwaysAllow
        MatchRules --> DenyRule : alwaysDeny
        MatchRules --> AskRule : alwaysAsk
        MatchRules --> NoRule : No match
    }

    AllowRule --> Allowed
    DenyRule --> Denied
    AskRule --> InteractiveDecision

    NoRule --> ModeBasedDecision

    state ModeBasedDecision {
        [*] --> CheckMode
        CheckMode --> InteractiveDecision : default
        CheckMode --> Allowed : bypassPermissions
        CheckMode --> EditCheck : acceptEdits
        CheckMode --> Denied : dontAsk
        CheckMode --> ClassifierDecision : auto
    }

    EditCheck --> Allowed : Is file edit
    EditCheck --> InteractiveDecision : Not file edit

    state ClassifierDecision {
        [*] --> RunClassifier
        RunClassifier --> Allowed : Classified safe
        RunClassifier --> InteractiveDecision : Uncertain
    }

    state InteractiveDecision {
        [*] --> ShowDialog
        ShowDialog --> Racing

        state Racing {
            [*] --> WaitForFirst
            WaitForFirst --> HookWins : Hook resolves
            WaitForFirst --> ClassifierWins : Classifier resolves
            WaitForFirst --> BridgeWins : Bridge resolves
            WaitForFirst --> ChannelWins : Channel resolves
            WaitForFirst --> UserWins : User responds
            WaitForFirst --> AbortWins : Abort signal
        }

        HookWins --> Allowed
        ClassifierWins --> Allowed
        BridgeWins --> Allowed
        ChannelWins --> Allowed

        UserWins --> UserAllows : Allow (once/always)
        UserWins --> UserDenies : Reject

        AbortWins --> Denied
    end

    UserAllows --> PersistCheck
    PersistCheck --> Allowed : Allow always → write to disk
    PersistCheck --> Allowed : Allow once → session only

    UserDenies --> Denied

    Allowed --> [*] : Execute tool
    Denied --> [*] : Skip tool
```

## MCP Connection State Machine

```mermaid
stateDiagram-v2
    [*] --> Discovered : Config loaded

    Discovered --> Connecting : initialize()

    Connecting --> TransportSetup

    state TransportSetup {
        [*] --> CreateTransport
        CreateTransport --> StdioTransport : type: stdio
        CreateTransport --> SSETransport : type: sse
        CreateTransport --> HTTPTransport : type: http
        CreateTransport --> WSTransport : type: ws
        CreateTransport --> SDKTransport : type: sdk
    }

    TransportSetup --> Connected : Transport established
    TransportSetup --> NeedsAuth : OAuth required
    TransportSetup --> ConnectionFailed : Error

    NeedsAuth --> OAuthFlow

    state OAuthFlow {
        [*] --> UserApproves
        UserApproves --> OpenBrowser
        OpenBrowser --> WaitForCallback
        WaitForCallback --> TokenExchange
        TokenExchange --> AuthSuccess
        TokenExchange --> AuthFailure
    }

    AuthSuccess --> Connected
    AuthFailure --> ConnectionFailed

    Connected --> ToolDiscovery : listTools()
    ToolDiscovery --> Ready : Tools loaded
    ToolDiscovery --> ConnectionFailed : RPC error

    Ready --> ToolExecution : Tool call
    ToolExecution --> Ready : Result

    Ready --> Disconnected : Connection lost
    Disconnected --> Reconnecting : Auto-reconnect

    state Reconnecting {
        [*] --> Backoff
        Backoff --> RetryConnect : After delay
        RetryConnect --> Connected : Success
        RetryConnect --> Backoff : Failure (count++)
        Backoff --> MaxRetries : 5 attempts
    }

    MaxRetries --> Disabled : Give up

    Ready --> Disabled : User disables
    Disabled --> Connecting : User re-enables
    ConnectionFailed --> Reconnecting : Retry eligible
    ConnectionFailed --> Disabled : Non-retryable
```

## Bridge Session State Machine

```mermaid
stateDiagram-v2
    [*] --> Registering : bridgeMain() called

    Registering --> Registered : POST /environments/bridge
    Registering --> Fatal : 401/403/404

    Registered --> Polling : Start work polling

    state Polling {
        [*] --> LongPoll
        LongPoll --> WorkReceived : WorkResponse
        LongPoll --> NoWork : Timeout
        NoWork --> Backoff : Increase delay
        Backoff --> LongPoll : After delay
        note right of Backoff : 2s → 2m → 10m
    }

    WorkReceived --> Acknowledging : PATCH /work/{id}

    state "Session Type" as SType {
        Acknowledging --> SessionWork : type: session
        Acknowledging --> HealthCheck : type: healthcheck
    }

    HealthCheck --> Polling : Respond OK

    SessionWork --> DecodingJWT : Decode secret

    DecodingJWT --> WebSocketConnect : JWT valid

    state WebSocketConnect {
        [*] --> Connecting2
        Connecting2 --> Connected2 : WS established
        Connected2 --> Active : Ready for messages
    }

    state Active {
        [*] --> Idle
        Idle --> ProcessingMessage : Inbound message
        ProcessingMessage --> Responding : Generate response
        Responding --> Idle : Response sent

        Idle --> PermissionRequest : Control request
        PermissionRequest --> Idle : Response sent

        note right of Active : Heartbeat every 30s<br/>UUID dedup active
    }

    Active --> SessionEnd : Work completed
    Active --> Disconnected2 : Connection lost

    Disconnected2 --> Reconnecting2 : Auto-reconnect
    Reconnecting2 --> Active : Reconnected
    Reconnecting2 --> SessionEnd : Max retries

    SessionEnd --> Polling : Resume polling

    Fatal --> [*] : Exit with error
```

## Context Compaction State Machine

```mermaid
stateDiagram-v2
    [*] --> Monitoring : Token usage tracking

    state Monitoring {
        [*] --> BelowThreshold
        BelowThreshold --> Warning : usage > warning_threshold
        Warning --> Critical : usage > error_threshold
        Critical --> TriggerCompact : usage ≥ auto_compact_threshold
    }

    TriggerCompact --> CircuitBreakerCheck

    state CircuitBreakerCheck {
        [*] --> CheckFailures
        CheckFailures --> Proceed : failures < 3
        CheckFailures --> Disabled : failures ≥ 3
    }

    Disabled --> [*] : Stop monitoring

    Proceed --> SelectStrategy

    state SelectStrategy {
        [*] --> ChooseStrategy
        ChooseStrategy --> AutoCompact : Token threshold
        ChooseStrategy --> ReactiveCompact : prompt_too_long error
        ChooseStrategy --> MicroCompact : Token-conservative
        ChooseStrategy --> SnipCompact : Targeted segments
    }

    SelectStrategy --> Compacting

    state Compacting {
        [*] --> GroupMessages
        GroupMessages --> BuildPrompt
        BuildPrompt --> CallAPI
        CallAPI --> Success : Summary generated
        CallAPI --> Failure : API error
    }

    Success --> InsertBoundary : compact_boundary message
    InsertBoundary --> DiscardOld : Remove old messages
    DiscardOld --> Cleanup : postCompactCleanup()
    Cleanup --> Monitoring : Reset counters

    Failure --> IncrementFailures : failures++
    IncrementFailures --> Monitoring : Try again later
```

## Complete Module Interaction Map

```mermaid
graph TB
    subgraph "User Interaction"
        Terminal["Terminal I/O"]
        IDE["IDE Extension"]
        WebApp["claude.ai"]
        SDK_API["SDK API"]
    end

    subgraph "Entry & Routing"
        CLI["cli.tsx"]
        Main["main.tsx"]
        Bridge["bridge/"]
    end

    subgraph "Core Loop"
        QE["QueryEngine"]
        Query["query.ts"]
        ToolOrch["toolOrchestration"]
    end

    subgraph "AI Provider"
        APIClient["api/client.ts"]
        Claude["api/claude.ts"]
        Retry["api/withRetry.ts"]
    end

    subgraph "Tool Ecosystem"
        ToolReg["tools.ts"]
        BashT["BashTool"]
        FileT["File Tools"]
        AgentT["AgentTool"]
        MCPT["MCP Tools"]
        SkillT["SkillTool"]
    end

    subgraph "Safety"
        PermSys["Permission System"]
        Hooks["Hook System"]
        Classifier["Bash Classifier"]
    end

    subgraph "State & Config"
        AppState2["AppState (Zustand)"]
        BootState["bootstrap/state.ts"]
        Settings["Settings System"]
        ClaudeMD["CLAUDE.md Loader"]
    end

    subgraph "Services"
        MCP_Svc["MCP Service"]
        LSP_Svc["LSP Service"]
        OAuth_Svc["OAuth Service"]
        Compact_Svc["Compact Service"]
        Analytics_Svc["Analytics"]
    end

    subgraph "UI"
        REPL2["REPL"]
        Components4["Components"]
        InkRender["Ink Renderer"]
    end

    Terminal --> CLI --> Main --> REPL2
    IDE --> Bridge --> QE
    WebApp --> Bridge
    SDK_API --> QE

    REPL2 --> QE --> Query
    Query --> Claude --> APIClient --> Retry
    Query --> ToolOrch --> ToolReg

    ToolReg --> BashT & FileT & AgentT & MCPT & SkillT
    ToolOrch --> PermSys
    PermSys --> Hooks & Classifier

    REPL2 --> AppState2
    Main --> BootState & Settings & ClaudeMD
    QE --> Compact_Svc
    MCPT --> MCP_Svc
    Main --> LSP_Svc & OAuth_Svc & Analytics_Svc

    REPL2 --> Components4 --> InkRender --> Terminal

    AgentT -.->|spawns| QE
```
