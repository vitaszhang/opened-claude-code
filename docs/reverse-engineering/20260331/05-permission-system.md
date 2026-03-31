# Permission System

## Architecture Overview

The permission system (`src/hooks/toolPermission/`) is the security backbone of Claude Code. Every tool invocation passes through multi-layered permission checks before execution.

```mermaid
graph TB
    subgraph "Permission Layers"
        L1["Layer 1: Deny Rules<br/>(blanket blocks from settings)"]
        L2["Layer 2: Input Validation<br/>(tool-specific schema checks)"]
        L3["Layer 3: Tool Permissions<br/>(tool.checkPermissions())"]
        L4["Layer 4: Hook System<br/>(pre-permission hooks)"]
        L5["Layer 5: Permission Rules<br/>(alwaysAllow/Deny/Ask)"]
        L6["Layer 6: Classifier<br/>(ML-based auto-approval)"]
        L7["Layer 7: User Consent<br/>(interactive dialog)"]
    end

    ToolCall["Tool Invocation"] --> L1
    L1 -->|pass| L2
    L2 -->|pass| L3
    L3 -->|pass| L4
    L4 -->|pass| L5
    L5 -->|pass| L6
    L6 -->|pass| L7
    L7 -->|approved| Execute["Tool Execution"]

    L1 -->|blocked| Deny["Deny"]
    L2 -->|invalid| Deny
    L3 -->|deny| Deny
    L4 -->|deny| Deny
    L5 -->|deny| Deny
    L6 -->|deny| Deny
    L7 -->|rejected| Deny
```

## Permission Modes

```mermaid
stateDiagram-v2
    [*] --> Default : Normal startup
    [*] --> Auto : TRANSCRIPT_CLASSIFIER enabled
    [*] --> BypassPermissions : bypass permissions flag

    Default --> Plan : /plan command
    Plan --> Default : ExitPlanMode

    Default --> AcceptEdits : User switches mode
    Default --> DontAsk : User switches mode
    AcceptEdits --> Default : User switches back
    DontAsk --> Default : User switches back

    state Default {
        [*] --> D_CheckRules
        D_CheckRules --> D_Allow : alwaysAllow match
        D_CheckRules --> D_Deny : alwaysDeny match
        D_CheckRules --> D_AskUser : No matching rule
        D_AskUser --> D_Allow : User approves
        D_AskUser --> D_Deny : User rejects
    }

    state Auto {
        [*] --> A_ClassifierCheck
        A_ClassifierCheck --> A_Allow : Classified safe
        A_ClassifierCheck --> A_AskUser : Uncertain
        A_AskUser --> A_Allow : User approves
        A_AskUser --> A_Deny : User rejects
    }

    state BypassPermissions {
        [*] --> BP_AllowAll : All tools auto approved
    }

    state Plan {
        [*] --> P_ReadOnly : Read only tools only
    }

    state AcceptEdits {
        [*] --> AE_AutoEdits : File edits auto approved
        AE_AutoEdits --> AE_AskOther : Non edit tools prompt
    }

    state DontAsk {
        [*] --> DA_DenyUnknown : Deny unless explicitly allowed
    }
```

| Mode | Behavior | Risk Level |
|------|----------|-----------|
| `default` | Ask user when no rule matches | Low |
| `plan` | Read-only tools only, explicit approval | Lowest |
| `acceptEdits` | Auto-allow file edits, ask for others | Medium |
| `bypassPermissions` | Allow everything | Highest |
| `dontAsk` | Deny unless explicitly allowed | Low |
| `auto` | ML classifier decides (ant-only) | Medium |

## Permission Decision Flow

```mermaid
flowchart TD
    Start["canUseTool(tool, input, context)"] --> DenyRules{"Step 1:<br/>Deny rules match?"}

    DenyRules -->|yes| Denied["Return DENY<br/>(blanket block)"]
    DenyRules -->|no| Validate{"Step 2:<br/>validateInput()?"}

    Validate -->|invalid| Denied
    Validate -->|valid| ToolCheck{"Step 3:<br/>tool.checkPermissions()"}

    ToolCheck -->|deny| Denied
    ToolCheck -->|allow| Allowed["Return ALLOW"]
    ToolCheck -->|ask| Hooks{"Step 4:<br/>Permission hooks?"}

    Hooks -->|hook resolves| HookDecision{"Hook decision?"}
    HookDecision -->|allow| Allowed
    HookDecision -->|deny| Denied
    Hooks -->|no hook| Rules{"Step 5:<br/>Permission rules match?"}

    Rules -->|alwaysAllow| Allowed
    Rules -->|alwaysDeny| Denied
    Rules -->|alwaysAsk| AskUser
    Rules -->|no match| ModeCheck{"Step 6:<br/>Permission mode?"}

    ModeCheck -->|default| AskUser["Show interactive dialog"]
    ModeCheck -->|bypass| Allowed
    ModeCheck -->|acceptEdits| EditCheck{"Is file edit?"}
    ModeCheck -->|dontAsk| Denied
    ModeCheck -->|auto| Classifier["Step 6b:<br/>Run classifier"]

    EditCheck -->|yes| Allowed
    EditCheck -->|no| AskUser

    Classifier -->|safe| Allowed
    Classifier -->|unsafe| AskUser
    Classifier -->|uncertain| AskUser

    AskUser --> UserDecision{"User response?"}
    UserDecision -->|Allow once| Allowed
    UserDecision -->|Allow always| PersistAllow["Persist rule + ALLOW"]
    UserDecision -->|Reject| Denied
    UserDecision -->|Abort (Ctrl+C)| Denied
```

