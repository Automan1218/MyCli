# MyCli (PaiCLI)

**English** | [简体中文](README.zh-CN.md)

> A production-grade Java Agent CLI, built as an answer to Claude Code. It evolved from a single-agent ReAct loop into Multi-Agent + TUI + MCP + Skills + image input across 21 development phases.

Banner version `v16.1.0`, Maven artifact `paicli-1.0-SNAPSHOT.jar` (the mismatch is expected).

```text
   ████████    PaiCLI π  v16.1.0
     ██  ██    Model glm-5.1 (glm)
     ██  ██    MCP 4/4 · 61 tools · 2/2 skills · ReAct
     ██  ██    ReAct · Plan · MCP · Browser · Image
     ██  ██

Tips for getting started:
1. Type / for commands and Tab completion
2. Ask coding questions, edit code or run commands
3. Attach context with @path or @image:
```

---

## Core Capabilities

| Capability | Description |
|------------|-------------|
| **Three execution paths** | ReAct (default) · Plan-and-Execute + DAG (`/plan`) · Multi-Agent (`/team`), sharing one tool/memory/snapshot stack |
| **Multi-model** | GLM-5.1 / GLM-5V-Turbo / DeepSeek V4 / StepFun / Kimi K2.6 / FreeLLMAPI, switch at runtime via `/model` |
| **Code search** | Real-time `glob_files` → `grep_code` → `read_file`; RAG semantic search (`search_code`) as a fallback |
| **Memory** | Short-term + long-term (explicit `/save`) + dynamic long-context budgeting + summary compaction |
| **MCP protocol** | stdio + Streamable HTTP servers; tools / resources / prompts auto-registered |
| **Web + browser** | `web_search` / `web_fetch` + Chrome DevTools MCP (SPA / logged-in / anti-scraping fallback) |
| **HITL + policy layer** | Human approval for dangerous ops + path fence + command fast-reject + structured audit (not a sandbox) |
| **Skill system** | Reusable "expert handbook" units the LLM loads on demand by scenario |
| **TUI** | inline streaming (default) / lanterna full-screen / plain fallback, switchable |
| **Image input** | Ctrl+V / `@image:` local images + MCP image content fed back into context |
| **Background tasks + Runtime API** | SQLite-persisted task queue + localhost HTTP API |
| **Side-Git snapshots** | Auto snapshot before/after each turn, one-key `/restore`, never touches your project `.git` |
| **LSP diagnostics** | Lightweight syntax diagnostics injected after `write_file` |

---

## Quick Start

### 1. Configure an API key

Copy `.env.example` to `.env` and fill in at least one provider key:

```bash
cp .env.example .env
# Edit .env, add any one of GLM / DeepSeek / StepFun / Kimi / FreeLLMAPI API key
```

Or use environment variables:

```bash
export GLM_API_KEY=your_api_key_here          # default provider
# or
export STEP_API_KEY=your_step_api_key_here
export KIMI_API_KEY=your_kimi_api_key_here
export FREELLMAPI_API_KEY=your_unified_key_here
export FREELLMAPI_BASE_URL=http://localhost:5173/v1
```

You can also persist config to `~/.paicli/config.json` from inside PaiCLI:

```text
/config provider freellmapi --base-url http://localhost:5173/v1 --api-key <key> --model auto
/model freellmapi
```

### 2. Build & run

```bash
mvn clean package                              # skips tests by default, produces a runnable jar
java -jar target/paicli-1.0-SNAPSHOT.jar
```

Or run from source:

```bash
mvn clean compile exec:java -Dexec.mainClass="com.paicli.cli.Main"
```

**Prerequisites:** Java 17+ / Maven / at least one API key. `ripgrep` optional (`grep_code` prefers it, falls back to a Java scan). RAG indexing needs a local Ollama running with `nomic-embed-text` pulled.

---

## Models

