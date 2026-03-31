# Data Flow and State Machines

## End-to-End Data Flow

### Happy Path: User Prompt to Response

```mermaid
sequenceDiagram
    participant User as User
    participant Ink as Ink Renderer
    participant REPL as REPL Component
    participant QE as QueryEngine
    participant Query as Query Loop
    participant API as Claude API
    participant Tools as Tool Executor
    participant State as AppState

    User->>Ink: Keyboard input
    Ink->>REPL: Parsed keypress events
    REPL->>REPL: Buffer input in PromptInput

    User->>REPL: Press Enter
    REPL->>REPL: Parse input

    alt Slash Command
        REPL->>REPL: Find and dispatch command
    else User Prompt
        REPL->>QE: submitMessage
        QE->>QE: processUserInput
        Note over QE: Handle attachments and memory files

        QE->>State: Update messages
        QE-->>REPL: yield system_init

        QE->>Query: start query

        loop Agentic Loop
            Query->>Query: Pre-process context
            Query->>API: queryModelWithStreaming

            loop Streaming
                API-->>Query: content_block_delta
                Query-->>QE: yield stream_event
                QE-->>REPL: yield stream_event
                REPL->>Ink: Update AssistantTextMessage
                Ink->>User: Render streamed text
            end

            API-->>Query: message_stop

            alt Tool calls present
                Query->>Tools: runTools

                loop Per Tool
                    Tools->>Tools: Schema validation
                    Tools->>Tools: Permission check
                    Tools->>Tools: Execute tool
                    Tools-->>REPL: Progress updates
                    REPL->>Ink: Update tool progress UI
                end

                Tools-->>Query: Tool results
                Query->>Query: Append results and continue
            else End turn
                Query-->>QE: Terminal result
            end
        end

        QE-->>REPL: yield result message
        REPL->>State: Update usage and messages
        REPL->>Ink: Render final state
        Ink->>User: Display response
    end
```

### Error Recovery Flow

```mermaid
sequenceDiagram
    participant Query as Query Loop
    participant Retry as withRetry
    participant API as Claude API
    participant Compact as Compaction
    participant User as User

    Query->>API: queryModelWithStreaming
    API-->>Retry: 429 Too Many Requests

    alt Short retry after
        Retry->>Retry: Sleep then retry
        Retry->>API: Retry same model
    else Long retry after
        Retry->>Retry: Enter fast-mode cooldown
        Retry->>API: Retry with fallback model
    end

    API-->>Retry: 529 Server Overloaded
    Retry-->>Query: yield SystemAPIErrorMessage
    Query-->>User: Show retry indicator

    Retry->>API: Retry attempt 2 of 10
    API-->>Query: Success and stream begins

    Note over Query: Later in conversation

    Query->>API: queryModelWithStreaming
    API-->>Query: Error prompt_too_long

    Query->>Compact: reactiveCompact
    Compact->>Compact: Summarize old messages
    Compact-->>Query: Compacted conversation

    Query->>API: Retry with compacted context
    API-->>Query: Success
```

## Session Lifecycle State Machine

```mermaid
stateDiagram-v2
    [*] --> FastPathCheck : CLI invoked

    FastPathCheck --> FastPathExit : Fast path match
    FastPathCheck --> FullLoad : No fast path

    FastPathExit --> [*] : Output and exit

    FullLoad --> PreAction : Modules loaded
    PreAction --> InitPhase : preAction hook
    InitPhase --> SetupScreens : init complete
    SetupScreens --> ToolLoad : Trust and auth done
    ToolLoad --> Ready : Tools and MCP loaded

    Ready --> Interactive : launchRepl
    Ready --> Headless : print flag

    state Interactive {
        [*] --> WaitingForInput
        WaitingForInput --> ProcessingInput : User submits
        ProcessingInput --> PreProcessCtx : Start query
        PreProcessCtx --> CallingAPI : Context optimized
        CallingAPI --> StreamingResp : API streaming
        StreamingResp --> ExecTools : tool use detected
        StreamingResp --> RespComplete : end turn
        ExecTools --> PreProcessCtx : results appended
        RespComplete --> WaitingForInput : Back to input
        WaitingForInput --> RunSlashCmd : slash command
        RunSlashCmd --> WaitingForInput : Command complete
    }

    state Headless {
        [*] --> SingleQuery
        SingleQuery --> StreamOutput : Query started
        StreamOutput --> HeadlessDone : Output complete
    }

    Interactive --> Compacting : Auto compact triggered
    Compacting --> Interactive : Compaction complete

    Interactive --> PlanMode : enter plan
    PlanMode --> Interactive : ExitPlanMode

    Interactive --> Backgrounding : background session
    Backgrounding --> Interactive : attach

    Interactive --> Shutdown : exit or Ctrl C
    Headless --> Shutdown : Output complete

    state Shutdown {
        [*] --> FlushTelemetry
        FlushTelemetry --> ShutdownLSP
        ShutdownLSP --> CleanupTeams
        CleanupTeams --> ResetCursor
    }

    Shutdown --> [*] : Cleanup done
```

