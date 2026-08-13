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
> `src/lib/agent-runtime/`. The consumer is `src/lib/exploration/runtime/agent-runner.ts`
> (NBE) and `src/lib/roundtable-brt/runner.ts` (BRT). dsh's surface lives in
> `packages/sdk/{client,protocol,server}` and `packages/bundle/{base,web-app,headless}`.

## 1. Lumos's current runtime shape (what we are evaluating against)

### 1.1 The runtime is two layers

| Layer | Today | dsh analogue |
|---|---|---|
| **Host runtime** (the Next.js process) | `E2BAgentSandboxProvider` (`src/lib/agent-runtime/e2b-sandbox.ts`): creates / connects / snapshots / disposes E2B sandboxes, applies `denyOrKillAgentSandboxNetwork`, recovers from capacity errors, exposes `AgentSandboxProvider` interface | `dsh-base` + a new `dsh-sandbox-e2b` plugin could mount E2B behind `ctx.sandbox` |
| **Sandbox runtime** (inside the E2B VM) | `ClaudeAgentSdkExecutor` (`src/lib/agent-runtime/claude-agent-sdk.ts`): writes a Node ESM runner that imports `@anthropic-ai/claude-agent-sdk@0.3.220` and calls `query({ prompt, options })` over stdio; the SDK then spawns the bundled `@anthropic-ai/claude-code@2.1.220` CLI which is what actually does the work | dsh's own `dsh-jsonrpc-agent` is the same kind of thing — a runtime you launch over stdio. To replace Claude Agent SDK, the sandbox runner would import `@deepseek-ai/dsh-sdk-client` and call `DeepSeekHarness.run(prompt)` instead. |

### 1.2 What Lumos does on top of the SDK

The dsh-shaped concerns live in *Lumos's host runtime*, not in the SDK call itself:

| Concern | File | Notes |
|---|---|---|
| Sandbox lifecycle / checkpoint / snapshot / pause / dispose | `e2b-sandbox.ts`, `remote-files.ts`, `timeouts.ts` | stateful, owns a long-lived sandbox for hours at a time |
| Skillpack sync (workspace `skills/` → sandbox `.lumos-agent-sdk/skills/` and `.claude/skills/`) | `skillpack.ts` | fingerprinted cache; declarative `resolveSkillRoot()` |
| Secret / env scrubbing | `security.ts` (`sanitizeAgentSandboxEnvs`, `assertAgentRuntimeProductionBoundary`) | hard-coded `SAFE_SANDBOX_ENV_NAMES` allowlist, `AgentRuntimeSecurityError` short-circuits production |
| Network egress allowlist + broker | `broker-egress.ts`, `broker-grants.ts`, `broker-config.ts`, `broker-request.ts` | **Crucial**: production must go through a Lumos-owned `/api/internal/agent-broker/anthropic` proxy that mints short-lived grants; the sandbox network is denied by default (`denyOrKillAgentSandboxNetwork`) |
| Process cleanup / detached stream recovery | `claude-code.ts` (`cleanupClaudeCodeProcesses`), `claude-agent-sdk.ts` (`looksDetachedCommandStreamFailure`) | heuristic process scanning inside the sandbox |
| Tool allowlists | `allowed-tools.ts` (`NBE_AGENT_ALLOWED_TOOLS`, `BRT_ALLOWED_TOOLS`) | module-specific allowlists (5-tool for NBE, 5-tool for BRT) |
| Template governance | `scripts/e2b/build-nbe-template.ts`, `scripts/e2b/build-brt-template.ts`, `scripts/e2b/install-claude-agent-sdk.sh` | builds a minimal image: Node `24.18.0` (digest-pinned), Claude Code CLI, Claude Agent SDK package, exact npm lockfile, no node version manager |
| Phase / checkpoint / quality-gate / artifact / report gate | `src/lib/exploration/runtime/agent-runner.ts` and surrounding phases | **NBE-specific**; out of scope for the agent runtime itself |
| Multi-agent discussion (moderator + 2–3 partners) | `src/lib/roundtable-brt/runner.ts`, `moderator-planner.ts`, `memo-orchestrator.ts`, etc. | **BRT-specific**; out of scope for the agent runtime itself |

