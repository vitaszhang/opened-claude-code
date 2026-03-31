# Configuration and Context Assembly

## Settings Hierarchy

Claude Code loads configuration from 5+ sources with strict priority ordering:

```mermaid
flowchart TD
    subgraph "Priority: Highest"
        Policy["policySettings<br/>/etc/claude-code/managed-settings.json<br/>+ /etc/claude-code/managed-settings.d/*.json<br/>(org-managed, immutable)"]
    end

    subgraph "Priority: High"
        Flag["flagSettings<br/>--settings CLI flag<br/>(session override)"]
    end

    subgraph "Priority: Medium"
        Local["localSettings<br/>.claude.local/settings.json<br/>(gitignored, personal)"]
    end

    subgraph "Priority: Low"
        Project["projectSettings<br/>.claude/settings.json<br/>(git-tracked, shared)"]
    end

    subgraph "Priority: Lowest"
        User["userSettings<br/>~/.claude/settings.json<br/>(global personal)"]
    end

    Policy --> Flag --> Local --> Project --> User

    subgraph "Merge Process"
        Parse["Parse each source"]
        Validate["Validate against<br/>SettingsSchema (Zod)"]
        Merge["Custom merge function<br/>(priority-based override)"]
        Cache["Cache parsed result"]
    end

    Policy & Flag & Local & Project & User --> Parse --> Validate --> Merge --> Cache
```

## CLAUDE.md Loading System

Memory/instruction files are loaded from multiple scopes with `@include` directive support:

```mermaid
flowchart TD
    subgraph "CLAUDE.md Sources"
        Managed["Managed CLAUDE.md<br/>(enterprise policy)"]
        UserClaude["~/.claude/CLAUDE.md<br/>(user global)"]
        ProjectClaude[".claude/CLAUDE.md<br/>(project root)"]
        ProjectRules[".claude/rules/*.md<br/>(project rules)"]
        LocalClaude["CLAUDE.local.md<br/>(gitignored, personal)"]
        AutoMemory["Auto-generated memory<br/>(optional)"]
        TeamMemory["Team memory<br/>(optional feature)"]
    end

    subgraph "Loading Process"
        Traverse["Traverse from cwd<br/>up to filesystem root"]
        Scan["Scan for CLAUDE.md,<br/>.claude/CLAUDE.md,<br/>.claude/rules/*.md"]
        ParseFrontmatter["Parse frontmatter<br/>(conditional rules)"]
        ExpandIncludes["Expand @include<br/>directives"]
        Dedup["Deduplicate by inode"]
        MaxSize["Cap at 40K chars/file"]
    end

    Managed & UserClaude & ProjectClaude & ProjectRules & LocalClaude & AutoMemory & TeamMemory --> Traverse
    Traverse --> Scan --> ParseFrontmatter --> ExpandIncludes --> Dedup --> MaxSize

    MaxSize --> MemoryFiles["MemoryFile[]<br/>(path, type, tokens, source)"]
```

### Memory File Types

```mermaid
classDiagram
    class MemoryFile {
        +string path
        +MemoryType type
        +number tokenCount
        +string source
        +string content
    }

    class MemoryType {
        <<enumeration>>
        User
        Project
        Local
        Managed
        AutoMem
        TeamMem
    }

    MemoryFile --> MemoryType
```

## System Prompt Assembly

The system prompt is built from multiple layers with a priority hierarchy:

```mermaid
flowchart TD
    subgraph "Prompt Priority"
        Override{"Override<br/>system prompt?"}
        Override -->|yes| UseOverride["Use override<br/>(loop mode)"]
        Override -->|no| CoordCheck{"Coordinator mode<br/>and no agent?"}
        CoordCheck -->|yes| UseCoord["Use coordinator prompt"]
        CoordCheck -->|no| AgentCheck{"Agent defined?"}
        AgentCheck -->|yes| ProactiveCheck{"Proactive mode?"}
        ProactiveCheck -->|yes| AppendAgent["Default + agent prompt"]
        ProactiveCheck -->|no| UseAgent["Agent prompt only"]
        AgentCheck -->|no| CustomCheck{"Custom prompt?"}
        CustomCheck -->|yes| UseCustom["Use custom prompt"]
        CustomCheck -->|no| UseDefault["Use default prompt"]
    end

    subgraph "Append"
        Append["+ appendSystemPrompt<br/>(always appended if specified)"]
    end

    UseOverride & UseCoord & AppendAgent & UseAgent & UseCustom & UseDefault --> Append
```

