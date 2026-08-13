# Lumos × dsh — 接入评估

> 快照日期：2026-08-14。当日实测的权威版本：
> Lumos 在 `/Users/harvey/Desktop/Harvey-files/08Myprojects/lumos/Lumos` (Next.js 16 + Bun，当前 `master`)，
> dsh 在 `0.1.0-rc.5` (TS) / `0.0.0.dev0` (Py)。
>
> 本文档把 Lumos 当前的 agent runtime 映射到 dsh 的插件树上，并回答一个考虑 dsh 的团队实际上会问的三个问题：
>
> 1. **Lumos 现在用 Claude Agent SDK 做什么，哪些用途是 dsh-shape？**
> 2. **dsh 能替换 Claude Agent SDK 作为 Lumos 的 harness 底座吗？**
> 3. **边界应该画在哪里，迁移成本是多少？**
>
> 读源：Lumos 的 runtime 在 `src/lib/agent-runtime/`。消费者是 `src/lib/exploration/runtime/agent-runner.ts`（NBE）和 `src/lib/roundtable-brt/runner.ts`（BRT）。dsh 的面在 `packages/sdk/{client,protocol,server}` 和 `packages/bundle/{base,web-app,headless}`。
>
> **这是一份深入评估，不是移植计划。** 只想要 8 阶段计划请看 §10。只想要逐文件行数和风险请看 §7。想要架构推理从 §3 开始。

## 1. Lumos 当前的 runtime 形态（评估的对象）

### 1.1 Runtime 是两层

| 层 | 现状 | dsh 类比 |
|---|---|---|
| **Host runtime**（Next.js 进程） | `E2BAgentSandboxProvider`（`src/lib/agent-runtime/e2b-sandbox.ts`）：创建/连接/快照/释放 E2B 沙箱，应用 `denyOrKillAgentSandboxNetwork`，从容量错误中恢复，暴露 `AgentSandboxProvider` 接口 | `dsh-base` + 一个新的 `dsh-sandbox-e2b` 插件可以把 E2B 挂在 `ctx.sandbox` 后面 |
| **Sandbox runtime**（E2B VM 内部） | `ClaudeAgentSdkExecutor`（`src/lib/agent-runtime/claude-agent-sdk.ts`）：写一个 Node ESM runner，import `@anthropic-ai/claude-agent-sdk@0.3.220`，在 stdio 上调 `query({ prompt, options })`；SDK 再派生 bundling 的 `@anthropic-ai/claude-code@2.1.220` CLI，那才是真正干活的 | dsh 自己的 `dsh-jsonrpc-agent` 是同一种东西——一个通过 stdio 启动的 runtime。要替换 Claude Agent SDK，sandbox runner 应该 import `@deepseek-ai/dsh-sdk-client` 并调 `DeepSeekHarness.run(prompt)`。或者——这是 SDK 对比文档 §0.1 打开的新选项——保留 Claude Agent SDK，通过 `subagent-claude-code` 把它当 dsh session 内的 subagent backend 用 |

### 1.2 Lumos 在 SDK 之上做什么

dsh-shape 的关注点都在 *Lumos 的 host runtime*，不在 SDK 调用本身：

| 关注点 | 文件 | 行数 | 备注 |
|---|---|---|---|
| 沙箱生命周期 / checkpoint / 快照 / pause / dispose | `e2b-sandbox.ts`、`remote-files.ts`、`timeouts.ts` | 394 | 有状态，掌管一个长寿命的沙箱长达数小时 |
| Skillpack 同步（workspace `skills/` → sandbox `.lumos-agent-sdk/skills/` 和 `.claude/skills/`） | `skillpack.ts` | 162 | 带 fingerprint 缓存；声明式 `resolveSkillRoot()` |
| 密钥 / env 清洗 | `security.ts`（`sanitizeAgentSandboxEnvs`、`assertAgentRuntimeProductionBoundary`） | 125 | 硬编码 `SAFE_SANDBOX_ENV_NAMES` allowlist；`AgentRuntimeSecurityError` 在生产环境直接短路 |
| 网络出口 allowlist + broker | `broker-egress.ts`、`broker-grants.ts`、`broker-config.ts`、`broker-request.ts`、`broker-search-proof-core.ts`、`broker-search-proofs.ts` | 1,786 | **关键**：生产环境必须通过 Lumos 自有的 `/api/internal/agent-broker/anthropic` 代理，它会铸造短期 grant；sandbox 网络默认被拒绝（`denyOrKillAgentSandboxNetwork`） |
| 进程清理 / 断流恢复 | `claude-code.ts`（`cleanupClaudeCodeProcesses`）、`claude-agent-sdk.ts`（`looksDetachedCommandStreamFailure`） | ~80 | 启发式进程扫描在 sandbox 内 |
| 工具 allowlist | `allowed-tools.ts`（`NBE_AGENT_ALLOWED_TOOLS`、`BRT_ALLOWED_TOOLS`） | 15 | 模块专属 allowlist（NBE：12 工具；BRT：5 工具） |
| 模板治理 | `scripts/e2b/build-nbe-template.ts`、`scripts/e2b/build-brt-template.ts`、`scripts/e2b/install-claude-agent-sdk.sh` | ~600 | 构建一个最小镜像：Node `24.18.0`（digest 钉死）、Claude Code CLI、Claude Agent SDK 包、精确的 npm lockfile、不要 node 版本管理器 |
| Phase / checkpoint / 质量门 / artifact / 报告门 | `src/lib/exploration/runtime/agent-runner.ts` 及周围 phase | 280 | **NBE 专属**；不在 agent runtime 本身范围 |
| 多 agent 讨论（moderator + 2-3 个 partner） | `src/lib/roundtable-brt/runner.ts`（4,612 行）、`moderator-planner.ts`、`memo-orchestrator.ts` 等 | 4,600+ | **BRT 专属**；不在 agent runtime 本身范围 |

### 1.3 两个 SDK 消费者

#### (a) NBE — Notebook Exploration（`src/lib/exploration/`）

- 长跑 phase 管道（intake、research、analysis、report）。每个 phase 是一次 Claude Code 调用。
- 硬约束：NBE adapter "固定实例化 `ClaudeAgentSdkExecutor`；不存在可切换到 CLI 的运行时配置" —— **今天没有运行时切换**。SDK 是唯一执行路径。
- Skill 同步："NBE 真实执行固定读取仓库内 `skills/` 并同步到 E2B" —— 仅仓库本地 skills。
- Final gate：`NBE_CLAUDE_EXECUTION_MODE=lumos_controller | orchestrator_parity` 和 `NBE_REPORT_READY_GATE=release` 产生 hard/publish/semantic 门；`PASS_WITH_WARNINGS` 是内部候选态。
- 报告修复轮次：`NBE_REPORT_REPAIR_MAX_ROUNDS=3` —— 在同一 sandbox 内重调 agent 来修复/扩展报告。
- **多工具集**：NBE 允许 12 工具（`Task, Glob, Grep, LS, Edit, MultiEdit, Write, NotebookRead, NotebookEdit, TodoWrite, WebSearch, …`）。这是 NBE 的宏观/市场证据路径；broker 主机按该精确集合 review。
- `Phase1StructuredRunnerPayload` 已经声明 `runtimeBackend` 和 `sandboxBackend` 字段 —— 架构层面已经有"多 backend"的概念，只是没到 runtime 层。

