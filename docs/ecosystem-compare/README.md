# Ecosystem comparison

This directory holds side-by-side comparisons of dsh against other agent
runtime SDKs in the broader agent ecosystem, plus concrete adoption
evaluations for projects that are considering dsh as a base.

## Documents

| File | 中文 |
|---|---|
| [dsh-vs-claude-agent-sdk-vs-codex-sdk.md](./dsh-vs-claude-agent-sdk-vs-codex-sdk.md) | [中文](./dsh-vs-claude-agent-sdk-vs-codex-sdk.zh.md) |
| [lumos-adoption-evaluation.md](./lumos-adoption-evaluation.md) | [中文](./lumos-adoption-evaluation.zh.md) |

### dsh vs Claude Agent SDK vs Codex SDK

Exhaustive feature matrix: dsh SDK vs `@anthropic-ai/claude-agent-sdk` vs
`@openai/codex-sdk`. Covers transport, API surface, configuration, subagents,
sessions, tools, hooks, sandboxing, telemetry, and every flagged beta /
experimental surface. The most important reframing in §0: **dsh is in a
parent category — a meta-runtime that *contains* Claude Code and Codex as
subagent backends, not as competitors.**

### Lumos × dsh — adoption evaluation

Detailed adoption evaluation of dsh for the Lumos project (Next.js 16 + Bun
+ Claude Agent SDK + E2B). Maps Lumos's `src/lib/agent-runtime/` against
dsh's plugin tree, lists an 8-stage migration plan, identifies concerns
that should stay in Lumos (agent broker, BRT/NBE product logic, E2B
template governance), and proposes three adoption paths ordered by risk
(composition / sandbox replacement / full base replacement). The
recommended starting point is the composition path: keep Claude Agent
SDK, register it as a dsh subagent backend via `subagent-claude-code`.

## Source discipline

- dsh facts come from this repo (`packages/`, `docs/architecture.md`,
  `.agents/notes/`).
- Claude Agent SDK facts come from the npm package
  (`@anthropic-ai/claude-agent-sdk@0.3.231`), the GitHub changelog
  (`anthropics/claude-agent-sdk-typescript/CHANGELOG.md`), and the Python
  README (`anthropics/claude-agent-sdk-python/README.md`).
- Codex SDK facts come from the npm package
  (`@openai/codex-sdk@0.147.0`) and the GitHub README
  (`openai/codex/sdk/typescript/README.md`).
- Lumos facts come from the Lumos repository at
  `/Users/harvey/Desktop/Harvey-files/08Myprojects/lumos/Lumos` and its
  internal docs under `Lumos/docs/`.

Where a fact is not directly verified against a source, the document says
so explicitly. Beta / experimental surface is annotated per item, not
collapsed into a single "this is beta" header.

## Bilingual convention

Each document in this directory ships as a `.md` (English, primary) + `.zh.md`
(Chinese mirror) + `.i18n.yaml` (pairing record), following the dsh repo
convention. The two sides carry equal authority. After editing either
side, run `pnpm run verify-translation-pairing --write` from the repo
root to refresh the pairing record.
