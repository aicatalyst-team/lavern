> [!WARNING]
> **This repository is archived.**
>
> Archived on 2026-09-08 by the AI Catalyst Platform Team.
> It is read-only and no longer maintained.

---# Lavern

**A multi-agent legal system. Apache 2.0.**

A team of 67 agent prompts (specialists plus orchestrators) coordinating through a debate protocol. The system reads documents, posts findings with cited evidence, runs three layers of verification, and pauses at a human gate before critical decisions land. Runs against Anthropic, Mistral (EU), or fully local via Ollama.

Built by a law firm founder over six months because the prevailing "AI as a junior associate" framing felt like the wrong analogy to start from. Whether this is the right one is what we're publishing to find out.

[**Watch the demo →**](https://lavern.ai/demo/) · [**lavern.ai**](https://lavern.ai) · [**Architecture deep-dive →**](https://lavern.ai/architecture/) · [**Quick Start →**](QUICKSTART.md)

> **This is not a product.** Lavern is best understood as a collection of ideas in one repo. A polished demo of how we could build in the future.
>
> You can criticise it for not working as a "product". That misses the mark. It is not meant to be a product. It is a source of inspiration.
>
> It is at least ten things, several of which are, on their own, products somebody could build a company around. They are sitting in the repo. Take whichever ones you want.

> **What is and isn't stress-tested.** The architecture is real and the code is open. The pipeline runs, the agents debate, the verification loops fire, the precedent board persists, the tests pass. What hasn't been independently validated is the *quality bar*: whether all this machinery produces materially better outputs than a well-prompted single LLM on a representative sample of real legal work. We have internal evaluation; we don't have a public benchmark. Treat the engineering as the contribution and the legal-quality claims as a hypothesis.

## Install

macOS / Linux:

```bash
curl -fsSL lavern.ai/install.sh | sh
```

Windows (PowerShell):

```powershell
irm lavern.ai/install.ps1 | iex
```

Either script clones the repo, installs backend and frontend dependencies, and gives you a `lavern` command. The 60-second walkthrough is in [QUICKSTART.md](QUICKSTART.md).

By hand:

```bash
git clone https://github.com/AnttiHero/lavern.git
cd lavern
npm install
(cd viz && npm install)
```

Then:

```bash
npm run serve:dev          # API server on :3000 (LOCAL MODE, no API key needed)
cd viz && npm run dev      # Dashboard on :5173, hot reload
open http://localhost:5173
```

Demo mode runs the dashboard, the Clawern view, and the cinematic guided tour without an API key. To run real engagements, add `ANTHROPIC_API_KEY` (or `MISTRAL_API_KEY` for the EU provider) to `.env`. It's auto-created from `.env.example` on first run.

## What's in the box

- **67 agent prompts**: 59 specialists, 7 workflow-specific orchestrators (`orchestrator-{adversarial,counsel,full-bench,review,roundtable,tabulate,verification}.ts`), and 1 generic base orchestrator (`orchestrator.ts`)
- **21 MCP tools** for debate, scoring, verification, grounding, memory, knowledge base, quality checks
- **9 workflows** from a single-specialist counsel call to a full adversarial review
- **5 seeded legal datasets**: CUAD, MAUD, ACORD, UNFAIR-ToS, LEDGAR (each under its own license; see NOTICE). ContractNLI was previously included; it was removed because its CC BY-NC-SA 4.0 license is incompatible with Apache 2.0. Fetch it yourself if you need it.
- **3 inference providers**: Anthropic Claude (US), Mistral AI (EU), or local via Ollama
- **1,677 tests** across 105 files. Clean `tsc --noEmit` on backend and frontend.

## Three modes

**Interactive.** Open the dashboard, brief the system in plain language, watch the agents work in a live activity feed, approve critical calls at human gates. Each engagement produces the deliverable plus an audit bundle: structured findings, debate resolutions, verification results, and a cost log.

**Clawern (autonomous).** Point it at a folder. Clawern processes new documents on a 30-minute heartbeat, accumulates a precedent board across past reviews, and pushes findings to Telegram, email, or macOS notifications. Includes weekly digest, multi-client isolation, audit trail, cost forecasting, hybrid local-plus-frontier processing.

**EU mode.** Set `LAVERN_PROVIDER=mistral` and the orchestrator, agents, debate, verification, briefing analyser, partner consult, agent-builder, and Clawern processing all route through Mistral AI in Paris instead of Anthropic. One feature, the Lavern Challenge (blind document comparison) at `src/api/routes/challenge.ts`, still instantiates the Anthropic client directly and will hit Anthropic even when Mistral is selected. If you need a strict EU boundary, avoid that feature for now. Use `claw start --ethical` to enforce Mistral-only with conservative risk posture across the rest of the pipeline.

## What "67 agents" actually means

Each agent is a specialised system prompt with its own role, MCP tool permissions, and slot in the debate protocol. All 67 run on the same underlying frontier LLM (Claude or Mistral, your choice), so yes, at the bottom of the stack it's an LLM.

The work isn't the prompts. The work is the four things wrapped around them:

- **The debate protocol.** Agents must cite specific text from the parsed document. Findings without citations don't enter the board. Agents can challenge each other; the challenger has to cite text too.
- **Three-layer verification.** Evaluator gate (drops weak findings) → adversarial debate (red team / blue team) → 10-pass verification pipeline (`src/workflows/templates/verification.ts`). The 10 passes are: `context`, `ux`, `clarity`, `structure`, `accuracy`, `completeness`, `risk`, `formatting`, `legal_design`, `delivery` (see `src/types/verification.ts` for the full pass definitions). Each layer fails closed independently. Separate from this pipeline, the mechanical grounding verifier in `src/mcp/tools/grounding-verifier.ts` cross-checks every cited quote against the parsed document via string matching.
- **Human gates.** Critical findings don't auto-deliver. The orchestrator surfaces the call and waits for a human to approve or override.
- **Precedent Board.** Persistent memory across engagements. Findings that recur across documents get reinforced; stale ones decay. A new pattern enters as "tentative" and gets promoted to "confirmed" once it recurs enough times with consistent verdicts.

Whether all of that actually adds up to materially better outputs than a single well-prompted LLM is an open empirical question. We have structures in place to test it; we don't claim to have settled it.

## LOCAL MODE (the default in v0.15.0)

Out of the box Lavern runs single-user on your machine. No login, no signup, no cookies. Every request is the synthetic `local-user`. Account routes, Google OAuth, email verification, password reset, Stripe billing, and the referral system are all gated behind `LAVERN_AUTH_ENABLED=true` and don't register at startup.

Flip the flag if you want to host Lavern for a team. The database schema is unchanged; the v0.14 cookie-based multi-user flow comes back online with no migration.

## Dashboard

React SPA, editorial design language (Newsreader serif, Geist sans, cream paper). WCAG AA accessible, responsive across desktop, tablet, and phone.

**Flow:** Landing → Briefing → Strategy → Team → Working → Delivery.

The Working view shows agents posting findings, debating, and resolving disputes in real time. The Delivery view includes confidence scores, grounding indicators, and the audit trail.

## API

```bash
npm run dev -- --serve     # Start API server, default localhost:3000
```

| Endpoint | Description |
|---|---|
| `POST /api/sessions` | Create analysis session |
| `GET /api/sessions/:id/events` | WebSocket event stream |
| `POST /api/sessions/:id/gate` | Submit gate decision |
| `GET /api/sessions/:id/download` | Download work product |
| `POST /api/engage` | Agent-native engagement (sync plus webhook) |
| `GET /api/capabilities` | Report auth state, billing state, provider, version |
| `GET /.well-known/agent.json` | A2A agent card |

See [`.env.example`](.env.example) for full configuration. Key variables:

- `ANTHROPIC_API_KEY`. Anthropic API key. Optional in demo mode.
- `MISTRAL_API_KEY`. Mistral API key. Required if `LAVERN_PROVIDER=mistral`.
- `LAVERN_PROVIDER`. `anthropic` (default), `mistral`, or `local` (Ollama).
- `LAVERN_AUTH_ENABLED`. Set `true` to enable cookie auth, Stripe billing, and Google OAuth. Default: off.
- `SHEM_PORT`. Server port. Default: 3000.
- `SHEM_DEFAULT_BUDGET`. Per-session budget in USD. Default: 5.0.

## Clawern commands

```bash
npm run dev -- claw init       # Interactive setup
npm run dev -- claw validate   # Configuration health check
npm run dev -- claw start      # Start watching
npm run dev -- claw pause      # Pause processing
npm run dev -- claw resume     # Resume processing
```

Includes a 30-minute heartbeat, Telegram bot, email alerts, weekly digest, scheduled re-review, change detection, cost forecasting, portfolio intelligence, hybrid local-plus-frontier processing, multi-client isolation, audit trail, and Prometheus metrics. `claw start --ethical` runs in Mistral EU only with a conservative risk posture.

## Development

```bash
npm test                  # 1,677 tests across 105 files
npm run typecheck:all     # TypeScript check (backend plus frontend)
npm run build             # Build backend
cd viz && npm run build   # Build dashboard
```

## Project structure

```
src/
├── agents/             67 agent prompts plus profiles plus definitions
├── api/                Fastify API server, WebSocket, middleware
├── assembly/           Document assembly, format conversion
├── claw/               Clawern: 28 modules (watch, plan, process, deliver,
│                       precedents, audit, backup, telegram, multi-client,
│                       hybrid analysis, anonymization)
├── db/                 SQLite persistence
├── documents/          Parser (PDF, DOCX, MD, TXT) plus sanitization
├── mcp/tools/          21 MCP tools
├── mcp/remote-bridge/  JSON-RPC bridge exposing Counsel tools to Anthropic
│                       Managed Agents (gated by env flag)
├── providers/          LLM provider abstraction (Claude, Mistral, local)
├── workflows/          9 templates plus executor
├── config.ts           Centralised config (env-var backed)
└── index.ts            Entry point

viz/                    React dashboard
site/                   Marketing site (static, Netlify)
menubar/                macOS menu bar app (SwiftUI)
tests/                  1,677 tests across 105 files
```

Full architectural detail in [CLAUDE.md](CLAUDE.md), the same context Claude Code reads when working on this repo.

## Connectors

The 21 MCP tools, 5 bundled legal datasets, 3 inference providers (Anthropic, Mistral, Ollama), and the remote MCP bridge are documented in [CONNECTORS.md](CONNECTORS.md).

## Status and known limitations

Lavern is at **v0.15.0**, the initial public open-source release. The engine, Clawern daemon, and dashboard are stable enough to run end-to-end against real documents; the test suite passes; `tsc --noEmit` is clean on both backend and frontend.

Known limitations we're not hiding:

- **No public benchmark.** Internal evaluation only. The quality-of-output claim is a hypothesis. We'd welcome help building a defensible benchmark.
- **Multi-agent debate is imperfect.** Agents sometimes don't listen to each other. Sometimes one dominates. Sometimes they swing to the opposite extreme when challenged, not because the challenge was stronger but because it was newer. We've built structure around the problem (evidence requirements, confidence thresholds, adversarial roles, escalation protocols), not solved it.
- **67 agents is probably more than needed.** Started with about a ten, just kept adding. There is also the agent builder mode if you want to build more. And Jude Claw.
- **No dense or vector retrieval.** The knowledge base uses BM25-style full-text search (SQLite FTS5). There's no embedding layer, no hybrid retrieval, no semantic precedent search. For high-recall retrieval across large document sets, that's the obvious next move; we haven't done it yet.
- **No durable task queue.** Long-running work flows through an in-process event bus and the Clawern daemon, not a Redis-backed queue with retry, dead-letter, and priority semantics. Session state persists (archive, precedent board), but if the server restarts mid-engagement the work needs to be re-kicked. Fine for the demo demonstrating the ideas; you'd want a real queue before running this at volume.
- **Counsel deliveries are slow.** A Counsel engagement on Cloud (Anthropic) takes 5–10 minutes end-to-end. Three things stack: the agent produces a long free-form transcript (rich audit trail, by design), the assembler runs a Sonnet cleanup pass to strip the chatter, and assembly calls are non-streaming so the dashboard sits on "Assembling…" with no progress feedback until the call completes. Other workflows (Review, Roundtable, Full Bench) are faster because they produce structured findings the assembler can format deterministically. Streaming assembly + a Haiku cleanup pass + marker-wrapped deliverables are the obvious wins for v0.15.1.
- **EU mode has one known gap.** The Lavern Challenge route (`src/api/routes/challenge.ts`) still instantiates Anthropic directly even when `LAVERN_PROVIDER=mistral`. The rest of the pipeline routes through Mistral. If you need strict EU boundary, avoid that feature until it's ported.
- **Remote MCP bridge is preview.** Gated behind a feature flag.
- **The HTTP API is evolving.** Expect non-breaking additions before a v1.0 freeze; pin a tag if you depend on it.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). New agents, new MCP tools, new workflow templates are all welcomed. The agent prompts are pure markdown, editable in any editor.

## Security

Found a vulnerability? See [SECURITY.md](SECURITY.md). Email `hello@lavern.ai`. Please don't file public GitHub issues for security bugs.

## License

[Apache 2.0](LICENSE). Copyright 2025–2026 Antti Innanen.

See [NOTICE](NOTICE) for third-party attributions and dataset licenses.