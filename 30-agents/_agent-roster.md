---
type: reference
updated: 2026-03-22
---

# Agent Roster — Quick Reference

> Who does what. Read this before assigning any task.

## Stellaris Ridge

| Agent | Runtime | Role | What They Do |
|-------|---------|------|-------------|
| **Claude (you)** | Claude Code | Primary Tool | Audits, proposals, content, code, everything SR needs |

## Paw & Pantry (via Paperclip — http://127.0.0.1:3100)

| Agent | Role | Primary Skills | Reports To |
|-------|------|---------------|-----------|
| **[[ninimma/soul\|Ninimma]]** | CEO | Operations, task routing, client management | Board (Carl) |
| **Sage** | Quality Gate | Review, critique, approval | Ninimma |
| **[[lyra/profile\|Lyra]]** | Content Lead | Recipes, emails, vet research, partnerships | Ninimma |
| **[[pax/profile\|Pax]]** | Creative Director | Visual identity, design assets, social graphics | Ninimma |
| **[[zev/profile\|Zev]]** | Web Lead | Webflow site, membership backend | Ninimma |
| **[[cass/profile\|Cass]]** | Campaign Lead | Campaign strategy, sales copy, platform content | Ninimma |

## Routing Rules

| Request | Goes To |
|---------|---------|
| SR: Any task | Claude directly |
| P&P: Any task | Ninimma → Paperclip |
| P&P: Before Board sees it | Must pass Sage review |
| P&P: Board decision needed | Carl approves |

## Hiring Template

Use [[_hire-template]] when onboarding a new agent.