### 1.3 The two SDK consumers

#### (a) NBE — Notebook Exploration (`src/lib/exploration/`)

- Long-running phase pipeline (intake, research, analysis, report). Each phase is a Claude Code invocation.
- Hard contract: NBE adapter "固定实例化 `ClaudeAgentSdkExecutor`; 不存在可切换到 CLI 的运行时配置" — there is **no runtime switch today**. The SDK is the single execution path.
- Skill sync: "NBE 真实执行固定读取仓库内 `skills/` 并同步到 E2B" — repo-local skills only.
- Final gate: `NBE_CLAUDE_EXECUTION_MODE=lumos_controller | orchestrator_parity` and `NBE_REPORT_READY_GATE=release` produce hard/publish/semantic gates; `PASS_WITH_WARNINGS` is an internal candidate-only state.
- Report repair rounds: `NBE_REPORT_REPAIR_MAX_ROUNDS=3` — re-invokes the agent inside the same sandbox to fix/extend the report.

#### (b) BRT — Business Roundtable (`src/lib/roundtable-brt/`)

- 2–3 "合伙人" sub-agents + a moderator, each is a `subagent_type`. They run in the same sandbox via the `Agent` tool's subagent mechanism, with five allowed tools (`Agent`, `Glob`, `Grep`, `LS`, `TodoWrite`).
- Subagent executor is injected via `createRoundtableClaudeDiscussionRuntime({ subagentExecutor })`. The DB-observable process event is `claude-agent-sdk-subagents`, audited as `done | queued | error`.
- Model is "圆桌对话固定 Opus 4.6" with **no automatic model downgrade** — a 503 from the default token group surfaces as `subagentRuntimeSucceeded=false` and a hard failure, not a silent retry on a smaller model. This is a **deliberate** product choice that dsh does not currently mirror.

## 2. dsh's surface, mapped to Lumos

| dsh concept | dsh location | Maps to Lumos today | Migration impact |
|---|---|---|---|
| `dsh-base` profile | `packages/bundle/base` | n/a — Lumos doesn't ship a profile at all; it ships a hand-rolled host runtime | **large** — would replace the entire host runtime scaffold |
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

| File | Lines | dsh-shape? |
|---|---|---|
| `e2b-sandbox.ts` | 275 | **Yes** — a `dsh-sandbox-e2b` plugin |
| `remote-files.ts` | 101 | **Yes** — a `dsh-fs-remote` provider |
| `timeouts.ts` | 18 | **Yes** — a config validation plugin |
| `security.ts` | 125 | **Yes** — a `dsh-guard-lumos` plugin (`assertAgentRuntimeProductionBoundary`, `sanitizeAgentSandboxEnvs`) |
| `skillpack.ts` | 162 | **Yes** — a `dsh-skill-lumos-bridge` |
| `claude-code.ts` | 344 | **Partial** — the `cleanupClaudeCodeProcesses` heuristic is Claude-CLI-specific; the `CLAUDE_CODE_EXIT_CODE_MARKER` protocol is Claude-CLI-specific; replace with dsh's own exit / stream detachment machinery |
| `claude-agent-sdk.ts` | 538 | **No** — replaced by `@deepseek-ai/dsh-sdk-client` calls |
| `broker-egress.ts` / `broker-grants.ts` / `broker-config.ts` / `broker-request.ts` | 1,190 | **Partial** — Lumos's broker is HTTP-API-shaped, not LLM-adapter-shaped. dsh has no `ctx.llm` adapter that can talk to a proxy expecting `/api/internal/agent-broker/anthropic/{...}` payload validation. You'd either build a custom `dsh-llm-lumos-broker` adapter, or refactor the broker to be a generic proxy that mounts behind `ctx.llm` |
| `config.ts` | 214 | **Partial** — dsh's typed config can host `LUMOS_AGENT_RUNTIME_*` keys; the hard ceilings (`hardMax`) are dsh-shaped, the env-name allowlist (`SAFE_SANDBOX_ENV_NAMES`) belongs in a guard plugin |
| `types.ts` | 112 | **Yes** — dsh's capability seam types are the right home; the `AgentSandboxProvider` / `ClaudeCodeExecutor` interfaces are dsh-shaped (a "service definition" + a "service provider") |
| `*.test.ts` | 1,500+ | **No** — Lumos's test suite is Vitest, dsh's is a different harness; this is mechanical port work |

