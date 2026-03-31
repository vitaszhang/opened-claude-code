# Tools and Commands

## Tool Type System

Every tool in Claude Code implements the `Tool` interface defined in `src/Tool.ts`:

```mermaid
classDiagram
    class Tool~Input Output Progress~ {
        +string name
        +string[] aliases
        +string searchHint
        +ZodSchema inputSchema
        +JSONSchema inputJSONSchema
        +ZodSchema outputSchema
        +number maxResultSizeChars
        +boolean strict
        +boolean shouldDefer
        +boolean alwaysLoad
        +call(input, context, canUseTool, msg, onProgress) ToolResult
        +description(input, options) string
        +prompt(options) string
        +checkPermissions(input, context) PermissionResult
        +validateInput(input, context) ValidationResult
        +isConcurrencySafe(input) boolean
        +isReadOnly(input) boolean
        +isDestructive(input) boolean
        +isEnabled() boolean
        +renderToolUseMessage(input, options) ReactNode
        +renderToolResultMessage(content, progress, options) ReactNode
        +renderToolUseErrorMessage(result, options) ReactNode
        +renderToolUseRejectedMessage(input, options) ReactNode
        +renderGroupedToolUse(toolUses, options) ReactNode
    }

    class ToolDef~Input Output Progress~ {
        <<partial>>
        +Optional isEnabled
        +Optional isConcurrencySafe
        +Optional isReadOnly
        +Optional isDestructive
        +Optional checkPermissions
    }

    class ToolResult~Output~ {
        +Output data
        +Message[] newMessages
        +Function contextModifier
        +MCPMeta mcpMeta
    }

    class ToolUseContext {
        +Tools tools
        +Command[] commands
        +MCPServerConnection[] mcpClients
        +AgentDefinition[] agents
        +AbortController abortController
        +FileStateCache readFileState
        +Function getAppState
        +Function setAppState
        +Function canUseTool
        +Function setToolJSX
        +Message[] messages
    }

    class PermissionResult {
        +behavior: allow|deny|ask
        +Input updatedInput
        +string decisionReason
    }

    Tool --> ToolResult : returns
    Tool --> ToolUseContext : receives
    Tool --> PermissionResult : permission check
    ToolDef --> Tool : buildTool()
```

## Tool Registry

Tools are registered via `src/tools.ts` with conditional loading based on feature flags, environment, and user type:

```mermaid
flowchart TD
    subgraph "Registration Flow"
        AllBase["getAllBaseTools()"] --> FeatureGate{"Feature flags<br/>+ USER_TYPE?"}
        FeatureGate --> FilterEnabled["Filter isEnabled()"]
        FilterEnabled --> FilterDeny["filterToolsByDenyRules()"]
        FilterDeny --> GetTools["getTools(permissionContext)"]
    end

    subgraph "Assembly"
        GetTools --> AssemblePool["assembleToolPool()"]
        MCPTools["MCP Server Tools"] --> AssemblePool
        AssemblePool --> Dedup["Deduplicate<br/>(built-in wins)"]
        Dedup --> Sort["Sort for prompt<br/>cache stability"]
        Sort --> FinalTools["Final Tool List"]
    end

    subgraph "Tool Search Integration"
        FinalTools --> Threshold{"Tool count<br/>> ~50?"}
        Threshold -->|yes| DeferTools["Deferred tools:<br/>defer_loading: true"]
        Threshold -->|no| AllLoaded["All tools loaded<br/>in prompt"]
        DeferTools --> ToolSearch["ToolSearchTool<br/>finds deferred tools"]
    end
```

## Complete Tool Inventory

### Core Tools (Always Available)

| Tool | Read-Only | Concurrent | Description |
|------|-----------|-----------|-------------|
| **BashTool** | No | No | Execute shell commands |
| **FileReadTool** | Yes | Yes | Read files (images, PDFs, notebooks) |
| **FileEditTool** | No | No | Diff-based file editing |
| **FileWriteTool** | No | No | Create/overwrite files |
| **GlobTool** | Yes | Yes | File pattern matching |
| **GrepTool** | Yes | Yes | Content search (ripgrep) |
| **AgentTool** | No | No | Spawn sub-agents |
| **WebFetchTool** | Yes | Yes | Fetch web pages |
| **WebSearchTool** | Yes | Yes | Web search (Bing) |
| **NotebookEditTool** | No | No | Jupyter cell editing |
| **SkillTool** | No | No | Invoke skills/workflows |
| **AskUserQuestionTool** | No | No | Prompt user for input |
| **ToolSearchTool** | Yes | Yes | Discover deferred tools |