## Tool Execution State Machine

```mermaid
stateDiagram-v2
    [*] --> TE_Received : ToolUseBlock from API

    TE_Received --> TE_SchemaValidation : Parse input

    TE_SchemaValidation --> TE_SchemaFailed : Invalid input
    TE_SchemaValidation --> TE_InputValidation : Schema valid

    TE_SchemaFailed --> TE_ErrorResult : Return error

    TE_InputValidation --> TE_ValidationFailed : Validation fails
    TE_InputValidation --> TE_PermissionCheck : Input valid

    TE_ValidationFailed --> TE_ErrorResult : Return error

    TE_PermissionCheck --> TE_CheckDenyRules : canUseTool called
    TE_CheckDenyRules --> TE_Denied : Deny rule match
    TE_CheckDenyRules --> TE_ToolSpecific : No deny match
    TE_ToolSpecific --> TE_Denied : Tool denies
    TE_ToolSpecific --> TE_HookCheck : Tool allows
    TE_HookCheck --> TE_Denied : Hook denies
    TE_HookCheck --> TE_RuleCheck : Hooks pass
    TE_RuleCheck --> TE_Allowed : Allow rule match
    TE_RuleCheck --> TE_Denied : Deny rule match
    TE_RuleCheck --> TE_ModeCheck : No rule match
    TE_ModeCheck --> TE_UserPrompt : Default mode
    TE_ModeCheck --> TE_Allowed : Bypass mode
    TE_ModeCheck --> TE_ClassifierCheck : Auto mode
    TE_ClassifierCheck --> TE_Allowed : Safe
    TE_ClassifierCheck --> TE_UserPrompt : Uncertain
    TE_UserPrompt --> TE_Allowed : User approves
    TE_UserPrompt --> TE_Denied : User rejects

    TE_Denied --> TE_RejectedResult : Return rejection

    TE_Allowed --> TE_Running : Execute tool
    TE_Running --> TE_Progress : onProgress callback
    TE_Progress --> TE_Running : Continue
    TE_Running --> TE_Success : Completed
    TE_Running --> TE_Failed : Error

    TE_Success --> TE_ProcessResult : Map to result block
    TE_Failed --> TE_ErrorResult : Return error

    TE_ProcessResult --> TE_ApplyBudget : Check result size
    TE_ApplyBudget --> TE_ApplyContextMod : Apply contextModifier
    TE_ApplyContextMod --> [*] : Return result

    TE_ErrorResult --> [*] : Return error
    TE_RejectedResult --> [*] : Return rejection
```

## Permission Decision State Machine

```mermaid
stateDiagram-v2
    [*] --> PD_Evaluating : Tool invocation

    PD_Evaluating --> PD_ScanDenyRules : Start evaluation
    PD_ScanDenyRules --> PD_Denied : Deny rule matches
    PD_ScanDenyRules --> PD_Validate : No deny rule

    PD_Validate --> PD_Denied : Input invalid
    PD_Validate --> PD_ToolCheck : Input valid

    PD_ToolCheck --> PD_Allowed : Tool allows
    PD_ToolCheck --> PD_Denied : Tool denies
    PD_ToolCheck --> PD_RunHooks : Tool asks

    PD_RunHooks --> PD_Allowed : Hook allows
    PD_RunHooks --> PD_Denied : Hook denies
    PD_RunHooks --> PD_MatchRules : No hook match

    PD_MatchRules --> PD_Allowed : alwaysAllow rule
    PD_MatchRules --> PD_Denied : alwaysDeny rule
    PD_MatchRules --> PD_ShowDialog : alwaysAsk rule
    PD_MatchRules --> PD_CheckMode : No rule match

    PD_CheckMode --> PD_ShowDialog : Default mode
    PD_CheckMode --> PD_Allowed : Bypass mode
    PD_CheckMode --> PD_EditCheck : AcceptEdits mode
    PD_CheckMode --> PD_Denied : DontAsk mode
    PD_CheckMode --> PD_RunClassifier : Auto mode

    PD_EditCheck --> PD_Allowed : Is file edit
    PD_EditCheck --> PD_ShowDialog : Not file edit

    PD_RunClassifier --> PD_Allowed : Classified safe
    PD_RunClassifier --> PD_ShowDialog : Uncertain

    PD_ShowDialog --> PD_Racing : Display dialog

    PD_Racing --> PD_Allowed : Hook wins race
    PD_Racing --> PD_Allowed : Classifier wins race
    PD_Racing --> PD_Allowed : Bridge wins race
    PD_Racing --> PD_Allowed : Channel wins race
    PD_Racing --> PD_UserAllows : User approves
    PD_Racing --> PD_UserDenies : User rejects
    PD_Racing --> PD_Denied : Abort signal

    PD_UserAllows --> PD_PersistCheck : Check persistence
    PD_PersistCheck --> PD_Allowed : Write rule to disk
    PD_UserDenies --> PD_Denied : Rejection recorded

    PD_Allowed --> [*] : Execute tool
    PD_Denied --> [*] : Skip tool
```

