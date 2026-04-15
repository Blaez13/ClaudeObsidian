---
type: reference
updated: 2026-04-15
---

# Agent Roster — Quick Reference

> Who does what. Read this before assigning any task.

## Board of Directors

Carl holds the chair directly (as of 2026-04-15). Three executive agents sit alongside him. Ninimma is Executive Assistant, not a board seat, but carries real operational authority.

| Seat | Name | Runtime | Model | Provider | Role |
|------|------|---------|-------|----------|------|
| **Chairman** | **Carl** | human | — | — | Final decisions, vision, client relationships |
| **CMO** | Katara | Hermes | Gemini 3 Pro | OpenRouter | All marketing — campaigns, content, ads, reputation, sales, GEO strategy |
| **CTO** | Mech | Claude Max | Opus 4.6 | Anthropic | All technology — infrastructure, deployments, SEO technicals, Conclave, tooling |
| **CFO** | Sokka | Claude Max | Haiku 4.5 | Anthropic | All finance — budgets, legal, ROI, pricing, cost tracking, compliance |

## Executive Assistant

| Name | Runtime | Model | Provider | Role |
|------|---------|-------|----------|------|
| Ninimma (Nin) | Hermes | MiniMax 2.7 | OpenRouter | Triage + routing to board specialists; runs Librarian nightly; Carl's #2 for execution |

## Sub-agents (background)

| Name | Under | Runtime | Model | Role |
|------|-------|---------|-------|------|
| The Librarian | Nin | Hermes | Qwen-Plus | Nightly bidirectional sync: Board memory (Honcho) ↔ Obsidian vault |

## Execution Engines

| Tool | Runtime | Role |
|------|---------|------|
| Conclave dashboard | FastAPI (Windows) | Orchestration, task queue, hive mind, @-mention routing |
| Claude Max CLI | Windows | Mech + Sokka invocations via `claude-agent-sdk` subprocess |
| Hermes | WSL Ubuntu | Katara + Nin + Librarian — session history, Honcho memory, cron |
| Claude Code (WSL) | WSL | Occasional WSL-side system work |

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

1. **Carl messages** — Discord (soon), Telegram (existing Hermes bots), or Conclave dashboard
2. **Nin triages** — routes to specialist via `@katara` / `@mech` / `@sokka` mentions
3. **Specialist answers or delegates** — claude-agent-sdk for Mech/Sokka, Hermes subprocess for Katara
4. **Conclave tracks** — tasks table + hive activity log, surfaced on the dashboard
5. **Librarian syncs** — nightly at 2am, Board decisions flow to vault, vault activity flows to Honcho
6. **Carl reviews** — morning catch-up via dashboard Activity tab

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

These Hermes profiles existed under the old structure but are no longer in active use. Kept for now (archive later):

- `chairman` — was Nin's role pre-2026-04-13; Carl now holds the chair directly
- `cto` — Mech is now Claude-native, not Hermes
- `cfo` — Sokka is now Claude-native, not Hermes
- `strategist` — was Uncle Iroh shell; TBD whether to resurrect as a proper agent