### Task Management Tools

| Tool | Description |
|------|-------------|
| **TaskCreateTool** | Create tasks |
| **TaskGetTool** | Retrieve task details |
| **TaskUpdateTool** | Update task status |
| **TaskListTool** | List all tasks |
| **TaskStopTool** | Stop running tasks |
| **TaskOutputTool** | Get task output |
| **TodoWriteTool** | Update todo panel |

### Mode Tools

| Tool | Description |
|------|-------------|
| **EnterPlanModeTool** | Enter plan mode |
| **ExitPlanModeV2Tool** | Exit plan mode |
| **EnterWorktreeTool** | Create git worktree |
| **ExitWorktreeTool** | Exit worktree |

### MCP Integration Tools

| Tool | Description |
|------|-------------|
| **MCPTool** | Call MCP server tools |
| **ListMcpResourcesTool** | List MCP resources |
| **ReadMcpResourceTool** | Read MCP resources |
| **McpAuthTool** | Handle MCP auth |

### Feature-Gated Tools

| Tool | Feature Gate | Description |
|------|-------------|-------------|
| **SendMessageTool** | (lazy) | Message other agents |
| **CronCreateTool** | AGENT_TRIGGERS | Create scheduled tasks |
| **CronDeleteTool** | AGENT_TRIGGERS | Delete scheduled tasks |
| **CronListTool** | AGENT_TRIGGERS | List scheduled tasks |
| **RemoteTriggerTool** | AGENT_TRIGGERS_REMOTE | Remote execution |
| **SleepTool** | PROACTIVE/KAIROS | Wait/sleep |
| **WebBrowserTool** | WEB_BROWSER_TOOL | Browser automation |
| **LSPTool** | (plugins) | Language server protocol |
| **PowerShellTool** | (enabled) | PowerShell execution |
| **WorkflowTool** | WORKFLOW_SCRIPTS | Workflow execution |
| **SnipTool** | HISTORY_SNIP | History snipping |
| **MonitorTool** | MONITOR_TOOL | Resource monitoring |

### Internal/Ant-Only Tools

| Tool | Description |
|------|-------------|
| **REPLTool** | REPL/VM execution |
| **ConfigTool** | Modify settings.json |
| **TeamCreateTool** | Create agent teams |
| **TeamDeleteTool** | Delete agent teams |
| **TungstenTool** | Internal integration |
| **SuggestBackgroundPRTool** | Background PR suggestions |

## Tool Execution Lifecycle

```mermaid
sequenceDiagram
    participant Model as Claude API
    participant Extract as Tool Extractor
    participant Schema as Schema Validator
    participant Partition as Partitioner
    participant Perm as Permission System
    participant Exec as Tool Executor
    participant Result as Result Processor

    Model->>Extract: AssistantMessage with tool_use blocks
    Extract->>Extract: Extract ToolUseBlock[]

    loop For each tool call
        Extract->>Schema: tool.inputSchema.safeParse(input)

        alt Schema valid
            Schema->>Partition: Valid tool call
        else Schema invalid
            Schema-->>Result: Schema error result
        end
    end

    Partition->>Partition: Classify tools
    Note over Partition: isReadOnly + isConcurrencySafe?

    alt Read-only batch
        Partition->>Perm: Check permissions (parallel)
        Perm->>Exec: runToolsConcurrently(batch, max=10)
    else Write/exec tool
        Partition->>Perm: Check permission (serial)
        Perm->>Exec: runToolsSerially(tool)
    end

    Exec->>Exec: tool.validateInput(input, context)
    Exec->>Exec: tool.call(input, context, ...)

    Exec->>Result: ToolResult with data
    Result->>Result: applyToolResultBudget()
    Result->>Result: Apply contextModifier (if any)
    Result-->>Model: ToolResultBlockParam
```

## Tool Concurrency Model