## MCP Connection State Machine

```mermaid
stateDiagram-v2
    [*] --> MC_Discovered : Config loaded

    MC_Discovered --> MC_Connecting : Initialize

    MC_Connecting --> MC_CreateTransport : Select transport type
    MC_CreateTransport --> MC_StdioTransport : stdio
    MC_CreateTransport --> MC_SSETransport : sse
    MC_CreateTransport --> MC_HTTPTransport : http
    MC_CreateTransport --> MC_WSTransport : ws
    MC_CreateTransport --> MC_SDKTransport : sdk

    MC_StdioTransport --> MC_Connected : Transport established
    MC_SSETransport --> MC_Connected : Transport established
    MC_HTTPTransport --> MC_Connected : Transport established
    MC_WSTransport --> MC_Connected : Transport established
    MC_SDKTransport --> MC_Connected : Transport established

    MC_Connecting --> MC_NeedsAuth : OAuth required
    MC_Connecting --> MC_ConnFailed : Connection error

    MC_NeedsAuth --> MC_UserApproves : Prompt user
    MC_UserApproves --> MC_OpenBrowser : Start OAuth
    MC_OpenBrowser --> MC_WaitCallback : Wait for redirect
    MC_WaitCallback --> MC_TokenExchange : Code received
    MC_TokenExchange --> MC_Connected : Auth success
    MC_TokenExchange --> MC_ConnFailed : Auth failure

    MC_Connected --> MC_ToolDiscovery : List tools
    MC_ToolDiscovery --> MC_Ready : Tools loaded
    MC_ToolDiscovery --> MC_ConnFailed : RPC error

    MC_Ready --> MC_ToolExec : Tool call
    MC_ToolExec --> MC_Ready : Result returned

    MC_Ready --> MC_Disconnected : Connection lost
    MC_Disconnected --> MC_Backoff : Auto reconnect

    MC_Backoff --> MC_RetryConnect : After delay
    MC_RetryConnect --> MC_Connected : Reconnected
    MC_RetryConnect --> MC_Backoff : Retry failed
    MC_Backoff --> MC_Disabled : 5 attempts exceeded

    MC_Ready --> MC_Disabled : User disables
    MC_Disabled --> MC_Connecting : User re-enables
    MC_ConnFailed --> MC_Backoff : Retry eligible
    MC_ConnFailed --> MC_Disabled : Non retryable
```

## Bridge Session State Machine

```mermaid
stateDiagram-v2
    [*] --> BS_Registering : bridgeMain called

    BS_Registering --> BS_Registered : Register environment
    BS_Registering --> BS_Fatal : Auth or not found error

    BS_Registered --> BS_LongPoll : Start work polling

    BS_LongPoll --> BS_WorkReceived : WorkResponse received
    BS_LongPoll --> BS_NoWork : Timeout
    BS_NoWork --> BS_PollBackoff : Increase delay
    BS_PollBackoff --> BS_LongPoll : Resume polling

    BS_WorkReceived --> BS_Acknowledging : Acknowledge work

    BS_Acknowledging --> BS_SessionWork : Session type
    BS_Acknowledging --> BS_HealthCheck : Healthcheck type

    BS_HealthCheck --> BS_LongPoll : Respond OK

    BS_SessionWork --> BS_DecodingJWT : Decode secret

    BS_DecodingJWT --> BS_WSConnecting : JWT valid
    BS_WSConnecting --> BS_WSConnected : WebSocket established
    BS_WSConnected --> BS_Active : Ready for messages

    BS_Active --> BS_Processing : Inbound message
    BS_Processing --> BS_Responding : Generate response
    BS_Responding --> BS_Active : Response sent
    BS_Active --> BS_PermRequest : Control request
    BS_PermRequest --> BS_Active : Response sent

    BS_Active --> BS_SessionEnd : Work completed
    BS_Active --> BS_WSDisconnected : Connection lost

    BS_WSDisconnected --> BS_WSReconnecting : Auto reconnect
    BS_WSReconnecting --> BS_Active : Reconnected
    BS_WSReconnecting --> BS_SessionEnd : Max retries

    BS_SessionEnd --> BS_LongPoll : Resume polling

    BS_Fatal --> [*] : Exit with error
```

