# MyCli (PaiCLI)

[English](README.md) | **简体中文**

> 面向商业使用的 Java Agent CLI，对标 Claude Code。从 ReAct 单代理循环演进到 Multi-Agent + TUI + MCP + Skill + 图片输入，已交付 21 期。

Banner 版本 `v16.1.0`，Maven 产物 `paicli-1.0-SNAPSHOT.jar`（两者版本号不一致是正常状态）。

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

## 核心能力

| 能力 | 说明 |
|------|------|
| **三条执行路径** | ReAct（默认）· Plan-and-Execute + DAG（`/plan`）· Multi-Agent 协作（`/team`），共享同一套工具 / 记忆 / 快照 |
| **多模型** | GLM-5.1 / GLM-5V-Turbo / DeepSeek V4 / StepFun / Kimi K2.6 / FreeLLMAPI，运行时 `/model` 切换 |
| **代码搜索** | `glob_files` → `grep_code` → `read_file` 实时探索；RAG 语义检索（`search_code`）作辅助 |
| **记忆系统** | 短期记忆 + 长期记忆（显式 `/save`）+ 长上下文动态预算 + 摘要压缩 |
| **MCP 协议** | stdio + Streamable HTTP server，工具 / resources / prompts 自动注册 |
| **联网 + 浏览器** | `web_search` / `web_fetch` + Chrome DevTools MCP（SPA / 登录态 / 防爬墙兜底） |
| **HITL + 策略层** | 危险操作人工审批 + 路径围栏 + 命令快速拒绝 + 结构化审计（非沙箱） |
| **Skill 系统** | 可复用「专家手册」单元，LLM 按场景自决加载 |
| **TUI** | inline 流式（默认）/ lanterna 全屏 / plain 兜底，三形态可切换 |
| **图片输入** | Ctrl+V / `@image:` 本地图片 + MCP image content 回灌 |
| **后台任务 + Runtime API** | SQLite 持久化任务队列 + localhost HTTP API |
| **Side-Git 快照** | 每个 turn 前后自动快照，`/restore` 一键回滚，不写用户项目 `.git` |
| **LSP 诊断** | `write_file` 后注入轻量语法诊断 |

---

## 快速开始

### 1. 配置 API Key

复制 `.env.example` 为 `.env`，至少填一个 provider 的 Key：

```bash
cp .env.example .env
# 编辑 .env，填入 GLM / DeepSeek / StepFun / Kimi / FreeLLMAPI 任一 API Key
```

或走环境变量：

```bash
export GLM_API_KEY=your_api_key_here          # 默认 provider
# 或
export STEP_API_KEY=your_step_api_key_here
export KIMI_API_KEY=your_kimi_api_key_here
export FREELLMAPI_API_KEY=your_unified_key_here
export FREELLMAPI_BASE_URL=http://localhost:5173/v1
```

也可在 PaiCLI 内写入 `~/.paicli/config.json`：

```text
/config provider freellmapi --base-url http://localhost:5173/v1 --api-key <key> --model auto
/model freellmapi
```

### 2. 编译运行

```bash
mvn clean package                              # 默认跳过测试，产出可手工验收的 jar
java -jar target/paicli-1.0-SNAPSHOT.jar
```

或直接跑源码：

```bash
mvn clean compile exec:java -Dexec.mainClass="com.paicli.cli.Main"
```

**运行前提：** Java 17+ / Maven / 至少一个 API Key。`ripgrep` 可选（`grep_code` 优先用，未装自动回退 Java 扫描）。RAG 索引需本地 Ollama 已启动并拉取 `nomic-embed-text`。

---

## 模型

| Provider | 切换命令 | 默认模型 | 上下文窗口 |
|----------|----------|----------|-----------|
| GLM | `/model glm-5.1` / `/model glm-5v-turbo` | glm-5.1 | 200k |
| DeepSeek | `/model deepseek` | deepseek-v4-flash | 1M |
| StepFun | `/model step` | step-3.5-flash | 256k |
| Kimi | `/model kimi` | kimi-k2.6 | 256k |
| FreeLLMAPI | `/model freellmapi` | auto | 128k（保守） |

- `LlmClient` 接口 + `AbstractOpenAiCompatibleClient` 模板基类，新增 provider 约 20 行
- 默认模型持久化到 `~/.paicli/config.json`，API Key 可从配置 / 环境变量 / `.env` 读取
- `glm-5v-turbo` 支持图片输入；DeepSeek V4 / Kimi thinking 模式下 `reasoning_content` 随下一轮请求带回

---

## 工具

核心内置 11 个工具：

