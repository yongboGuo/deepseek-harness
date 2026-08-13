# Lumos × dsh — adoption evaluation

> Snapshot date: 2026-08-14. Authoritative versions probed on that date:
> Lumos at `/Users/harvey/Desktop/Harvey-files/08Myprojects/lumos/Lumos` (Next.js 16 + Bun, current `master`),
> dsh at `0.1.0-rc.5` (TS) / `0.0.0.dev0` (Py).
>
> This document maps Lumos's current agent runtime to dsh's plugin tree, and answers
> the three questions a team considering dsh would actually ask:
>
> 1. **What does Lumos use Claude Agent SDK for today, and which of those uses are dsh-shaped?**
> 2. **Can dsh replace Claude Agent SDK as Lumos's harness base?**
> 3. **Where should the boundary be drawn, and what is the migration cost?**
>
> Reading the source: Lumos's runtime lives in
> `src/lib/agent-runtime/`. The consumers are `src/lib/exploration/runtime/agent-runner.ts`
> (NBE) and `src/lib/roundtable-brt/runner.ts` (BRT). dsh's surface lives in
> `packages/sdk/{client,protocol,server}` and `packages/bundle/{base,web-app,headless}`.
>
> **This is a deep evaluation, not a port plan.** If you want the 8-stage plan only, see
> §10. If you want the file-by-file line count and risk, see §7. If you want the
> architectural reasoning, start at §3.

## 1. Lumos's current runtime shape (what we are evaluating against)

### 1.1 The runtime is two layers

| Layer | Today | dsh analogue |
|---|---|---|
| **Host runtime** (the Next.js process) | `E2BAgentSandboxProvider` (`src/lib/agent-runtime/e2b-sandbox.ts`): creates / connects / snapshots / disposes E2B sandboxes, applies `denyOrKillAgentSandboxNetwork`, recovers from capacity errors, exposes `AgentSandboxProvider` interface | `dsh-base` + a new `dsh-sandbox-e2b` plugin could mount E2B behind `ctx.sandbox` |
| **Sandbox runtime** (inside the E2B VM) | `ClaudeAgentSdkExecutor` (`src/lib/agent-runtime/claude-agent-sdk.ts`): writes a Node ESM runner that imports `@anthropic-ai/claude-agent-sdk@0.3.220` and calls `query({ prompt, options })` over stdio; the SDK then spawns the bundled `@anthropic-ai/claude-code@2.1.220` CLI which is what actually does the work | dsh's own `dsh-jsonrpc-agent` is the same kind of thing — a runtime you launch over stdio. To replace Claude Agent SDK, the sandbox runner would import `@deepseek-ai/dsh-sdk-client` and call `DeepSeekHarness.run(prompt)` instead. Or — and this is the new option §0.1 of the comparison doc opens up — keep Claude Agent SDK in place and route it through `subagent-claude-code` as a subagent backend inside a dsh session. |

### 1.2 What Lumos does on top of the SDK

The dsh-shaped concerns live in *Lumos's host runtime*, not in the SDK call itself:

| Concern | File | LOC | Notes |
|---|---|---|---|
| Sandbox lifecycle / checkpoint / snapshot / pause / dispose | `e2b-sandbox.ts`, `remote-files.ts`, `timeouts.ts` | 394 | stateful, owns a long-lived sandbox for hours at a time |
| Skillpack sync (workspace `skills/` → sandbox `.lumos-agent-sdk/skills/` and `.claude/skills/`) | `skillpack.ts` | 162 | fingerprinted cache; declarative `resolveSkillRoot()` |
| Secret / env scrubbing | `security.ts` (`sanitizeAgentSandboxEnvs`, `assertAgentRuntimeProductionBoundary`) | 125 | hard-coded `SAFE_SANDBOX_ENV_NAMES` allowlist, `AgentRuntimeSecurityError` short-circuits production |
| Network egress allowlist + broker | `broker-egress.ts`, `broker-grants.ts`, `broker-config.ts`, `broker-request.ts`, `broker-search-proof-core.ts`, `broker-search-proofs.ts` | 1,786 | **Crucial**: production must go through a Lumos-owned `/api/internal/agent-broker/anthropic` proxy that mints short-lived grants; the sandbox network is denied by default (`denyOrKillAgentSandboxNetwork`) |
| Process cleanup / detached stream recovery | `claude-code.ts` (`cleanupClaudeCodeProcesses`), `claude-agent-sdk.ts` (`looksDetachedCommandStreamFailure`) | ~80 | heuristic process scanning inside the sandbox |
| Tool allowlists | `allowed-tools.ts` (`NBE_AGENT_ALLOWED_TOOLS`, `BRT_ALLOWED_TOOLS`) | 15 | module-specific allowlists (NBE: 12 tools; BRT: 5 tools) |
| Template governance | `scripts/e2b/build-nbe-template.ts`, `scripts/e2b/build-brt-template.ts`, `scripts/e2b/install-claude-agent-sdk.sh` | ~600 | builds a minimal image: Node `24.18.0` (digest-pinned), Claude Code CLI, Claude Agent SDK package, exact npm lockfile, no node version manager |
| Phase / checkpoint / quality-gate / artifact / report gate | `src/lib/exploration/runtime/agent-runner.ts` and surrounding phases | 280 | **NBE-specific**; out of scope for the agent runtime itself |
| Multi-agent discussion (moderator + 2–3 partners) | `src/lib/roundtable-brt/runner.ts` (4,612 LOC), `moderator-planner.ts`, `memo-orchestrator.ts`, etc. | 4,600+ | **BRT-specific**; out of scope for the agent runtime itself |

### 1.3 The two SDK consumers

#### (a) NBE — Notebook Exploration (`src/lib/exploration/`)

- Long-running phase pipeline (intake, research, analysis, report). Each phase is a Claude Code invocation.
- Hard contract: NBE adapter "固定实例化 `ClaudeAgentSdkExecutor`; 不存在可切换到 CLI 的运行时配置" — there is **no runtime switch today**. The SDK is the single execution path.
- Skill sync: "NBE 真实执行固定读取仓库内 `skills/` 并同步到 E2B" — repo-local skills only.
- Final gate: `NBE_CLAUDE_EXECUTION_MODE=lumos_controller | orchestrator_parity` and `NBE_REPORT_READY_GATE=release` produce hard/publish/semantic gates; `PASS_WITH_WARNINGS` is an internal candidate-only state.
- Report repair rounds: `NBE_REPORT_REPAIR_MAX_ROUNDS=3` — re-invokes the agent inside the same sandbox to fix/extend the report.
- **Multi-tool set**: NBE allows 12 tools (`Task, Glob, Grep, LS, Edit, MultiEdit, Write, NotebookRead, NotebookEdit, TodoWrite, WebSearch, …`). This is the NBE macro/market evidence path; the broker host is reviewed for that exact set.
- The `Phase1StructuredRunnerPayload` already declares `runtimeBackend` and `sandboxBackend` fields — the architecture has the concept of multiple backends, just not at the runtime level.