| Provider | Switch command | Default model | Context window |
|----------|----------------|---------------|----------------|
| GLM | `/model glm-5.1` / `/model glm-5v-turbo` | glm-5.1 | 200k |
| DeepSeek | `/model deepseek` | deepseek-v4-flash | 1M |
| StepFun | `/model step` | step-3.5-flash | 256k |
| Kimi | `/model kimi` | kimi-k2.6 | 256k |
| FreeLLMAPI | `/model freellmapi` | auto | 128k (conservative) |

- `LlmClient` interface + `AbstractOpenAiCompatibleClient` template base — adding a provider is ~20 lines
- Default model persisted to `~/.paicli/config.json`; API keys read from config / env / `.env`
- `glm-5v-turbo` supports image input; under DeepSeek V4 / Kimi thinking mode, `reasoning_content` is carried back into the next request

---

## Tools

11 core built-in tools:

| Tool | Purpose |
|------|---------|
| `read_file` | Read file content |
| `write_file` | Write a file (5MB cap, triggers LSP diagnostics on success) |
| `list_dir` | List a directory |
| `glob_files` | Glob project files by name in real time (read-only, skips build/dep dirs) |
| `grep_code` | Keyword/regex search in real time, prefers ripgrep, returns line numbers, context, partial state |
| `execute_command` | Run a short shell command (60s timeout, blacklist rejects destructive commands) |
| `create_project` | Scaffold a project (java/python/node) |
| `search_code` | RAG semantic search (natural-language query; fuzzy or when normal search fails) |
| `web_search` | Search the web for real-time info |
| `web_fetch` | Fetch a URL and extract Markdown body |
| `revert_turn` | Restore to the Nth most recent pre-turn snapshot (via HITL + audit) |

Plus dynamic MCP tools `mcp__{server}__{tool}` and resource virtual tools.

**Parallel execution:** when a turn returns multiple `tool_calls`, they run in parallel (max 4 concurrent), results fed back in original order. All three paths share one batch execution entry.

**Policy layer:** file tools are confined to the project root; `execute_command` blacklists `sudo` / `rm -rf` of root / `mkfs` / `dd of=/dev` / fork bombs / `curl|sh`; all `mcp__` tools trigger HITL + audit. See `/policy`.

---

## Commands

**Execution modes**
- `/plan [task]` — Plan-and-Execute mode (with a task, runs it directly, then returns to ReAct)
- `/team [task]` — Multi-Agent collaboration mode
- `/cancel` — request cancellation of the running task

**Security / approval**
- `/hitl` / `/hitl on` / `/hitl off` — approval status / enable / disable
- `/policy` — view security policy state
- `/audit [N]` — view the last N dangerous-tool audit entries today (default 10)

**MCP**
- `/mcp` — all server states
- `/mcp restart|logs|disable|enable <name>` — manage one server
- `/mcp resources|prompts <name>` — view a server's resources / prompts

**Memory**
- `/memory` / `/mem` — memory system state
- `/memory list|search <kw>|delete <id>|clear` — manage long-term memory
- `/save <fact>` — save a project-scoped fact; `/save --global <fact>` for cross-project prefs

**Codebase / RAG**
- `/index [path]` — index the codebase (default current dir)
- `/search <query>` — semantic search
- `/graph <class>` — view code relationship graph

**Snapshot / browser / tasks**
- `/snapshot` / `/snapshot status|clean` — Side-Git snapshots
- `/restore <N>` — restore to the Nth most recent pre-turn snapshot
- `/browser status|connect [port]|disconnect|tabs` — CDP session reuse
- `/task` / `/task add <task>|cancel <id>|log <id>` — background tasks

**Skill / general**
- `/skill list|show <name>|on <name>|off <name>|reload` — Skill management
- `/clear` — clear conversation history / short-term memory / Skill buffer (long-term memory kept)
- `/context` — context mode / cache mode / RAG topK state
- `/model <provider>` — switch model
- `/config` — configuration
- `/exit` / `/quit` — exit