| 工具 | 作用 |
|------|------|
| `read_file` | 读取文件内容 |
| `write_file` | 写入文件（5MB 上限，成功后触发 LSP 诊断） |
| `list_dir` | 列出目录 |
| `glob_files` | 按文件名 glob 实时查找（只读，自动跳过构建/依赖目录） |
| `grep_code` | 关键字/正则实时搜索，优先 ripgrep，返回行号、上下文、partial 状态 |
| `execute_command` | 执行短时 Shell 命令（60 秒超时，黑名单拦截破坏性命令） |
| `create_project` | 创建项目结构（java/python/node） |
| `search_code` | RAG 语义检索（自然语言查询，模糊语义或常规搜索无果时辅助） |
| `web_search` | 联网搜索实时信息 |
| `web_fetch` | 抓取 URL 并提取正文 Markdown |
| `revert_turn` | 恢复到最近第 N 个 pre-turn 快照（走 HITL + 审计） |

外加 MCP 动态工具 `mcp__{server}__{tool}` 及 resources 虚拟工具。

**并行执行：** 同一轮返回多个 `tool_calls` 时并行执行（最多 4 并发），结果按原始顺序回灌。三条路径共用统一批量执行入口。

**策略层：** 文件类工具路径强制限定项目根内；`execute_command` 黑名单拦截 `sudo` / `rm -rf 全盘` / `mkfs` / `dd of=/dev` / fork bomb / `curl|sh` 等；所有 `mcp__` 工具默认触发 HITL + 审计。详见 `/policy`。

---

## 命令

**执行模式**
- `/plan [任务]` — Plan-and-Execute 模式（带任务则直接执行，完成后回 ReAct）
- `/team [任务]` — Multi-Agent 协作模式
- `/cancel` — 请求取消运行中任务

**安全 / 审批**
- `/hitl` / `/hitl on` / `/hitl off` — 人工审批状态 / 启用 / 关闭
- `/policy` — 查看安全策略状态
- `/audit [N]` — 查看今日最近 N 条危险工具审计（默认 10）

**MCP**
- `/mcp` — 所有 server 状态
- `/mcp restart|logs|disable|enable <name>` — 管理单个 server
- `/mcp resources|prompts <name>` — 查看 server 暴露的 resources / prompts

**记忆**
- `/memory` / `/mem` — 记忆系统状态
- `/memory list|search <关键词>|delete <id>|clear` — 管理长期记忆
- `/save <事实>` — 保存项目级事实；`/save --global <事实>` 保存跨项目偏好

**代码库 / RAG**
- `/index [路径]` — 索引代码库（默认当前目录）
- `/search <查询>` — 语义检索
- `/graph <类名>` — 查看代码关系图谱

**快照 / 浏览器 / 任务**
- `/snapshot` / `/snapshot status|clean` — Side-Git 快照
- `/restore <N>` — 恢复到最近第 N 个 pre-turn 快照
- `/browser status|connect [port]|disconnect|tabs` — CDP 会话复用
- `/task` / `/task add <任务>|cancel <id>|log <id>` — 后台任务

**Skill / 通用**
- `/skill list|show <name>|on <name>|off <name>|reload` — Skill 管理
- `/clear` — 清空对话历史 / 短期记忆 / Skill buffer（长期记忆保留）
- `/context` — 上下文模式 / cache 模式 / RAG topK 状态
- `/model <provider>` — 切换模型
- `/config` — 配置
- `/exit` / `/quit` — 退出

> 未识别的 `/xxx` 在 CLI 层直接报「未知命令」，不回退给 Agent。

---

## TUI 形态

| 形态 | 启用方式 | 风格 |
|------|----------|------|
| **inline 流式**（默认） | 直接运行 / `PAICLI_RENDERER=inline` | Claude Code 风格：π 彩色开屏、JLine 底部 dock、行内可折叠工具块、行内 diff、单字符 HITL 提示 |
| **lanterna 全屏** | `PAICLI_RENDERER=lanterna`（旧 `PAICLI_TUI=true`） | 三栏全屏：文件树 + 对话流 + 状态栏 + 输入栏 |
| **plain 兜底** | `PAICLI_RENDERER=plain` | 纯 println，无折叠 / 状态栏 |

- `PAICLI_NO_STATUSBAR=true` — inline 模式禁用底部 dock
- `NO_COLOR=1` — 禁用所有 ANSI 颜色，保留布局
- 对话历史保存到 `~/.paicli/history/session_*.jsonl`

---

## MCP 配置

MCP 子系统默认开启。`~/.paicli/mcp.json` 不存在时自动创建默认 chrome-devtools 配置。合并用户级 `~/.paicli/mcp.json` 与项目级 `.paicli/mcp.json`（项目级按 server 名覆盖）：

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

