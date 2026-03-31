# Query Engine and API Client

## QueryEngine Overview

The `QueryEngine` (`src/QueryEngine.ts`) is the central orchestrator for conversations with Claude. One instance exists per conversation session, owning message state, usage tracking, and the agentic tool-call loop.

```mermaid
classDiagram
    class QueryEngine {
        -Message[] mutableMessages
        -NonNullableUsage totalUsage
        -SDKPermissionDenial[] permissionDenials
        -Set~string~ discoveredSkillNames
        -Set~string~ loadedNestedMemoryPaths
        -QueryEngineConfig config
        +submitMessage(prompt, options) AsyncGenerator~SDKMessage~
        -processUserInput(input) Message[]
        -query(messages, options) AsyncGenerator
    }

    class QueryEngineConfig {
        +string cwd
        +Tools tools
        +Command[] commands
        +MCPServerConnection[] mcpClients
        +AgentDefinition[] agents
        +CanUseTool canUseTool
        +string customSystemPrompt
        +string appendSystemPrompt
        +ThinkingConfig thinkingConfig
        +number maxTurns
        +number maxBudgetUsd
        +JSONSchema jsonSchema
        +Function snipReplay
    }

    class NonNullableUsage {
        +number input_tokens
        +number cache_creation_input_tokens
        +number cache_read_input_tokens
        +number output_tokens
        +ServerToolUse server_tool_use
        +string service_tier
        +CacheCreation cache_creation
        +string inference_geo
        +number iterations
        +string speed
    }

    QueryEngine --> QueryEngineConfig
    QueryEngine --> NonNullableUsage
```

## Message Flow

```mermaid
sequenceDiagram
    participant User as User Input
    participant QE as QueryEngine
    participant Process as processUserInput()
    participant Query as query() Loop
    participant API as Claude API
    participant Tools as Tool Executor
    participant Result as Result Message

    User->>QE: submitMessage(prompt, options)

    QE->>Process: processUserInput(prompt)
    Process-->>QE: Message[] (with attachments)

    QE-->>User: yield system_init message

    QE->>Query: query(messages, options)

    loop Agentic Loop
        Query->>Query: Pre-process messages<br/>(compact, snip, microcompact)
        Query->>API: queryModelWithStreaming()
        API-->>Query: Stream events (text, tool_use, thinking)

        alt Tool calls present
            Query->>Tools: runTools(toolUseBlocks)
            Tools-->>Query: Tool results (UserMessage[])
            Query->>Query: Append results to messages
            Note over Query: Continue loop
        else No tool calls (end_turn)
            Query-->>QE: Terminal result
        end
    end

    QE-->>User: yield result message<br/>(usage, cost, errors)
```

## API Client Architecture

The API client (`src/services/api/client.ts`) supports multiple authentication providers:

```mermaid
flowchart TD
    subgraph "getAnthropicClient()"
        AuthCheck{"Authentication<br/>Method?"}

        AuthCheck -->|ANTHROPIC_API_KEY| DirectAPI["Direct API Client<br/>(api.anthropic.com)"]
        AuthCheck -->|OAuth Tokens| ClaudeAI["Claude.ai OAuth<br/>(with subscriber check)"]
        AuthCheck -->|AWS Credentials| Bedrock["AWS Bedrock Client<br/>(region override for Haiku)"]
        AuthCheck -->|Foundry Key| Foundry["Azure Foundry Client<br/>(DefaultAzureCredential)"]
        AuthCheck -->|Google Auth| Vertex["Vertex AI Client<br/>(project ID fallback)"]
    end

    subgraph "Client Configuration"
        Config["maxRetries: 10<br/>timeout: 600s<br/>dangerouslyAllowBrowser: true"]
        Headers["x-app: cli<br/>User-Agent: custom<br/>X-Claude-Code-Session-Id: UUID<br/>x-anthropic-additional-protection"]
        Proxy["Proxy via getProxyFetchOptions()"]
    end

    DirectAPI --> Config
    ClaudeAI --> Config
    Bedrock --> Config
    Foundry --> Config
    Vertex --> Config
    Config --> Headers
    Config --> Proxy
```