The hard things dsh does **not** give Lumos:

- **Postgres-backed broker grants** (`broker-grants.ts`). dsh has no equivalent; you'd build a `dsh-grant-lumos-postgres` plugin.
- **The Lumos model allowlist pattern**: 5-tool for NBE, 5-tool for BRT, with module-specific `ConsumedAgentBrokerGrant["module"]` types. dsh's `ctx.tools` is shape-equivalent but does not ship module-keyed allowlists.
- **The "圆桌对话固定 Opus 4.6" policy**. dsh has `maxTokens` per agent but no product-level "no model downgrade" guarantee; the adapter just routes to whatever the provider returns. If you want the same guarantee, you have to add a `dsh-llm-strict-model` plugin that rejects any `model !==` the configured one.
- **The Lumos CI / template-build pipeline.** `scripts/e2b/build-nbe-template.ts` is a custom Node 24.18.0 digest-pinned image builder. dsh does not currently ship an E2B template builder; you'd carry this over as-is.

### 3.3 What does dsh give Lumos that today costs custom code?

This is the most useful question. The list is not empty.

| Today, custom-built in Lumos | What dsh would replace it with |
|---|---|
| `claude-code.ts` process cleanup heuristic (`cleanupClaudeCodeProcesses` — Python `ps`/`kill` script) | dsh SDK's `HarnessClient.close()` private SIGKILL ladder — `disposeEofGraceMs` / `disposeGraceMs` / `shutdownTimeoutMs` are knobs you can tune, and the ladder is idempotent |
| `looksDetachedCommandStreamFailure` regex | dsh SDK's `TransportClosedError` (typed) |
| `claude-agent-sdk.ts`'s `runPrompt` contract (a single `prompt → result` facade over the SDK) | dsh SDK's `DeepSeekHarness.run()` directly — but you still want a Lumos-side wrapper for the E2B lifecycle, so the savings is the JSON-RPC layer, not the wrapper |
| `assertAgentRuntimeProductionBoundary` and `AgentRuntimeSecurityError` | dsh's `ctx.guard` capability seam — would need a `dsh-guard-lumos` plugin to wire the env-name allowlist, but the seam itself is provided |
| `sanitizeAgentSandboxEnvs` (env-name allowlist + credential regex) | dsh's `dsh-subprocess` `scrubbedParentEnv` plus a guard plugin |
| `BrokerRequestError` schema (8-key top-level allowlist, 32-tool cap, byte caps on request and system) | dsh does **not** ship a request-schema guard. You'd carry this as a `dsh-llm-lumos-broker` plugin |
| `DefaultSkillpackSyncer` (fingerprint + sync into sandbox) | dsh's `dsh-skill` + `extensions` are the canonical home; you'd write a `dsh-skill-lumos-bridge` that maps `skills/` into the `dsh.extensions` tree |
| `E2BAgentSandboxProvider.acquire()` (connect → snapshot-restore → template; capacity recovery; `denyOrKillAgentSandboxNetwork` on every connect) | dsh's `ctx.sandbox` would host this, but the *acquire algorithm* is Lumos-specific (the connect-then-snapshot-then-template ordering) and would carry over as a `dsh-sandbox-e2b-lumos` plugin |
| `claude-code.ts` `CLAUDE_CODE_EXIT_CODE_MARKER` (a sentinel line printed by the runner so the host can parse the Claude Code exit code) | dsh SDK returns a typed `RunResult` with `finishReason` (`completed` / `max-tokens` / `error`); the marker becomes unnecessary |
| `BashToolOutput.timedOutAfterMs` handling (Claude-CLI-specific timeout reporting) | dsh's `SessionEvent` includes the same fact; the host can subscribe and react |

