# UI Architecture

## Terminal Rendering Stack

Claude Code uses a custom fork of Ink (React for terminals) with Yoga for flexbox layout. The rendering pipeline converts a React component tree into optimized ANSI terminal output.

```mermaid
flowchart TD
    subgraph "React Layer"
        Components["React Components<br/>(140+ components)"]
        Reconciler["react-reconciler<br/>(ConcurrentRoot)"]
    end

    subgraph "Virtual DOM"
        DOM["Custom DOM Tree<br/>(element/text nodes)"]
        Props["Property Diffing<br/>& Mutation Tracking"]
    end

    subgraph "Layout Engine"
        Yoga["Yoga (Native)<br/>Flexbox Layout"]
        Engine["Layout Computation"]
        Geometry["Position/Size<br/>Calculations"]
    end

    subgraph "Render Pipeline"
        RenderNode["render-node-to-output.ts<br/>Walk tree post-layout"]
        Segments["Output Segments<br/>(text, colors, styles)"]
        Borders["Border Rendering"]
        Wrap["Text Wrapping"]
    end

    subgraph "Screen Buffer"
        Grid["Cell Grid<br/>(width × height)"]
        Pools["StylePool, CharPool,<br/>HyperlinkPool"]
        Diff["Diff Algorithm<br/>(old vs new cells)"]
        Patches["Minimal Cell Updates"]
    end

    subgraph "Terminal Output"
        ANSI["ANSI Escape Sequences"]
        CSI["CSI: cursor, color, modes"]
        DEC["DEC: mouse tracking, alt-screen"]
        OSC["OSC: hyperlinks, title"]
    end

    Components --> Reconciler --> DOM --> Props
    DOM --> Yoga --> Engine --> Geometry
    Geometry --> RenderNode --> Segments --> Borders --> Wrap
    Wrap --> Grid --> Pools --> Diff --> Patches
    Patches --> ANSI --> CSI & DEC & OSC
```

## Component Tree Structure

```mermaid
graph TD
    App["App<br/>(AppStateProvider + FpsMetrics + Stats)"]
    AppState["AppStateProvider<br/>(Zustand store)"]
    Mailbox["MailboxProvider<br/>(interprocess)"]
    Voice["VoiceProvider<br/>(conditional)"]

    App --> AppState --> Mailbox --> Voice

    Voice --> REPL["REPL<br/>(main interactive component)"]

    REPL --> InkApp["Ink App<br/>(terminal renderer)"]

    InkApp --> Contexts["Context Providers"]
    Contexts --> TermSize["TerminalSizeContext"]
    Contexts --> TermFocus["TerminalFocusContext"]
    Contexts --> Stdin["StdinContext"]
    Contexts --> Clock["ClockContext"]
    Contexts --> Cursor["CursorDeclarationContext"]

    REPL --> MessageList["VirtualMessageList<br/>(scrollable viewport)"]
    REPL --> PromptInput["PromptInput<br/>(command input)"]
    REPL --> Dialogs["Dialogs & Modals"]
    REPL --> Status["Status Components"]
    REPL --> Tasks["Background Tasks"]

    MessageList --> MsgComponents["Message Components<br/>(39 variants)"]
    MessageList --> Selection["Selection Overlay"]

    PromptInput --> BaseTextInput["BaseTextInput"]
    PromptInput --> Footer["PromptInputFooter"]
    PromptInput --> QueuedCmds["QueuedCommands"]
    PromptInput --> CtxSuggestions["ContextSuggestions"]

    Dialogs --> QuickOpen["QuickOpenDialog<br/>(Ctrl+K)"]
    Dialogs --> Settings["Settings Panel<br/>(Ctrl+,)"]
    Dialogs --> BridgeDialog["BridgeDialog"]
    Dialogs --> PermDialog["Permission Dialogs"]
    Dialogs --> MCPDialog["MCP Approval Dialogs"]

    Status --> StatusLine["StatusLine<br/>(footer)"]
    Status --> Spinner["SpinnerWithVerb"]
    Status --> Notices["StatusNotices"]

    Tasks --> TaskDialog["BackgroundTasksDialog"]
    Tasks --> ShellProgress["ShellProgress"]
    Tasks --> AgentDetail["AsyncAgentDetailDialog"]
```

## Component Categories