## Streaming Implementation

```mermaid
sequenceDiagram
    participant QM as queryModel()
    participant API as Claude API
    participant Stream as Stream Handler
    participant Usage as Usage Tracker

    QM->>API: messages.create(stream: true)

    API-->>Stream: message_start (initial usage snapshot)
    Stream->>Usage: Reset currentMessageUsage

    loop Content Streaming
        API-->>Stream: content_block_start (text/tool_use/thinking)

        loop Token Streaming
            API-->>Stream: content_block_delta (text/thinking tokens)
            Stream-->>QM: yield StreamEvent
        end

        API-->>Stream: content_block_stop
    end

    API-->>Stream: message_delta (usage update, stop_reason)
    Stream->>Usage: updateUsage(currentMessageUsage, delta)

    API-->>Stream: message_stop
    Usage->>Usage: accumulateUsage(totalUsage, currentMessageUsage)

    Stream-->>QM: yield AssistantMessage
```

## Tool Call Loop State Machine

```mermaid
stateDiagram-v2
    [*] --> PreProcess : Start iteration

    PreProcess --> CompactHistory : Messages ready
    CompactHistory --> MicroCompact : History compacted
    MicroCompact --> ContextCollapse : Micro-compact done
    ContextCollapse --> APICall : Context optimized

    APICall --> StreamResponse : API streaming

    StreamResponse --> ExtractTools : stop_reason = tool_use
    StreamResponse --> BuildResult : stop_reason = end_turn
    StreamResponse --> HandleError : API error

    ExtractTools --> PermissionCheck : ToolUseBlock[] extracted

    PermissionCheck --> ExecuteTools : All tools permitted
    PermissionCheck --> DenyTool : Permission denied

    DenyTool --> AppendDenial : Denial recorded
    AppendDenial --> PreProcess : Continue with denial message

    ExecuteTools --> PartitionTools : Tools ready

    state PartitionTools {
        [*] --> ClassifyTool
        ClassifyTool --> ReadOnlyBatch : concurrency safe and read only
        ClassifyTool --> WriteBatch : not read only
        ReadOnlyBatch --> ConcurrentExec : max 10 parallel
        WriteBatch --> SerialExec : one at a time
    }

    PartitionTools --> CollectResults : All tools complete
    CollectResults --> ApplyBudget : Results collected
    ApplyBudget --> AppendResults : Budget applied
    AppendResults --> PreProcess : Continue loop

    HandleError --> MaxOutputRecovery : max output tokens
    HandleError --> ReactiveCompact : prompt too long
    HandleError --> RetryWithBackoff : Transient 429 or 529
    HandleError --> FatalError : Non retryable

    MaxOutputRecovery --> PreProcess : Retry with budget override
    ReactiveCompact --> PreProcess : After compaction
    RetryWithBackoff --> APICall : After delay

    BuildResult --> [*] : Return Terminal
    FatalError --> [*] : Return error result
```

## Retry Logic

The retry system (`src/services/api/withRetry.ts`) handles transient failures with provider-specific strategies:

```mermaid
flowchart TD
    APICall["API Call"] --> Error{"Error Type?"}

    Error -->|"401/403<br/>(Auth)"| AuthRetry["Clear auth cache<br/>Refresh tokens<br/>Get fresh client"]
    Error -->|"429/529<br/>(Capacity)"| CapacityCheck{"Retry-After<br/>duration?"}
    Error -->|"ECONNRESET<br/>EPIPE"| ConnRetry["Disable keep-alive<br/>Get fresh client"]
    Error -->|"AWS/GCP<br/>Credential"| CredRetry["Force credential<br/>refresh"]
    Error -->|"Other"| FatalCheck{"Retries<br/>remaining?"}

    CapacityCheck -->|"< 2s"| ShortRetry["Sleep + retry<br/>same model"]
    CapacityCheck -->|"> 2s"| LongRetry["Fast-mode cooldown<br/>Use fallback model"]

    AuthRetry --> RetryAttempt
    ShortRetry --> RetryAttempt
    LongRetry --> RetryAttempt
    ConnRetry --> RetryAttempt
    CredRetry --> RetryAttempt

    RetryAttempt{"Attempt ≤ 10?"} -->|yes| YieldError["Yield SystemAPIErrorMessage<br/>(attempt #, delay)"]
    RetryAttempt -->|no| StreamFallback{"Streaming<br/>fallback?"}

    YieldError --> APICall

    StreamFallback -->|yes| NonStream["Non-streaming request<br/>timeout: 120-300s"]
    StreamFallback -->|no| ThrowError["Throw CannotRetryError"]

    FatalCheck -->|yes| RetryAttempt
    FatalCheck -->|no| ThrowError

    NonStream --> Success{"Success?"}
    Success -->|yes| Return["Return message"]
    Success -->|no| ThrowError
```

### Retry by Query Source

| Query Source | Retries 529? | Background? | Notes |
|-------------|-------------|-------------|-------|
| `repl_main_thread*` | Yes | No | User-blocking, full retry |
| `sdk` | Yes | No | Programmatic, full retry |
| `agent:*` | Yes | No | Sub-agent work |
| `verification_agent` | Yes | No | Critical verification |
| `auto_mode` | Yes | No | Security classifier |
| `summaries` | No | Yes | Non-critical background |
| `titles` | No | Yes | Non-critical background |
| `extract_memories` | No | Yes | Non-critical background |

## API Request Construction

```mermaid
flowchart LR
    subgraph "Request Parameters"
        Model["model: resolved model ID"]
        Messages["messages: conversation history"]
        System["system: assembled prompt"]
        Tools["tools: JSON schemas"]
        MaxTokens["max_tokens: capped default"]
        Betas["betas: merged beta headers"]
        Metadata["metadata: user_id composite"]
        Thinking["thinking: adaptive/enabled/disabled"]
        Effort["effort: high/medium/low"]
        Extra["extra_body: from env var"]
    end

    subgraph "Beta Headers"
        B1["interleaved-thinking-2025-05-14"]
        B2["context-1m-2025-08-07"]
        B3["context-management-2025-06-27"]
        B4["structured-outputs-2025-12-15"]
        B5["web-search-2025-03-05"]
        B6["advanced-tool-use-2025-11-20"]
        B7["effort-2025-11-24"]
        B8["task-budgets-2026-03-13"]
        B9["fast-mode-2026-02-01"]
        B10["prompt-caching-scope-2026-01-05"]
    end

    Model --> Request["BetaMessageStreamParams"]
    Messages --> Request
    System --> Request
    Tools --> Request
    MaxTokens --> Request
    Betas --> Request
    Metadata --> Request
    Thinking --> Request
    Effort --> Request
    Extra --> Request

    B1 --> Betas
    B2 --> Betas
    B3 --> Betas
    B4 --> Betas
    B5 --> Betas
    B6 --> Betas
    B7 --> Betas
    B8 --> Betas
    B9 --> Betas
    B10 --> Betas
```

## Thinking Mode Configuration

```mermaid
flowchart TD
    Check{"Model supports<br/>thinking?"}

    Check -->|"Claude 4.6+<br/>(Opus/Sonnet)"| Adaptive["type: 'adaptive'<br/>Model chooses when to think"]
    Check -->|"Claude 4+<br/>(1P/Foundry)"| Enabled["type: 'enabled'<br/>budgetTokens: configurable"]
    Check -->|"Older models"| Disabled["type: 'disabled'<br/>No thinking"]

    subgraph "Configuration Sources"
        EnvVar["MAX_THINKING_TOKENS env var"]
        Settings["Settings JSON"]
        Default["Default: enabled"]
    end

    EnvVar --> Enabled
    Settings --> Enabled
    Default --> Enabled
```

## Token Counting and Cost Tracking