## Context Compaction State Machine

```mermaid
stateDiagram-v2
    [*] --> CC_BelowThreshold : Token usage tracking

    CC_BelowThreshold --> CC_Warning : Usage exceeds warning threshold
    CC_Warning --> CC_Critical : Usage exceeds error threshold
    CC_Critical --> CC_TriggerCompact : Usage exceeds compact threshold

    CC_TriggerCompact --> CC_CheckFailures : Check circuit breaker
    CC_CheckFailures --> CC_SelectStrategy : Fewer than 3 failures
    CC_CheckFailures --> CC_Disabled : 3 or more failures

    CC_Disabled --> [*] : Stop monitoring

    CC_SelectStrategy --> CC_AutoCompact : Token threshold trigger
    CC_SelectStrategy --> CC_ReactiveCompact : Prompt too long recovery
    CC_SelectStrategy --> CC_MicroCompact : Token conservative mode
    CC_SelectStrategy --> CC_SnipCompact : Targeted segments

    CC_AutoCompact --> CC_GroupMessages : Begin compaction
    CC_ReactiveCompact --> CC_GroupMessages : Begin compaction
    CC_MicroCompact --> CC_GroupMessages : Begin compaction
    CC_SnipCompact --> CC_GroupMessages : Begin compaction

    CC_GroupMessages --> CC_BuildPrompt : Messages grouped
    CC_BuildPrompt --> CC_CallAPI : Prompt ready
    CC_CallAPI --> CC_Success : Summary generated
    CC_CallAPI --> CC_Failure : API error

    CC_Success --> CC_InsertBoundary : Insert compact boundary
    CC_InsertBoundary --> CC_DiscardOld : Remove old messages
    CC_DiscardOld --> CC_Cleanup : Post compact cleanup
    CC_Cleanup --> CC_BelowThreshold : Reset counters

    CC_Failure --> CC_IncrementFailures : Increment failure count
    CC_IncrementFailures --> CC_BelowThreshold : Try again later
```

## Complete Module Interaction Map

```mermaid
graph TB
    subgraph "User Interaction"
        Terminal["Terminal IO"]
        IDE["IDE Extension"]
        WebApp["claude.ai"]
        SDKAPI["SDK API"]
    end

    subgraph "Entry and Routing"
        CLIEntry["cli.tsx"]
        MainEntry["main.tsx"]
        BridgeEntry["bridge/"]
    end

    subgraph "Core Loop"
        QEngine["QueryEngine"]
        QueryLoop["query.ts"]
        ToolOrch["toolOrchestration"]
    end

    subgraph "AI Provider"
        APIClient["api/client.ts"]
        ClaudeAPI["api/claude.ts"]
        RetryLogic["api/withRetry.ts"]
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
        HookSys["Hook System"]
        BashClassifier["Bash Classifier"]
    end

    subgraph "State and Config"
        AppStore["AppState Zustand"]
        BootState["bootstrap/state.ts"]
        SettingsSys["Settings System"]
        ClaudeMDLoader["CLAUDE.md Loader"]
    end

    subgraph "Services"
        MCPSvc["MCP Service"]
        LSPSvc["LSP Service"]
        OAuthSvc["OAuth Service"]
        CompactSvc["Compact Service"]
        AnalyticsSvc["Analytics"]
    end

    subgraph "UI"
        REPLComp["REPL"]
        UIComponents["Components"]
        InkRenderer["Ink Renderer"]
    end

    Terminal --> CLIEntry --> MainEntry --> REPLComp
    IDE --> BridgeEntry --> QEngine
    WebApp --> BridgeEntry
    SDKAPI --> QEngine

    REPLComp --> QEngine --> QueryLoop
    QueryLoop --> ClaudeAPI --> APIClient --> RetryLogic
    QueryLoop --> ToolOrch --> ToolReg

    ToolReg --> BashT
    ToolReg --> FileT
    ToolReg --> AgentT
    ToolReg --> MCPT
    ToolReg --> SkillT
    ToolOrch --> PermSys
    PermSys --> HookSys
    PermSys --> BashClassifier

    REPLComp --> AppStore
    MainEntry --> BootState
    MainEntry --> SettingsSys
    MainEntry --> ClaudeMDLoader
    QEngine --> CompactSvc
    MCPT --> MCPSvc
    MainEntry --> LSPSvc
    MainEntry --> OAuthSvc
    MainEntry --> AnalyticsSvc

    REPLComp --> UIComponents --> InkRenderer --> Terminal

    AgentT -.-> QEngine
```