#### (b) BRT — Business Roundtable (`src/lib/roundtable-brt/`)

- 2–3 "合伙人" sub-agents + a moderator, each is a `subagent_type`. They run in the same sandbox via the `Agent` tool's subagent mechanism, with five allowed tools (`Agent`, `Glob`, `Grep`, `LS`, `TodoWrite`).
- Subagent executor is injected via `createRoundtableClaudeDiscussionRuntime({ subagentExecutor })`. The DB-observable process event is `claude-agent-sdk-subagents`, audited as `done | queued | error`.
- The runner is **4,612 LOC**. It is the largest single consumer of `claude-agent-sdk` and contains the multi-agent orchestration logic, the moderator planner, the memo orchestrator, the public-turn/memo/decision-board/state/event-adapter subsystems, and the lifecycle/abort/turn-lease/preflight pipeline.
- Model is "圆桌对话固定 Opus 4.6" with **no automatic model downgrade** — a 503 from the default token group surfaces as `subagentRuntimeSucceeded=false` and a hard failure, not a silent retry on a smaller model. This is a **deliberate** product choice that dsh does not currently mirror; the closest dsh construct is a `ctx.llm` adapter with a `providerRetryPolicy()` that rejects model downgrades, but it is not a default.
- BRT passes `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` to the sandbox when configured. This is the Claude-side experimental surface for cross-agent collaboration; Lumos uses it when the moderator planner decides to enable it.
- BRT uses `experimental_repairText` (the Claude Agent SDK's structured-output repair callback) in `moderator-planner.ts`, `crisis.ts`, and elsewhere. This is a Claude-specific API; dsh's structured-output path is the `dsh-system-prompt` + `tools/post-execute` decision path.

### 1.4 What runs alongside the SDK but does not depend on it

These are LLM-shape but not Claude-shape, and would be a relatively safe co-adoption target if Lumos wanted to incrementally introduce dsh without disrupting Claude:

- The Lumos chat UX: `src/lib/chat-content`, `src/components/chat/*`, `src/app/chat/*`. Uses Vercel AI SDK (`@ai-sdk/anthropic`, `@ai-sdk/openai`) directly, not Claude Agent SDK. **dsh would not touch this**.
- The diagnosis graph executor (`src/lib/diagnosis/`): Vercel AI SDK only, no Claude Agent SDK. **dsh would not touch this**.
- The year-plan workflow (`src/lib/year-plan/`): Vercel AI SDK only, no Claude Agent SDK. **dsh would not touch this**.
- The roundtable debate (the moderator + partners in BRT) is on the Claude Agent SDK; the *roundtable product* (state, public turns, memo, decision board, billing, dedupe) is on Lumos's own DB. **dsh would touch the SDK side, not the product side.**

The implication is significant: the surface area that depends on Claude Agent SDK is concentrated in two pipelines (NBE and BRT) out of the whole Lumos app. Everything else is independent.

## 2. dsh's surface, mapped to Lumos

### 2.1 The dsh plugin tree, by capability family

dsh organizes its 50+ packages into capability families, each a `Service Definition` (the seam) plus 0..N `Service Provider`s (backends that register on the seam) plus optional `Consumer`s (model-facing tools that expose the seam). This is the only structural pattern in the comparison; you should internalize it before reading the rest of the doc.

| Family | dsh key | Service Definition | Shipped backends | Lumos analogue |
|---|---|---|---|---|
| `llm` | `ctx.llm` | `LlmRuntime` (adapter registry + stream API) | `dsh-llm-deepseek`, `dsh-llm-pi-ai`, … | broker (different shape) |
| `core/tools` | `ctx.tools` | `ToolRegistry` + waterfall pipeline | bash, fs, search, todo, subagent, session-query | `NBE_AGENT_ALLOWED_TOOLS` / `BRT_ALLOWED_TOOLS` (hard-coded arrays) |
| `core/agent` | `ctx.agents` | `AgentRegistry` + `agent/*` events | `core/agent-loop` (default driver) | n/a (Lumos builds its own loop) |
| `core/session` | `ctx.sessions` | `SessionStore` + `Session` event-sourced log | persistence plugin (JSONL) | Postgres-backed session persistence in Lumos |
| `sandbox` | `ctx.sandbox` | process-confinement seam | `dsh-sandbox-local` (POSIX ACL, win32 ACL) | `E2BAgentSandboxProvider` (remote, not process-level) |
| `fs` | `ctx.fs` | filesystem seam | `dsh-fs-local`, `dsh-fs-remote` | `remote-files.ts` (push files into sandbox) |
| `subagent` | `ctx.subagents` | subagent provider registry | `subagent-spawn-in-process`, `subagent-fork-in-process`, `subagent-acp`, **`subagent-codex`**, **`subagent-claude-code`**, **`subagent-dsh-sdk`** | n/a (Lumos's BRT has its own subagent flow via `Agent` tool) |
| `compaction` | `ctx.compaction` | compaction seam | `compaction-basic`, `compaction-tool-result-pruner`, `command-compact` | n/a (Lumos relies on Claude auto-compaction) |
| `guard` | n/a (event consumer) | behavior guards | `repeat-tool-reminder`, `timeout-policy` | `assertAgentRuntimeProductionBoundary`, `sanitizeAgentSandboxEnvs` |
| `mcp` | n/a | MCP server mount | various MCP packages | n/a (Claude Agent SDK has in-process MCP; dsh exposes it as a plugin) |
| `hooks` | `ctx.hooks` | hook bridge (intercepts at lifecycle points) | `hooks-claude-code`, `hooks-codex` | n/a (Lumos has its own broker-side validation) |
| `interaction/approval` | `ctx.approval` | user approval service | shipped with `dsh-base` | n/a (Lumos's `NBE_REPORT_READY_GATE`) |
| `host` | `ctx.webServer`, `ctx.apiProxy` | web host half | webserver, apiproxy, plugin-inventory | Lumos's Next.js app |
| `bundle/base` | (composition) | first layer of every profile | model adapters + tools + persistence + policy + Claude/Codex providers (dormant) | n/a |
| `bundle/web-app` | (composition) | web host | webserver, browser plugin roster, persona, `web-runtime` | Lumos's Next.js app |
| `bundle/headless` | (composition) | one-shot runner | persona, Code Mode worker, `headless-runner` | Lumos's BRT single-shot discussion |
| `acp/acp` | n/a | automation-only ACP server | `@deepseek-ai/dsh-acp` (stdio JSON-RPC) | n/a |
| `acp` | n/a | ACP client for subagent | `subagent-acp` | n/a |

### 2.2 The exact match table (dsh concept → Lumos file)

| dsh concept | dsh location | Maps to Lumos today | Migration impact |
|---|---|---|---|
| `dsh-base` profile | `packages/bundle/base` (with `cordis.patch.yml` listing ~200 rows) | n/a — Lumos doesn't ship a profile at all; it ships a hand-rolled host runtime | **large** — would replace the entire host runtime scaffold |
| `dsh-headless` profile | `packages/bundle/headless` | partial — `LUMOS_ROUNDTABLE_AGENT_RUNTIME=e2b` paths are headless; NBE has long-running artifacts that don't fit "one prompt in, one answer out" | **medium** — the BRT single-shot path could be replaced; NBE's multi-phase repair loop cannot, by itself |
| `cordis.patch.yml` layering | `packages/bundle/*/cordis.yml` + `packages/boot/app-boot` | today: hard-coded `claude-agent-sdk.ts` + `claude-code.ts` + `e2b-sandbox.ts` | **medium** — patch layers are a natural fit for Lumos's "module-specific runtime" model (NBE vs BRT vs future modules) |
| `ctx.sandbox` capability seam | `packages/sandbox` | `AgentSandboxProvider` (Lumos's own seam) | **low** — dsh's `ctx.sandbox` could host a `dsh-sandbox-e2b` plugin that does exactly what `E2BAgentSandboxProvider` does today |
| `ctx.fs` provider | `packages/fs` | `remote-files.ts` (push files into sandbox) | **low** — but the seam would have to support a "remote + on-demand sync" mode, not just a local provider |
| `ctx.tools` registry + `tools/*` events | `packages/core/tools` | `NBE_AGENT_ALLOWED_TOOLS` / `BRT_ALLOWED_TOOLS` (hard-coded arrays) | **low–medium** — allowlists become plugin-level config; tool allowlist events become waterfalls |
| `ctx.hooks` lifecycle | `packages/hooks` + `dsh-hook-protocol-lib` + `dsh-hook-bridges` | none directly; Lumos relies on Claude Code's `PreToolUse`/`PostToolUse` etc. via the SDK | **medium** — dsh's hook system is generic; Lumos would have to wire its broker, scrubber, and gate logic as plugins |
| `ctx.llm` adapter seam | `packages/llm/llm`, `packages/llm/*` | `DMX_BASE_URL` + `DMX_API_KEY` in `cordis.yml` for the **host**; the **sandbox** goes through the Lumos agent broker | **high** — this is the key compatibility question. See §3. |
| `ctx.guard` policy seam | `packages/guard` | `assertAgentRuntimeProductionBoundary`, `isExplicitStagingAgentBrokerEnabled` | **low** — same shape, different file |
| `ctx.subprocess` (Lumos's exception case) | `packages/subprocess` | `claude-agent-sdk.ts`'s private `disposeEofGraceMs` ladder | **low** — the SDK client's private SIGKILL ladder already does this; no change needed |
| `dsh-skill` + `extensions` | `packages/skill` + `packages/extensions` | `DefaultSkillpackSyncer` (Lumos's) | **medium** — would need a `dsh-skill-lumos-bridge` to feed Lumos's repo-local skills into dsh's extensions tree |
| `dsh-compaction` capability seam | `packages/compaction` | none — Lumos relies on Claude's auto-compaction | **low** — when dsh's `recallable-compaction` lands, it could replace auto-compaction with auditable seams |
| `dsh-session` + `dsh-session-query` | `packages/session`, `packages/session-query` | Lumos DB tables (`exploration_jobs`, `chat_state_snapshots`, `roundtable_*`) | **high** — Lumos's session persistence is application-level (Postgres + Better Auth + chat_state), not the agent's; the agent's session log is currently the SDK's CLI files |
| `dsh-job` + `ctx.jobs` background work | `packages/jobs` | `nbe:worker`, `roundtable:worker` (Bun workers in the Next.js process) | **medium** — dsh's job service runs *inside* the dsh process; Lumos's workers run *outside* the E2B sandbox. These are two different job systems, not replacements |
| `dsh-tool-terminal` / `ctx.terminals` | `packages/extension/terminal` | `cleanupClaudeCodeProcesses` in `claude-code.ts` | **low** — same problem, different mechanism |
| `dsh-mcp` | `packages/mcp` | Claude Agent SDK supports in-process MCP via `@tool` + `create_sdk_mcp_server`; Lumos does not use it today | **low** — dsh mounts MCP as a plugin; equivalent in shape, more verbose to set up |
| `dsh-subagent` capability seam | `packages/subagent` | BRT's multi-agent flow | **medium–high** — dsh's subagent surface is provider-portable; BRT's current subagent execution assumes Claude's `Agent` tool. Porting BRT means choosing a subagent backend |
| `dsh-acp` automation server | `packages/acp/acp` | n/a | **opportunity** — Lumos could expose its NBE/BRT runner as an ACP automation server, allowing external tools (Zed, JetBrains, custom editors) to drive the same pipeline. dsh already ships the server. |

### 2.3 The unique position: dsh can *use* Claude Agent SDK as a backend

Per §0.1 of the SDK comparison: dsh ships `subagent-claude-code`, which uses the official Claude Agent SDK as a subagent backend inside a dsh session. This means **dsh does not require Lumos to abandon Claude Agent SDK**.

A migration path that does not require abandoning Claude:

1. Lumos wraps its current `ClaudeAgentSdkExecutor` in a thin adapter that registers as a dsh subagent provider (`@deepseek-ai/dsh-subagent-lumos-claude-sdk`).
2. Lumos's NBE/BRT runners route through dsh's `subagent` tool instead of importing `@anthropic-ai/claude-agent-sdk` directly.
3. The dsh session hosts the orchestration, the broker, the persistence, the policy; the Claude Code child does the model work.
4. dsh's `hooks-claude-code` bridge translates Claude's `hooks.json` into dsh's interception points, so existing Claude hooks keep working.

This is a fundamentally different migration from "replace Claude with dsh". It is "host Claude inside dsh". It does not require any of the model-binding changes in §3.1.

## 3. The hard questions

### 3.1 Can the sandbox-side runtime be swapped from Claude Agent SDK to dsh?

**Yes, mechanically, but with caveats.**

The current sandbox runner (`claude-agent-sdk.ts`) is a Node ESM script that:

1. Imports `@anthropic-ai/claude-agent-sdk`.
2. Calls `query({ prompt, options })` with options built from `AgentRuntimeConfig`.
3. Streams messages back to the host via E2B's `commands.run`.

Replacing it with dsh means:

```ts
// instead of
import { query } from '@anthropic-ai/claude-agent-sdk'
async for await (const msg of query({ prompt, options })) { ... }

// you write
import { DeepSeekHarness } from '@deepseek-ai/dsh-sdk-client'
await using harness = new DeepSeekHarness({
  launch: { command: 'node', args: ['lib/bin.js', 'cordis.yml'] },
  provider: 'deepseek-official',  // or whatever cordis.yml registers
  model: '<model>',
  maxTokens: 49_152,
})
const result = await harness.run(prompt)
```

This is mechanical. The reasons **not** to do it today, in order of severity:

1. **dsh is `0.1.0-rc.5`.** "THERE WILL BE COMPATIBILITY-BREAKING CHANGES." Lumos's current SDK pin is `0.3.220` with a tested 8-month history and a vendor-bundled CLI.
2. **The Claude Code CLI is what the sandbox actually runs.** Replacing the SDK does *not* remove the dependency on a CLI in the sandbox — dsh needs its own runtime to be in the sandbox. Today, that means `node lib/bin.js cordis.yml` (the SDK contract assumes you bring the runtime). You have to build a `nbe-latest-dsh-terminal-mvp` template, parallel to `nbe-latest-claude-terminal-mvp`, with a pinned Node 24.18.0 and a bundled dsh runtime.
3. **The model binding is the real product decision.** Today, NBE/BRT are pinned to Claude (Opus 4.6 for roundtable, configurable for NBE) with an `anthropicBaseUrl` that goes through Lumos's broker. dsh's `cordis.yml` decides the adapter; you'd register a DeepSeek route and possibly an OpenAI route, and lose Anthropic entirely (unless you also mount `llm-pi-ai`, which is a third-party adapter; the dsh project ships `llm-pi-ai` as a separate package, so it's a one-line mount).
4. **The BRT subagent flow is Claude-CLI-shaped.** dsh's `ctx.subagent` is a generic capability seam; BRT's current subagents rely on Claude Code's `Agent` tool with five specific tool names. Porting BRT means rewriting `roundtable-brt/runner.ts` to either spawn dsh subagents (one harness per subagent) or to compose one parent harness with the subagent backend mounted.

### 3.2 Can dsh replace the *host* runtime?

**Partially. The seam-aligned parts are good fits; the application-level parts are not.**

`src/lib/agent-runtime/` is ~6,300 lines. Splitting it by dsh-shape:

| File | Lines | dsh-shape? | What would move |
|---|---|---|---|
| `e2b-sandbox.ts` | 275 | **Yes** | a `dsh-sandbox-e2b-lumos` plugin (or just `dsh-sandbox-e2b` if Lumos contributes the upstream provider) |
| `remote-files.ts` | 101 | **Yes** | a `dsh-fs-remote-lumos` provider |
| `timeouts.ts` | 18 | **Yes** | a config validation plugin (or just `dsh-cmdline` runtime config) |
| `security.ts` | 125 | **Yes** | a `dsh-guard-lumos` plugin (`assertAgentRuntimeProductionBoundary`, `sanitizeAgentSandboxEnvs`) |
| `skillpack.ts` | 162 | **Yes** | a `dsh-skill-lumos-bridge` extension |
| `claude-code.ts` | 344 | **Partial** | the `cleanupClaudeCodeProcesses` heuristic is Claude-CLI-specific; the `CLAUDE_CODE_EXIT_CODE_MARKER` protocol is Claude-CLI-specific; replace with dsh's own exit / stream detachment machinery (or keep Claude as a subagent backend, see §2.3) |
| `claude-agent-sdk.ts` | 538 | **No** | replaced by `@deepseek-ai/dsh-sdk-client` calls (or, per §2.3, by `subagent-claude-code`) |
| `broker-egress.ts` / `broker-grants.ts` / `broker-config.ts` / `broker-request.ts` / `broker-search-proof-core.ts` / `broker-search-proofs.ts` | 1,786 | **Partial** | Lumos's broker is HTTP-API-shaped, not LLM-adapter-shaped. dsh has no `ctx.llm` adapter that can talk to a proxy expecting `/api/internal/agent-broker/anthropic/{...}` payload validation. You'd either build a custom `dsh-llm-lumos-broker` adapter, or refactor the broker to be a generic proxy that mounts behind `ctx.llm` |
| `config.ts` | 214 | **Partial** | dsh's typed config can host `LUMOS_AGENT_RUNTIME_*` keys; the hard ceilings (`hardMax`) are dsh-shaped, the env-name allowlist (`SAFE_SANDBOX_ENV_NAMES`) belongs in a guard plugin |
| `types.ts` | 112 | **Yes** | dsh's capability seam types are the right home; the `AgentSandboxProvider` / `ClaudeCodeExecutor` interfaces are dsh-shaped (a "service definition" + a "service provider") |
| `*.test.ts` | 1,500+ | **No** | Lumos's test suite is Vitest, dsh's is a different harness; this is mechanical port work |

The hard things dsh does **not** give Lumos:

- **Postgres-backed broker grants** (`broker-grants.ts`). dsh has no equivalent; you'd build a `dsh-grant-lumos-postgres` plugin.
- **The Lumos model allowlist pattern**: 5-tool for BRT, 12-tool for NBE, with module-specific `ConsumedAgentBrokerGrant["module"]` types. dsh's `ctx.tools` is shape-equivalent but does not ship module-keyed allowlists. You'd write a `dsh-tool-allowlist-by-module` extension that maps `ConsumedAgentBrokerGrant["module"]` to a `ctx.tools` filter.
- **The "圆桌对话固定 Opus 4.6" policy**. dsh has `maxTokens` per agent but no product-level "no model downgrade" guarantee; the adapter just routes to whatever the provider returns. If you want the same guarantee, you have to add a `dsh-llm-strict-model` plugin that rejects any `model !==` the configured one.
- **The Lumos CI / template-build pipeline.** `scripts/e2b/build-nbe-template.ts` is a custom Node 24.18.0 digest-pinned image builder. dsh does not currently ship an E2B template builder; you'd carry this over as-is.
- **The `experimental_repairText` callback** that BRT uses for structured-output repair. This is a Claude Agent SDK API; dsh's structured-output path is the `dsh-system-prompt` + `tools/post-execute` decision path, which is a different model. A port would need a `dsh-structured-repair` plugin that wraps `tools/post-execute` to perform schema repair.

### 3.3 What does dsh give Lumos that today costs custom code?

This is the most useful question. The list is not empty.

| Today, custom-built in Lumos | What dsh would replace it with |
|---|---|
| `claude-code.ts` process cleanup heuristic (`cleanupClaudeCodeProcesses` — Python `ps`/`kill` script) | dsh SDK's `HarnessClient.close()` private SIGKILL ladder — `disposeEofGraceMs` / `disposeGraceMs` / `shutdownTimeoutMs` are knobs you can tune, and the ladder is idempotent |
| `looksDetachedCommandStreamFailure` regex | dsh SDK's `TransportClosedError` (typed) |
| `claude-agent-sdk.ts`'s `runPrompt` contract (a single `prompt → result` facade over the SDK) | dsh SDK's `DeepSeekHarness.run()` directly — but you still want a Lumos-side wrapper for the E2B lifecycle, so the savings is the JSON-RPC layer, not the wrapper |
| `assertAgentRuntimeProductionBoundary` and `AgentRuntimeSecurityError` | dsh's `ctx.guard` capability seam — would need a `dsh-guard-lumos` plugin to wire the env-name allowlist, but the seam itself is provided |
| `sanitizeAgentSandboxEnvs` (env-name allowlist + credential regex) | dsh's `dsh-subprocess` `scrubbedParentEnv` plus a guard plugin |
| `BrokerRequestError` schema (8-key top-level allowlist, 32-tool cap, 8 MB request / 1 MB system byte caps) | dsh does **not** ship a request-schema guard. You'd carry this as a `dsh-llm-lumos-broker` plugin |
| `DefaultSkillpackSyncer` (fingerprint + sync into sandbox) | dsh's `dsh-skill` + `extensions` are the canonical home; you'd write a `dsh-skill-lumos-bridge` that maps `skills/` into the `dsh.extensions` tree |
| `E2BAgentSandboxProvider.acquire()` (connect → snapshot-restore → template; capacity recovery; `denyOrKillAgentSandboxNetwork` on every connect) | dsh's `ctx.sandbox` would host this, but the *acquire algorithm* is Lumos-specific (the connect-then-snapshot-then-template ordering) and would carry over as a `dsh-sandbox-e2b-lumos` plugin |
| `claude-code.ts` `CLAUDE_CODE_EXIT_CODE_MARKER` (a sentinel line printed by the runner so the host can parse the Claude Code exit code) | dsh SDK returns a typed `RunResult` with `finishReason` (`completed` / `max-tokens` / `error`); the marker becomes unnecessary |
| `BashToolOutput.timedOutAfterMs` handling (Claude-CLI-specific timeout reporting) | dsh's `SessionEvent` includes the same fact; the host can subscribe and react |
| `SandboxEgress.denyOrKillAgentSandboxNetwork` (E2B-specific network policy) | dsh's `dsh-sandbox-local` (POSIX ACL) or a `dsh-sandbox-e2b` plugin (E2B-specific) — same shape, different file |
| `MAX_REQUEST_BYTES = 8 * 1_024 * 1_024` and `MAX_SYSTEM_BYTES = 1 * 1_024 * 1_024` in `broker-request.ts` | dsh does not enforce byte caps at the LLM seam; the closest analogue is the adapter's own `prepareCall` reject path. The caps would stay in the `dsh-llm-lumos-broker` plugin |
| `ConsumedAgentBrokerGrant["module"]`-keyed `SAFE_TOOLS` map | dsh's `ctx.tools` is shape-equivalent; a `dsh-tool-allowlist-by-module` extension maps the broker module to a tool filter |

## 4. Risk analysis per concern

| Concern | If you migrate it wrong | Mitigation |
|---|---|---|
| Sandbox lifecycle | orphaned sandboxes, leaked E2B cost | dsh SDK's idempotent close + the dsh runtime's `ctx.sandbox.dispose()` callback; ship a metric for sandbox-hours/day |
| Broker | open proxy = open model billing | keep broker in Lumos; never let dsh route model calls directly; only route through `dsh-llm-lumos-broker` |
| Skillpack | missing skills → silent tool failures | fingerprint-based invalidation (already in `DefaultSkillpackSyncer`); keep the syncer, just re-target it at `dsh.extensions` |
| Process cleanup | zombie claude processes inside sandbox; detached streams | dsh SDK's SIGKILL ladder handles it; verify with `claude --version` smoke after E2B template change |
| Tool allowlist | model calls a tool that should not be available | dsh `ctx.tools` is per-agent-scope; bind the allowlist to the agent, not the session |
| Subagent (BRT) | wrong tool name in subagent definition | dsh's `subagent` tool has typed provider config; validate `subagent_type` at composition time, not at runtime |
| Persistence | session log divergence between host DB and runtime | either commit to dsh's session as source of truth, or mirror dsh's session into Lumos's DB via a `session/event` listener |
| Permissions | auto-accept on edit, miss approval gate | dsh's `ctx.approval` seam replaces Claude's `permission_mode`; map Lumos's gate flags to `ctx.approval` rules |
| Compaction | silent context loss | dsh's `compaction-basic` is a `CompactionResult`-yielding provider; pair with `compaction-tool-result-pruner` to bound tool result sizes |
| Telemetry | missing cost attribution | dsh's `ctx.telemetry` and `runtime-diagnostics` emit model-call events; the broker's `costUSD` computation stays in Lumos |
| Model downgrade | "圆桌对话固定 Opus 4.6" policy violated | ship a `dsh-llm-strict-model` plugin (per §3.2); verify in CI by setting the model to Sonnet and asserting the run rejects |
| Agent Teams | `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` flow changes | if you keep Claude as a backend via `subagent-claude-code`, the experimental flag passes through; if you don't, BRT's cross-agent collaboration needs a dsh-native replacement (no clean drop-in today) |

## 5. Backwards compatibility and feature parity

| Lumos feature | Today | With dsh (full replacement) | With dsh (composition) |
|---|---|---|---|
| `claude-agent-sdk` `query({ prompt, options })` | ✓ | replaced with `DeepSeekHarness.run()` | kept behind `subagent-claude-code` |
| `Agent` tool (subagent invocation) | ✓ | replaced with dsh's `subagent` tool, provider = `claude-code` | unchanged |
| `TodoWrite` (plan tool) | ✓ | replaced with dsh's `todo-write-tool` (2026-06-29) | unchanged (Claude backend) |
| `Bash` + `BashOutput` + `KillBash` | ✓ | replaced with dsh's `tool-bash` + `tool-pwsh` (POSIX/win32 split in `dsh-base` patch) | unchanged |
| `Read` / `Write` / `Edit` / `Glob` / `Grep` / `LS` | ✓ | replaced with dsh's `tool-fs-*` family | unchanged |
| `WebSearch` (NBE-only, via Anthropic server-side search) | ✓ | replaced with dsh's `tool-web-search` (would need a backend plugin) | unchanged |
| `WebFetch` (no current Lumos use) | ✓ | replaced with dsh's `tool-web-fetch` | unchanged |
| `NotebookRead` / `NotebookEdit` (NBE-only) | ✓ | replaced with dsh's `tool-notebook-*` | unchanged |
| `MultiEdit` (NBE-only) | ✓ | replaced with dsh's `tool-edit-batch` | unchanged |
| `Skill` (Claude's skill loader) | ✓ | replaced with dsh's `dsh-skill` + `extensions` | kept (Claude's skill loader still works inside the Claude child) |
| `PreToolUse` / `PostToolUse` hooks | ✓ | replaced with dsh's `ctx.hooks` interception points | dsh's `hooks-claude-code` bridge translates Claude's `hooks.json` |
| `SubagentStop` / `Stop` / `Notification` | ✓ | replaced with dsh's `agent/*` events | dsh's `hooks-claude-code` bridge |
| `SessionStart` / `SessionEnd` / `PreCompact` | ✓ | replaced with dsh's `session/*` events | dsh's `hooks-claude-code` bridge |
| `PreCompact` (manual compaction) | ✓ | replaced with dsh's `command-compact` (`ctx.commands`) | unchanged |
| `can_use_tool` async callback | ✓ | replaced with dsh's `tools/execute` waterfall | dsh's `hooks-claude-code` bridge (passthrough) |
| `experimental_repairText` (BRT structured output) | ✓ | replaced with dsh's `tools/post-execute` accept/reject | replaced with a `dsh-structured-repair` extension that watches the Claude child |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` | ✓ | not available in dsh (no drop-in) | passes through to the Claude child (unchanged) |
| `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` / `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` | ✓ | replaced with dsh's per-plugin depth/concurrency | passes through to the Claude child |
| `permission_mode: 'manual' \| 'acceptEdits' \| ...` | ✓ | replaced with dsh's `ctx.guard` + `ctx.approval` | mapped by `hooks-claude-code` (degrades if not in map) |
| `api_error_status: 429 / 529` fields | ✓ | dsh's `LlmRuntime` returns a typed `finish { kind: 'error', failure }`; the original API status is in the failure object | passes through |
| `cancel_queued` (capability `interrupt_cancel_queued_v1`) | ✓ | replaced with dsh's `HarnessClient.close()` ladder | passes through |
| `crossSessionInbound` / `dialogExpiry` | ✓ | dsh has `interactive-side-sessions` (proposed) | passes through |
| `Settings.source: 'archive'` (plugin install over HTTPS) | ✓ | n/a (dsh has its own plugin loading, not zip over HTTPS) | unchanged |
| `Settings.decode: 'jwt'` / `maskClaims` / `awsPairs` / `sigv4` | ✓ | dsh's `dsh-subprocess` `scrubbedParentEnv` is the closest | passes through |
| `ResumeSessionAt` / `resumeDropsTurn` | ✓ | dsh's `Session` is replayable from the log; no separate "truncating resume" verb needed | unchanged |

## 6. Test impact

Lumos's `src/lib/agent-runtime/` ships ~1,500 LOC of Vitest tests. A migration should expect:

| Test file | LOC | What needs to change |
|---|---|---|
| `agent-runtime.test.ts` | 1,179 | most tests will need re-baselining; the surface they test (process cleanup, env scrubbing, model ID validation) is dsh-shaped but the API is different |
| `broker-egress.test.ts` | 148 | broker tests stay (the broker is in Lumos) |
| `broker-grants.test.ts` | 72 | broker tests stay |
| `broker-grants.integration.test.ts` | 430 | integration tests need a running dsh runtime in the test environment |
| `broker-request.test.ts` | 329 | broker tests stay |
| `broker-search-proof-core.test.ts` | 263 | broker tests stay |
| `remote-files.test.ts` | 36 | re-baseline for dsh's `ctx.fs` API |
| `sandbox-entrypoints.test.ts` | 277 | re-baseline for dsh's `ctx.sandbox` API |

**Rule of thumb:** plan 1 LOC of test work per 3 LOC of production code change, then add another 30% for the integration test scaffolding (E2B + dsh runtime in CI).

## 7. What is the **migration cost**, in concrete terms?

A staged plan if you wanted to do it. Cost estimates are line counts, not calendar.

| Stage | What | LOC estimate (replaced / ported) | Risk | When |
|---|---|---|---|---|
| **0. Stand up dsh in parallel** | Add dsh as a workspace dep, build a `nbe-dsh-canary` E2B template, write a `DshBridgeExecutor` that implements `ClaudeCodeExecutor` and forwards to `DeepSeekHarness.run()` | ~250 LOC of new code; no removals | low | week 1–2 |
| **1. Replace the sandbox-side SDK** | Swap `claude-agent-sdk.ts` for a `dsh-bridge.ts` that uses `@deepseek-ai/dsh-sdk-client`; keep the `runPrompt` contract identical | ~538 LOC removed (claude-agent-sdk.ts) → ~400 LOC replaced; net ~140 LOC removed | medium (model binding decision) | week 3–6 |
| **2. Move process cleanup to dsh's SIGKILL ladder** | Remove `cleanupClaudeCodeProcesses` from `claude-code.ts`; rely on `HarnessClient.close()` | ~80 LOC removed; tests for the heuristic deleted | low | week 7 |
| **3. Mount E2B as `dsh-sandbox-e2b-lumos`** | Move `E2BAgentSandboxProvider` into a dsh plugin | ~275 LOC moved; the `AgentSandboxProvider` interface becomes a dsh `Service Definition` | low | week 8–9 |
| **4. Move security / guard / sanitization to a dsh plugin** | Mount `dsh-guard-lumos` with `assertAgentRuntimeProductionBoundary` + `sanitizeAgentSandboxEnvs` | ~225 LOC moved | low | week 10 |
| **5. Move skillpack syncing to a dsh extension** | Mount `dsh-skill-lumos-bridge` | ~162 LOC moved | low | week 11 |
| **6. Refactor the broker to a `ctx.llm` adapter** | Write `dsh-llm-lumos-broker`; remove the in-process `BrokerRequestError` schema validation; move it into the plugin | ~1,786 LOC moved / replaced | **high** (model-binding product decision) | week 12–15 |
| **7. Refactor BRT's subagent flow** | Replace `Agent` tool calls with dsh subagent composition | ~600 LOC of `roundtable-brt/runner.ts` rewritten (it's 4,612 LOC, the subagent-specific portion is ~600); `subagent_type` → dsh `subagent_id`; tests re-baselined | **high** (product-visible) | week 16–20 |
| **8. Decide on Claude** | The model is now a `cordis.yml` decision. If Lumos wants to keep Claude for NBE/BRT, mount `llm-pi-ai` and route to the broker through it. If Lumos wants to drop Claude, delete `llm-pi-ai` and register `dsh-llm-deepseek` (or a custom DeepSeek adapter) | 0 LOC by itself; flips the rest of the configuration | medium (product) | week 21 |

**Net estimate:** ~3,800 LOC moved/replaced, ~1,500 LOC deleted (mostly process cleanup heuristics, sentinel markers, and module-specific allowlists that become dsh config). Tests re-baselined throughout. The total is comparable to a major Lumos refactor — about **half** of the runtime (was a quarter in the first draft; the broker refactor is bigger than I initially estimated).

**Calendar estimate (rough):** 1–2 engineers, **3–4 months**, with a long overlap window where both runtimes ship. The BRT refactor (stage 7) is the highest-risk step because it changes the user-visible product surface.

## 8. Where the boundary should be drawn

If Lumos adopts dsh, the suggested split is:

| Concern | Where |
|---|---|
| Agent loop, model adapter, session log, hooks, tool registry, subagent backend, in-process plugin tree | **dsh** (`dsh-base` + a `dsh-profile-lumos` profile that mounts all Lumos-specific plugins) |
| E2B sandbox provider, template governance, image builder, snapshot policy | **Lumos** (a `dsh-sandbox-e2b-lumos` plugin maintained in the Lumos repo) |
| Broker, grants, request-schema validation, model allowlists | **Lumos** (a `dsh-llm-lumos-broker` plugin; the broker is a Lumos product, not a dsh primitive) |
| NBE phases, report gate, repair rounds, quality gates | **Lumos** (unchanged; they live above the runtime) |
| BRT roundtable flow, moderator planner, memo orchestrator | **Lumos** (unchanged; but the subagent execution moves from Claude Code's `Agent` tool to dsh's `ctx.subagent`) |
| Skillpack content (the actual `skills/` directory) | **Lumos** (unchanged; `dsh-skill-lumos-bridge` only handles the *delivery* of skills into dsh's tree) |
| DB schema, Better Auth, Next.js app, design system, CI | **Lumos** (unchanged; dsh is a backend concern) |

This split means:

- Lumos keeps its application identity (Next.js, Better Auth, E2B, the broker).
- dsh takes the parts that should be reusable across profiles (loop, model, session, hooks, subagent).
- The E2B template builder and the broker stay in Lumos because they encode product decisions (where data lives, what "no automatic model downgrade" means).
- The boundary is enforced by **patch layering**: Lumos-specific plugins are mounted as the topmost layer over `dsh-base`, and `cordis.patch.yml` (per profile) is what the operator edits.

## 9. What dsh is **not** a good fit for in Lumos

These are not failures of dsh; they are cases where Lumos's domain is its own and dsh should stay out.

1. **The Lumos agent broker.** The broker is an HTTP proxy that mints short-lived grants, validates a strict request schema (8 top-level keys, 32-tool cap, 8 MB request / 1 MB system byte caps), and enforces a per-job / per-user / global request rate limit. This is application-level identity and rate-limiting, not an LLM adapter. dsh has no `ctx.llm` shape that matches. Keep it in Lumos.

2. **The Next.js chat UX, the chat state snapshot model, the workspace memory CRUD, the design system, the Playwright smoke harness.** None of this is agent-runtime concern. Leave it in Lumos.

3. **The roundtable BRT product decisions**: who the partners are, the moderator persona, the memo template, the formal-speech sanitizer, the dedupe rules, the live-patches flow. All of this is BRT product. Leave it in Lumos.

4. **The NBE product decisions**: phase ordering, quality gates, report repair rounds, artifact-store schema. All NBE. Leave it in Lumos.

5. **E2B template governance**: digest-pinned Node, exact npm lockfile, named-template uniqueness, build ID + name suffix, tag-free `assignTags` policy. This is operational. Carry it into the `dsh-sandbox-e2b-lumos` plugin if you adopt; do not expect dsh to ship it.

6. **Vercel AI SDK consumers** (chat UX, diagnosis graph, year-plan workflow). They do not use Claude Agent SDK; do not touch them.

7. **The "圆桌对话固定 Opus 4.6" product rule.** This is a BRT product decision. Encode it as a `dsh-llm-strict-model` plugin if you migrate, but the policy is Lumos's, not dsh's.

## 10. Verdict and adoption paths

**dsh is a good architectural fit for the agent-runtime layer, but a poor fit for the broker and the product layers.**

The mechanical work to swap Claude Agent SDK → dsh in the sandbox is real but bounded (~3,800 LOC moved/replaced, ~1,500 LOC deleted, 3–4 months of overlap with the current runtime). The architectural work — replacing the entire host runtime scaffold with `dsh-base` + a Lumos profile — is the larger investment and the one that actually pays off, because it makes the loop, the model, the persistence, and the tool registry swappable instead of hand-rolled.

### 10.1 Three adoption paths, ordered by risk

| Path | What | Cost | Win |
|---|---|---|---|
| **A. Composition (lowest risk)** | Keep Claude Agent SDK; register it as a dsh subagent backend via `subagent-claude-code`. dsh hosts the orchestration; the broker stays in Lumos; the persistence layer (Phase1, exploration_jobs, chat_state_snapshots) stays in Lumos. | ~600 LOC of plugin wiring, no removals. 4–6 weeks. | dsh becomes the *outer* runtime; Claude becomes an inner backend. The "Meta-runtime" advantage of dsh (composability) is realized without touching the model binding. **This is the recommended starting point.** |
| **B. Sandbox replacement (medium risk)** | Replace the Claude Agent SDK call with `DeepSeekHarness.run()` inside the E2B sandbox. dsh is now the inner runtime; Claude is gone. | ~3,800 LOC moved/replaced, ~1,500 LOC deleted. 3–4 months. | The "loop is swappable" advantage is realized. The model binding is now dsh's. |
| **C. Full base replacement (highest risk)** | Replace the entire `src/lib/agent-runtime/` scaffold with `dsh-base` + a `dsh-profile-lumos` profile. The Next.js process boots a dsh runtime in-process; the E2B sandbox becomes a dsh `ctx.sandbox` provider; the broker becomes a `dsh-llm-lumos-broker` adapter. | ~6,300 LOC moved/replaced, ~2,000 LOC deleted. 5–6 months. | The whole stack is a composition; the operator edits one `cordis.patch.yml` to change policy. The downside is the largest migration surface. |

### 10.2 The recommended path

**Path A first.** Stand up dsh as a thin orchestration layer; keep Claude Agent SDK as a subagent backend via `subagent-claude-code`. This is ~600 LOC of plugin wiring, 4–6 weeks, and gives you the composability story without touching the model binding.

Then, **only if the model binding is a real product decision** (you want to add a DeepSeek route, or a local model route, without rebuilding the broker), do **Path B**. Path B's main cost is the E2B template build (you need a dsh runtime in the sandbox), not the code change.

**Path C is the right answer for a new project, not a migration.** If Lumos were greenfield, it would be the obvious choice. For an existing project with a tested broker, a tested E2B pipeline, and a product surface already on Claude, Path C is a 6-month commitment that should be the explicit subject of a separate decision.

### 10.3 When to use dsh in Lumos

| If you want to… | Recommendation |
|---|---|
| Add a third LLM provider alongside Claude without re-engineering the broker | **Path A only** — register Claude as a dsh subagent backend; the broker is still Claude-bound. To add a provider, write a dsh subagent backend for it. |
| Replace the Claude Code CLI in the sandbox with a smaller, auditable runtime | **Path B, stages 0–2 only** — get the SDK swap working, leave the rest; the biggest gain is killing `claude-code.ts`'s heuristics |
| Get rid of the custom process-cleanup / detached-stream / exit-marker logic | **Path A first, then Path B stage 2** — this is the cleanest win; ~80 LOC of bash + regex goes away |
| Make the agent loop swappable (e.g. swap Claude Code for a smaller local model) | **Path C, all stages** — this is the architectural win, and it's the only one that justifies the calendar cost |
| Make the broker model-agnostic | **Path B, stage 6** — dsh's `ctx.llm` is the right shape; the work is contained |
| Run Lumos workflows without E2B (e.g. on a local dev machine) | **Path A only** — dsh-base + dsh-headless gives you a "no sandbox" mode that's a real profile, not a stub |
| Run Lumos workflows in a multi-tenant hosted dsh | **do not adopt dsh for this** — dsh does not currently ship multi-tenant primitives (per-user rate limits, per-user storage roots, per-user audit); Lumos would build them, but they'd be in `src/lib/`, not `packages/`, and the win is small |
| Expose NBE / BRT as an automation server (Zed, JetBrains, custom editors) | **Path A, opportunistic** — dsh's `dsh-acp` is the right shape, but you'd expose a Lumos-specific ACP server in front of the BRT/NBE runners; the dsh ACP server is reference, not drop-in |

### 10.4 When not to adopt dsh

- Lumos is happy with Claude for the foreseeable future and just wants a smaller sandbox runtime. Adopt `dsh-base` + a headless profile (Path A); skip the broker refactor.
- Lumos needs a multi-tenant hosted dsh. The primitives are not there; building them in Lumos and skipping dsh is faster.
- Lumos needs to ship in the next 4 weeks. dsh is `0.1.0-rc.5`; the calendar cost of a parallel canary is not negligible.
- The BRT roundtable depends on `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` and the `experimental_repairText` callback. dsh has no drop-in for either. Path A keeps them via `subagent-claude-code`; Path B/C loses them unless you build replacements.

## 11. Source files cited

### dsh (this repo)
- `packages/sdk/client/README.md` — TS SDK surface
- `packages/sdk/server/README.md` — stdio JSON-RPC server
- `packages/sdk/protocol/README.md` — wire types
- `python/sdk/README.md` — Python SDK surface
- `docs/architecture.md` — plugin tree, sessions, events
- `docs/cordis-primer.md` — Cordis framework
- `packages/llm/llm/README.md` — LLM runtime
- `packages/core/agent/README.md` — agent registry
- `packages/core/tools/README.md` — tool execution pipeline
- `packages/core/session/README.md` — event-sourced session log
- `packages/subagent/README.md` — subagent family
- `packages/subagent/subagent-claude-code/README.md` — Claude Code as dsh subagent
- `packages/subagent/subagent-codex/README.md` — Codex as dsh subagent
- `packages/subagent/subagent-dsh-sdk/README.md` — dsh as dsh subagent (recursive)
- `packages/subagent/subagent-acp/README.md` — ACP as dsh subagent
- `packages/hooks/README.md` — Claude/Codex hook bridges
- `packages/acp/acp/README.md` — dsh's own ACP server
- `packages/bundle/base/README.md` — dsh-base bundle
- `packages/bundle/headless/README.md` — dsh-headless bundle
- `packages/bundle/web-app/README.md` — dsh-web-app bundle
- `packages/sandbox/README.md` — sandbox capability family
- `packages/guard/README.md` — loop-hygiene guards
- `packages/compaction/README.md` — compaction capability family
- `.agents/notes/implemented/feature/` — 510 implemented feature notes
- `.agents/notes/proposed/feature/` — proposed features
- `.agents/notes/proposed/architecture/` — proposed architecture

### Lumos (`/Users/harvey/Desktop/Harvey-files/08Myprojects/lumos/Lumos`)
- `src/lib/agent-runtime/types.ts` — capability interfaces (112 LOC)
- `src/lib/agent-runtime/claude-agent-sdk.ts` — Claude Agent SDK executor (538 LOC)
- `src/lib/agent-runtime/claude-code.ts` — Claude Code CLI wrapping (344 LOC)
- `src/lib/agent-runtime/e2b-sandbox.ts` — E2B sandbox lifecycle (275 LOC)
- `src/lib/agent-runtime/skillpack.ts` — skillpack syncing (162 LOC)
- `src/lib/agent-runtime/security.ts` — env scrubbing + production boundary (125 LOC)
- `src/lib/agent-runtime/config.ts` — runtime config (214 LOC)
- `src/lib/agent-runtime/remote-files.ts` — host→sandbox file sync (101 LOC)
- `src/lib/agent-runtime/allowed-tools.ts` — module-specific tool allowlists (15 LOC, NBE 12 + BRT 5)
- `src/lib/agent-runtime/timeouts.ts` — clamp helpers (18 LOC)
- `src/lib/agent-runtime/broker-egress.ts` — broker egress wiring (116 LOC)
- `src/lib/agent-runtime/broker-grants.ts` — short-lived grant minting (429 LOC)
- `src/lib/agent-runtime/broker-config.ts` — broker config validation (134 LOC)
- `src/lib/agent-runtime/broker-request.ts` — request schema validation (496 LOC)
- `src/lib/agent-runtime/broker-search-proof-core.ts` — proof artifacts (357 LOC)
- `src/lib/agent-runtime/broker-search-proofs.ts` — proof persistence (159 LOC)
- `src/lib/agent-runtime/agent-runtime.test.ts` — main test file (1,179 LOC)
- `src/lib/roundtable-brt/runner.ts` — BRT runner (4,612 LOC, the largest consumer)
- `src/lib/roundtable-brt/constants.ts` — BRT paths + `CLAUDE_AGENT_SDK_VERSION = "0.3.220"`
- `src/lib/roundtable-brt/moderator-planner.ts` — uses `experimental_repairText`
- `src/lib/roundtable-brt/crisis.ts` — uses `experimental_repairText`
- `src/lib/exploration/runtime/agent-runner.ts` — NBE runner (280 LOC, above the runtime)
- `src/lib/exploration/runtime/runtime-manager.ts` — NBE orchestration
- `src/lib/exploration/runtime/phase1-structured.ts` — has `runtimeBackend` + `sandboxBackend` payload fields
- `docs/current/Agent-Runtime-通用执行底座.md` — Lumos's own runtime contract
- `docs/current/E2B-Agent-模板测试服发布.md` — template build/promote flow
- `docs/code-intelligence/domains/agent-runtime.md` — codemap of the runtime
- `scripts/e2b/install-claude-agent-sdk.sh` — SDK install in the template
- `scripts/e2b/build-nbe-template.ts` and `scripts/e2b/build-brt-template.ts` — template builders