```mermaid
flowchart TD
    subgraph "Per-Message Tracking"
        Start["message_start event"]
        Delta["message_delta events"]
        Stop["message_stop event"]

        Start --> Reset["Reset currentMessageUsage<br/>= EMPTY_USAGE"]
        Delta --> Update["updateUsage(current, delta)<br/>input, output, cache tokens"]
        Stop --> Accumulate["accumulateUsage(total, current)"]
    end

    subgraph "Accumulated Fields"
        InputTokens["input_tokens"]
        OutputTokens["output_tokens"]
        CacheCreate["cache_creation_input_tokens"]
        CacheRead["cache_read_input_tokens"]
        ServerTool["server_tool_use<br/>(web_search, web_fetch)"]
        CacheBreakdown["cache_creation breakdown<br/>(1h vs 5m TTL)"]
        ServiceTier["service_tier"]
        InferGeo["inference_geo"]
        Iterations["iterations count"]
    end

    Accumulate --> InputTokens
    Accumulate --> OutputTokens
    Accumulate --> CacheCreate
    Accumulate --> CacheRead
    Accumulate --> ServerTool
    Accumulate --> CacheBreakdown
    Accumulate --> ServiceTier
    Accumulate --> InferGeo
    Accumulate --> Iterations
```

## Message Types

```mermaid
classDiagram
    class Message {
        <<union>>
    }

    class AssistantMessage {
        +type: "assistant"
        +uuid: string
        +timestamp: number
        +message.role: "assistant"
        +message.content: ContentBlock[]
        +requestId: string
        +apiError: string
        +stop_reason: string
        +usage: NonNullableUsage
    }

    class UserMessage {
        +type: "user"
        +uuid: string
        +timestamp: number
        +message.role: "user"
        +message.content: string | ContentBlockParam[]
        +toolUseResult: string
        +sourceToolAssistantUUID: string
        +isMeta: boolean
    }

    class SystemAPIErrorMessage {
        +type: "system"
        +subtype: "api_error"
        +attempt: number
        +maxRetries: number
        +retryDelayMs: number
        +errorClassification: string
    }

    class CompactBoundary {
        +type: "system"
        +subtype: "compact_boundary"
        +summary: string
    }

    class ToolUseSummary {
        +type: "tool_use_summary"
        +toolUses: ToolUseInfo[]
    }

    class AttachmentMessage {
        +type: "attachment"
        +toolResults: ToolResult[]
        +maxTurnsReached: boolean
        +queuedCommands: string[]
    }

    Message <|-- AssistantMessage
    Message <|-- UserMessage
    Message <|-- SystemAPIErrorMessage
    Message <|-- CompactBoundary
    Message <|-- ToolUseSummary
    Message <|-- AttachmentMessage
```

## Context Compaction

When the conversation approaches the context window limit, automatic compaction kicks in:

```mermaid
flowchart TD
    Monitor["Token usage monitor"] --> ThresholdCheck{"usage ≥<br/>effective_window - 13K?"}

    ThresholdCheck -->|no| Continue["Continue normally"]
    ThresholdCheck -->|yes| CircuitBreaker{"3 consecutive<br/>failures?"}

    CircuitBreaker -->|yes| Stop["Stop compaction<br/>(avoid API spam)"]
    CircuitBreaker -->|no| Compact["compactConversation()"]

    Compact --> GroupMessages["Group messages by API round"]
    GroupMessages --> BuildPrompt["Build compaction prompt"]
    BuildPrompt --> CallAPI["Call Claude API for summary"]
    CallAPI --> InsertBoundary["Insert compact_boundary<br/>message with summary"]
    InsertBoundary --> DiscardOld["Discard messages before boundary"]

    subgraph "Compaction Strategies"
        S1["autoCompact - Token threshold trigger"]
        S2["reactiveCompact - prompt_too_long recovery"]
        S3["microCompact - Token-conservative summaries"]
        S4["snipCompact - Snippet-based compaction"]
        S5["sessionMemoryCompact - Memory-backed"]
    end
```