## Interactive Permission Handler

When the decision is `ask`, the interactive handler races multiple approval mechanisms:

```mermaid
flowchart TD
    AskDecision["behavior: 'ask'"] --> PushQueue["Push ToolUseConfirm<br/>to React queue"]
    PushQueue --> RenderDialog["Render permission dialog"]

    RenderDialog --> Race["Race 6 parallel mechanisms"]

    subgraph "Parallel Racers (first wins)"
        R1["1. Hook Execution<br/>(PermissionRequest hooks)"]
        R2["2. Bash Classifier<br/>(async ML check)"]
        R3["3. Bridge Response<br/>(CCR / claude.ai)"]
        R4["4. Channel Relay<br/>(Telegram, iMessage, etc.)"]
        R5["5. User Interaction<br/>(local keyboard/mouse)"]
        R6["6. Abort Signal<br/>(session cancellation)"]
    end

    Race --> R1 & R2 & R3 & R4 & R5 & R6

    R1 & R2 & R3 & R4 & R5 & R6 --> Claim{"claim()<br/>atomic race winner"}

    Claim -->|"Winner: Hook"| HookAllow["handleHookAllow()"]
    Claim -->|"Winner: Classifier"| ClassifierAllow["Auto-approve + checkmark animation"]
    Claim -->|"Winner: Bridge"| BridgeAllow["Remote approval"]
    Claim -->|"Winner: Channel"| ChannelAllow["Channel-relayed approval"]
    Claim -->|"Winner: User"| UserAllow["handleUserAllow() or onReject()"]
    Claim -->|"Winner: Abort"| Aborted["cancelAndAbort()"]

    ClassifierAllow --> GracePeriod["200ms grace period<br/>(ignore accidental keypress)"]
    GracePeriod --> Checkmark["1-3s ✓ animation"]

    UserAllow --> Persist{"Allow always?"}
    Persist -->|yes| WriteDisk["persistPermissions()<br/>→ settings.json"]
    Persist -->|no| SessionOnly["Session-only approval"]
```

## Permission Rules

```mermaid
classDiagram
    class PermissionRule {
        +PermissionRuleSource source
        +RuleBehavior ruleBehavior
        +RuleValue ruleValue
    }

    class RuleValue {
        +string toolName
        +string ruleContent
    }

    class PermissionRuleSource {
        <<enumeration>>
        userSettings
        projectSettings
        localSettings
        flagSettings
        policySettings
        cliArg
        command
        session
    }

    class ToolPermissionContext {
        +PermissionMode mode
        +Map additionalWorkingDirectories
        +ToolPermissionRulesBySource alwaysAllowRules
        +ToolPermissionRulesBySource alwaysDenyRules
        +ToolPermissionRulesBySource alwaysAskRules
        +boolean isBypassPermissionsModeAvailable
        +ToolPermissionRulesBySource strippedDangerousRules
        +boolean shouldAvoidPermissionPrompts
        +boolean awaitAutomatedChecksBeforeDialog
    }

    PermissionRule --> RuleValue
    PermissionRule --> PermissionRuleSource
    ToolPermissionContext --> PermissionRule : contains rules
```

### Rule Format

Rules follow the pattern `ToolName[(content)]`:

| Rule | Meaning |
|------|---------|
| `Bash` | All bash commands |
| `Bash(python:*)` | Bash commands starting with `python:` |
| `Bash(git *)` | Bash commands starting with `git ` |
| `Edit(/path/to/file.ts)` | Edit a specific file |
| `FileRead` | All file reads |
| `mcp__server__tool` | Specific MCP tool |

### Settings File Hierarchy (Priority Order)

```mermaid
flowchart TD
    subgraph "Highest Priority"
        Policy["policySettings<br/>(org-managed, immutable)"]
    end

    subgraph "High Priority"
        Flag["flagSettings<br/>(--permission-* CLI flags)"]
    end

    subgraph "Medium Priority"
        Local["localSettings<br/>(.claude.local/settings.json)<br/>(gitignored)"]
    end

    subgraph "Low Priority"
        Project["projectSettings<br/>(.claude/settings.json)<br/>(git-tracked)"]
    end

    subgraph "Lowest Priority"
        User["userSettings<br/>(~/.claude/settings.json)"]
    end

    Policy --> Flag --> Local --> Project --> User
```

