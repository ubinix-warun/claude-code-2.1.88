# Claude Code — Software Stack & Architecture

> **Version:** 2.1.88  
> **Package:** `@anthropic-ai/claude-code`  
> **Runtime:** Node.js ≥ 18, bundled with Bun

---

## Table of Contents

1. [Software Stack Overview](#1-software-stack-overview)
2. [Top-Level Component Diagram](#2-top-level-component-diagram)
3. [Top-Level Sequence Diagram — Interactive Session](#3-top-level-sequence-diagram--interactive-session)
4. [Module: CLI Entry & Bootstrap](#4-module-cli-entry--bootstrap)
5. [Module: Terminal UI (Ink / React)](#5-module-terminal-ui-ink--react)
6. [Module: Query Engine](#6-module-query-engine)
7. [Module: Tools System](#7-module-tools-system)
8. [Module: State Management](#8-module-state-management)
9. [Module: API Service](#9-module-api-service)
10. [Module: MCP Service](#10-module-mcp-service)
11. [Module: Session Memory & Compact](#11-module-session-memory--compact)
12. [Module: Commands (Slash Commands)](#12-module-commands-slash-commands)
13. [Module: Multi-Agent / Swarm Coordinator](#13-module-multi-agent--swarm-coordinator)
14. [Module: Analytics & Telemetry](#14-module-analytics--telemetry)

---

## 1. Software Stack Overview

| Layer | Technology | Location |
|---|---|---|
| **Runtime** | Node.js ≥ 18, ESM | `cli.js` (self-contained bundle) |
| **Build toolchain** | Bun bundler + TypeScript | `source/src/**/*.ts(x)` |
| **Terminal UI** | [Ink](https://github.com/vadimdemedes/ink) (React for terminals) | `source/src/ink.ts`, `screens/`, `components/` |
| **UI framework** | React 18 (RSC-compiled via Bun React Compiler) | `source/src/components/` |
| **State management** | Custom zustand-style store | `source/src/state/` |
| **CLI argument parsing** | [Commander.js](https://github.com/tj/commander.js) | `source/src/main.tsx` |
| **AI API client** | `@anthropic-ai/sdk` (also Bedrock / Vertex via adapters) | `source/src/services/api/` |
| **Tool protocol** | [MCP – Model Context Protocol](https://modelcontextprotocol.io) | `source/src/services/mcp/` |
| **Analytics** | Internal + Datadog | `source/src/services/analytics/` |
| **Auth** | OAuth 2.0 / API key / AWS STS / GCP | `source/src/utils/auth.ts`, `services/oauth/` |
| **Image processing** | `@img/sharp` (optional, native binaries per platform) | `package.json` optionalDependencies |
| **Schema validation** | Zod v4 | `source/src/schemas/` |
| **Logging / tracing** | Internal sinks, Datadog, OpenTelemetry-style unary logging | `source/src/utils/log.ts`, `utils/telemetry/` |

### Key architectural characteristics

- **Single self-contained binary** — `cli.js` (≈13 MB) bundles all JavaScript + `node_modules`; only native add-ons (sharp) ship separately.
- **Feature flags** — `feature('FLAG_NAME')` calls at module level enable dead-code elimination at build time (Bun DCE).
- **Reactive UI** — Ink renders a virtual terminal DOM via React, enabling hooks-driven updates on every turn.
- **Pluggable tools** — Every AI capability (file edit, bash, web search, …) is a `Tool` object registered at startup.
- **MCP gateway** — External tools/resources are exposed to Claude through the Model Context Protocol.
- **Multi-agent swarm** — Teammate sub-agents can be spawned in-process or out-of-process; a Coordinator mode orchestrates them.

---

## 2. Top-Level Component Diagram

```mermaid
graph TB
    subgraph User["User / Terminal"]
        STDIN["stdin / keyboard"]
        STDOUT["stdout / terminal output"]
    end

    subgraph CLI["CLI Entry Layer\n(entrypoints/cli.tsx · main.tsx)"]
        CLIEntry["cli.tsx\n(fast-path bootstrap)"]
        MainTSX["main.tsx\n(Commander CLI, init, flags)"]
    end

    subgraph UI["Terminal UI Layer\n(Ink + React)"]
        InkRoot["ink.ts\n(Ink renderer root)"]
        App["components/App.tsx\n(AppState provider, contexts)"]
        REPL["screens/REPL.tsx\n(main interactive loop)"]
        PromptInput["components/PromptInput/\n(text input, history)"]
        Messages["components/Messages.tsx\n(message list renderer)"]
    end

    subgraph Engine["Query Engine\n(query.ts · QueryEngine.ts)"]
        QE["query()\nrunQuery()\n(turn orchestration)"]
        TokenBudget["query/tokenBudget.ts"]
        StopHooks["query/stopHooks.ts"]
    end

    subgraph StateLayer["State Management\n(state/)"]
        AppStore["AppStateStore.ts\n(immutable state atom)"]
        Store["store.ts\n(createStore, dispatch)"]
    end

    subgraph ToolsLayer["Tools System\n(tools.ts · Tool.ts · tools/)"]
        ToolRegistry["getTools()\n(tool list assembly)"]
        BashTool["BashTool"]
        FileTools["FileRead/Write/EditTool"]
        GlobGrep["GlobTool · GrepTool"]
        AgentTool["AgentTool\n(sub-agent spawner)"]
        WebTools["WebFetch/SearchTool"]
        MCPTool["MCPTool\n(MCP-based tools)"]
        OtherTools["TodoWrite · TaskCreate\nNotebookEdit · LSP …"]
    end

    subgraph ServicesLayer["Services Layer"]
        subgraph API["API Service\n(services/api/)"]
            APIClient["claude.ts\n(Anthropic SDK caller)"]
            WithRetry["withRetry.ts"]
            Bootstrap["bootstrap.ts"]
        end
        subgraph MCPSvc["MCP Service\n(services/mcp/)"]
            MCPMgr["MCPConnectionManager.tsx"]
            MCPClient["client.ts"]
        end
        subgraph MemSvc["Session Memory\n(services/SessionMemory/)"]
            MemStore["sessionMemory.ts"]
        end
        subgraph CompactSvc["Compact Service\n(services/compact/)"]
            AutoCompact["autoCompact.ts"]
            Compact["compact.ts"]
        end
        subgraph Analytics["Analytics\n(services/analytics/)"]
            EventLog["index.ts (logEvent)"]
            GrowthBook["growthbook.ts"]
        end
    end

    subgraph Commands["Commands\n(commands/)"]
        SlashCmds["Slash command handlers\n(/help /clear /compact …)"]
    end

    subgraph Coordinator["Coordinator / Swarm\n(coordinator/ · utils/swarm/)"]
        CoordMode["coordinatorMode.ts"]
        Teammate["utils/teammate.ts"]
        SwarmBackends["swarm/backends/"]
    end

    subgraph External["External Systems"]
        AnthropicAPI["Anthropic API\n(api.anthropic.com)"]
        BedrockVertex["AWS Bedrock / GCP Vertex"]
        MCPServers["MCP Servers\n(local or remote)"]
        Filesystem["Local Filesystem"]
        ShellOS["OS / Shell"]
        WebInternet["Web / Internet"]
    end

    STDIN --> CLIEntry
    CLIEntry --> MainTSX
    MainTSX --> InkRoot
    InkRoot --> App
    App --> REPL
    REPL --> PromptInput
    REPL --> Messages
    REPL --> QE
    QE --> AppStore
    QE --> ToolRegistry
    QE --> APIClient
    AppStore --> Store
    ToolRegistry --> BashTool
    ToolRegistry --> FileTools
    ToolRegistry --> GlobGrep
    ToolRegistry --> AgentTool
    ToolRegistry --> WebTools
    ToolRegistry --> MCPTool
    ToolRegistry --> OtherTools
    BashTool --> ShellOS
    FileTools --> Filesystem
    GlobGrep --> Filesystem
    WebTools --> WebInternet
    MCPTool --> MCPMgr
    MCPMgr --> MCPClient
    MCPClient --> MCPServers
    APIClient --> WithRetry
    APIClient --> AnthropicAPI
    APIClient --> BedrockVertex
    QE --> MemStore
    QE --> AutoCompact
    QE --> EventLog
    AgentTool --> Coordinator
    Coordinator --> CoordMode
    Coordinator --> Teammate
    Teammate --> SwarmBackends
    REPL --> SlashCmds
    Messages --> STDOUT
```

---

## 3. Top-Level Sequence Diagram — Interactive Session

```mermaid
sequenceDiagram
    actor User
    participant CLI as cli.tsx / main.tsx
    participant Init as init.ts (startup)
    participant UI as REPL.tsx (Ink/React)
    participant QE as query.ts (Query Engine)
    participant Tools as Tools System
    participant API as services/api/claude.ts
    participant Anthropic as Anthropic API

    User->>CLI: run `claude`
    CLI->>Init: init() — load config, auth, MCP, GrowthBook, policy
    Init-->>CLI: ready
    CLI->>UI: launchRepl() → render <App><REPL/>

    loop Each conversation turn
        User->>UI: type prompt, press Enter
        UI->>QE: runQuery(messages, tools, systemPrompt)
        QE->>API: streamMessage(messages, tools)
        API->>Anthropic: POST /v1/messages (streaming)
        Anthropic-->>API: token stream (text + tool_use blocks)
        API-->>QE: StreamEvent stream

        loop Tool use blocks
            QE->>Tools: executeTool(toolName, input)
            alt BashTool
                Tools->>Tools: spawn shell command
            else FileEditTool
                Tools->>Tools: read/patch/write file
            else AgentTool
                Tools->>Tools: fork sub-agent (recursive query)
            else MCPTool
                Tools->>Tools: call MCP server
            else WebFetchTool
                Tools->>Tools: HTTP request
            end
            Tools-->>QE: ToolResult
        end

        QE->>QE: check autoCompact / token budget
        QE-->>UI: updated messages (AssistantMessage + ToolResults)
        UI-->>User: render response in terminal
    end

    User->>UI: /exit or Ctrl-C
    UI->>CLI: graceful shutdown
```

---

## 4. Module: CLI Entry & Bootstrap

### Purpose
`entrypoints/cli.tsx` is the program's true entry point. It handles fast-paths (version, MCP server modes) before loading the full `main.tsx`. `main.tsx` uses **Commander.js** to parse all flags/subcommands and calls `init()` to prepare the runtime environment.

### Component Diagram

```mermaid
graph LR
    subgraph CLIEntry["entrypoints/cli.tsx"]
        FastVersion["--version fast-path\n(zero imports)"]
        SlowPaths["other flags\n→ load main.tsx"]
    end

    subgraph MainTSX["main.tsx"]
        CmdrRoot["Commander root program"]
        subgraph Commands["sub-commands"]
            Default["default (REPL)"]
            NonInteractive["-p prompt (headless)"]
            Resume["resume"]
            APIKeyCmd["apikey subcommands"]
        end
        Init["init() — entrypoints/init.ts"]
    end

    subgraph InitModule["entrypoints/init.ts"]
        MDMRead["MDM raw read (settings)"]
        KeychainPrefetch["Keychain prefetch"]
        ConfigLoad["enableConfigs()"]
        AuthCheck["checkAuth()"]
        MCPInit["initMCP()"]
        GrowthBook["initGrowthBook()"]
        PolicyLoad["loadPolicyLimits()"]
        RemoteSettings["loadRemoteManagedSettings()"]
    end

    CLIEntry --> MainTSX
    MainTSX --> Init
    Init --> InitModule
    CmdrRoot --> Default
    CmdrRoot --> NonInteractive
    CmdrRoot --> Resume
    CmdrRoot --> APIKeyCmd
```

### Sequence Diagram — Startup

```mermaid
sequenceDiagram
    participant OS as OS / Node.js
    participant CLI as cli.tsx
    participant Main as main.tsx
    participant Init as init.ts
    participant Config as utils/config.ts
    participant Auth as utils/auth.ts
    participant MCP as services/mcp/
    participant GB as services/analytics/growthbook.ts
    participant Policy as services/policyLimits/

    OS->>CLI: node cli.js [args]
    CLI->>CLI: parse args (fast-path --version?)
    CLI->>Main: dynamic import main.tsx
    Main->>Main: startMdmRawRead() + startKeychainPrefetch()
    Main->>Init: init()
    Init->>Config: enableConfigs() — load global + project CLAUDE.md
    Init->>Auth: checkHasTrustDialogAccepted()
    Init->>Auth: getGlobalConfig() — API key / OAuth
    Init->>MCP: prefetchOfficialMcpUrls()
    Init->>GB: initializeGrowthBook()
    Init->>Policy: loadPolicyLimits()
    Init-->>Main: ready
    Main->>Main: Commander parse → choose subcommand
    Main->>Main: launchRepl() or headless run
```

---

## 5. Module: Terminal UI (Ink / React)

### Purpose
The UI layer renders an interactive terminal application using **Ink** (a React renderer for terminals). It manages:
- The prompt input area
- The scrollable message history
- Dialogs (permissions, settings, onboarding, MCP approval, …)
- Status line (model, cost, connection)

### Component Diagram

```mermaid
graph TB
    subgraph InkRenderer["ink.ts — Ink Renderer Root"]
        InkRender["render() / renderAndRun()"]
    end

    subgraph AppComponent["components/App.tsx"]
        AppStateProvider["AppStateProvider\n(state context)"]
        VoiceProvider["VoiceProvider (opt)"]
        MailboxProvider["MailboxProvider"]
        SettingsChange["useSettingsChange hook"]
    end

    subgraph REPLScreen["screens/REPL.tsx"]
        PromptInput["PromptInput/\n(multi-line editor)"]
        VirtualMessageList["VirtualMessageList.tsx\n(virtualized scroll)"]
        StatusLine["StatusLine.tsx\n(model · cost · mode)"]
        Dialogs["Dialogs layer\n(MCPApproval · TrustDialog · …)"]
    end

    subgraph MessageComponents["Message Rendering"]
        MsgRow["MessageRow.tsx"]
        MsgResponse["MessageResponse.tsx"]
        Markdown["Markdown.tsx"]
        StructuredDiff["StructuredDiff.tsx"]
        HighlightedCode["HighlightedCode.tsx"]
        FileEditDiff["FileEditToolDiff.tsx"]
    end

    subgraph HooksUI["React Hooks (hooks/)"]
        useCommandQueue["useCommandQueue"]
        useQueueProcessor["useQueueProcessor"]
        useGlobalKeybindings["useGlobalKeybindings"]
        useIDEIntegration["useIDEIntegration"]
        usePasteHandler["usePasteHandler"]
    end

    InkRender --> AppComponent
    AppComponent --> REPLScreen
    REPLScreen --> PromptInput
    REPLScreen --> VirtualMessageList
    REPLScreen --> StatusLine
    REPLScreen --> Dialogs
    VirtualMessageList --> MsgRow
    MsgRow --> MsgResponse
    MsgResponse --> Markdown
    MsgResponse --> StructuredDiff
    MsgResponse --> HighlightedCode
    MsgResponse --> FileEditDiff
    REPLScreen --> HooksUI
```

### Sequence Diagram — User Input → Render

```mermaid
sequenceDiagram
    actor User
    participant PI as PromptInput
    participant RH as useCommandQueue / useQueueProcessor
    participant REPL as REPL.tsx
    participant QE as query.ts
    participant VML as VirtualMessageList
    participant MR as MessageRow / MessageResponse

    User->>PI: type text + Enter
    PI->>RH: enqueue(userMessage)
    RH->>REPL: process queue item
    REPL->>QE: runQuery(messages, tools)
    Note over REPL,QE: streaming begins
    QE-->>REPL: StreamEvent (text delta)
    REPL->>VML: update messages state
    VML->>MR: re-render row
    MR-->>User: incremental text appears
    QE-->>REPL: StreamEvent (tool_use)
    REPL-->>User: show spinner / tool progress
    QE-->>REPL: StreamEvent (stop)
    REPL->>VML: finalize messages
    VML-->>User: complete response visible
```

---

## 6. Module: Query Engine

### Purpose
`query.ts` / `QueryEngine.ts` is the **core conversation loop**. For each user turn it:
1. Prepends system context + memory attachments
2. Calls the Anthropic API with streaming
3. Executes tool calls in the response
4. Appends tool results and recurses (multi-turn tool use)
5. Manages token budget, auto-compact, and stop hooks

### Component Diagram

```mermaid
graph TB
    subgraph QE["query.ts — runQuery()"]
        PrepMessages["prepareMessages()\n(normalise + attach memories)"]
        APICall["callAPI()\n→ services/api/claude.ts"]
        StreamProc["processStream()\n(handle StreamEvents)"]
        ToolDispatch["dispatchTools()\n(parallel tool execution)"]
        RecurseCheck["recurse?\n(more tool calls?)"]
        CompactCheck["autoCompact check\n(services/compact/)"]
        StopHooks["stopHooks.ts\n(halt conditions)"]
        TokenBudget["tokenBudget.ts\n(warn / enforce limits)"]
    end

    subgraph Attach["utils/attachments.ts"]
        MemoryAttach["memory files"]
        ContextAttach["context injections"]
    end

    subgraph MsgUtils["utils/messages.ts"]
        NormMsg["normalizeMessagesForAPI()"]
        CreateMsg["createUserMessage() / createAssistantMessage()"]
    end

    QE --> Attach
    QE --> MsgUtils
    QE --> APICall
    QE --> ToolDispatch
    QE --> CompactCheck
    QE --> StopHooks
    QE --> TokenBudget
    RecurseCheck -->|yes| APICall
    RecurseCheck -->|no| QE
```

### Sequence Diagram — Multi-Turn Tool Use

```mermaid
sequenceDiagram
    participant REPL as REPL.tsx
    participant QE as query.ts (runQuery)
    participant Attach as attachments.ts
    participant API as services/api/claude.ts
    participant ToolSys as Tools System
    participant Compact as services/compact/

    REPL->>QE: runQuery(messages, tools, systemPrompt)
    QE->>Attach: getAttachmentMessages() — inject memories/context
    QE->>API: streamMessage(preparedMessages, tools)
    API-->>QE: stream: text delta …

    loop tool_use in response
        API-->>QE: stream: tool_use block (name, input)
        QE->>ToolSys: executeTool(name, input, context)
        ToolSys-->>QE: ToolResult
        QE->>QE: append tool_result to messages
    end

    API-->>QE: stream: stop_reason = end_turn | tool_use
    QE->>QE: stopHooks.ts — check for early exit conditions
    QE->>Compact: calculateTokenWarningState()
    alt token budget exceeded
        Compact->>QE: trigger autoCompact()
        QE->>API: compact conversation → summary
    end

    alt more tool results pending
        QE->>API: streamMessage(messages + tool_results)
        Note over QE,API: recursive turn
    end

    QE-->>REPL: final messages array
```

---

## 7. Module: Tools System

### Purpose
Every capability Claude can invoke is a **`Tool`** object defined in `Tool.ts`. `tools.ts` assembles the active tool list based on feature flags and user context. Tools are split between:
- **Built-in tools** (shipped with Claude Code)
- **MCP tools** (dynamically from MCP servers)

### Component Diagram

```mermaid
graph TB
    subgraph Registry["tools.ts — getTools()"]
        Assembly["assemble tool list\n(feature flags + user type)"]
    end

    subgraph ToolInterface["Tool.ts — Tool interface"]
        Name["name: string"]
        Description["description"]
        InputSchema["inputSchema (Zod / JSON Schema)"]
        Call["call(input, ctx) → ToolResult"]
        RenderResult["renderToolUseMessage() (React)"]
        Permissions["isReadOnly · needsPermission"]
    end

    subgraph BuiltIn["Built-in Tools"]
        BashTool["BashTool\n(shell command execution)"]
        FileReadTool["FileReadTool\n(read file or range)"]
        FileWriteTool["FileWriteTool\n(create file)"]
        FileEditTool["FileEditTool\n(str-replace patch)"]
        GlobTool["GlobTool\n(glob pattern search)"]
        GrepTool["GrepTool\n(ripgrep-based search)"]
        WebFetchTool["WebFetchTool\n(HTTP GET, HTML→MD)"]
        WebSearchTool["WebSearchTool\n(search API)"]
        AgentTool["AgentTool\n(sub-agent fork)"]
        TodoWriteTool["TodoWriteTool\n(task list)"]
        NotebookEditTool["NotebookEditTool\n(Jupyter)"]
        TaskTools["TaskCreate/List/Stop/…\n(background tasks)"]
    end

    subgraph MCPToolLayer["MCP-based Tools"]
        MCPTool["MCPTool\n(proxy to MCP server)"]
        MCPMgr["MCPConnectionManager"]
    end

    subgraph Security["Permission / Security"]
        PermCheck["bashPermissions.ts\n(allow/deny lists)"]
        BashSec["bashSecurity.ts\n(dangerous command detection)"]
        PathVal["pathValidation.ts\n(sandbox boundaries)"]
    end

    Assembly --> BuiltIn
    Assembly --> MCPToolLayer
    BuiltIn --> ToolInterface
    MCPToolLayer --> ToolInterface
    BashTool --> Security
    FileEditTool --> Security
    MCPTool --> MCPMgr
```

### Sequence Diagram — Tool Execution (BashTool example)

```mermaid
sequenceDiagram
    participant QE as query.ts
    participant TR as Tool Registry
    participant BT as BashTool
    participant Perm as Permission Check
    participant Shell as OS Shell (execa)
    participant UI as REPL.tsx (UI)

    QE->>TR: findToolByName("Bash")
    TR-->>QE: BashTool instance
    QE->>BT: call({ command: "ls -la" }, context)
    BT->>Perm: checkPermission(command, context)
    alt permission denied
        Perm-->>BT: PermissionResult.deny
        BT-->>QE: ToolResult error "Permission denied"
    else permission granted
        Perm-->>BT: PermissionResult.allow
        BT->>UI: emit BashProgress (running…)
        BT->>Shell: execa(command, { timeout, cwd })
        Shell-->>BT: stdout / stderr / exitCode
        BT->>UI: emit BashProgress (done)
        BT-->>QE: ToolResult { output, exitCode }
    end
```

### Sequence Diagram — AgentTool (sub-agent fork)

```mermaid
sequenceDiagram
    participant QE as Parent Query Engine
    participant AT as AgentTool
    participant Fork as forkSubagent.ts
    participant SubQE as Sub-agent Query Engine
    participant API as Anthropic API

    QE->>AT: call({ prompt, tools[] })
    AT->>Fork: forkSubagent(agentDef, parentContext)
    Fork->>SubQE: new runQuery(subMessages, subTools)
    SubQE->>API: streamMessage (sub-agent system prompt)
    API-->>SubQE: response + tool_uses
    SubQE->>SubQE: execute sub-tools recursively
    SubQE-->>Fork: final result
    Fork-->>AT: AgentResult
    AT-->>QE: ToolResult (sub-agent output)
```

---

## 8. Module: State Management

### Purpose
`state/` implements a minimal **reactive state store** (similar to Zustand) wrapping React's `useSyncExternalStore`. `AppState` is an immutable snapshot; mutations go through `setState()`.

### Component Diagram

```mermaid
graph TB
    subgraph StateModule["state/"]
        AppStateStore["AppStateStore.ts\n— AppState type\n— getDefaultAppState()"]
        Store["store.ts\n— createStore()\n— getState() / setState() / subscribe()"]
        AppStateTSX["AppState.tsx\n— AppStateProvider (React context)\n— AppStoreContext"]
        Selectors["selectors.ts\n— derived selectors"]
        OnChange["onChangeAppState.ts\n— side-effect callbacks"]
    end

    subgraph Consumers["Consumers"]
        REPL["REPL.tsx"]
        QE["query.ts"]
        Components["UI components\n(via useContext / useSyncExternalStore)"]
    end

    AppStateStore --> Store
    Store --> AppStateTSX
    AppStateTSX --> REPL
    AppStateTSX --> Components
    Store --> QE
    Selectors --> Components
    OnChange --> Store
```

### Sequence Diagram — State Update

```mermaid
sequenceDiagram
    participant QE as query.ts
    participant Store as store.ts
    participant Subs as React subscribers
    participant UI as UI components

    QE->>Store: setState({ messages: [...newMessages] })
    Store->>Store: merge + freeze new state snapshot
    Store->>Subs: notify all subscribers
    Subs->>UI: useSyncExternalStore re-render
    UI-->>User: updated terminal output
```

---

## 9. Module: API Service

### Purpose
`services/api/` wraps the **Anthropic SDK** (and Bedrock/Vertex adapters). Key responsibilities:
- Building the API request (model, messages, tools, betas, cache headers)
- Streaming token-by-token
- Retry logic with back-off
- Cost / token tracking
- Error normalisation

### Component Diagram

```mermaid
graph TB
    subgraph APIService["services/api/"]
        ClaudeTS["claude.ts\n(streamMessage — main entry)"]
        Client["client.ts\n(SDK client factory)"]
        WithRetry["withRetry.ts\n(exponential back-off)"]
        Errors["errors.ts / errorUtils.ts"]
        Usage["usage.ts\n(token accounting)"]
        Bootstrap["bootstrap.ts\n(prefetch config)"]
        FilesAPI["filesApi.ts\n(file upload / download)"]
        Logging["logging.ts\n(API request logs)"]
    end

    subgraph SDK["@anthropic-ai/sdk"]
        BetaMessages["beta.messages.stream()"]
    end

    subgraph Providers["Cloud Providers"]
        Anthropic["api.anthropic.com"]
        Bedrock["AWS Bedrock"]
        Vertex["GCP Vertex AI"]
    end

    ClaudeTS --> Client
    ClaudeTS --> WithRetry
    ClaudeTS --> Errors
    ClaudeTS --> Usage
    ClaudeTS --> Logging
    Client --> SDK
    SDK --> Anthropic
    Client --> Bedrock
    Client --> Vertex
    Bootstrap --> ClaudeTS
```

### Sequence Diagram — Stream API Call

```mermaid
sequenceDiagram
    participant QE as query.ts
    participant Claude as claude.ts (streamMessage)
    participant Retry as withRetry.ts
    participant Client as client.ts (SDK)
    participant Anthropic as Anthropic API

    QE->>Claude: streamMessage(messages, tools, systemPrompt, model)
    Claude->>Claude: buildRequest (add betas, cache headers, attribution)
    Claude->>Retry: withRetry(() => client.stream(request))
    Retry->>Client: beta.messages.stream(params)
    Client->>Anthropic: HTTPS POST /v1/messages (SSE)
    loop SSE events
        Anthropic-->>Client: text_delta / tool_use / usage
        Client-->>Retry: StreamEvent
        Retry-->>Claude: StreamEvent
        Claude-->>QE: yield StreamEvent
    end
    Anthropic-->>Client: message_stop
    Client-->>Claude: final usage stats
    Claude->>Claude: logTokenUsage()
    Claude-->>QE: done
```

---

## 10. Module: MCP Service

### Purpose
`services/mcp/` manages **Model Context Protocol** server connections. It:
- Discovers servers from config (local, remote, VS Code SDK)
- Maintains persistent connections (stdio / SSE / WebSocket)
- Exposes MCP tools and resources to Claude
- Handles OAuth for remote MCP servers

### Component Diagram

```mermaid
graph TB
    subgraph MCPService["services/mcp/"]
        ConnMgr["MCPConnectionManager.tsx\n(lifecycle: connect/disconnect)"]
        Client["client.ts\n(MCP SDK client wrapper)"]
        Config["config.ts\n(load server configs)"]
        Types["types.ts\n(MCPServerConfig · ServerResource)"]
        Auth["auth.ts\n(OAuth for remote MCP)"]
        OfficialReg["officialRegistry.ts\n(Anthropic-curated servers)"]
        InProcess["InProcessTransport.ts\n(in-process MCP)"]
        SdkControl["SdkControlTransport.ts\n(VS Code SDK bridge)"]
        Elicitation["elicitationHandler.ts\n(interactive prompts from MCP)"]
    end

    subgraph MCPServers["MCP Servers (external)"]
        LocalStdio["Local stdio server\n(spawned subprocess)"]
        RemoteSSE["Remote SSE server\n(HTTPS)"]
        VSCodeSdk["VS Code SDK MCP\n(in-process)"]
    end

    subgraph ToolsLayer["Tool Registry"]
        MCPTool["MCPTool instances\n(one per MCP tool)"]
    end

    ConnMgr --> Client
    ConnMgr --> Config
    ConnMgr --> Auth
    Client --> LocalStdio
    Client --> RemoteSSE
    Client --> VSCodeSdk
    ConnMgr --> MCPTool
    InProcess --> Client
    SdkControl --> Client
    Elicitation --> ConnMgr
```

### Sequence Diagram — MCP Tool Call

```mermaid
sequenceDiagram
    participant QE as query.ts
    participant MCPTool as MCPTool.call()
    participant ConnMgr as MCPConnectionManager
    participant MCPClient as MCP Client (SDK)
    participant Server as MCP Server (external)

    QE->>MCPTool: call({ toolName, input })
    MCPTool->>ConnMgr: getConnection(serverName)
    ConnMgr-->>MCPTool: MCPServerConnection
    MCPTool->>MCPClient: client.callTool({ name, arguments })
    MCPClient->>Server: JSON-RPC tools/call request
    Server-->>MCPClient: JSON-RPC result / error
    MCPClient-->>MCPTool: ToolResult
    MCPTool-->>QE: ToolResult
```

---

## 11. Module: Session Memory & Compact

### Purpose

**Session Memory** (`services/SessionMemory/`) persists facts learned during a session into `~/.claude/memory/` files, which are injected as attachments on subsequent turns.

**Compact** (`services/compact/`) manages the context window: when the conversation approaches the token limit it either auto-compacts (summarises older turns) or warns the user.

### Component Diagram

```mermaid
graph TB
    subgraph MemModule["services/SessionMemory/"]
        MemStore["sessionMemory.ts\n(read / write memory files)"]
        MemUtils["sessionMemoryUtils.ts\n(parse · format)"]
        MemPrompts["prompts.ts\n(memory extraction system prompt)"]
    end

    subgraph CompactModule["services/compact/"]
        AutoCompact["autoCompact.ts\n(shouldAutoCompact · trigger)"]
        CompactFn["compact.ts\n(buildPostCompactMessages)"]
        Micro["microCompact.ts\n(light-weight within-turn compact)"]
        Prompt["prompt.ts\n(compact system prompt)"]
        TimeConfig["timeBasedMCConfig.ts"]
    end

    subgraph Attach["utils/attachments.ts"]
        InjectMem["filterDuplicateMemoryAttachments()\ngetAttachmentMessages()"]
    end

    subgraph QE["query.ts"]
        TokenCheck["calculateTokenWarningState()"]
    end

    MemStore --> MemUtils
    MemStore --> MemPrompts
    InjectMem --> MemStore
    QE --> InjectMem
    QE --> AutoCompact
    AutoCompact --> CompactFn
    CompactFn --> Prompt
    Micro --> CompactFn
```

### Sequence Diagram — Auto-Compact

```mermaid
sequenceDiagram
    participant QE as query.ts
    participant AC as autoCompact.ts
    participant CF as compact.ts
    participant API as Anthropic API
    participant State as AppStateStore

    QE->>AC: calculateTokenWarningState(usage, model)
    AC-->>QE: { shouldCompact: true, tokenCount }
    QE->>CF: buildPostCompactMessages(messages)
    CF->>API: streamMessage(compactPrompt + messages)
    API-->>CF: summary text
    CF-->>QE: [systemMessage(summary), …recentMessages]
    QE->>State: setState({ messages: compacted })
    Note over QE: conversation continues with compacted context
```

---

## 12. Module: Commands (Slash Commands)

### Purpose
`commands/` contains handlers for every **slash command** (`/help`, `/clear`, `/compact`, `/memory`, `/mcp`, `/model`, `/permissions`, …). Each command is a `Command` object with a `handler` function and optional React UI.

### Component Diagram

```mermaid
graph TB
    subgraph CmdRegistry["commands.ts — getCommands()"]
        CmdList["Command[] list\n(assembled at startup)"]
    end

    subgraph CmdInterface["types/command.ts — Command interface"]
        CName["name: string"]
        CDescription["description"]
        CHandler["handler(args, context) → void"]
        CRenderUI["renderInChat? (React component)"]
    end

    subgraph SampleCmds["Sample command modules"]
        Help["commands/help/"]
        Clear["commands/clear/"]
        Compact["commands/compact/"]
        Memory["commands/memory/"]
        MCPCmd["commands/mcp/"]
        Model["commands/model/"]
        Permissions["commands/permissions/"]
        Skills["commands/skills/"]
        Tasks["commands/tasks/"]
        Config["commands/config/"]
        Login["commands/login/"]
        Logout["commands/logout/"]
    end

    subgraph Parser["utils/slashCommandParsing.ts"]
        Parse["parseSlashCommand(input)"]
    end

    CmdRegistry --> SampleCmds
    SampleCmds --> CmdInterface
    Parser --> CmdRegistry
```

### Sequence Diagram — Slash Command Execution

```mermaid
sequenceDiagram
    actor User
    participant PI as PromptInput
    participant Parser as slashCommandParsing.ts
    participant Registry as commands.ts
    participant CmdHandler as Command.handler()
    participant UI as REPL.tsx

    User->>PI: type "/compact" + Enter
    PI->>Parser: parseSlashCommand("/compact")
    Parser-->>PI: { name: "compact", args: [] }
    PI->>Registry: findCommand("compact")
    Registry-->>PI: CompactCommand
    PI->>CmdHandler: handler(args, context)
    CmdHandler->>UI: dispatch compact action
    UI-->>User: show compact progress
```

---

## 13. Module: Multi-Agent / Swarm Coordinator

### Purpose
Claude Code supports **multi-agent workflows** via the Swarm system. The coordinator (`coordinator/coordinatorMode.ts`) orchestrates a team of Teammate agents. Each agent runs its own query loop and communicates via mailbox messages.

### Component Diagram

```mermaid
graph TB
    subgraph CoordModule["coordinator/"]
        CoordMode["coordinatorMode.ts\n(orchestrate team)"]
    end

    subgraph SwarmUtils["utils/swarm/"]
        Backends["backends/\n(in-process · remote)"]
        InProcessRunner["inProcessRunner.ts"]
        SpawnUtils["spawnUtils.ts"]
        TeamHelpers["teamHelpers.ts"]
        TeammateInit["teammateInit.ts"]
        LayoutMgr["teammateLayoutManager.ts"]
        PermSync["permissionSync.ts"]
        LeaderBridge["leaderPermissionBridge.ts"]
    end

    subgraph TeammateUtils["utils/teammate.ts"]
        TmUtils["teammate lifecycle\n(create · connect · shutdown)"]
    end

    subgraph AgentTool["tools/AgentTool/"]
        Fork["forkSubagent.ts"]
        RunAgent["runAgent.ts"]
        Resume["resumeAgent.ts"]
    end

    subgraph Mailbox["context/mailbox.ts"]
        MailboxProv["MailboxProvider\n(message queue between agents)"]
    end

    CoordMode --> TeammateUtils
    TeammateUtils --> SwarmUtils
    SwarmUtils --> Backends
    Backends --> InProcessRunner
    InProcessRunner --> Fork
    Fork --> RunAgent
    AgentTool --> RunAgent
    RunAgent --> Mailbox
    PermSync --> LeaderBridge
```

### Sequence Diagram — Coordinator spawning Teammates

```mermaid
sequenceDiagram
    participant User
    participant Coord as coordinatorMode.ts
    participant TmUtils as utils/teammate.ts
    participant TmInit as swarm/teammateInit.ts
    participant TmAgent as Teammate Query Loop
    participant Mailbox as Mailbox (message bus)
    participant API as Anthropic API

    User->>Coord: start coordinator session
    Coord->>TmUtils: createTeammates(agentDefs)

    loop for each Teammate
        TmUtils->>TmInit: initTeammate(agentDef)
        TmInit->>TmAgent: spawn sub-process / in-process agent
        TmAgent->>Mailbox: subscribe(agentId)
    end

    Coord->>Mailbox: send task message to Teammate A
    Mailbox->>TmAgent: deliver message
    TmAgent->>API: streamMessage (agent turn)
    API-->>TmAgent: response
    TmAgent->>Mailbox: send result to Coordinator
    Mailbox->>Coord: deliver result
    Coord-->>User: aggregated output
```

---

## 14. Module: Analytics & Telemetry

### Purpose
`services/analytics/` provides an event-logging pipeline. Events are logged via `logEvent()` and routed to one or more **sinks** (Datadog, internal first-party endpoint, GrowthBook). Feature flags use GrowthBook for A/B experiments.

### Component Diagram

```mermaid
graph TB
    subgraph Analytics["services/analytics/"]
        LogEvent["index.ts — logEvent(eventName, metadata)"]
        Sink["sink.ts — EventSink interface"]
        Datadog["datadog.ts — DatadogSink"]
        FirstParty["firstPartyEventLogger.ts\n(internal endpoint)"]
        GrowthBook["growthbook.ts\n(feature flags · A/B)"]
        Metadata["metadata.ts\n(session · user context)"]
        SinkKill["sinkKillswitch.ts"]
    end

    subgraph Callers["Callers (throughout codebase)"]
        QE["query.ts"]
        Tools["tools/BashTool/ …"]
        Commands["commands/"]
        UI["components/"]
    end

    Callers --> LogEvent
    LogEvent --> Sink
    Sink --> Datadog
    Sink --> FirstParty
    LogEvent --> GrowthBook
    LogEvent --> Metadata
    Sink --> SinkKill
```

### Sequence Diagram — Event Logging

```mermaid
sequenceDiagram
    participant Tool as BashTool (example)
    participant LE as logEvent()
    participant Meta as metadata.ts
    participant Sink as EventSink(s)
    participant DD as Datadog
    participant FP as First-Party Endpoint

    Tool->>LE: logEvent("bash_execute", { exitCode, duration })
    LE->>Meta: enrich(sessionId, userId, model, …)
    Meta-->>LE: enriched event
    LE->>Sink: emit(enrichedEvent)
    Sink->>DD: POST metrics/events (async, batched)
    Sink->>FP: POST internal endpoint (async, batched)
```

---

## Appendix: Directory Quick-Reference

```
source/src/
├── entrypoints/          # cli.tsx (entry), init.ts (startup), mcp.ts, sdk/
├── main.tsx              # Commander CLI, flag parsing, launchRepl()
├── query.ts              # Core conversation loop (runQuery)
├── QueryEngine.ts        # QueryEngine class (wraps query.ts)
├── Tool.ts               # Tool interface + helpers
├── tools.ts              # getTools() — assembles tool list
├── tools/                # Individual tool implementations
│   ├── BashTool/         # Shell command execution
│   ├── FileReadTool/     # Read files
│   ├── FileWriteTool/    # Write/create files
│   ├── FileEditTool/     # Str-replace patch editing
│   ├── GlobTool/         # Glob pattern matching
│   ├── GrepTool/         # Ripgrep-based search
│   ├── AgentTool/        # Sub-agent fork & run
│   ├── WebFetchTool/     # HTTP fetch + HTML→Markdown
│   ├── WebSearchTool/    # Web search API
│   ├── MCPTool/          # Proxy to MCP server tool
│   ├── TodoWriteTool/    # Task/todo list management
│   ├── TaskCreateTool/   # Background task creation
│   └── NotebookEditTool/ # Jupyter notebook editing
├── state/                # AppState store (zustand-style)
├── components/           # React/Ink UI components
├── screens/              # Top-level screens (REPL, Doctor, Resume)
├── hooks/                # React hooks
├── commands/             # Slash command handlers
├── services/
│   ├── api/              # Anthropic SDK client, retry, usage
│   ├── mcp/              # MCP connection manager
│   ├── compact/          # Context window management
│   ├── SessionMemory/    # Persistent memory files
│   ├── analytics/        # Event logging, GrowthBook
│   ├── oauth/            # OAuth 2.0 flows
│   └── lsp/              # Language Server Protocol integration
├── coordinator/          # Multi-agent coordinator mode
├── utils/
│   ├── auth.ts           # Auth logic (API key, OAuth, Bedrock, Vertex)
│   ├── config.ts         # Global + project config (CLAUDE.md)
│   ├── git.ts            # Git helpers
│   ├── permissions/      # Tool permission framework
│   ├── swarm/            # Teammate / swarm helpers
│   ├── model/            # Model selection, cost
│   ├── settings/         # MDM, remote managed settings
│   └── memory/           # Memory file utilities
├── constants/            # Prompts, product config, OAuth scopes
└── types/                # Shared TypeScript types
```
