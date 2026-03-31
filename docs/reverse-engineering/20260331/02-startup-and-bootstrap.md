# Startup and Bootstrap

## Entry Point Decision Tree

The CLI entry point (`src/entrypoints/cli.tsx`) implements a fast-path routing system that avoids loading the full application for simple operations.

```mermaid
flowchart TD
    Start["CLI Invocation"] --> EnvSetup["Environment Setup<br/>COREPACK_ENABLE_AUTO_PIN=0<br/>NODE_OPTIONS (if remote)"]
    EnvSetup --> Ablation{"ABLATION_BASELINE<br/>feature flag?"}
    Ablation -->|yes| SetAblation["Set experiment flags:<br/>SIMPLE, DISABLE_THINKING,<br/>DISABLE_COMPACT, etc."]
    Ablation -->|no| CheckArgs
    SetAblation --> CheckArgs

    CheckArgs --> Version{"--version/-v/-V?"}
    Version -->|yes| PrintVersion["Print MACRO.VERSION<br/>(zero imports, instant)"]

    Version -->|no| DumpPrompt{"--dump-system-prompt?<br/>(ant-only)"}
    DumpPrompt -->|yes| DumpSP["Dump system prompt<br/>(minimal imports)"]

    DumpPrompt -->|no| ChromeMCP{"--claude-in-chrome-mcp?"}
    ChromeMCP -->|yes| LaunchChromeMCP["Start Chrome MCP Server"]

    ChromeMCP -->|no| NativeHost{"--chrome-native-host?"}
    NativeHost -->|yes| LaunchNative["Start Native Host"]

    NativeHost -->|no| ComputerMCP{"--computer-use-mcp?<br/>(CHICAGO_MCP)"}
    ComputerMCP -->|yes| LaunchCUMCP["Start Computer Use MCP"]

    ComputerMCP -->|no| DaemonWorker{"--daemon-worker=kind?<br/>(DAEMON)"}
    DaemonWorker -->|yes| RunWorker["runDaemonWorker(kind)<br/>(no config, no sinks)"]

    DaemonWorker -->|no| BridgeCmd{"remote-control/rc/<br/>remote/sync/bridge?<br/>(BRIDGE_MODE)"}
    BridgeCmd -->|yes| BridgeMain["enableConfigs()<br/>→ bridgeMain(args)"]

    BridgeCmd -->|no| DaemonCmd{"daemon subcommand?<br/>(DAEMON)"}
    DaemonCmd -->|yes| DaemonMain["enableConfigs()<br/>→ daemonMain(args)"]

    DaemonCmd -->|no| BGCmd{"ps/logs/attach/kill/<br/>--bg/--background?<br/>(BG_SESSIONS)"}
    BGCmd -->|yes| BGHandler["enableConfigs()<br/>→ bg.js handler"]

    BGCmd -->|no| Templates{"new/list/reply?<br/>(TEMPLATES)"}
    Templates -->|yes| TemplateHandler["Template job handler"]

    Templates -->|no| EnvRunner{"environment-runner?<br/>(BYOC)"}
    EnvRunner -->|yes| EnvRunnerMain["BYOC runner"]

    EnvRunner -->|no| SelfHosted{"self-hosted-runner?<br/>(SELF_HOSTED)"}
    SelfHosted -->|yes| SHRunner["Self-hosted runner"]

    SelfHosted -->|no| Worktree{"--worktree --tmux?"}
    Worktree -->|yes| WorktreeMode["Worktree + tmux spawn"]

    Worktree -->|no| Update{"--update/--upgrade?"}
    Update -->|yes| UpdateCmd["Redirect to update subcommand"]

    Update -->|no| LoadMain["import main.tsx<br/>(full CLI load ~135ms)"]
    LoadMain --> MainFn["main()"]
```

## Full Initialization Sequence

When the fast-path check falls through, the full CLI loads via `main.tsx`:

```mermaid
sequenceDiagram
    participant CLI as cli.tsx
    participant Main as main.tsx (module load)
    participant Prefetch as Async Prefetches
    participant PreAction as preAction Hook
    participant Action as Action Handler
    participant REPL as REPL Loop

    CLI->>Main: import('./main.tsx')

    Note over Main: Module-level side effects begin

    Main->>Prefetch: startMdmRawRead() (async subprocess)
    Main->>Prefetch: startKeychainPrefetch() (async keychain read)

    Note over Main: ~135ms of import evaluation<br/>(200+ modules)

    Main->>Main: Export main() function
    CLI->>Main: main()

    Main->>Main: Security hardening<br/>(NoDefaultCurrentDirectoryInExePath)
    Main->>Main: Signal handlers<br/>(SIGINT → gracefulShutdown)
    Main->>Main: URL/URI pre-processing<br/>(cc://, deep links, SSH)
    Main->>Main: Commander.js program setup

    Main->>PreAction: preAction hook fires

    PreAction->>Prefetch: await ensureMdmSettingsLoaded()
    PreAction->>Prefetch: await ensureKeychainPrefetchCompleted()

    PreAction->>PreAction: init() [memoized, runs once]
    Note over PreAction: enableConfigs()<br/>applySafeConfigEnvironmentVariables()<br/>setupGracefulShutdown()<br/>applyExtraCACertsFromConfig()<br/>configureGlobalMTLS()<br/>configureGlobalAgents()<br/>preconnectAnthropicApi()

    PreAction->>PreAction: initSinks() (analytics)
    PreAction->>PreAction: runMigrations()
    PreAction->>PreAction: loadRemoteManagedSettings() (async)
    PreAction->>PreAction: loadPolicyLimits() (async)

    PreAction->>Action: Action handler executes

    Action->>Action: showSetupScreens() (trust, auth)
    Action->>Action: initializeGrowthBook()

    par Parallel Prefetches
        Action->>Action: prefetchAwsCredentials()
        Action->>Action: prefetchGcpCredentials()
        Action->>Action: prefetchFastModeStatus()
        Action->>Action: prefetchAllMcpResources()
        Action->>Action: refreshModelCapabilities()
    end

    Action->>Action: getTools() + getMcpToolsCommandsAndResources()
    Action->>Action: resolveInitialModel()
    Action->>Action: getSystemPrompt()
    Action->>Action: Build CLI options

    Action->>REPL: launchRepl()

    Note over REPL: Interactive conversation loop begins
```

## Bootstrap State

The global session state (`src/bootstrap/state.ts`) is a mutable singleton that holds all session-scoped configuration:

```mermaid
classDiagram
    class State {
        +string originalCwd
        +string projectRoot
        +string cwd
        +SessionId sessionId
        +SessionId parentSessionId
        +ModelSetting mainLoopModelOverride
        +ModelSetting initialMainLoopModel
        +ModelStrings modelStrings
        +boolean isInteractive
        +boolean kairosActive
        +boolean isRemoteMode
        +Meter meter
        +AttributedCounter sessionCounter
        +AttributedCounter locCounter
        +AttributedCounter prCounter
        +AttributedCounter commitCounter
        +AttributedCounter costCounter
        +AttributedCounter tokenCounter
        +boolean sessionBypassPermissionsMode
        +boolean useCoworkPlugins
        +boolean chromeFlagOverride
        +SessionCronTask[] sessionCronTasks
        +Set~string~ sessionCreatedTeams
        +boolean sessionTrustAccepted
        +boolean sessionPersistenceDisabled
        +ChannelEntry[] allowedChannels
        +boolean hasDevChannels
        +string[] inlinePlugins
        +string[] promptCache1hAllowlist
        +boolean promptCache1hEligible
        +RegisteredHookMatcher[] registeredHooks
    }

    class StateAccessors {
        +getSessionId() SessionId
        +switchSession(id) void
        +setOriginalCwd(cwd) void
        +setCwdState(cwd) void
        +setKairosActive(bool) void
        +setIsRemoteMode(bool) void
        +setMeter(meter) void
        +getSessionCounter() AttributedCounter
    }

    State --> StateAccessors : accessed via
```

## Feature Flag System

Feature flags control conditional compilation and runtime behavior:

```mermaid
flowchart LR
    subgraph "Build Time (Production)"
        BunBuild["bun build --define<br/>FEATURE_X=true/false"]
        DCE["Dead Code Elimination<br/>(entire branches removed)"]
        BunBuild --> DCE
    end

    subgraph "Runtime (Development)"
        EnvVar["FEATURES=BRIDGE_MODE,KAIROS"]
        Parse["Parse comma-separated<br/>into Set"]
        EnvVar --> Parse
    end

    subgraph "shims/bun-bundle.ts"
        FeatureFn["feature(name): boolean"]
    end

    DCE --> FeatureFn
    Parse --> FeatureFn

    subgraph "Feature Gates"
        F1["BRIDGE_MODE"]
        F2["DAEMON"]
        F3["BG_SESSIONS"]
        F4["KAIROS"]
        F5["VOICE_MODE"]
        F6["PROACTIVE"]
        F7["COORDINATOR_MODE"]
        F8["SSH_REMOTE"]
        F9["AGENT_TRIGGERS"]
        F10["WEB_BROWSER_TOOL"]
        F11["TRANSCRIPT_CLASSIFIER"]
        F12["~80 more flags..."]
    end

    FeatureFn --> F1
    FeatureFn --> F2
    FeatureFn --> F3
    FeatureFn --> F4
    FeatureFn --> F5
    FeatureFn --> F6
    FeatureFn --> F7
    FeatureFn --> F8
    FeatureFn --> F9
    FeatureFn --> F10
    FeatureFn --> F11
    FeatureFn --> F12
```