## Permission Decision Types

```mermaid
classDiagram
    class PermissionDecision {
        <<union>>
    }

    class AllowDecision {
        +behavior: "allow"
        +Input updatedInput
        +boolean userModified
        +PermissionDecisionReason decisionReason
        +string acceptFeedback
        +ContentBlockParam[] contentBlocks
    }

    class AskDecision {
        +behavior: "ask"
        +string message
        +Input updatedInput
        +PermissionDecisionReason decisionReason
        +PermissionUpdate[] suggestions
        +string blockedPath
        +PendingClassifierCheck pendingClassifierCheck
    }

    class DenyDecision {
        +behavior: "deny"
        +string message
        +PermissionDecisionReason decisionReason
    }

    class PermissionDecisionReason {
        <<enumeration>>
        rule
        mode
        subcommandResults
        hook
        classifier
        sandboxOverride
        workingDir
        safetyCheck
        asyncAgent
        permissionPromptTool
        other
    }

    PermissionDecision <|-- AllowDecision
    PermissionDecision <|-- AskDecision
    PermissionDecision <|-- DenyDecision
    AllowDecision --> PermissionDecisionReason
    AskDecision --> PermissionDecisionReason
    DenyDecision --> PermissionDecisionReason
```

## Three Permission Handlers

Different execution contexts use different permission handlers:

```mermaid
flowchart TD
    ToolInvocation["Tool Invocation"] --> ContextCheck{"Execution<br/>context?"}

    ContextCheck -->|"Main agent<br/>(interactive)"| Interactive["interactiveHandler.ts"]
    ContextCheck -->|"Coordinator<br/>worker"| Coordinator["coordinatorHandler.ts"]
    ContextCheck -->|"Swarm<br/>worker"| Swarm["swarmWorkerHandler.ts"]

    subgraph "Interactive Handler"
        I1["Push to UI queue"]
        I2["Race 6 mechanisms"]
        I3["User dialog"]
        I1 --> I2 --> I3
    end

    subgraph "Coordinator Handler"
        C1["Await hooks (sequential)"]
        C2["Await classifier (sequential)"]
        C3["Fall through to interactive"]
        C1 --> C2 --> C3
    end

    subgraph "Swarm Worker Handler"
        S1["Try classifier"]
        S2["Forward to leader via mailbox"]
        S3["Wait for leader response"]
        S4["Show 'waiting for approval'"]
        S1 --> S2 --> S3
        S2 --> S4
    end

    Interactive --> I1
    Coordinator --> C1
    Swarm --> S1
```

## Denial Tracking

```mermaid
stateDiagram-v2
    [*] --> Normal : consecutiveDenials = 0

    Normal --> Tracking : Tool denied
    Tracking --> Tracking : Another denial (count++)
    Tracking --> Normal : Tool approved (count = 0)
    Tracking --> Fallback : consecutiveDenials >= 20

    Fallback --> Normal : shouldFallbackToPrompting() returns true
    Note right of Fallback : Forces interactive prompting<br/>regardless of mode
```

## Permission Persistence Flow

```mermaid
sequenceDiagram
    participant User as User
    participant Dialog as Permission Dialog
    participant Context as PermissionContext
    participant Persist as persistPermissions()
    participant Settings as settings.json
    participant State as ToolPermissionContext

    User->>Dialog: Click "Allow Always"
    Dialog->>Context: handleUserAllow(input, updates)

    Context->>Persist: persistPermissionUpdates(updates)

    loop For each PermissionUpdate
        Persist->>Settings: updateSettingsForSource()
        Note over Settings: Write to appropriate file:<br/>~/.claude/settings.json or<br/>.claude/settings.json
    end

    Context->>State: applyPermissionUpdates(context, updates)
    Note over State: Merge into runtime<br/>alwaysAllow/Deny/Ask rules

    Context-->>Dialog: PermissionAllowDecision
    Dialog-->>User: Tool executes
```

## Analytics Events

| Event | Trigger |
|-------|---------|
| `tengu_tool_use_granted_in_config` | Auto-approved by allowlist |
| `tengu_tool_use_granted_by_classifier` | Classifier auto-approved |
| `tengu_tool_use_granted_in_prompt_permanent` | User approved "always" |
| `tengu_tool_use_granted_in_prompt_temporary` | User approved "once" |
| `tengu_tool_use_granted_by_permission_hook` | Hook auto-approved |
| `tengu_tool_use_denied_in_config` | Denied by denylist |
| `tengu_tool_use_rejected_in_prompt` | User rejected |
