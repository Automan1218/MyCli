# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**源头真相优先级**：代码实际行为 > `AGENTS.md` > `README.md` > `ROADMAP.md` > 本文件。`ROADMAP.md` 是演进方向，不代表已交付。

---

## 项目概况

**PaiCLI** — 面向商业使用的 Java Agent CLI，对标 Claude Code。已交付 21 期，从 ReAct 单代理演进到 Multi-Agent + TUI + MCP + Skill + 图片输入。Banner 版本 `v16.1.0`，Maven 产物 `paicli-1.0-SNAPSHOT.jar`（两者不一致是正常状态）。

---

## 构建与测试

```bash
cp .env.example .env                                  # 首次配置
mvn clean package                                     # 打包（默认跳过测试）
java -jar target/paicli-1.0-SNAPSHOT.jar              # 运行

mvn test -Pquick                                      # 常规快速回归
mvn test -Pphase16-smoke                              # TUI / inline renderer 冒烟
mvn test -Dtest=XxxTest -DskipTests=false             # 单个测试类
mvn test -DskipTests=false                            # 全量（大范围重构/发版前用）
```

**按模块选测试：**

| 场景 | 命令 |
|------|------|
| 代码搜索工具 | `mvn test -Dtest=ToolRegistryTest,CodeSearchGoldenSetTest,ApprovalPolicyTest` |
| 命令解析 | `mvn test -Dtest=CliCommandParserTest,PlanReviewInputParserTest,MainInputNormalizationTest` |
| DAG/Plan | `mvn test -Dtest=ExecutionPlanTest` |
| Multi-Agent | `mvn test -Dtest=AgentRoleTest,AgentMessageTest,AgentOrchestratorTest` |
| RAG | `mvn test -Dtest=CodeChunkerTest,CodeAnalyzerTest,VectorStoreTest,CodeIndexTest` |

**运行前提：** Java 17+ / Maven / 至少一个 API Key（`GLM_API_KEY` / `DEEPSEEK_API_KEY` / `STEP_API_KEY` / `KIMI_API_KEY` / `FREELLMAPI_API_KEY`）。`ripgrep` 可选，`grep_code` 优先使用，未安装时自动回退 Java 扫描。

---

## 三条主执行路径

| 路径 | 入口类 | 触发方式 |
|------|--------|----------|
| ReAct | `Agent.java` | 默认 |
| Plan-and-Execute + DAG | `PlanExecuteAgent.java` | `/plan` |
| Multi-Agent | `AgentOrchestrator.java` | `/team` |

三条路径共享 `ToolRegistry` / `MemoryManager` / `SnapshotService`，并行工具调用统一走 `executeTools()`，不手写 for-loop，最多 4 个并发，结果保持原始顺序。

---

## 模块导航

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
└── prompt/      PromptAssembler, PromptContext, PromptRepository
```

**任务定位：**

| 任务类型 | 先看 |
|----------|------|
| CLI 命令 | `Main.java` + `CliCommandParser.java` |
| 规划/DAG | `PlanExecuteAgent.java` + `Planner.java` + `ExecutionPlan.java` |
| 工具调用 | `ToolRegistry.java` + `Agent.java` |
| 模型/API | `llm/*Client.java` + `LlmClientFactory.java` |
| RAG 语义辅助 | `CodeRetriever.java` + `CodeIndex.java` + `VectorStore.java` |
| Multi-Agent | `AgentOrchestrator.java` + `SubAgent.java` |
| MCP | `McpServerManager.java` + `McpClient.java` |
| 渲染/TUI | `render/Renderer.java` + `RendererFactory.java` |

---

## 改动联动规则（必须遵守）

| 改动类型 | 必须同步更新 |
|----------|-------------|
| 改行为 | `AGENTS.md` / `README.md` / `ROADMAP.md`（状态变化时） |
| 改命令入口 | `Main.java` + `CliCommandParser.java` + 测试 + `README.md` + `AGENTS.md` |
| 改 Plan 审阅交互 | `Main.java` + `PlanReviewInputParser.java` + 测试 + 手工验证 |
| 改工具集 | `ToolRegistry.java` + Agent/PlanExecuteAgent/SubAgent 提示词 + 可能 Planner 提示词 + 文档 |
| 改模型/接口 | 对应 Client + `LlmClientFactory.java` + `.env.example` + 文档 |
| 改 Memory | `MemoryManager` + `LongTermMemory` + `TokenBudget` + 测试 + 文档 |
| 改 HITL/策略 | `policy/` + ToolRegistry + HitlToolRegistry + 提示词 + `.env.example` + 文档 + 测试 |
| 改 MCP | `mcp/` + ToolRegistry + HITL + AuditLog + 提示词 + 文档 + 测试 |

未识别的 `/xxx` 在 CLI 层直接报"未知命令"，不回退给 Agent。

---

## 关键行为约束

**HITL 拦截顺序：** `HitlToolRegistry → ToolRegistry → PathGuard/CommandGuard`。`PathGuard` 强制路径限定在项目根内；用户无法批准策略层拒绝的请求。

**Memory：** 长期记忆只通过 `/save` 或用户明确要求保存，不自动提取事实。`shortTermMemory` 压缩 ≠ `conversationHistory` 压缩（后者防 window 超限）。

**Plan 审阅快捷键：** `Enter` 执行 / `Ctrl+O` 展开 / `ESC` 取消 / `I` 补充重规划。方向键不应被误判为 ESC。

**DeepSeek V4 / Kimi thinking 模式：** `reasoning_content` 必须随下一轮请求历史带回；其他 provider 默认只写日志。

**Web 联网预检：** 含时效性问题（最新/当前/今天/今年/2026/趋势/新闻/版本）时，ReAct 进入第一轮 LLM 前先执行一次内置 `web_search` 并注入本轮上下文；用户明确"不要联网"时跳过。

**MCP 启动：** 最多等待 8 秒（`PAICLI_MCP_STARTUP_WAIT_SECONDS` 可调），超时保留 `STARTING` 后台继续初始化；`/mcp` 查看最新状态。MCP 配置合并 `~/.paicli/mcp.json`（用户级）与 `.paicli/mcp.json`（项目级），支持 `${VAR}` 环境变量展开。

**代码搜索策略：** 默认 `glob_files` 找候选文件 → `grep_code` 精确定位 → `read_file` 按需读取行段。`search_code`（RAG 语义）仅在关键词不明确或常规搜索无果时使用，不作为精确定位首选。

---

## 不要做的事

- 不提交 `.env` / 真实 API Key / `target/` 产物
- 不把 `ROADMAP.md` 中"将来要做"误读成"现在已有"
- 交互主路径不新增裸 `System.out.println`（除 fatal bootstrap / runtime API / legacy TUI 降级）
- 渲染输出优先走 `Renderer.stream()`，不直接争抢 stdout

**当前已知未交付边界：** 容器/VM 沙箱 / MCP OAuth + sampling + server 自动重启。