> An unrecognized `/xxx` is reported as "unknown command" at the CLI layer and not passed to the Agent.

---

## TUI variants

| Variant | How to enable | Style |
|---------|---------------|-------|
| **inline streaming** (default) | run directly / `PAICLI_RENDERER=inline` | Claude Code style: π color splash, JLine bottom dock, inline collapsible tool blocks, inline diff, single-char HITL prompt |
| **lanterna full-screen** | `PAICLI_RENDERER=lanterna` (legacy `PAICLI_TUI=true`) | three-pane full-screen: file tree + conversation + status bar + input bar |
| **plain fallback** | `PAICLI_RENDERER=plain` | raw println, no folding / status bar |

- `PAICLI_NO_STATUSBAR=true` — disable the bottom dock in inline mode
- `NO_COLOR=1` — disable all ANSI colors, keep layout
- Conversation history saved to `~/.paicli/history/session_*.jsonl`

---

## MCP configuration

The MCP subsystem is on by default. If `~/.paicli/mcp.json` is missing, a default chrome-devtools config is created. User-level `~/.paicli/mcp.json` and project-level `.paicli/mcp.json` are merged (project-level overrides by server name):

```json
{
  "mcpServers": {
    "fetch":  { "command": "uvx", "args": ["mcp-server-fetch"] },
    "git":    { "command": "uvx", "args": ["mcp-server-git", "--repository", "${PROJECT_DIR}"] },
    "remote": { "url": "https://mcp.example.com/v1", "headers": {"Authorization": "Bearer ${REMOTE_TOKEN}"} },
    "step_search": { "url": "https://api.stepfun.com/step_plan/v1/mcp/web_search/mcp", "headers": {"Authorization": "Bearer ${STEP_API_KEY}"} }
  }
}
```

- `command` = stdio server, `url` = Streamable HTTP server
- `${PROJECT_DIR}` / `${HOME}` are built-in; other `${VAR}` read from env / system properties / project `.env` / user `~/.env`
- When `STEP_API_KEY` is detected, a built-in `step_search` remote MCP is added; for `step-3.7-flash*` models, `web_search` / `web_fetch` proxy to it preferentially
- Startup MCP never blocks the first screen: waits up to 8s by default (`PAICLI_MCP_STARTUP_WAIT_SECONDS`), keeps unfinished servers `STARTING` in the background; `/mcp` shows latest state
- OAuth and `sampling/createMessage` are not yet implemented

**Browser login state:** on Chrome 144+, open `chrome://inspect/#remote-debugging` and check `Allow remote debugging`; the Agent calls `browser_connect` automatically on a login page. On older versions, start Chrome with `--remote-debugging-port` then `/browser connect <port>`.

---

## Runtime API

```bash
PAICLI_RUNTIME_API_KEY=your_local_api_key \
  java -jar target/paicli-1.0-SNAPSHOT.jar serve --http --port 8080
```

- Listens on `127.0.0.1` only, requires `PAICLI_RUNTIME_API_KEY`
- Endpoints: `POST /v1/threads`, `POST /v1/threads/{id}/turns`, `GET /v1/threads/{id}/events`
- Headers: `Authorization: Bearer <key>` or `X-PaiCLI-API-Key: <key>`

---

## Security model (why it's not called a sandbox)

Local Agent CLIs (cf. Claude Code / Cursor / Aider) don't ship container/VM sandboxes by default — sandboxes weaken the Agent, give false security, and hurt UX. Real production Agent sandboxes are microVM-level (Devin / Modal / Anthropic Computer Use use Firecracker / gVisor). PaiCLI's security model is **HITL + path checks + command fast-reject + audit**, not isolation.

---

## Testing

`mvn clean package` skips tests by default to produce a verifiable jar. Pick a regression scope by change area:

```bash
mvn test -Pquick                              # routine fast regression
mvn test -Pphase16-smoke                      # TUI / inline renderer smoke
mvn test -Dtest=XxxTest -DskipTests=false     # a single test class
mvn test -DskipTests=false                    # full (before release / large refactor)
```

| Module | Command |
|--------|---------|
| Code search | `mvn test -Dtest=ToolRegistryTest,CodeSearchGoldenSetTest,ApprovalPolicyTest` |
| Command parsing | `mvn test -Dtest=CliCommandParserTest,PlanReviewInputParserTest,MainInputNormalizationTest` |
| DAG/Plan | `mvn test -Dtest=ExecutionPlanTest` |
| Multi-Agent | `mvn test -Dtest=AgentRoleTest,AgentMessageTest,AgentOrchestratorTest` |
| RAG | `mvn test -Dtest=CodeChunkerTest,CodeAnalyzerTest,VectorStoreTest,CodeIndexTest` |

---

## Tech stack

Java 17 · Maven · OkHttp · Jackson · JLine 4 (terminal interaction / Status / input widgets) · SQLite (vector / graph / task persistence) · JavaParser (AST analysis + LSP syntax diagnostics) · JGit (Side-Git snapshots) · Jsoup (HTML extraction) · Ollama (local embeddings)

---

## Project structure

```
src/main/java/com/paicli/
├── agent/       Agent.java, PlanExecuteAgent.java, SubAgent.java, AgentOrchestrator.java
├── cli/         Main.java, CliCommandParser.java, PlanReviewInputParser.java
├── llm/         GLMClient, DeepSeekClient, StepClient, KimiClient, FreeLlmApiClient
├── context/     ContextProfile, ContextMode, TokenUsageFormatter
├── memory/      MemoryManager, ConversationHistoryCompactor, LongTermMemory
├── plan/        Planner, ExecutionPlan, Task
├── rag/         CodeIndex, CodeRetriever, VectorStore, CodeChunker
├── tool/        ToolRegistry
├── mcp/         McpClient, McpServerManager, transport/, resources/, mention/
├── hitl/        HitlToolRegistry, ApprovalPolicy, TerminalHitlHandler
├── web/         SearchProvider, WebFetcher, HtmlExtractor, NetworkPolicy
├── policy/      PathGuard, CommandGuard, AuditLog
├── skill/       SkillRegistry, SkillContextBuffer
├── render/      Renderer, InlineRenderer, PlainRenderer, RendererFactory
├── snapshot/    SideGitManager, SnapshotService
├── runtime/     api/ (RuntimeApiServer) + task/ (DurableTaskManager)
├── lsp/         LspManager, LspDiagnosticFormatter
├── browser/     BrowserSession, BrowserGuard, SensitivePagePolicy
├── image/       ImageReferenceParser
└── prompt/      PromptAssembler, PromptContext, PromptRepository
```

---

## Data & config directories

| Path | Purpose |
|------|---------|
| `~/.paicli/config.json` | Default model / provider config |
| `~/.paicli/memory/long_term_memory.json` | Long-term memory |
| `~/.paicli/rag/codebase.db` | Code index |
| `~/.paicli/snapshots/` | Side-Git snapshots (never touches your project `.git`) |
| `~/.paicli/tasks/tasks.db` | Background task queue |
| `~/.paicli/audit/` | Dangerous-tool audit JSONL (daily) |
| `~/.paicli/history/` | Conversation & input history |
| `~/.paicli/logs/paicli.log` | Debug log (auto rotate / cleanup) |
| `~/.paicli/skills/` | User-level Skills |
| `~/.paicli/mcp.json` | User-level MCP config |

> Long-term memory only stores stable facts saved with explicit intent (`/save`, or the Agent calling `save_memory` when you clearly say "remember this"). It is never auto-extracted, is project-scoped by default, and is auditable and deletable.

---

## Known boundaries

On the roadmap but **not delivered**: container/VM sandbox · MCP OAuth + sampling + server auto-restart · video / audio / image generation · TUI sixel image preview.