### 3.4 What is the **migration cost**, in concrete terms?

A staged plan if you wanted to do it. Cost estimates are line counts, not calendar.

| Stage | What | LOC estimate (replaced / ported) |
|---|---|---|
| **0. Stand up dsh in parallel** | Add dsh as a workspace dep, build a `nbe-dsh-canary` E2B template, write a `DshBridgeExecutor` that implements `ClaudeCodeExecutor` and forwards to `DeepSeekHarness.run()` | ~250 LOC of new code; no removals |
| **1. Replace the sandbox-side SDK** | Swap `claude-agent-sdk.ts` for a `dsh-bridge.ts` that uses `@deepseek-ai/dsh-sdk-client`; keep the `runPrompt` contract identical | ~538 LOC removed (claude-agent-sdk.ts) → ~400 LOC replaced; net ~140 LOC removed |
| **2. Move process cleanup to dsh's SIGKILL ladder** | Remove `cleanupClaudeCodeProcesses` from `claude-code.ts`; rely on `HarnessClient.close()` | ~80 LOC removed; tests for the heuristic deleted |
| **3. Mount E2B as `dsh-sandbox-e2b-lumos`** | Move `E2BAgentSandboxProvider` into a dsh plugin | ~275 LOC moved; the `AgentSandboxProvider` interface becomes a dsh `Service Definition` |
| **4. Move security / guard / sanitization to a dsh plugin** | Mount `dsh-guard-lumos` with `assertAgentRuntimeProductionBoundary` + `sanitizeAgentSandboxEnvs` | ~225 LOC moved |
| **5. Move skillpack syncing to a dsh extension** | Mount `dsh-skill-lumos-bridge` | ~162 LOC moved |
| **6. Refactor the broker to a `ctx.llm` adapter** | Write `dsh-llm-lumos-broker`; remove the in-process `BrokerRequestError` schema validation; move it into the plugin | ~600 LOC moved / replaced |
| **7. Refactor BRT's subagent flow** | Replace `Agent` tool calls with dsh subagent composition | ~200 LOC of `roundtable-brt/runner.ts` rewritten; `subagent_type` → dsh `subagent_id`; tests re-baselined |
| **8. Decide on Claude** | The model is now a `cordis.yml` decision. If Lumos wants to keep Claude for NBE/BRT, mount `llm-pi-ai` and route to the broker through it. If Lumos wants to drop Claude, delete `llm-pi-ai` and register `dsh-llm-deepseek` (or a custom DeepSeek adapter) | 0 LOC by itself; flips the rest of the configuration |

**Net estimate:** ~1,800 LOC moved/replaced, ~1,200 LOC deleted (mostly process cleanup heuristics, sentinel markers, and module-specific allowlists that become dsh config). Tests re-baselined throughout. The total is comparable to a major Lumos refactor — about a quarter of the runtime.

**Calendar estimate (rough):** 1–2 engineers, 2–3 months, with a long overlap window where both runtimes ship. The BRT refactor (stage 7) is the highest-risk step because it changes the user-visible product surface.

## 4. Decision matrix: when to use dsh in Lumos

