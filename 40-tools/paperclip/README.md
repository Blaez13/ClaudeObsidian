# Paperclip — superseded

**Status:** retired 2026-04-15.

Paperclip at `localhost:3100` was never built. It's been superseded by **Celestial Conclave** at `C:\Stellaris\conclave\` (separate git repo: [Blaez13/Conclave](https://github.com/Blaez13/Conclave)), which provides the agent orchestration, task queue, and dashboard that the Paperclip concept described.

## What moved where

| Paperclip concept | Conclave equivalent |
|---|---|
| `localhost:3100` routing layer | FastAPI at `localhost:3141` |
| Task queue for P&P | `tasks` table in Conclave SQLite + Tasks tab in dashboard |
| Agent delegation | `@mention` auto-dispatch (Phase 2) |
| Hive mind for agent coordination | `hive_activity` table (Phase 2) |

## Where to look now

- **Code:** `C:\Stellaris\conclave\`
- **Spec:** `C:\Stellaris\docs\superpowers\specs\2026-04-14-clawos-design.md`
- **Plans:** `C:\Stellaris\docs\superpowers\plans\2026-04-*-conclave-*.md`

## Historical notes

Original Paperclip design memo and any related files live alongside this README. Do not delete — keep for context. Just don't build anything new against `localhost:3100` — build against Conclave.