```mermaid
flowchart TD
    ToolCalls["Tool Use Blocks from Model"] --> Partition["partitionToolCalls()"]

    Partition --> Classify{"For each tool:<br/>isReadOnly?<br/>isConcurrencySafe?"}

    Classify -->|"Both true"| ReadBatch["Add to Read-Only Batch"]
    Classify -->|"Either false"| WriteSingle["Separate Write Batch"]

    ReadBatch --> NextTool{"Next tool?"}
    NextTool -->|"Read-only"| ReadBatch
    NextTool -->|"Write/exec"| FlushRead["Flush: Execute read batch<br/>(max 10 concurrent)"]
    FlushRead --> WriteSingle

    WriteSingle --> ExecSerial["Execute single tool<br/>(serial)"]

    ExecSerial --> ApplyCtx["Apply contextModifier"]
    ApplyCtx --> NextBatch{"More batches?"}

    NextBatch -->|yes| Classify
    NextBatch -->|no| CollectResults["Collect all tool results"]
```

## Slash Command System

Commands are registered in `src/commands.ts` and come in three types:

```mermaid
classDiagram
    class Command {
        <<interface>>
        +string name
        +string[] aliases
        +string description
        +CommandType type
        +CommandSource loadedFrom
        +boolean userInvocable
        +boolean disableModelInvocation
    }

    class PromptCommand {
        +type: "prompt"
        +getPromptForCommand(args, ctx) ContentBlockParam[]
    }

    class LocalCommand {
        +type: "local"
        +load() LocalCommandModule
    }

    class LocalJSXCommand {
        +type: "local-jsx"
        +load() LocalJSXCommandModule
    }

    Command <|-- PromptCommand
    Command <|-- LocalCommand
    Command <|-- LocalJSXCommand

    note for PromptCommand "Expands to text for model<br/>e.g., /commit, /review"
    note for LocalCommand "Runs locally, not sent to model<br/>e.g., /clear, /exit, /help"
    note for LocalJSXCommand "Renders interactive Ink UI<br/>e.g., /settings, /mcp"
```

### Command Categories

```mermaid
mindmap
    root((Slash Commands))
        File/Repository
            /addDir
            /branch
            /commit
            /diff
            /files
            /rewind
        Session
            /clear
            /compact
            /context
            /copy
            /exit
            /session
            /status
            /summary
        Configuration
            /config
            /hooks
            /keybindings
            /permissions
            /theme
            /vim
        Development
            /cost
            /doctor
            /ide
            /model
            /output-style
            /tasks
            /upgrade
        Information
            /advisor
            /brief
            /help
            /memory
            /usage
            /version
        Auth
            /login
            /logout
            /privacy-settings
        Extensions
            /agents
            /mcp
            /plugin
            /skills
```

### Command Discovery Flow

```mermaid
flowchart TD
    LoadAll["loadAllCommands(cwd)"] --> Sources

    subgraph Sources ["Command Sources"]
        Bundled["getBundledSkills()"]
        BuiltinPlugin["getBuiltinPluginSkillCommands()"]
        SkillDir["getSkillDirCommands(cwd)"]
        Workflow["getWorkflowCommands(cwd)"]
        PluginCmd["getPluginCommands()"]
        PluginSkill["getPluginSkills()"]
        Dynamic["getDynamicSkills()"]
        Builtin["COMMANDS()"]
    end

    Sources --> Merge["Merge all sources"]
    Merge --> Availability["meetsAvailabilityRequirement()<br/>(auth checks)"]
    Availability --> Enabled["isCommandEnabled()<br/>(feature flags)"]
    Enabled --> Dedup["Deduplicate by name"]
    Dedup --> CommandList["Final Command List"]

    CommandList --> SlashCommands["Slash commands<br/>(user-invocable)"]
    CommandList --> SkillToolCommands["Skill tool commands<br/>(model-invocable)"]
```

### Command-Tool Interaction

```mermaid
flowchart LR
    subgraph "REPL Input"
        UserInput["User types /command args"]
    end

    subgraph "Command Dispatch"
        Parse["Parse command name"]
        Find["findCommand(name, commands)"]
        TypeCheck{"Command type?"}
    end

    subgraph "Execution"
        Local["load().call(args, context)<br/>Returns text/compact/skip"]
        LocalJSX["load().call(onDone, context, args)<br/>Renders React UI"]
        Prompt["getPromptForCommand(args, context)<br/>Expands to ContentBlockParam[]"]
    end

    subgraph "Result"
        LocalResult["Display in REPL<br/>(not sent to model)"]
        JSXResult["Render interactive UI"]
        PromptResult["Send expanded text<br/>as user message to model"]
    end

    UserInput --> Parse --> Find --> TypeCheck
    TypeCheck -->|local| Local --> LocalResult
    TypeCheck -->|local-jsx| LocalJSX --> JSXResult
    TypeCheck -->|prompt| Prompt --> PromptResult
```