| If you want to… | Recommendation |
|---|---|
| Add a third LLM provider alongside Claude without re-engineering the broker | **do not adopt dsh for this** — write a `ctx.llm` analogue in Lumos directly; the broker is HTTP-shaped, dsh is JSON-RPC-shaped, and the mismatch is larger than the win |
| Replace the Claude Code CLI in the sandbox with a smaller, auditable runtime | **adopt dsh at stages 0–2 only** — get the SDK swap working, leave the rest; the biggest gain is killing `claude-code.ts`'s heuristics |
| Get rid of the custom process-cleanup / detached-stream / exit-marker logic | **adopt dsh at stage 2** — this is the cleanest win; ~80 LOC of bash + regex goes away |
| Make the agent loop swappable (e.g. swap Claude Code for a smaller local model) | **adopt dsh at all stages** — this is the architectural win, and it's the only one that justifies the calendar cost |
| Make the broker model-agnostic | **adopt dsh at stage 6** — dsh's `ctx.llm` is the right shape; the work is contained |
| Run Lumos workflows without E2B (e.g. on a local dev machine) | **adopt dsh at stages 0–2** — `dsh-base` + `dsh-headless` gives you a "no sandbox" mode that's a real profile, not a stub |
| Run Lumos workflows in a multi-tenant hosted dsh | **do not adopt dsh for this** — dsh does not currently ship multi-tenant primitives (per-user rate limits, per-user storage roots, per-user audit); Lumos would build them, but they'd be in `src/lib/`, not `packages/`, and the win is small |

## 5. Where the boundary should be drawn

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

## 6. What dsh is **not** a good fit for in Lumos

These are not failures of dsh; they are cases where Lumos's domain is its own and dsh should stay out.

1. **The Lumos agent broker.** The broker is an HTTP proxy that mints short-lived grants, validates a strict request schema (8 top-level keys, 32-tool cap, 8 MB request / 1 MB system byte caps), and enforces a per-job / per-user / global request rate limit. This is application-level identity and rate-limiting, not an LLM adapter. dsh has no `ctx.llm` shape that matches. Keep it in Lumos.

2. **The Next.js chat UX, the chat state snapshot model, the workspace memory CRUD, the design system, the Playwright smoke harness.** None of this is agent-runtime concern. Leave it in Lumos.

3. **The roundtable BRT product decisions**: who the partners are, the moderator persona, the memo template, the formal-speech sanitizer, the dedupe rules, the live-patches flow. All of this is BRT product. Leave it in Lumos.

4. **The NBE product decisions**: phase ordering, quality gates, report repair rounds, artifact-store schema. All NBE. Leave it in Lumos.

5. **E2B template governance**: digest-pinned Node, exact npm lockfile, named-template uniqueness, build ID + name suffix, tag-free tag-free `assignTags` policy. This is operational. Carry it into the `dsh-sandbox-e2b-lumos` plugin if you adopt; do not expect dsh to ship it.

## 7. Verdict

**dsh is a good architectural fit for the agent-runtime layer, but a poor fit for the broker and the product layers.**

The mechanical work to swap Claude Agent SDK → dsh in the sandbox is real but bounded (~1,800 LOC moved/replaced, ~1,200 LOC deleted, 2–3 months of overlap with the current runtime). The architectural work — replacing the entire host runtime scaffold with `dsh-base` + a Lumos profile — is the larger investment and the one that actually pays off, because it makes the loop, the model, the persistence, and the tool registry swappable instead of hand-rolled.

**Recommended path if adopting:**

1. **Stages 0–2 first** (parallel canary, SDK swap, process-cleanup cleanup). Lowest risk, highest short-term win. ~3–6 weeks.
2. **Stages 3–5 second** (E2B / guard / skill plugins into dsh). Medium risk, large LOC moved. ~4–6 weeks.
3. **Stage 6 third** (broker as a `ctx.llm` adapter). High risk; this is where the model-binding product decision gets made explicit. ~3–4 weeks.
4. **Stage 7 last** (BRT subagent refactor). Highest product risk; gate on whether Claude is still in the model mix.

**Not recommended now if:**

- Lumos is happy with Claude for the foreseeable future and just wants a smaller sandbox runtime. Adopt `dsh-base` + a headless profile; skip the broker refactor.
- Lumos needs a multi-tenant hosted dsh. The primitives are not there; building them in Lumos and skipping dsh is faster.
- Lumos needs to ship in the next 4 weeks. dsh is `0.1.0-rc.5`; the calendar cost of a parallel canary is not negligible.

