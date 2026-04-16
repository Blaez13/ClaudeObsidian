---
type: reference
updated: 2026-04-15
---

# Agent Roster — Quick Reference

> Who does what. Read this before assigning any task.

## Board of Directors

Carl holds the chair directly. Three executive agents sit alongside him on the board.

| Seat | Name | Runtime | Model | Provider | Role |
|------|------|---------|-------|----------|------|
| **Chairman** | **Carl** | human | — | — | Final decisions, vision, client relationships |
| **CMO** | Katara | Hermes | Gemini 3 Pro | OpenRouter | Marketing, sales, ads, reputation, GEO |
| **CTO** | Mech | Hermes | GLM 5.1 | Z.AI (direct) | Tech / infra / deployments / implementation |
| **CFO** | Sokka | Claude Max | Haiku 4.5 | Anthropic | Finance, legal, budgets, ROI |

## Strategist (on-demand, not a voting seat)

| Name | Runtime | Model | Provider | Role |
|------|---------|-------|----------|------|
| Uncle Iroh | Claude Max | Opus 4.6 | Anthropic | **AI agency business strategist** — offer design, positioning, pricing, portfolio shape, what kills/scales small agencies. Not a tech/product/marketing specialist — he's the meta-level, industry-veteran voice that pressure-tests plans. Invoked sparingly. |

## Chief Operating Officer

Not a board seat, but carries real operational authority — triages everything, runs Librarian, manages execution pipeline.

| Name | Runtime | Model | Provider | Role |
|------|---------|-------|----------|------|
| Ninimma (Nin) | Hermes | MiniMax 2.7 | OpenRouter | Triage + routing; runs Librarian nightly; Carl's #2 for execution |

## Sub-agents (background)

| Name | Under | Runtime | Model | Role |
|------|-------|---------|-------|------|
| The Librarian | Nin | Hermes | Qwen 3.6 Plus | Nightly bidirectional sync: Board memory (Honcho) ↔ Obsidian vault. Uses Alibaba DashScope direct (not OpenRouter). |

## Execution Engines

| Tool | Runtime | Role |
|------|---------|------|
| Conclave dashboard | FastAPI (Windows, :3141) | Orchestration, task queue, hive mind, @-mention routing |
| Claude Max CLI | Windows | Sokka (Haiku) + Iroh (Opus) via `claude-agent-sdk` subprocess |
| Hermes | WSL Ubuntu | Katara, Mech, Nin, Librarian — session history, Honcho memory, cron |

## Claude Max slot allocation (avoid rate-limit contention)

Max supports concurrent Opus + Haiku. We keep **one** of each:
- **Opus 4.6** → Iroh (strategic, low-frequency)
- **Haiku 4.5** → Sokka (finance checks, high-frequency)

Everything else (Katara, Mech, Nin, Librarian) routes via Hermes → OpenRouter
and has no contention.

## Claude Code Skill Suites (Windows — available to board via MCP)

| Suite | Skills | Owner |
|-------|--------|-------|
| Marketing (`/market`) | 15 | Katara |
| Sales (`/sales`) | 14 | Katara |
| GEO/AEO (`/geo`) | 14 | Katara (strategy) + Mech (technical) |
| Ads (`/ads`) | 16 | Katara |
| Reputation (`/reputation`) | 15 | Katara |
| Legal (`/legal`) | 14 | Sokka |
| Frontend/Design | Various | Mech |
| Infrastructure / Conclave / Hermes | Various | Mech |

## How It Works

1. **Carl messages** — Telegram (existing Hermes bots), Conclave dashboard, or terminal
2. **Nin triages** — routes to specialist via `@katara` / `@mech` / `@sokka` / `@iroh`
3. **Specialist answers or delegates** — Claude SDK for Sokka/Iroh, Hermes subprocess for Katara/Mech/Nin
4. **For major decisions: pull Iroh** — Mech writes `@iroh: pressure-test this plan` before committing wide-blast-radius changes
5. **Conclave tracks** — tasks table + hive activity log, surfaced on dashboard
6. **Librarian syncs** — nightly at 2am (Hermes cron), Board decisions → vault, vault activity → Honcho
7. **Carl reviews** — morning catch-up via dashboard Activity tab

## Department Building

Each board member can build their own department by proposing sub-agents to Carl. Sub-agents are added as YAML files to `C:\Stellaris\conclave\agents\` (for Conclave visibility) plus a skill under their parent's Hermes profile (for Hermes-side runtime). See `C:\Stellaris\conclave\agents\librarian.yaml` for the canonical sub-agent pattern.

## Side Project

**Paw & Pantry** — Premium dog food brand. Dormant but viable. Context preserved in `20-paw-and-pantry/`. When Carl reactivates, it routes through Katara (CMO) as a brand initiative.

## Legacy Agents (Archived)

The following agents were part of the old P&P structure and are no longer active:

- Sage, Lyra, Pax, Zev, Cass — old P&P sub-agents (context in Honcho + `30-agents/`)
- Ridge — old Jarod sales assistant (archived to `~/.hermes/archive/`)
- Aang — old CEO role (deleted; Carl is Chairman now)

## Retired Hermes Profiles (as of 2026-04-15)

Archived at `~/.hermes/archive/profiles-2026-04-15/profiles/`:

- `chairman` — was Nin's role pre-2026-04-13
- `cto` — Mech moved to Hermes's default profile + OpenRouter GLM
- `cfo` — Sokka is now Claude Max (Hermes profile retired)
- `strategist` — replaced by Uncle Iroh as a proper Claude-native agent