```mermaid
mindmap
    root((UI Components))
        Messages (39)
            AssistantTextMessage
            AssistantThinkingMessage
            AssistantToolUseMessage
            UserPromptMessage
            UserToolResultMessage/
                SuccessMessage
                ErrorMessage
                RejectMessage
                CanceledMessage
            UserBashOutputMessage
            SystemTextMessage
            CompactBoundaryMessage
            RateLimitMessage
            TaskAssignmentMessage
            HookProgressMessage
            AttachmentMessage
            PlanApprovalMessage
            GroupedToolUseContent
        Dialogs (15+)
            QuickOpenDialog
            BridgeDialog
            ExportDialog
            Settings
            MCPServerApproval
            CostThreshold
            BypassPermissionsMode
            InvalidConfig
            ConsoleOAuthFlow
        Tasks (13)
            BackgroundTasksDialog
            BackgroundTaskStatus
            ShellDetailDialog
            ShellProgress
            AsyncAgentDetail
            RemoteSessionDetail
        Input
            PromptInput
            BaseTextInput
            TextInput
            ContextSuggestions
        Status
            StatusLine
            StatusNotices
            SpinnerWithVerb
        Layout
            FullscreenLayout
            Markdown
            MessageResponse
        Ink Primitives
            Box
            Text
            ScrollBox
            Button
            Link
            AlternateScreen
```

## State Management (Zustand)

```mermaid
classDiagram
    class AppState {
        +SettingsJson settings
        +boolean verbose
        +ModelSetting mainLoopModel
        +string statusLineText
        +ExpandedView expandedView
        +number selectedIPAgentIndex
        +FooterItem footerSelection
        +ToolPermissionContext toolPermissionContext
        +string agent
        +boolean kairosEnabled
        +string remoteSessionUrl
        +ConnectionStatus remoteConnectionStatus
        +boolean replBridgeEnabled
        +boolean replBridgeConnected
        +boolean replBridgeSessionActive
        +string replBridgeError
        +MCPState mcp
        +TaskMap tasks
        +string foregroundedTaskId
    }

    class MCPState {
        +MCPServerConnection[] clients
        +Tool[] tools
        +Command[] commands
        +Record resources
    }

    class AppStateStore {
        +useAppState(selector) T
        +useSetAppState() Function
    }

    AppState --> MCPState
    AppStateStore --> AppState : manages
```

### State Flow

```mermaid
flowchart LR
    subgraph "State Sources"
        UserInput["User Input"]
        APIResponse["API Response"]
        ToolResult["Tool Results"]
        MCP["MCP Events"]
        Bridge["Bridge Messages"]
        Hooks["Hook Events"]
    end

    subgraph "State Updates"
        SetState["useSetAppState()"]
        Selectors["useAppState(selector)<br/>Object.is comparison"]
        OnChange["onChangeAppState<br/>callback"]
    end

    subgraph "Consumers"
        Components2["React Components"]
        REPL2["REPL Loop"]
        QueryEngine["QueryEngine"]
        Permissions["Permission System"]
    end

    UserInput --> SetState
    APIResponse --> SetState
    ToolResult --> SetState
    MCP --> SetState
    Bridge --> SetState
    Hooks --> SetState

    SetState --> Selectors --> Components2
    SetState --> OnChange --> REPL2 & QueryEngine & Permissions
```

## Ink Rendering Pipeline

```mermaid
sequenceDiagram
    participant React as React Reconciler
    participant DOM as Virtual DOM
    participant Yoga as Yoga Layout
    participant Render as Render Pipeline
    participant Screen as Screen Buffer
    participant Term as Terminal

    React->>DOM: Commit fiber updates
    DOM->>DOM: Apply property mutations

    DOM->>Yoga: Request layout computation
    Yoga->>Yoga: Flexbox calculation<br/>(width, height, position)
    Yoga-->>DOM: Layout results

    DOM->>Render: render-node-to-output()
    Render->>Render: Walk component tree
    Render->>Render: Generate segments<br/>(text, style, color)
    Render->>Render: Handle borders, wrapping
    Render-->>Screen: Output segments

    Screen->>Screen: Build cell grid
    Screen->>Screen: Diff old vs new cells
    Screen->>Screen: Compute minimal patches

    Screen->>Term: Emit ANSI sequences
    Note over Term: CSI: cursor positioning<br/>SGR: colors, bold, etc.<br/>OSC: hyperlinks<br/>DEC: alt-screen, mouse
```

## Input Handling