## Operational Modes

```mermaid
stateDiagram-v2
    [*] --> CLIEntry : CLI invocation

    CLIEntry --> FastPath : Fast-path match
    CLIEntry --> FullCLI : No fast-path

    state FastPath {
        [*] --> Version : --version
        [*] --> MCP_Server : --claude-in-chrome-mcp
        [*] --> DaemonWorker : --daemon-worker
        [*] --> BridgeMode : remote-control/rc
        [*] --> DaemonMode : daemon subcommand
        [*] --> BGSession : ps/logs/attach/kill
        [*] --> Templates : new/list/reply
        [*] --> EnvRunner : environment-runner
        [*] --> WorktreeMode : --worktree --tmux
    }

    state FullCLI {
        [*] --> Interactive : No -p flag
        [*] --> Headless : -p / --print flag
        [*] --> Assistant : assistant subcommand
        [*] --> SSHRemote : ssh host
        [*] --> DirectConnect : cc:// URL
    }

    Interactive --> REPLLoop : launchRepl()
    Headless --> PrintLoop : print.ts (no TUI)
    Assistant --> AssistantREPL : Briefing mode
    SSHRemote --> SpawnSSH : SSH session
    DirectConnect --> OpenSession : Parsed URL

    REPLLoop --> [*] : exit/Ctrl+C
    PrintLoop --> [*] : output complete
```

## Environment Variables

| Variable | Phase | Effect |
|----------|-------|--------|
| `CLAUDE_CODE_REMOTE` | Module load | Increase max heap to 8GB |
| `CLAUDE_CODE_ABLATION_BASELINE` | Module load | Set experiment flags |
| `COREPACK_ENABLE_AUTO_PIN` | Module load | Prevent yarn auto-pinning |
| `CLAUDE_CODE_SIMPLE` | Runtime | Minimal mode (skip hooks, LSP, plugins) |
| `CLAUDE_CODE_PROFILE_STARTUP` | Runtime | Enable startup profiling |
| `FEATURES` | Module load | Enable feature flags (dev mode) |
| `USER_TYPE` | Runtime | `ant` enables internal features |
| `CLAUDE_CODE_DISABLE_TERMINAL_TITLE` | Runtime | Skip process title |
| `API_TIMEOUT_MS` | Runtime | API request timeout |
| `CLAUDE_CODE_MAX_CONTEXT_TOKENS` | Runtime | Override context window size |
| `CLAUDE_CODE_EXTRA_BODY` | Runtime | Extra API request body params |
| `ANTHROPIC_API_KEY` | Runtime | Direct API authentication |
| `ANTHROPIC_CUSTOM_HEADERS` | Runtime | Custom API headers |

## Graceful Shutdown

```mermaid
sequenceDiagram
    participant Signal as SIGINT/exit
    participant Shutdown as gracefulShutdown()
    participant Telemetry as Analytics
    participant LSP as LSP Servers
    participant Teams as Teams
    participant Cursor as Terminal

    Signal->>Shutdown: Trigger

    par Cleanup Operations
        Shutdown->>Telemetry: Flush events
        Shutdown->>LSP: Shutdown all servers
        Shutdown->>Teams: Clean up session teams
    end

    Shutdown->>Cursor: resetCursor()
    Shutdown->>Shutdown: process.exit()
```

## Startup Performance Profiling

The startup profiler tracks 40+ checkpoints with production sampling:

```mermaid
gantt
    title Startup Timeline (Approximate)
    dateFormat X
    axisFormat %L ms

    section cli.tsx
    Environment setup     : 0, 5
    Fast-path checks      : 5, 15
    Import main.tsx       : 15, 20

    section main.tsx (module)
    Start MDM read (async)     : 20, 25
    Start keychain (async)     : 20, 25
    Heavy imports (~135ms)     : 25, 160

    section preAction
    Await MDM + keychain       : 160, 170
    init() - configs           : 170, 185
    init() - network setup     : 185, 200
    init() - API preconnect    : 200, 210
    initSinks()                : 210, 215
    Migrations                 : 215, 220

    section Action Handler
    Setup screens              : 220, 260
    GrowthBook init            : 260, 270
    Parallel prefetches        : 270, 310
    Load tools + MCP           : 310, 350
    System prompt build        : 350, 370
    Launch REPL                : 370, 380
```
