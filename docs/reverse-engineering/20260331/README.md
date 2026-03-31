# Claude Code CLI — Reverse Engineering Documentation

> Generated 2026-03-31 from source snapshot extracted via npm package source map.

## Overview

Claude Code is Anthropic's official terminal-based AI coding assistant. It operates as an **agentic loop** — accepting user input, querying the Claude API with streaming, executing tools (file edits, shell commands, web searches, etc.), and iterating until the task is complete.

**Tech Stack:** Bun + TypeScript | React 19 + Ink (terminal UI) | Yoga (flexbox layout) | Commander.js (CLI) | Zod v4 (schemas) | Zustand (state)

**Scale:** ~800K lines of source across 1,500+ files, 55+ tools, 60+ slash commands, 90+ React hooks, 140+ UI components, 340+ utilities.

## Table of Contents

| # | Document | Description |
|---|----------|-------------|
| 01 | [System Architecture](01-system-architecture.md) | High-level architecture, module dependency graph, subsystem overview, technology stack, design philosophy |
| 02 | [Startup and Bootstrap](02-startup-and-bootstrap.md) | Entry point decision tree, initialization sequence, bootstrap state, feature flags, operational modes, environment variables |
| 03 | [Query Engine](03-query-engine.md) | QueryEngine class, message flow, API client (multi-provider), streaming, tool call loop, retry logic, thinking mode, token counting, context compaction |
| 04 | [Tools and Commands](04-tools-and-commands.md) | Tool type system, complete tool inventory (55+), tool execution lifecycle, concurrency model, slash command system (60+), command discovery |
| 05 | [Permission System](05-permission-system.md) | Multi-layer architecture, 6 permission modes, decision flow, interactive handler with 6-way race, permission rules, persistence, analytics |
| 06 | [Services](06-services.md) | MCP protocol (25 files), OAuth, LSP, context compaction, analytics/GrowthBook, policy limits, tool orchestration, initialization order |
| 07 | [UI Architecture](07-ui-architecture.md) | Ink rendering pipeline (React → Virtual DOM → Yoga → Screen buffer → ANSI), component tree, 140+ components, Zustand state management, input handling |
| 08 | [Bridge and Coordinator](08-bridge-and-coordinator.md) | IDE communication (VS Code, JetBrains, claude.ai), JWT auth, WebSocket protocol, multi-agent coordinator, skills system, swarm permissions |
| 09 | [Configuration and Context](09-configuration-and-context.md) | 5-source settings hierarchy, CLAUDE.md loading, system prompt assembly, context window management, SDK schemas (Zod v4), hook system, utility modules |
| 10 | [Data Flow and State Machines](10-data-flow-and-state-machines.md) | End-to-end data flow sequences, session lifecycle, tool execution, permission decision, MCP connection, bridge session, compaction state machines |

## Architecture at a Glance

```mermaid
graph TB
    subgraph "User Interfaces"
        Terminal["Terminal (CLI)"]
        IDE["IDE Extensions"]
        Web["claude.ai"]
        SDK["SDK (Programmatic)"]
    end

    subgraph "Core Engine"
        Entry["Entry Points<br/>cli.tsx → main.tsx"]
        QE["QueryEngine<br/>(Agentic Loop)"]
        Tools["Tool System<br/>(55+ tools)"]
        Perms["Permission System<br/>(6 modes)"]
    end

    subgraph "Services"
        API["Claude API<br/>(multi-provider)"]
        MCP["MCP Servers<br/>(external tools)"]
        LSP["LSP Servers<br/>(language intelligence)"]
    end

    subgraph "UI Layer"
        React["React 19 + Ink<br/>(terminal rendering)"]
        State["Zustand Store<br/>(state management)"]
    end

    Terminal --> Entry --> QE
    IDE --> Entry
    Web --> Entry
    SDK --> QE

    QE --> Tools --> Perms
    QE --> API
    Tools --> MCP
    Tools --> LSP

    QE --> React --> Terminal
    React --> State
```

## Key Design Decisions

1. **Fast-path routing** — `--version` and similar flags return instantly without loading the 200+ module dependency tree
2. **Parallel prefetch** — MDM settings, keychain, and API connection fire asynchronously during module import (~135ms)
3. **Build-time dead-code elimination** — ~90 feature flags via `bun:bundle` compile out entire subsystems from non-Anthropic builds
4. **Multi-layer permission system** — 7 check layers with ML classifier, hooks, rules, and user consent racing in parallel
5. **Concurrent tool execution** — Read-only tools run in parallel (max 10); write tools execute serially with context propagation
6. **Context compaction** — Automatic conversation summarization with circuit breaker (3 failures) to manage context window

## Reconstructed Environment Notes

This documentation was produced from a reconstructed development environment:

- **~150 stub files** exist for modules missing from the source map extraction
- **TypeScript strict mode is relaxed** (`noImplicitAny: false`, `strictNullChecks: false`)
- **Proprietary packages** (`@ant/*`, `@anthropic-ai/sandbox-runtime`, etc.) are replaced with empty stubs
- **Native modules** (`color-diff-napi`, `modifiers-napi`, `audio-capture-napi`) are stubbed

Some implementation details may be incomplete where stubs obscure the actual logic.