### Default System Prompt Components

```mermaid
flowchart TD
    subgraph "System Prompt Assembly (prompts.ts)"
        Prefix["CLI Prefix<br/>'You are Claude Code...'"]
        ToolDesc["Tool Descriptions<br/>(JSON schemas + prompts)"]
        MemoryInstr["Memory/Instructions<br/>(CLAUDE.md files)"]
        Agents["Available Agents<br/>(agent definitions)"]
        Skills["Available Skills<br/>(slash commands)"]
        MCPTools["MCP Server Tools<br/>(external tools)"]
        ModelInfo["Model Info<br/>(name, capabilities)"]
        DateInfo["Current Date"]
        GitStatus["Git Status<br/>(branch, recent commits)"]
    end

    Prefix --> ToolDesc --> MemoryInstr --> Agents --> Skills --> MCPTools --> ModelInfo --> DateInfo --> GitStatus

    GitStatus --> FinalPrompt["Final System Prompt"]

    subgraph "Cache Control"
        CachedSections["Cached sections<br/>(memoized until /clear)"]
        VolatileSections["Volatile sections<br/>(breaks cache on change)"]
    end

    CachedSections --> FinalPrompt
    VolatileSections --> FinalPrompt
```

## Context Window Management

```mermaid
flowchart TD
    subgraph "Window Size Determination"
        Default["Default: 200,000 tokens"]
        OneMillion["1M context flag:<br/>model suffix '[1m]'"]
        EnvOverride["CLAUDE_CODE_MAX_CONTEXT_TOKENS"]
        BetaHeader["Beta: context-1m-2025-08-07"]
    end

    Default --> WindowSize["Effective window size"]
    OneMillion --> WindowSize
    EnvOverride --> WindowSize
    BetaHeader --> WindowSize

    subgraph "Budget Allocation"
        WindowSize --> Reserved["Reserved for output<br/>(model max_output_tokens)"]
        WindowSize --> Effective["Effective input budget"]
        Effective --> SystemPrompt2["System prompt tokens"]
        Effective --> Messages2["Conversation messages"]
        Effective --> Tools2["Tool schemas"]
        Effective --> Buffer["13K buffer for auto-compact"]
    end

    subgraph "Monitoring"
        TokenCount["analyzeContext.ts<br/>Token counting per category"]
        Warning["Warning threshold<br/>(approaching limit)"]
        AutoCompact["Auto-compact trigger<br/>(at threshold)"]
    end

    Effective --> TokenCount --> Warning --> AutoCompact
```

## SDK Schemas (Zod v4)

The SDK types are defined as Zod schemas in `src/entrypoints/sdk/` and auto-generated to TypeScript:

```mermaid
classDiagram
    class CoreSchemas {
        +ModelUsageSchema
        +OutputFormatSchema
        +ApiKeySourceSchema
        +ConfigScopeSchema
        +SdkBetaSchema
        +ThinkingConfigSchema
        +McpStdioServerConfigSchema
        +McpSSEServerConfigSchema
        +McpHttpServerConfigSchema
        +McpSdkServerConfigSchema
        +McpServerStatusSchema
        +PermissionUpdateSchema
        +HookEventSchema
        +SDKMessageSchema
        +SDKStreamlinedTextMessageSchema
        +SlashCommandSchema
        +AgentDefinitionSchema
    }

    class ControlSchemas {
        +SDKControlInitializeRequestSchema
        +SDKControlPermissionRequestSchema
        +SDKControlSetPermissionModeRequestSchema
    }

    class GeneratedTypes {
        +coreTypes.generated.ts (5,127 LOC)
        <<auto-generated from Zod schemas>>
    }

    CoreSchemas --> GeneratedTypes : scripts/generate-sdk-types.ts
    ControlSchemas --> GeneratedTypes
```

### SDK Message Schema