- `command` = stdio server，`url` = Streamable HTTP server
- `${PROJECT_DIR}` / `${HOME}` 内置变量；其他 `${VAR}` 从系统环境变量 / 系统属性 / 项目 `.env` / 用户 `~/.env` 读取
- 检测到 `STEP_API_KEY` 时自动内置 `step_search` 远程 MCP；模型为 `step-3.7-flash*` 时 `web_search` / `web_fetch` 优先代理到它
- 启动期 MCP 不阻塞首屏：默认最多等 8 秒（`PAICLI_MCP_STARTUP_WAIT_SECONDS` 可调），超时保留 `STARTING` 后台继续，`/mcp` 看最新状态
- OAuth 与 `sampling/createMessage` 当前未实现

**浏览器登录态：** Chrome 144+ 推荐打开 `chrome://inspect/#remote-debugging` 勾选 `Allow remote debugging`，Agent 遇登录页会自动 `browser_connect`；旧版本可启动带 `--remote-debugging-port` 的 Chrome 后 `/browser connect <port>`。

---

## Runtime API

```bash
PAICLI_RUNTIME_API_KEY=your_local_api_key \
  java -jar target/paicli-1.0-SNAPSHOT.jar serve --http --port 8080
```

- 仅监听 `127.0.0.1`，强制要求 `PAICLI_RUNTIME_API_KEY`
- 端点：`POST /v1/threads`、`POST /v1/threads/{id}/turns`、`GET /v1/threads/{id}/events`
- 请求头：`Authorization: Bearer <key>` 或 `X-PaiCLI-API-Key: <key>`

---

## 安全模型（为什么不叫沙箱）

本地 Agent CLI（参考 Claude Code / Cursor / Aider）默认不做容器/VM 沙箱——沙箱削弱 Agent 能力、给虚假安全感、体验更差。生产级 Agent 沙箱实际是 microVM-level（Devin / Modal / Anthropic Computer Use 用 Firecracker / gVisor）。PaiCLI 的安全模型是 **HITL + 路径校验 + 命令快速拒绝 + 审计**，不是隔离。

---

## 测试

`mvn clean package` 默认跳过测试，优先产出可手工验收的 jar。按改动范围选择回归：

```bash
mvn test -Pquick                              # 常规快速回归
mvn test -Pphase16-smoke                      # TUI / inline renderer 冒烟
mvn test -Dtest=XxxTest -DskipTests=false     # 单个测试类
mvn test -DskipTests=false                    # 全量（发版 / 大重构前）
```

| 模块 | 命令 |
|------|------|
| 代码搜索 | `mvn test -Dtest=ToolRegistryTest,CodeSearchGoldenSetTest,ApprovalPolicyTest` |
| 命令解析 | `mvn test -Dtest=CliCommandParserTest,PlanReviewInputParserTest,MainInputNormalizationTest` |
| DAG/Plan | `mvn test -Dtest=ExecutionPlanTest` |
| Multi-Agent | `mvn test -Dtest=AgentRoleTest,AgentMessageTest,AgentOrchestratorTest` |
| RAG | `mvn test -Dtest=CodeChunkerTest,CodeAnalyzerTest,VectorStoreTest,CodeIndexTest` |

---

## 技术栈

Java 17 · Maven · OkHttp · Jackson · JLine 4（终端交互 / Status / 输入 widgets）· SQLite（向量 / 图谱 / 任务持久化）· JavaParser（AST 分析 + LSP 语法诊断）· JGit（Side-Git 快照）· Jsoup（HTML 提取）· Ollama（本地 Embedding）

---

## 项目结构

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

## 数据与配置目录

| 路径 | 用途 |
|------|------|
| `~/.paicli/config.json` | 默认模型 / provider 配置 |
| `~/.paicli/memory/long_term_memory.json` | 长期记忆 |
| `~/.paicli/rag/codebase.db` | 代码索引 |
| `~/.paicli/snapshots/` | Side-Git 快照（不写用户项目 `.git`） |
| `~/.paicli/tasks/tasks.db` | 后台任务队列 |
| `~/.paicli/audit/` | 危险工具审计 JSONL（按天） |
| `~/.paicli/history/` | 对话与输入历史 |
| `~/.paicli/logs/paicli.log` | 调试日志（自动滚动 / 清理） |
| `~/.paicli/skills/` | 用户级 Skill |
| `~/.paicli/mcp.json` | 用户级 MCP 配置 |

> 长期记忆只保存显式保存意图下的稳定事实（`/save` 或用户明确说「记住」时由 Agent 调 `save_memory`），不自动提取，默认项目级作用域，可审计可删除。

---

## 已知边界

以下在路线图但**未交付**：容器 / VM 沙箱 · MCP OAuth + sampling + server 自动重启 · 视频 / 音频 / 图像生成 · TUI sixel 图片预览。