## 8. Source files cited

### dsh (this repo)
- `packages/sdk/client/README.md` — TS SDK surface
- `packages/sdk/server/README.md` — stdio JSON-RPC server
- `packages/sdk/protocol/README.md` — wire types
- `python/sdk/README.md` — Python SDK surface
- `docs/architecture.md` — plugin tree, sessions, events
- `.agents/notes/implemented/feature/` — 510 implemented feature notes (e.g. `2026-06-14-acp-multi-session`, `2026-06-18-compaction-capability-seam`, `2026-06-21-subagent-capability-seam`, `2026-06-29-todo-write-tool`, `2026-06-30-hook-bridges`)
- `.agents/notes/proposed/feature/` — proposed features (`2026-07-06-recallable-compaction`, `2026-07-08-interactive-side-sessions`, `2026-08-04-task-surface`, `2026-06-30-pre-tool-input-rewrite`)
- `.agents/notes/proposed/architecture/` — proposed architecture (`2026-07-19-required-cancellation-through-tool-capability-seams`, `2026-07-24-domain-kv-storage-and-workspace`, `2026-07-25-client-settings-locale-theme`, `2026-07-27-session-projection-and-command-log`, `2026-07-28-storage-root-and-derived-medium-recovery`, `2026-07-29-durable-last-activity-index`, `2026-06-16-typed-event-schemas`)

### Lumos (`/Users/harvey/Desktop/Harvey-files/08Myprojects/lumos/Lumos`)
- `src/lib/agent-runtime/types.ts` — capability interfaces
- `src/lib/agent-runtime/claude-agent-sdk.ts` — Claude Agent SDK executor (538 LOC)
- `src/lib/agent-runtime/claude-code.ts` — Claude Code CLI wrapping (344 LOC)
- `src/lib/agent-runtime/e2b-sandbox.ts` — E2B sandbox lifecycle (275 LOC)
- `src/lib/agent-runtime/skillpack.ts` — skillpack syncing (162 LOC)
- `src/lib/agent-runtime/security.ts` — env scrubbing + production boundary (125 LOC)
- `src/lib/agent-runtime/config.ts` — runtime config (214 LOC)
- `src/lib/agent-runtime/remote-files.ts` — host→sandbox file sync (101 LOC)
- `src/lib/agent-runtime/allowed-tools.ts` — module-specific tool allowlists (15 LOC)
- `src/lib/agent-runtime/timeouts.ts` — clamp helpers (18 LOC)
- `src/lib/agent-runtime/broker-egress.ts` — broker egress wiring (116 LOC)
- `src/lib/agent-runtime/broker-grants.ts` — short-lived grant minting (429 LOC)
- `src/lib/agent-runtime/broker-config.ts` — broker config validation (134 LOC)
- `src/lib/agent-runtime/broker-request.ts` — request schema validation (496 LOC)
- `src/lib/agent-runtime/broker-search-proof-core.ts` — proof artifacts (357 LOC)
- `src/lib/agent-runtime/broker-search-proofs.ts` — proof persistence (159 LOC)
- `src/lib/roundtable-brt/runner.ts` — BRT runner (above the runtime)
- `src/lib/roundtable-brt/constants.ts` — BRT paths + `CLAUDE_AGENT_SDK_VERSION`
- `src/lib/exploration/runtime/agent-runner.ts` — NBE runner (above the runtime)
- `docs/current/Agent-Runtime-通用执行底座.md` — Lumos's own runtime contract
- `docs/current/E2B-Agent-模板测试服发布.md` — template build/promote flow
- `docs/code-intelligence/domains/agent-runtime.md` — codemap of the runtime
- `scripts/e2b/install-claude-agent-sdk.sh` — SDK install in the template
- `scripts/e2b/build-nbe-template.ts` and `scripts/e2b/build-brt-template.ts` — template builders