```mermaid
classDiagram
    class SDKMessage {
        <<union>>
    }

    class UserMessage_SDK {
        +type: "user"
        +content: string | ContentBlock[]
    }

    class AssistantMessage_SDK {
        +type: "assistant"
        +content: ContentBlock[]
        +stop_reason: string
        +usage: ModelUsage
    }

    class SystemMessage_SDK {
        +type: "system"
        +subtype: string
        +content: string
    }

    class ToolUseSummary_SDK {
        +type: "tool_use_summary"
        +toolUses: ToolUseInfo[]
    }

    class ResultMessage_SDK {
        +type: "result"
        +usage: ModelUsage
        +total_cost_usd: number
        +stop_reason: string
        +errors: Error[]
    }

    SDKMessage <|-- UserMessage_SDK
    SDKMessage <|-- AssistantMessage_SDK
    SDKMessage <|-- SystemMessage_SDK
    SDKMessage <|-- ToolUseSummary_SDK
    SDKMessage <|-- ResultMessage_SDK
```

## Hook System

Hooks allow external commands to run in response to Claude Code events:

```mermaid
flowchart TD
    subgraph "Hook Events"
        PreTool["pre-tool-use<br/>Before tool execution"]
        PostTool["post-tool-use<br/>After tool execution"]
        PermReq["permission-request<br/>When permission needed"]
        SessionStart["session-start<br/>On session begin"]
        Setup["setup<br/>On first run"]
        Stop["stop<br/>On session end"]
        PreSampling["pre-sampling<br/>Before API call"]
        PostSampling["post-sampling<br/>After API response"]
    end

    subgraph "Hook Configuration (settings.json)"
        HookDef["hooks: {<br/>  'pre-tool-use': [{<br/>    matcher: { tool_name: 'Bash' },<br/>    command: 'validate-bash.sh'<br/>  }]<br/>}"]
    end

    subgraph "Hook Execution"
        Match["Match event to hook definitions"]
        Spawn["Spawn hook command"]
        IO["Provide JSON input via stdin"]
        Parse["Parse JSON output from stdout"]
        Apply["Apply hook result<br/>(allow, deny, modify)"]
    end

    PreTool & PostTool & PermReq & SessionStart & Setup & Stop & PreSampling & PostSampling --> Match
    HookDef --> Match
    Match --> Spawn --> IO --> Parse --> Apply
```

### Hook Input/Output Flow

```mermaid
sequenceDiagram
    participant Engine as Query Engine
    participant Hooks as Hook System
    participant Cmd as Hook Command

    Engine->>Hooks: Event: pre-tool-use (BashTool, input)

    Hooks->>Hooks: Match matchers against event
    Hooks->>Cmd: Spawn command<br/>stdin: JSON {tool_name, input}

    Cmd->>Cmd: Validate/transform

    alt Hook approves
        Cmd-->>Hooks: stdout: {decision: "allow", updatedInput: {...}}
        Hooks-->>Engine: Allow with modified input
    else Hook denies
        Cmd-->>Hooks: stdout: {decision: "deny", reason: "..."}
        Hooks-->>Engine: Deny with reason
    else Hook errors
        Cmd-->>Hooks: Exit code != 0
        Hooks-->>Engine: Treat as deny (safe default)
    end
```

## Utility Module Categories

```mermaid
mindmap
    root((src/utils/ ~341 files))
        API & Networking (26)
            api.ts
            http.ts
            proxy.ts
            apiPreconnect.ts
            mtls.ts
            auth.ts
        Context & Memory (18)
            analyzeContext.ts
            context.ts
            claudemd.ts
            attachments.ts
            agentContext.ts
        Configuration (32)
            config.ts
            settings/
            markdownConfigLoader.ts
            env.ts
        File & Git (28)
            git/
            git.ts
            file.ts
            fsOperations.ts
            path.ts
        System Prompt (24)
            systemPrompt.ts
            model/
            tokens.ts
            thinking.ts
        Bash & Shell (18)
            bash/parser.ts
            bash/commands.ts
            bash/shellCompletion.ts
        Permissions (24)
            permissions/
            bashClassifier.ts
            yoloClassifier.ts
        Session & State (22)
            sessionStart.ts
            sessionState.ts
            sessionStorage.ts
        Hooks & Events (18)
            hooks.ts
            attributionHooks.ts
        Caching (16)
            completionCache.ts
            memoize.ts
            CircularBuffer.ts
        Display (14)
            ansiToPng.ts
            format.ts
            theme.ts
        Search (12)
            ripgrep.ts
            glob.ts
            transcriptSearch.ts
        Telemetry (10)
            telemetry/
```