```mermaid
flowchart TD
    Stdin["stdin"] --> Parser["parseMultipleKeypresses()"]

    Parser --> Route{"Event type?"}

    Route -->|Keyboard| KeyBindings["Global keybinding check"]
    Route -->|Mouse| MouseHandler["Mouse event handler"]
    Route -->|Paste| PasteHandler["Clipboard paste handler"]

    KeyBindings --> CommandKB{"Command<br/>keybinding?"}
    CommandKB -->|yes| Dispatch["Dispatch command<br/>(Ctrl+K, Ctrl+C, etc.)"]
    CommandKB -->|no| VimCheck{"Vim mode?"}

    VimCheck -->|yes| VimInput["Vim keybinding handler"]
    VimCheck -->|no| TextInput["Text input to PromptInput"]

    TextInput --> SlashCheck{"Starts with /?"}
    SlashCheck -->|yes| Typeahead["Show command typeahead"]
    SlashCheck -->|no| BufferInput["Buffer in input field"]

    subgraph "On Submit (Enter)"
        ParseCmd["Parse command/prompt"]
        Submit["handlePromptSubmit()"]
        AddMsg["Add to messages[]"]
        Trigger["Trigger query loop"]
    end

    BufferInput -->|Enter| ParseCmd --> Submit --> AddMsg --> Trigger
    Typeahead -->|Enter| ParseCmd
```

## Frame Management

```mermaid
flowchart LR
    subgraph "Double Buffer"
        Front["Front Buffer<br/>(currently displayed)"]
        Back["Back Buffer<br/>(being composed)"]
    end

    subgraph "Frame Cycle"
        Compose["Compose frame"] --> Diff["Diff buffers"]
        Diff --> Patches["Generate patches"]
        Patches --> Emit["Emit ANSI"]
        Emit --> Swap["Swap buffers"]
        Swap --> Compose
    end

    subgraph "Optimizations"
        DamageB["Damage bounds<br/>(repaint changed area)"]
        ScrollH["Scroll hints<br/>(DECSTBM hardware)"]
        LayoutS["Layout shift detection"]
        NodeCache["Node cache reuse"]
    end

    Back --> Compose
    Compose --> DamageB & ScrollH & LayoutS
    NodeCache --> Compose
```

## Ink Components (Primitives)

| Component | Purpose |
|-----------|---------|
| `App.tsx` | Root with stdin/stdout context, Ctrl+C handler |
| `AlternateScreen.tsx` | Fullscreen mode (alt-screen buffer) |
| `Box.tsx` | Flex container (flexDirection, margin, padding) |
| `Text.tsx` | Styled text node (colors, bold, etc.) |
| `ScrollBox.tsx` | Scrolling viewport with keyboard navigation |
| `Button.tsx` | Clickable button (mouse support) |
| `Link.tsx` | Hyperlink (OSC 8 protocol) |
| `NoSelect.tsx` | Exclude region from text selection |
| `RawAnsi.tsx` | Pass-through ANSI sequences |
| `Spacer.tsx` | Flexible space |
| `Newline.tsx` | Line break |

## Virtual Message List

The message list uses virtual scrolling for performance with large conversations:

```mermaid
flowchart TD
    Messages["Message[]<br/>(conversation history)"] --> Filter["Filter visible messages"]
    Filter --> Virtual["Virtual scroll calculation"]

    Virtual --> Viewport["Visible viewport<br/>(terminal height)"]
    Virtual --> OffscreenAbove["Off-screen above<br/>(not rendered)"]
    Virtual --> OffscreenBelow["Off-screen below<br/>(not rendered)"]

    Viewport --> Render["Render visible messages"]
    Render --> Dispatch{"Message type?"}

    Dispatch --> Assistant["AssistantTextMessage"]
    Dispatch --> Thinking["AssistantThinkingMessage"]
    Dispatch --> ToolUse["AssistantToolUseMessage"]
    Dispatch --> UserPrompt["UserPromptMessage"]
    Dispatch --> ToolResult["UserToolResultMessage/*"]
    Dispatch --> System["SystemTextMessage"]
    Dispatch --> Compact["CompactBoundaryMessage"]
    Dispatch --> Other["Other message types..."]

    subgraph "Scroll Controls"
        ArrowKeys["↑↓ Arrow keys"]
        PageUpDown["Page Up/Down"]
        Home["Home/End"]
        Mouse["Mouse wheel"]
    end

    ArrowKeys & PageUpDown & Home & Mouse --> Virtual
```

## Theme System

```mermaid
flowchart LR
    subgraph "Theme Sources"
        SystemTheme["System theme detection<br/>(dark/light)"]
        UserPref["User preference<br/>(/theme command)"]
        Settings["settings.json"]
    end

    subgraph "Theme Object"
        Colors["Text colors"]
        Borders["Border colors"]
        Status["Status colors"]
        Permission["Permission mode colors"]
        Syntax["Syntax highlighting"]
    end

    SystemTheme --> Theme["Resolved Theme"]
    UserPref --> Theme
    Settings --> Theme

    Theme --> Colors & Borders & Status & Permission & Syntax

    Colors --> Components3["All UI Components"]
    Borders --> Components3
    Status --> Components3
    Permission --> Components3
    Syntax --> Components3
```