#### (b) BRT — Business Roundtable（`src/lib/roundtable-brt/`）

- 2-3 个 "合伙人" subagent + 一个 moderator，每个是 `subagent_type`。它们通过 `Agent` 工具的 subagent 机制在同一个 sandbox 内跑，使用五个允许工具（`Agent`、`Glob`、`Grep`、`LS`、`TodoWrite`）。
- Subagent executor 通过 `createRoundtableClaudeDiscussionRuntime({ subagentExecutor })` 注入。DB 可见的进程事件是 `claude-agent-sdk-subagents`，审计为 `done | queued | error`。
- Runner 是 **4,612 行**。它是 `claude-agent-sdk` 最大的单一消费者，包含多 agent 编排逻辑、moderator planner、memo orchestrator、public-turn/memo/decision-board/state/event-adapter 子系统，以及 lifecycle/abort/turn-lease/preflight 管道。
- 模型是 "圆桌对话固定 Opus 4.6"，**不自动降级** —— 默认 token group 的 503 会以 `subagentRuntimeSucceeded=false` 和硬失败出现，而不是悄悄换更小的模型重试。这是一个**故意**的产品决定，dsh 当前没有镜像；最接近的 dsh 构造是带 `providerRetryPolicy()` 的 `ctx.llm` adapter，拒绝模型降级，但不是默认。
- BRT 在配置时向 sandbox 传 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`。这是 Claude 侧的跨 agent 协作实验面；moderator planner 决定启用时 Lumos 才用它。
- BRT 在 `moderator-planner.ts`、`crisis.ts` 等地方使用 `experimental_repairText`（Claude Agent SDK 的结构化输出修复回调）。这是 Claude 专属 API；dsh 的结构化输出路径是 `dsh-system-prompt` + `tools/post-execute` 决定路径，是不同的模型。

### 1.4 与 SDK 并行但不依赖 SDK 的部分

这些是 LLM-shape 但不是 Claude-shape，如果 Lumos 想渐进引入 dsh 而不干扰 Claude，它们是相对安全的共同采用目标：

- Lumos 的 chat UX：`src/lib/chat-content`、`src/components/chat/*`、`src/app/chat/*`。直接使用 Vercel AI SDK（`@ai-sdk/anthropic`、`@ai-sdk/openai`），不用 Claude Agent SDK。**dsh 不会动这部分**。
- Diagnosis graph executor（`src/lib/diagnosis/`）：仅 Vercel AI SDK，没有 Claude Agent SDK。**dsh 不会动这部分**。
- Year-plan workflow（`src/lib/year-plan/`）：仅 Vercel AI SDK，没有 Claude Agent SDK。**dsh 不会动这部分**。
- 圆桌辩论（BRT 中的 moderator + partners）在 Claude Agent SDK 上；*圆桌产品*（state、public turns、memo、decision board、billing、dedupe）在 Lumos 自己的 DB 上。**dsh 会动 SDK 那侧，不动产品那侧。**

这含义重大：依赖 Claude Agent SDK 的表面积集中在两个管道（NBE 和 BRT），相对于整个 Lumos app 而言。其他一切都独立。

## 2. dsh 的面，映射到 Lumos

### 2.1 dsh 插件树，按能力家族划分

dsh 把 50+ 个包组织成能力家族，每个家族是一个 `Service Definition`（seam）加 0..N 个 `Service Provider`（在该 seam 上注册的 backend）加可选的 `Consumer`（把 seam 暴露给模型的工具）。这是对比里唯一的结构模式；你应该先内化它再读后面。

| 家族 | dsh key | Service Definition | 出货的 backend | Lumos 类比 |
|---|---|---|---|---|
| `llm` | `ctx.llm` | `LlmRuntime`（adapter 注册表 + 流 API） | `dsh-llm-deepseek`、`dsh-llm-pi-ai`、… | broker（形状不同） |
| `core/tools` | `ctx.tools` | `ToolRegistry` + waterfall 管道 | bash、fs、search、todo、subagent、session-query | `NBE_AGENT_ALLOWED_TOOLS` / `BRT_ALLOWED_TOOLS`（硬编码数组） |
| `core/agent` | `ctx.agents` | `AgentRegistry` + `agent/*` 事件 | `core/agent-loop`（默认 driver） | n/a（Lumos 自建 loop） |
| `core/session` | `ctx.sessions` | `SessionStore` + `Session` 事件源 log | persistence 插件（JSONL） | Lumos 的 Postgres 持久化 |
| `sandbox` | `ctx.sandbox` | 进程隔离 seam | `dsh-sandbox-local`（POSIX ACL，win32 ACL） | `E2BAgentSandboxProvider`（远程，不是进程级） |
| `fs` | `ctx.fs` | filesystem seam | `dsh-fs-local`、`dsh-fs-remote` | `remote-files.ts`（把文件推入 sandbox） |
| `subagent` | `ctx.subagents` | subagent provider 注册表 | `subagent-spawn-in-process`、`subagent-fork-in-process`、`subagent-acp`、**`subagent-codex`**、**`subagent-claude-code`**、**`subagent-dsh-sdk`** | n/a（Lumos 的 BRT 通过 `Agent` 工具有自己的 subagent 流） |
| `compaction` | `ctx.compaction` | compaction seam | `compaction-basic`、`compaction-tool-result-pruner`、`command-compact` | n/a（Lumos 依赖 Claude 自动 compaction） |
| `guard` | n/a（事件 consumer） | 行为 guard | `repeat-tool-reminder`、`timeout-policy` | `assertAgentRuntimeProductionBoundary`、`sanitizeAgentSandboxEnvs` |
| `mcp` | n/a | MCP server 挂载 | 各种 MCP 包 | n/a（Claude Agent SDK 有进程内 MCP；dsh 把它暴露为插件） |
| `hooks` | `ctx.hooks` | hook bridge（在生命周期点拦截） | `hooks-claude-code`、`hooks-codex` | n/a（Lumos 自有 broker 侧校验） |
| `interaction/approval` | `ctx.approval` | 用户审批 service | 跟 `dsh-base` 一起出货 | n/a（Lumos 的 `NBE_REPORT_READY_GATE`） |
| `host` | `ctx.webServer`、`ctx.apiProxy` | web host 半边 | webserver、apiproxy、plugin-inventory | Lumos 的 Next.js app |
| `bundle/base` | （组合） | 每个 profile 的第一层 | model adapters + tools + persistence + policy + Claude/Codex providers（dormant） | n/a |
| `bundle/web-app` | （组合） | web host | webserver、browser 插件 roster、persona、`web-runtime` | Lumos 的 Next.js app |
| `bundle/headless` | （组合） | one-shot runner | persona、Code Mode worker、`headless-runner` | Lumos 的 BRT 单次讨论 |
| `acp/acp` | n/a | 纯自动化的 ACP server | `@deepseek-ai/dsh-acp`（stdio JSON-RPC） | n/a |
| `acp` | n/a | subagent 的 ACP client | `subagent-acp` | n/a |

### 2.2 精确匹配表（dsh 概念 → Lumos 文件）

| dsh 概念 | dsh 位置 | 对应 Lumos 现状 | 迁移影响 |
|---|---|---|---|
| `dsh-base` profile | `packages/bundle/base`（带 `cordis.patch.yml`，约 200 行） | n/a —— Lumos 完全不发布 profile；它发布一个手写的 host runtime | **大** —— 会替换整个 host runtime 脚手架 |
| `dsh-headless` profile | `packages/bundle/headless` | 部分 —— `LUMOS_ROUNDTABLE_AGENT_RUNTIME=e2b` 路径是 headless；NBE 有长跑 artifact，不适合"一个 prompt 进一个答案出" | **中** —— BRT 单次路径可以替换；NBE 的多 phase 修复循环本身不行 |
| `cordis.patch.yml` 分层 | `packages/bundle/*/cordis.yml` + `packages/boot/app-boot` | 今天：硬编码 `claude-agent-sdk.ts` + `claude-code.ts` + `e2b-sandbox.ts` | **中** —— patch 层天然契合 Lumos 的"模块专属 runtime"模型（NBE vs BRT vs 未来模块） |
| `ctx.sandbox` 能力 seam | `packages/sandbox` | `AgentSandboxProvider`（Lumos 自己的 seam） | **小** —— dsh 的 `ctx.sandbox` 可以托管一个 `dsh-sandbox-e2b` 插件，做今天 `E2BAgentSandboxProvider` 做的事 |
| `ctx.fs` provider | `packages/fs` | `remote-files.ts`（推文件入 sandbox） | **小** —— 但 seam 必须支持"remote + 按需同步"模式，不只是 local provider |
| `ctx.tools` 注册表 + `tools/*` 事件 | `packages/core/tools` | `NBE_AGENT_ALLOWED_TOOLS` / `BRT_ALLOWED_TOOLS`（硬编码数组） | **小-中** —— allowlist 变成插件级 config；tool allowlist 事件变成 waterfall |
| `ctx.hooks` 生命周期 | `packages/hooks` + `dsh-hook-protocol-lib` + `dsh-hook-bridges` | 没有直接对应；Lumos 通过 SDK 依赖 Claude Code 的 `PreToolUse`/`PostToolUse` 等 | **中** —— dsh 的 hook 系统是通用的；Lumos 需要把它的 broker、scrubber、gate 逻辑接成插件 |
| `ctx.llm` adapter seam | `packages/llm/llm`、`packages/llm/*` | **主机**用 `DMX_BASE_URL` + `DMX_API_KEY`（在 `cordis.yml`）；**sandbox**走 Lumos agent broker | **高** —— 这是关键兼容问题。详见 §3 |
| `ctx.guard` policy seam | `packages/guard` | `assertAgentRuntimeProductionBoundary`、`isExplicitStagingAgentBrokerEnabled` | **小** —— 同形不同文件 |
| `ctx.subprocess`（Lumos 的例外情形） | `packages/subprocess` | `claude-agent-sdk.ts` 的私有 `disposeEofGraceMs` 阶梯 | **小** —— SDK client 的私有 SIGKILL 阶梯已经在做这件事；不需要改 |
| `dsh-skill` + `extensions` | `packages/skill` + `packages/extensions` | `DefaultSkillpackSyncer`（Lumos 的） | **中** —— 需要一个 `dsh-skill-lumos-bridge` 把 Lumos 仓库本地 skills 喂给 dsh 的 extensions 树 |
| `dsh-compaction` 能力 seam | `packages/compaction` | 没有 —— Lumos 依赖 Claude 自动 compaction | **小** —— 等 dsh 的 `recallable-compaction` 落地后，可以用可审计的 seam 替换自动 compaction |
| `dsh-session` + `dsh-session-query` | `packages/session`、`packages/session-query` | Lumos 的 DB 表（`exploration_jobs`、`chat_state_snapshots`、`roundtable_*`） | **高** —— Lumos 的 session 持久化是应用级（Postgres + Better Auth + chat_state），不是 agent 的；agent 的 session log 目前是 SDK 的 CLI 文件 |
| `dsh-job` + `ctx.jobs` 后台工作 | `packages/jobs` | `nbe:worker`、`roundtable:worker`（Next.js 进程里的 Bun worker） | **中** —— dsh 的 job service 跑在 dsh 进程*内*；Lumos 的 worker 跑在 E2B sandbox *外*。这是两套不同的 job 系统，不是替换 |
| `dsh-tool-terminal` / `ctx.terminals` | `packages/extension/terminal` | `cleanupClaudeCodeProcesses` 在 `claude-code.ts` | **小** —— 同样的问题，不同的机制 |
| `dsh-mcp` | `packages/mcp` | Claude Agent SDK 通过 `@tool` + `create_sdk_mcp_server` 支持进程内 MCP；Lumos 今天没用 | **小** —— dsh 把 MCP 挂成插件；同形但搭建更繁琐 |
| `dsh-subagent` 能力 seam | `packages/subagent` | BRT 的多 agent 流 | **中-高** —— dsh 的 subagent 面是 provider-portable；BRT 当前 subagent 执行假设 Claude 的 `Agent` 工具。移植 BRT 意味着选一个 subagent backend |
| `dsh-acp` 自动化 server | `packages/acp/acp` | n/a | **机会** —— Lumos 可以把 NBE/BRT runner 暴露为 ACP 自动化 server，让外部工具（Zed、JetBrains、自定义编辑器）驱动同一管道。dsh 已经出货 server |

### 2.3 独特定位：dsh 能 *使用* Claude Agent SDK 作为 backend

根据 SDK 对比 §0.1：dsh 出货 `subagent-claude-code`，把官方 Claude Agent SDK 用作 dsh session 内的 subagent backend。这意味着 **dsh 不要求 Lumos 放弃 Claude Agent SDK**。

不要求放弃 Claude 的迁移路径：

1. Lumos 把当前的 `ClaudeAgentSdkExecutor` 包成一个薄 adapter，注册为 dsh subagent provider（`@deepseek-ai/dsh-subagent-lumos-claude-sdk`）。
2. Lumos 的 NBE/BRT runner 改走 dsh 的 `subagent` 工具，不再直接 import `@anthropic-ai/claude-agent-sdk`。
3. dsh session 托管编排、broker、持久化、policy；Claude Code 子做模型工作。
4. dsh 的 `hooks-claude-code` bridge 把 Claude 的 `hooks.json` 翻译到 dsh 的拦截点，所以已有的 Claude hooks 继续工作。

这与"用 dsh 替换 Claude"在根本上是不同的迁移。它是"在 dsh 里面托管 Claude"。它不需要 §3.1 里任何模型绑定变更。

## 3. 难题

### 3.1 能把 sandbox 侧 runtime 从 Claude Agent SDK 换成 dsh 吗？

**机械上能，但有 caveat。**

当前 sandbox runner（`claude-agent-sdk.ts`）是一个 Node ESM 脚本，做三件事：

1. Import `@anthropic-ai/claude-agent-sdk`。
2. 用从 `AgentRuntimeConfig` 构造的 options 调 `query({ prompt, options })`。
3. 通过 E2B 的 `commands.run` 流式把消息回传给 host。

替换为 dsh 意味着：

```ts
// 原来
import { query } from '@anthropic-ai/claude-agent-sdk'
async for await (const msg of query({ prompt, options })) { ... }

// 改成
import { DeepSeekHarness } from '@deepseek-ai/dsh-sdk-client'
await using harness = new DeepSeekHarness({
  launch: { command: 'node', args: ['lib/bin.js', 'cordis.yml'] },
  provider: 'deepseek-official',  // 或者 cordis.yml 注册的别的
  model: '<model>',
  maxTokens: 49_152,
})
const result = await harness.run(prompt)
```

机械上很简单。今天**不**这么做的原因，按严重度排序：

1. **dsh 是 `0.1.0-rc.5`。**"会有破坏性变更。"Lumos 当前的 SDK 钉是 `0.3.220`，有 8 个月的测试历史和厂商 bundling 的 CLI。
2. **真正在 sandbox 里跑的是 Claude Code CLI。** 替换 SDK **不**意味着把 sandbox 里的 CLI 依赖去掉——dsh 需要自己的 runtime 在 sandbox 里。今天，这意味着 `node lib/bin.js cordis.yml`（SDK 合约假设你带 runtime）。你得构建一个 `nbe-latest-dsh-terminal-mvp` 模板，平行于 `nbe-latest-claude-terminal-mvp`，带钉死的 Node 24.18.0 和 bundling 的 dsh runtime。
3. **模型绑定是真产品决定。** 今天，NBE/BRT 钉在 Claude（圆桌固定 Opus 4.6，NBE 可配），用走 Lumos broker 的 `anthropicBaseUrl`。dsh 的 `cordis.yml` 决定 adapter；你得注册一条 DeepSeek route，可能再加 OpenAI route，并彻底丢掉 Anthropic（除非你同时挂 `llm-pi-ai`，那是第三方 adapter；dsh 项目把 `llm-pi-ai` 作为独立包出货，所以一行就能挂上）。
4. **BRT subagent 流是 Claude-CLI 形的。** dsh 的 `ctx.subagent` 是通用的能力 seam；BRT 当前的 subagent 依赖 Claude Code 的 `Agent` 工具配合五个具体工具名。移植 BRT 意味着重写 `roundtable-brt/runner.ts`，要么派生 dsh subagent（每个 subagent 一个 harness），要么用挂载了 subagent backend 的单个父 harness 组合。

### 3.2 dsh 能替换 *host* runtime 吗？

**部分能。Seam 对齐的部分是好匹配；应用级的部分不是。**

`src/lib/agent-runtime/` ~6,300 行。按 dsh-shape 拆分：

| 文件 | 行数 | dsh-shape？ | 会搬到哪里 |
|---|---|---|---|
| `e2b-sandbox.ts` | 275 | **是** | 一个 `dsh-sandbox-e2b-lumos` 插件（如果 Lumos 向上游贡献 provider，就是 `dsh-sandbox-e2b`） |
| `remote-files.ts` | 101 | **是** | 一个 `dsh-fs-remote-lumos` provider |
| `timeouts.ts` | 18 | **是** | 一个 config 校验插件（或 `dsh-cmdline` runtime config） |
| `security.ts` | 125 | **是** | 一个 `dsh-guard-lumos` 插件（`assertAgentRuntimeProductionBoundary`、`sanitizeAgentSandboxEnvs`） |
| `skillpack.ts` | 162 | **是** | 一个 `dsh-skill-lumos-bridge` extension |
| `claude-code.ts` | 344 | **部分** | `cleanupClaudeCodeProcesses` 启发式是 Claude-CLI 专属；`CLAUDE_CODE_EXIT_CODE_MARKER` 协议是 Claude-CLI 专属；用 dsh 自己的 exit / 流断连机制替换（或保留 Claude 作为 subagent backend，见 §2.3） |
| `claude-agent-sdk.ts` | 538 | **否** | 被 `@deepseek-ai/dsh-sdk-client` 调用替换（或，按 §2.3，被 `subagent-claude-code`） |
| `broker-*.ts`（六个文件） | 1,786 | **部分** | Lumos 的 broker 是 HTTP-API 形的，不是 LLM-adapter 形的。dsh 没有 `ctx.llm` adapter 能跟期望 `/api/internal/agent-broker/anthropic/{...}` payload 校验的代理对话。你要么写一个自定义 `dsh-llm-lumos-broker` adapter，要么把 broker 重构成挂在 `ctx.llm` 后面的通用代理 |
| `config.ts` | 214 | **部分** | dsh 的类型化 config 可以托管 `LUMOS_AGENT_RUNTIME_*` key；硬上限（`hardMax`）是 dsh-shape 的；env 名 allowlist（`SAFE_SANDBOX_ENV_NAMES`）属于 guard 插件 |
| `types.ts` | 112 | **是** | dsh 的能力 seam 类型是正确的归宿；`AgentSandboxProvider` / `ClaudeCodeExecutor` 接口是 dsh-shape（"service definition" + "service provider"） |
| `*.test.ts` | 1,500+ | **否** | Lumos 的测试套件是 Vitest，dssh 是不同的 harness；这是机械移植工作 |

dsh **不**给 Lumos 的硬东西：

- **Postgres 后端的 broker grants**（`broker-grants.ts`）。dsh 没有等价物；你会建一个 `dsh-grant-lumos-postgres` 插件。
- **Lumos 的模型 allowlist 模式**：BRT 5 工具，NBE 12 工具，带模块专属 `ConsumedAgentBrokerGrant["module"]` 类型。dsh 的 `ctx.tools` 同形但没有模块键控 allowlist。你会写一个 `dsh-tool-allowlist-by-module` extension，把 `ConsumedAgentBrokerGrant["module"]` 映射到 `ctx.tools` 过滤器。
- **"圆桌对话固定 Opus 4.6" 策略**。dsh 有每个 agent 的 `maxTokens`，但没有产品级的"不降级"保证；adapter 只是路由到 provider 返回的。如果你要同样的保证，加一个 `dsh-llm-strict-model` 插件，拒绝任何 `model !==` 配置值的调用。
- **Lumos 的 CI / 模板构建管道。** `scripts/e2b/build-nbe-template.ts` 是一个自定义 Node 24.18.0 digest-钉死的镜像构建器。dsh 当前不出货 E2B 模板构建器；你会原样带过来。
- **`experimental_repairText` 回调** BRT 用来做结构化输出修复。这是 Claude Agent SDK API；dsh 的结构化输出路径是 `dsh-system-prompt` + `tools/post-execute` 决定路径，是不同模型。移植需要一个包裹 `tools/post-execute` 来执行 schema 修复的 `dsh-structured-repair` 插件。

### 3.3 dsh 给 Lumos 什么？目前是手写的？

这是最有价值的问题。清单不短。

| 今天，Lumos 手写的东西 | dsh 替换为什么 |
|---|---|
| `claude-code.ts` 进程清理启发式（`cleanupClaudeCodeProcesses` —— Python `ps`/`kill` 脚本） | dsh SDK 的 `HarnessClient.close()` 私有 SIGKILL 阶梯 —— `disposeEofGraceMs` / `disposeGraceMs` / `shutdownTimeoutMs` 是你可以调的旋钮，阶梯是幂等的 |
| `looksDetachedCommandStreamFailure` 正则 | dsh SDK 的 `TransportClosedError`（类型化） |
| `claude-agent-sdk.ts` 的 `runPrompt` 合约（SDK 之上的单 `prompt → result` 外观） | dsh SDK 的 `DeepSeekHarness.run()` 直接用 —— 但你仍想要一个 Lumos 侧的包装器管 E2B 生命周期，所以省下来的是 JSON-RPC 层，不是包装器 |
| `assertAgentRuntimeProductionBoundary` 和 `AgentRuntimeSecurityError` | dsh 的 `ctx.guard` 能力 seam —— 需要 `dsh-guard-lumos` 插件接 env 名 allowlist，但 seam 本身是现成的 |
| `sanitizeAgentSandboxEnvs`（env 名 allowlist + 凭据 regex） | dsh 的 `dsh-subprocess` `scrubbedParentEnv` 加 guard 插件 |
| `BrokerRequestError` schema（8 key 顶层 allowlist、32 工具上限、8 MB 请求 / 1 MB 系统字节上限） | dsh **不**出货 request-schema guard。你会把它带到 `dsh-llm-lumos-broker` 插件里 |
| `DefaultSkillpackSyncer`（fingerprint + 同步入 sandbox） | dsh 的 `dsh-skill` + `extensions` 是规范归宿；你写一个 `dsh-skill-lumos-bridge` 把 `skills/` 映射到 `dsh.extensions` 树 |
| `E2BAgentSandboxProvider.acquire()`（connect → snapshot-restore → template；容量恢复；每次 connect 都 `denyOrKillAgentSandboxNetwork`） | dsh 的 `ctx.sandbox` 会托管它，但 *acquire 算法* 是 Lumos 专属（connect-then-snapshot-then-template 顺序），会作为 `dsh-sandbox-e2b-lumos` 插件带过来 |
| `claude-code.ts` `CLAUDE_CODE_EXIT_CODE_MARKER`（runner 打印一个 sentinel 行让 host 解析 Claude Code 退出码） | dsh SDK 返回一个带 `finishReason`（`completed` / `max-tokens` / `error`）的类型化 `RunResult`；marker 不再需要 |
| `BashToolOutput.timedOutAfterMs` 处理（Claude-CLI 专属超时报告） | dsh 的 `SessionEvent` 包含同一事实；host 可以订阅并反应 |
| `SandboxEgress.denyOrKillAgentSandboxNetwork`（E2B 专属网络策略） | dsh 的 `dsh-sandbox-local`（POSIX ACL）或一个 `dsh-sandbox-e2b` 插件（E2B 专属）—— 同形不同文件 |
| `MAX_REQUEST_BYTES = 8 * 1_024 * 1_024` 和 `MAX_SYSTEM_BYTES = 1 * 1_024 * 1_024` 在 `broker-request.ts` | dsh 不在 LLM seam 上强制字节上限；最接近的是 adapter 自己的 `prepareCall` 拒绝路径。上限会留在 `dsh-llm-lumos-broker` 插件里 |
| `ConsumedAgentBrokerGrant["module"]` 键控的 `SAFE_TOOLS` map | dsh 的 `ctx.tools` 同形；`dsh-tool-allowlist-by-module` extension 把 broker module 映射到 tool 过滤器 |

## 4. 按关注点的风险分析

| 关注点 | 迁移出错时 | 缓解 |
|---|---|---|
| 沙箱生命周期 | 孤儿沙箱，泄漏 E2B 成本 | dsh SDK 的幂等 close + dsh runtime 的 `ctx.sandbox.dispose()` 回调；出一个 sandbox-hours/day 指标 |
| Broker | 开放代理 = 开放的模型账单 | 保持 broker 在 Lumos；绝不让 dsh 直接路由模型调用；只通过 `dsh-llm-lumos-broker` 路由 |
| Skillpack | 缺 skill → 静默工具失败 | 基于 fingerprint 的失效（已在 `DefaultSkillpackSyncer`）；保留 syncer，只是重新指向 `dsh.extensions` |
| 进程清理 | sandbox 内僵尸 claude 进程；断流 | dsh SDK 的 SIGKILL 阶梯处理；E2B 模板变更后用 `claude --version` smoke 验证 |
| 工具 allowlist | 模型调用不该可用的工具 | dsh `ctx.tools` 是 per-agent-scope；把 allowlist 绑到 agent，不是 session |
| Subagent（BRT） | subagent 定义里工具名错 | dsh 的 `subagent` 工具有类型化的 provider config；在组合时（不是运行时）校验 `subagent_type` |
| 持久化 | host DB 和 runtime 之间 session log 分叉 | 要么把 dsh 的 session 当作真源，要么通过 `session/event` 监听器把 dsh session 镜像到 Lumos DB |
| 权限 | 编辑时自动接受，漏掉审批门 | dsh 的 `ctx.approval` seam 替换 Claude 的 `permission_mode`；把 Lumos 的 gate flag 映射到 `ctx.approval` 规则 |
| Compaction | 静默上下文丢失 | dsh 的 `compaction-basic` 是出 `CompactionResult` 的 provider；配 `compaction-tool-result-pruner` 限制 tool result 大小 |
| 遥测 | 缺少成本归因 | dsh 的 `ctx.telemetry` 和 `runtime-diagnostics` 发出 model-call 事件；broker 的 `costUSD` 计算留在 Lumos |
| 模型降级 | "圆桌对话固定 Opus 4.6" 策略被破坏 | 出 `dsh-llm-strict-model` 插件（按 §3.2）；在 CI 中验证：把模型设成 Sonnet，断言 run 拒绝 |
| Agent Teams | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` 流改变 | 如果你通过 `subagent-claude-code` 保留 Claude 作为 backend，实验 flag 透传；如果你不保留，BRT 的跨 agent 协作需要 dsh-native 替代（今天没有干净的 drop-in） |

## 5. 向后兼容和功能对等

| Lumos 功能 | 现状 | 全面替换为 dsh | 组合式使用 dsh |
|---|---|---|---|
| `claude-agent-sdk` `query({ prompt, options })` | ✓ | 替换为 `DeepSeekHarness.run()` | 保留在 `subagent-claude-code` 后面 |
| `Agent` 工具（subagent 调用） | ✓ | 替换为 dsh 的 `subagent` 工具，provider = `claude-code` | 不变 |
| `TodoWrite`（plan 工具） | ✓ | 替换为 dsh 的 `todo-write-tool` (2026-06-29) | 不变（Claude backend） |
| `Bash` + `BashOutput` + `KillBash` | ✓ | 替换为 dsh 的 `tool-bash` + `tool-pwsh`（POSIX/win32 拆分在 `dsh-base` patch） | 不变 |
| `Read` / `Write` / `Edit` / `Glob` / `Grep` / `LS` | ✓ | 替换为 dsh 的 `tool-fs-*` 家族 | 不变 |
| `WebSearch`（仅 NBE，通过 Anthropic 服务端搜索） | ✓ | 替换为 dsh 的 `tool-web-search`（需要 backend 插件） | 不变 |
| `WebFetch`（Lumos 当前没用） | ✓ | 替换为 dsh 的 `tool-web-fetch` | 不变 |
| `NotebookRead` / `NotebookEdit`（仅 NBE） | ✓ | 替换为 dsh 的 `tool-notebook-*` | 不变 |
| `MultiEdit`（仅 NBE） | ✓ | 替换为 dsh 的 `tool-edit-batch` | 不变 |
| `Skill`（Claude 的 skill loader） | ✓ | 替换为 dsh 的 `dsh-skill` + `extensions` | 保留（Claude 的 skill loader 在 Claude 子里仍工作） |
| `PreToolUse` / `PostToolUse` hooks | ✓ | 替换为 dsh 的 `ctx.hooks` 拦截点 | dsh 的 `hooks-claude-code` bridge 翻译 Claude 的 `hooks.json` |
| `SubagentStop` / `Stop` / `Notification` | ✓ | 替换为 dsh 的 `agent/*` 事件 | dsh 的 `hooks-claude-code` bridge |
| `SessionStart` / `SessionEnd` / `PreCompact` | ✓ | 替换为 dsh 的 `session/*` 事件 | dsh 的 `hooks-claude-code` bridge |
| `PreCompact`（手动 compaction） | ✓ | 替换为 dsh 的 `command-compact`（`ctx.commands`） | 不变 |
| `can_use_tool` 异步回调 | ✓ | 替换为 dsh 的 `tools/execute` waterfall | dsh 的 `hooks-claude-code` bridge（透传） |
| `experimental_repairText`（BRT 结构化输出） | ✓ | 替换为 dsh 的 `tools/post-execute` accept/reject | 替换为一个监听 Claude 子的 `dsh-structured-repair` extension |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` | ✓ | dsh 中没有可用（没有 drop-in） | 透传到 Claude 子（不变） |
| `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` / `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` | ✓ | 替换为 dsh 的 per-plugin 深度/并发 | 透传到 Claude 子 |
| `permission_mode: 'manual' \| 'acceptEdits' \| ...` | ✓ | 替换为 dsh 的 `ctx.guard` + `ctx.approval` | 由 `hooks-claude-code` 映射（不在映射中就降级） |
| `api_error_status: 429 / 529` 字段 | ✓ | dsh 的 `LlmRuntime` 返回类型化的 `finish { kind: 'error', failure }`；原始 API 状态在 failure 对象里 | 透传 |
| `cancel_queued`（能力 `interrupt_cancel_queued_v1`） | ✓ | 替换为 dsh 的 `HarnessClient.close()` 阶梯 | 透传 |
| `crossSessionInbound` / `dialogExpiry` | ✓ | dsh 有 `interactive-side-sessions`（提议中） | 透传 |
| `Settings.source: 'archive'`（通过 HTTPS 安装插件） | ✓ | n/a（dsh 有自己的插件加载，不是 zip over HTTPS） | 不变 |
| `Settings.decode: 'jwt'` / `maskClaims` / `awsPairs` / `sigv4` | ✓ | dsh 的 `dsh-subprocess` `scrubbedParentEnv` 是最接近的 | 透传 |
| `ResumeSessionAt` / `resumeDropsTurn` | ✓ | dsh 的 `Session` 可从 log 重放；不需要单独的"截断式 resume"动词 | 不变 |

## 6. 测试影响

Lumos 的 `src/lib/agent-runtime/` 出货 ~1,500 行 Vitest 测试。一次迁移应该预期：

| 测试文件 | 行数 | 需要改什么 |
|---|---|---|
| `agent-runtime.test.ts` | 1,179 | 大部分测试需要 re-baseline；它们测的面（进程清理、env 清洗、模型 ID 校验）是 dsh-shape 但 API 不同 |
| `broker-egress.test.ts` | 148 | broker 测试保留（broker 在 Lumos） |
| `broker-grants.test.ts` | 72 | broker 测试保留 |
| `broker-grants.integration.test.ts` | 430 | 集成测试需要 CI 中跑一个 dsh runtime |
| `broker-request.test.ts` | 329 | broker 测试保留 |
| `broker-search-proof-core.test.ts` | 263 | broker 测试保留 |
| `remote-files.test.ts` | 36 | 为 dsh 的 `ctx.fs` API re-baseline |
| `sandbox-entrypoints.test.ts` | 277 | 为 dsh 的 `ctx.sandbox` API re-baseline |

**经验法则：** 计划每 3 行生产代码改动配 1 行测试工作，再加 30% 给集成测试脚手架（CI 中的 E2B + dsh runtime）。

## 7. 迁移成本，具体到行数

如果你要做的阶段计划。成本估算是行数，不是日历。

| 阶段 | 内容 | LOC 估算（替换/移植） | 风险 | 时间 |
|---|---|---|---|---|
| **0. 并行启动 dsh** | 加 dsh 作为 workspace 依赖，构建 `nbe-dsh-canary` E2B 模板，写一个 `DshBridgeExecutor` 实现 `ClaudeCodeExecutor` 并转发到 `DeepSeekHarness.run()` | 新代码 ~250 行；无删除 | 低 | 第 1-2 周 |
| **1. 替换 sandbox 侧 SDK** | 把 `claude-agent-sdk.ts` 换成用 `@deepseek-ai/dsh-sdk-client` 的 `dsh-bridge.ts`；保持 `runPrompt` 合约不变 | 删 `claude-agent-sdk.ts` ~538 行 → 替换 ~400 行；净 ~140 行删除 | 中（模型绑定决定） | 第 3-6 周 |
| **2. 把进程清理搬到 dsh 的 SIGKILL 阶梯** | 从 `claude-code.ts` 删 `cleanupClaudeCodeProcesses`；依赖 `HarnessClient.close()` | 删 ~80 行；删启发式的测试 | 低 | 第 7 周 |
| **3. 把 E2B 挂成 `dsh-sandbox-e2b-lumos`** | 把 `E2BAgentSandboxProvider` 搬进 dsh 插件 | 搬 ~275 行；`AgentSandboxProvider` 接口变成 dsh `Service Definition` | 低 | 第 8-9 周 |
| **4. 把 security / guard / sanitization 搬到 dsh 插件** | 挂 `dsh-guard-lumos`，带 `assertAgentRuntimeProductionBoundary` + `sanitizeAgentSandboxEnvs` | 搬 ~225 行 | 低 | 第 10 周 |
| **5. 把 skillpack 同步搬到 dsh extension** | 挂 `dsh-skill-lumos-bridge` | 搬 ~162 行 | 低 | 第 11 周 |
| **6. 把 broker 重构为 `ctx.llm` adapter** | 写 `dsh-llm-lumos-broker`；移除进程内的 `BrokerRequestError` schema 校验；搬进插件 | 搬/替换 ~1,786 行 | **高**（模型绑定产品决定） | 第 12-15 周 |
| **7. 重构 BRT 的 subagent 流** | 用 dsh subagent 组合替换 `Agent` 工具调用 | 重写 `roundtable-brt/runner.ts` ~600 行（4,612 行中 subagent 专属部分约 600）；`subagent_type` → dsh `subagent_id`；测试 re-baseline | **高**（产品可见） | 第 16-20 周 |
| **8. 决定 Claude 怎么办** | 模型现在是 `cordis.yml` 决定。如果 Lumos 想为 NBE/BRT 保留 Claude，挂 `llm-pi-ai` 并通过它路由到 broker。如果 Lumos 想丢掉 Claude，删 `llm-pi-ai` 并注册 `dsh-llm-deepseek`（或自定义 DeepSeek adapter） | 自身 0 行；翻动其他配置 | 中（产品） | 第 21 周 |

**净估算：** 搬/替换 ~3,800 行，删 ~1,500 行（主要是进程清理启发式、sentinel marker 和变成 dsh config 的模块专属 allowlist）。全程测试 re-baseline。总数可比 Lumos 一次大重构 —— 约 runtime 的**一半**（第一稿是一季度；broker 重构比初估大）。

**日历估算（粗）：** 1-2 个工程师，**3-4 个月**，有很长的 overlap 窗口两个 runtime 同时出货。BRT 重构（第 7 阶段）是风险最高的步骤，因为它改变了用户可见的产品面。

## 8. 边界应该画在哪里

如果 Lumos 采用 dsh，建议的拆分：

| 关注点 | 在哪 |
|---|---|
| Agent loop、model adapter、session log、hooks、tool 注册表、subagent backend、进程内插件树 | **dsh**（`dsh-base` + 一个 `dsh-profile-lumos` profile，挂所有 Lumos 专属插件） |
| E2B sandbox provider、template governance、image builder、snapshot 策略 | **Lumos**（一个 `dsh-sandbox-e2b-lumos` 插件，在 Lumos 仓库维护） |
| Broker、grants、request-schema 校验、模型 allowlist | **Lumos**（一个 `dsh-llm-lumos-broker` 插件；broker 是 Lumos 的产品，不是 dsh 的原语） |
| NBE phase、report gate、repair 轮次、quality gate | **Lumos**（不变；它们在 runtime 之上） |
| BRT 圆桌流、moderator planner、memo orchestrator | **Lumos**（不变；但 subagent 执行从 Claude Code 的 `Agent` 工具迁到 dsh 的 `ctx.subagent`） |
| Skillpack 内容（实际的 `skills/` 目录） | **Lumos**（不变；`dsh-skill-lumos-bridge` 只管把 skills *投递*到 dsh 树里） |
| DB schema、Better Auth、Next.js app、设计系统、CI | **Lumos**（不变；dsh 是后端关注点） |

这个拆分意味着：

- Lumos 保持它的应用身份（Next.js、Better Auth、E2B、broker）。
- dsh 拿走了那些应该跨 profile 可复用的部分（loop、model、session、hooks、subagent）。
- E2B 模板构建器和 broker 留在 Lumos，因为它们编码了产品决定（数据存哪、"不自动降级"是什么意思）。
- 边界由 **patch 分层** 强制：Lumos 专属插件作为最上层挂在 `dsh-base` 之上，操作员编辑每个 profile 的 `cordis.patch.yml`。

## 9. dsh 在 Lumos **不**适合的东西

这些不是 dsh 的失败；是 Lumos 的领域是它自己的，dsh 应该不插手。

1. **Lumos agent broker。** Broker 是铸造短期 grant、校验严格 request schema（8 顶层 key、32 工具上限、8 MB 请求 / 1 MB 系统字节上限）、强制每 job / 每 user / 全局速率限制的 HTTP 代理。这是应用级身份和速率限制，不是 LLM adapter。dsh 没有匹配的 `ctx.llm` 形状。留在 Lumos。

2. **Next.js chat UX、chat state snapshot 模型、workspace memory CRUD、设计系统、Playwright smoke harness。** 这些都不是 agent-runtime 关注点。留在 Lumos。

3. **圆桌 BRT 产品决定**：谁是 partners、moderator persona、memo 模板、formal-speech sanitizer、dedupe 规则、live-patches 流。这些都是 BRT 产品。留在 Lumos。

4. **NBE 产品决定**：phase 顺序、quality gate、report repair 轮次、artifact-store schema。都是 NBE。留在 Lumos。

5. **E2B 模板治理**：digest 钉死的 Node、精确的 npm lockfile、命名模板唯一性、build ID + 名字后缀、无 tag 的 `assignTags` 策略。这是运维。采用时搬到 `dsh-sandbox-e2b-lumos` 插件；别指望 dsh 出货。

6. **Vercel AI SDK 消费者**（chat UX、diagnosis graph、year-plan workflow）。它们不用 Claude Agent SDK；别动它们。

7. **"圆桌对话固定 Opus 4.6" 产品规则。** 这是 BRT 产品决定。如果迁移，把它编码为 `dsh-llm-strict-model` 插件，但策略是 Lumos 的，不是 dsh 的。

## 10. 结论与采用路径

**dsh 对 agent-runtime 层是好的架构契合，对 broker 层和产品层是差的契合。**

把 sandbox 里的 Claude Agent SDK 换成 dsh 的机械工作是真实的但有界的（~3,800 行搬/替换，~1,500 行删除，3-4 个月与当前 runtime 重叠）。架构工作 —— 把整个 host runtime 脚手架换成 `dsh-base` + 一个 Lumos profile —— 是更大的投资，也是真正有回报的，因为它让 loop、model、persistence、tool 注册表变成可换的而不是手写的。

### 10.1 三条采用路径，按风险排序

| 路径 | 内容 | 成本 | 收益 |
|---|---|---|---|
| **A. 组合（最低风险）** | 保留 Claude Agent SDK；通过 `subagent-claude-code` 把它注册为 dsh subagent backend。dsh 主持编排；broker 留在 Lumos；持久化层（Phase1、exploration_jobs、chat_state_snapshots）留在 Lumos。 | 插件接线 ~600 行，无删除。4-6 周。 | dsh 成为 *外部* runtime；Claude 成为内部 backend。dsh 的"Meta-runtime"优势（可组合性）被实现，模型绑定不动。**这是推荐的起点。** |
| **B. 替换 sandbox（中风险）** | 在 E2B sandbox 里把 Claude Agent SDK 调用换成 `DeepSeekHarness.run()`。dsh 现在是内部 runtime；Claude 没了。 | 搬/替换 ~3,800 行，删 ~1,500 行。3-4 个月。 | "loop 可换"的优势被实现。模型绑定现在是 dsh 的。 |
| **C. 全面替换 base（最高风险）** | 把整个 `src/lib/agent-runtime/` 脚手架换成 `dsh-base` + 一个 `dsh-profile-lumos` profile。Next.js 进程里启动一个 dsh runtime；E2B sandbox 变成 dsh `ctx.sandbox` provider；broker 变成 `dsh-llm-lumos-broker` adapter。 | 搬/替换 ~6,300 行，删 ~2,000 行。5-6 个月。 | 整个 stack 都是组合；操作员编辑一个 `cordis.patch.yml` 改策略。代价是最大的迁移面。 |

### 10.2 推荐的路径

**先走 A。** 把 dsh 立成薄编排层；通过 `subagent-claude-code` 保留 Claude Agent SDK 作为 subagent backend。这是 ~600 行插件接线，4-6 周，给你可组合性的故事而不动模型绑定。

然后，**只有当模型绑定是真产品决定**时（你想加 DeepSeek route 或本地模型 route 而不重建 broker），才走 **B**。B 的主要成本是 E2B 模板构建（sandbox 里需要 dsh runtime），不是代码改动。

**C 对新项目才是正确答案，不是迁移。** 如果 Lumos 是 greenfield，那会是显然的选择。对于一个已经测试过的 broker、测试过的 E2B 管道、已经上了 Claude 的产品面的现有项目，C 是 6 个月的承诺，应该作为单独决定的主题。

### 10.3 何时在 Lumos 用 dsh

| 如果你想… | 推荐 |
|---|---|
| 在不重做 broker 的前提下，加一个 Claude 之外的第三方 LLM provider | **只走 A** —— 把 Claude 注册为 dsh subagent backend；broker 还是 Claude 绑定的。要加 provider，给它写一个 dsh subagent backend |
| 在 sandbox 里把 Claude Code CLI 换成一个更小的、可审计的 runtime | **走 B，只到阶段 0-2** —— 让 SDK 切换工作，其他先不动；最大收益是干掉 `claude-code.ts` 的启发式 |
| 摆脱自定义的进程清理 / 断流 / 退出 marker 逻辑 | **先 A，然后 B 阶段 2** —— 这是最干净的胜利；~80 行 bash + 正则消失 |
| 让 agent loop 可换（比如把 Claude Code 换成本地小模型） | **走 C，所有阶段** —— 这是架构上的胜利，也是唯一能 justify 日历成本的 |
| 让 broker 与模型无关 | **走 B，阶段 6** —— dsh 的 `ctx.llm` 是正确的形状；工作是隔离的 |
| 在没有 E2B 的情况下跑 Lumos workflow（比如本地开发机） | **只走 A** —— `dsh-base` + `dsh-headless` 给你"无 sandbox"模式，是真正的 profile 而不是 stub |
| 在多租户 hosted dsh 中跑 Lumos workflow | **不为这个采用 dsh** —— dsh 当前不出货多租户原语（每 user 速率限制、每 user 存储根、每 user 审计）；Lumos 自己会建，但会在 `src/lib/` 而不是 `packages/`，收益小 |
| 把 NBE / BRT 暴露为自动化 server（Zed、JetBrains、自定义编辑器） | **A，机会主义** —— dsh 的 `dsh-acp` 是正确的形状，但你会在 BRT/NBE runner 前暴露一个 Lumos 专属的 ACP server；dsh ACP server 是参考，不是 drop-in |

### 10.4 何时不采用 dsh

- Lumos 短期内对 Claude 满意，只想要一个更小的 sandbox runtime。采用 `dsh-base` + 一个 headless profile（A 路径）；跳过 broker 重构。
- Lumos 需要多租户 hosted dsh。原语没有；在 Lumos 中自建并跳过 dsh 更快。
- Lumos 需要在接下来 4 周内出货。dsh 是 `0.1.0-rc.5`；并行 canary 的日历成本不可忽略。
- BRT 圆桌依赖 `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` 和 `experimental_repairText` 回调。dsh 对这两个都没有 drop-in。A 路径通过 `subagent-claude-code` 保留它们；B/C 路径会丢掉它们，除非你建替代品。

## 11. 引用的源文件

### dsh（本仓库）
- `packages/sdk/client/README.md` —— TS SDK 面
- `packages/sdk/server/README.md` —— stdio JSON-RPC server
- `packages/sdk/protocol/README.md` —— wire 类型
- `python/sdk/README.md` —— Python SDK 面
- `docs/architecture.md` —— 插件树、sessions、events
- `docs/cordis-primer.md` —— Cordis 框架
- `packages/llm/llm/README.md` —— LLM runtime
- `packages/core/agent/README.md` —— agent 注册表
- `packages/core/tools/README.md` —— 工具执行管道
- `packages/core/session/README.md` —— 事件源 session log
- `packages/subagent/README.md` —— subagent 家族
- `packages/subagent/subagent-claude-code/README.md` —— Claude Code 作为 dsh subagent
- `packages/subagent/subagent-codex/README.md` —— Codex 作为 dsh subagent
- `packages/subagent/subagent-dsh-sdk/README.md` —— dsh 作为 dsh subagent（递归）
- `packages/subagent/subagent-acp/README.md` —— ACP 作为 dsh subagent
- `packages/hooks/README.md` —— Claude/Codex hook bridges
- `packages/acp/acp/README.md` —— dsh 自己的 ACP server
- `packages/bundle/base/README.md` —— dsh-base bundle
- `packages/bundle/headless/README.md` —— dsh-headless bundle
- `packages/bundle/web-app/README.md` —— dsh-web-app bundle
- `packages/sandbox/README.md` —— sandbox 能力家族
- `packages/guard/README.md` —— loop-hygiene guards
- `packages/compaction/README.md` —— compaction 能力家族
- `.agents/notes/implemented/feature/` —— 510 个已实施特性 note
- `.agents/notes/proposed/feature/` —— 提议中的特性
- `.agents/notes/proposed/architecture/` —— 提议中的架构

### Lumos（`/Users/harvey/Desktop/Harvey-files/08Myprojects/lumos/Lumos`）
- `src/lib/agent-runtime/types.ts` —— 能力接口 (112 行)
- `src/lib/agent-runtime/claude-agent-sdk.ts` —— Claude Agent SDK executor (538 行)
- `src/lib/agent-runtime/claude-code.ts` —— Claude Code CLI 包装 (344 行)
- `src/lib/agent-runtime/e2b-sandbox.ts` —— E2B sandbox 生命周期 (275 行)
- `src/lib/agent-runtime/skillpack.ts` —— skillpack 同步 (162 行)
- `src/lib/agent-runtime/security.ts` —— env 清洗 + production 边界 (125 行)
- `src/lib/agent-runtime/config.ts` —— runtime config (214 行)
- `src/lib/agent-runtime/remote-files.ts` —— host→sandbox 文件同步 (101 行)
- `src/lib/agent-runtime/allowed-tools.ts` —— 模块专属工具 allowlist (15 行, NBE 12 + BRT 5)
- `src/lib/agent-runtime/timeouts.ts` —— clamp 助手 (18 行)
- `src/lib/agent-runtime/broker-egress.ts` —— broker egress 接线 (116 行)
- `src/lib/agent-runtime/broker-grants.ts` —— 短期 grant 铸造 (429 行)
- `src/lib/agent-runtime/broker-config.ts` —— broker config 校验 (134 行)
- `src/lib/agent-runtime/broker-request.ts` —— request schema 校验 (496 行)
- `src/lib/agent-runtime/broker-search-proof-core.ts` —— 证明 artifacts (357 行)
- `src/lib/agent-runtime/broker-search-proofs.ts` —— 证明持久化 (159 行)
- `src/lib/agent-runtime/agent-runtime.test.ts` —— 主测试文件 (1,179 行)
- `src/lib/roundtable-brt/runner.ts` —— BRT runner (4,612 行, 最大的消费者)
- `src/lib/roundtable-brt/constants.ts` —— BRT 路径 + `CLAUDE_AGENT_SDK_VERSION = "0.3.220"`
- `src/lib/roundtable-brt/moderator-planner.ts` —— 使用 `experimental_repairText`
- `src/lib/roundtable-brt/crisis.ts` —— 使用 `experimental_repairText`
- `src/lib/exploration/runtime/agent-runner.ts` —— NBE runner (280 行, runtime 之上)
- `src/lib/exploration/runtime/runtime-manager.ts` —— NBE 编排
- `src/lib/exploration/runtime/phase1-structured.ts` —— 有 `runtimeBackend` + `sandboxBackend` payload 字段
- `docs/current/Agent-Runtime-通用执行底座.md` —— Lumos 自己的 runtime 合约
- `docs/current/E2B-Agent-模板测试服发布.md` —— 模板 build/promote 流
- `docs/code-intelligence/domains/agent-runtime.md` —— runtime 的 codemap
- `scripts/e2b/install-claude-agent-sdk.sh` —— 模板里的 SDK 安装
- `scripts/e2b/build-nbe-template.ts` 和 `scripts/e2b/build-brt-template.ts` —— 模板构建器
