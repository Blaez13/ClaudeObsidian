# ClawOS — Design Spec

**Date:** 2026-04-14
**Status:** Approved by Carl. Ready for implementation plan.
**Supersedes / extends:** `C:\Users\ctrep\.claude\plans\floofy-cooking-tiger.md` (prior plan-mode session)
**Code lives at:** `C:\Stellaris\clawos\`

---

## 1. Goal

Build a single Python service ("ClawOS") that:

1. Replaces the empty Paperclip layer (`40-tools/paperclip/`).
2. Fronts the existing Hermes agent runtime at `~/.hermes/venv` via Python import (no HTTP shim).
3. Integrates the Boardroom skill into a running command (`/board <topic>`).
4. Exposes a LAN-reachable web dashboard accessible from Carl's basement PC, upstairs laptop, and Jarod's machine via Tailscale.
5. Hosts a War Room voice session where Hermes agents and Claude agents speak together as heterogeneous seats at the same table.

This is a Python port and extension of the ClaudeClawOS v2 reference architecture (Gumroad). The Gumroad package is a build-spec, not runnable code. We are not installing it as-is.

## 2. Stack

- **Python 3.11+**, managed by `uv`
- **FastAPI** — HTTP server, REST API, dashboard, WebSocket
- **SQLite** — WAL mode + FTS5 + field-level AES-256-GCM on secret columns
- **Alpine.js + HTMX** — embedded in single `templates/dashboard.html`. No build step.
- **Pipecat** (Phase 4) — War Room voice on port 7860, separate subprocess
- **discord.py** — Phase 2 gateway (only gateway; no Telegram/Slack in v1)
- **claude-agent-sdk** (Python) — subprocess spawns `claude` CLI with `bypassPermissions: true`, `maxTurns=30`, session resumption keyed to `(chat_id, agent_id)`
- **google-genai** — Phase 3 memory (Gemini extraction + embeddings)
- **NSSM** — Phase 4 Windows service installer

## 3. Location & Ports

- **Code:** `C:\Stellaris\clawos\` (top-level, outside the Obsidian markdown tree)
- **Dashboard + REST API:** `0.0.0.0:3141`
- **War Room voice:** `0.0.0.0:7860` (Phase 4 only)

## 4. Access Model

- Bind `0.0.0.0` on both ports — LAN-reachable from any machine on the home network.
- **Tailscale** for off-LAN access (Jarod's machine).
- **Auth:** per-user API token via `?token=...` query param on every request.
- `CLAWOS_TOKENS` env var holds JSON: `[{"name":"carl","token":"...","role":"admin"},{"name":"jarod","token":"...","role":"admin"}]`.
- Tailscale hostname allowlist (`CLAWOS_TAILSCALE_ALLOWLIST`) adds a second factor — request source hostname must match.
- **Jarod's role = admin** (option C from Q&A) — full capabilities, same as Carl. Audit log records `initiated_by` on every action so we can see who did what.
- No public HTTPS in v1.

## 5. Agent Roster

| Agent | Runtime | Role | Model | Provider |
|---|---|---|---|---|
| **Katara** (CMO) | hermes | Marketing / sales / ads / reputation / GEO | Gemini 3.1 | Hermes → OpenRouter |
| **Mech** (CTO) | claude | Tech / infra / deployments | Opus 4.6 (swappable) | Claude Max |
| **Sokka** (CFO) | claude | Finance / legal / budgets / ROI | Haiku 4.5 | Claude Max |
| **Ninimma** (PA) | hermes | Personal assistant (demoted from board) | MiniMax 2.7 | Hermes → OpenRouter |

**Board vote = Katara + Mech + Sokka (3 seats).** Ninimma excluded — she's @-addressable but doesn't vote.

**Excluded:** Sage, Lyra, Pax, Zev, Cass, Ridge — legacy P&P agents. Archive their `30-agents/<name>/` folders to `80-archive/legacy-pp-agents/`.

### Agent definition shape

Each agent is one YAML file at `agents/<id>.yaml`. Adding a new agent = drop one file, restart, no code changes.

```yaml
id: mech
name: Mech
runtime: claude          # or: hermes
model: claude-opus-4-6
cwd: C:\Stellaris        # working dir for claude CLI
system_prompt_path: agents/_prompts/mech.md
mcp_servers: [all]       # or specific list
voice_preset: charon     # Phase 4 War Room
demoted: false           # true = no board vote
enabled: true
color: "#4a9eff"         # dashboard
```

## 6. Architecture

```
                      ┌──────────────────────────────────────┐
                      │  ClawOS Core (FastAPI, port 3141)    │
                      │  - Embedded Alpine.js + HTMX UI      │
                      │  - SQLite WAL + FTS5 + AES fields    │
                      │  - Per-user token auth + Tailscale   │
                      └──────────────┬───────────────────────┘
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        │                            │                            │
  ┌─────▼─────┐              ┌───────▼───────┐            ┌───────▼──────┐
  │  Gateway  │              │  Orchestrator │            │   War Room   │
  │ Discord   │              │ runtime field │            │ Pipecat 7860 │
  │ Dashboard │              │ picks bridge  │            │ Gemini Live  │
  └───────────┘              └───────┬───────┘            └──────┬───────┘
                                     │                           │
                          ┌──────────┼──────────┐                │
                          │                     │                │
                   ┌──────▼──────┐       ┌──────▼─────┐          │
                   │   Hermes    │       │   Claude   │          │
                   │   Bridge    │       │    SDK     │◄─────────┘
                   │ Python      │       │ subprocess │
                   │ import      │       │            │
                   └─────────────┘       └────────────┘
                          │                     │
                          └──────────┬──────────┘
                                     │
                       ┌─────────────▼──────────────┐
                       │   6-Layer Memory (Gemini)  │
                       │   + Vault crystallization  │
                       │   + Hive-mind activity log │
                       └────────────────────────────┘
```

### Data flow (every message)

```
Gateway → Message Queue (asyncio FIFO per chat_id)
  → Security Gate (PIN check, rate limit, audit log)
  → Memory Inject (6-layer retrieval → system context)
  → Orchestrator (parses @agent addressing)
    ├─ runtime == "claude" → claude_sdk.invoke()
    └─ runtime == "hermes" → hermes.invoke()
  → Exfiltration Guard (regex scan, redact secrets)
  → Reply via originating gateway
  → Fire-and-forget: Memory Ingest + Relevance Evaluation + Crystallization
```

### Novel contribution vs ClaudeClawOS

ClaudeClawOS treats agents as homogeneous — all are Claude SDK calls with different system prompts.

ClawOS treats agents as **heterogeneous seats at the same table**:

- Claude agents invoked via `claude-agent-sdk` subprocess (`runtime: claude`)
- Hermes agents invoked via Python import of Hermes session API (`runtime: hermes`)
- Both read/write the **same SQLite memory layer** and **same hive-mind activity log**
- Both appear as voices in the **same War Room session**
- Router picks bridge based on the `runtime` field in agent YAML

## 7. Memory

### 6-layer retrieval (every turn)

1. **Semantic** — embedding cosine similarity ≥ 0.3, top 5
2. **FTS5 keyword** on `memories` table, top 5
3. **Recent high-importance** — last 48h, importance ≥ 0.7, top 5
4. **Consolidation insights** — latest 3 records
5. **Conversation history recall** — 7-day window, top 10
6. **Vault markdown FTS** — grep `c:/Stellaris/**/*.md`, top 5

Results deduplicated by content hash, formatted into a context string with section headers, injected as system context.

### Crystallization (writes to vault)

When a SQLite memory hits `importance ≥ 0.8`, `clawos/crystallize.py` appends to the relevant vault markdown file:

| Memory signal | Target | Format |
|---|---|---|
| Contains client name in `clients/` | `10-stellaris-ridge/clients/<name>/brief.md` | Append under `## Memory <date>` with fact + source session id |
| Topic includes "decision", "agreed", "chose" | Nearest `_decisions.md` | Date + rationale + agents involved |
| Today's session, reflective | `01-daily/<today>.md` | Bullet under `### Notes` |
| Client-business rule (pricing, SOP, process) | `10-stellaris-ridge/_context.md` | Append under matching section |
| SR company fact | `10-stellaris-ridge/_context.md` | Same |
| Paw & Pantry-related | `20-paw-and-pantry/_context.md` or `_decisions.md` | Same |
| Cannot classify with confidence | Skip — stay SQLite-only | — |

Classification: rule-based first. Fall back to Gemini (`gemini-3-flash-preview` classification prompt) when rules don't match. If Gemini confidence < 0.7, skip crystallization.

After append, `git add && git commit -m "crystallize: <summary>"` so the change flows through Carl's normal vault git history.

### Decay & supersession

- Pinned memories: 0% decay
- Importance ≥ 0.8: 1% decay/day
- Importance ≥ 0.5: 2% decay/day
- Importance < 0.5: 5% decay/day
- Hard delete when salience drops below 0.05
- Contradictions: newer memory's `importance` is checked, if higher, older memory gets `superseded_by` pointer

## 8. Phases

### Phase 1 — Core pipeline & dashboard (ships first)

**Goal:** Open dashboard from upstairs laptop → message Mech → reply → restart process → transcript reloads.

| File | Lines (est) | Purpose |
|---|---|---|
| `clawos/main.py` | 150 | FastAPI app, lifespan, mount routes, bind 0.0.0.0:3141 |
| `clawos/auth.py` | 80 | Token + Tailscale hostname check, returns User or 401 |
| `clawos/queue.py` | 60 | asyncio.Queue per chat_id with worker tasks |
| `clawos/orchestrator.py` (v1) | 120 | Single-agent routing, dispatches to Claude bridge |
| `clawos/agents/__init__.py` | 40 | YAML loader, agent registry dict |
| `clawos/agents/claude_sdk.py` | 180 | `claude-agent-sdk` subprocess wrapper |
| `clawos/routes/dashboard.py` | 120 | `/`, `/api/sessions`, `/api/messages`, `POST /api/send`, SSE `/api/events` |
| `clawos/routes/health.py` | 30 | `/healthz` (no auth), `/api/status` |
| `templates/dashboard.html` | 600 | Single-file UI, Alpine.js + HTMX, dark theme, 3-pane layout |
| `agents/mech.yaml` | 20 | Mech config |
| `agents/sokka.yaml` | 20 | Sokka config |
| `agents/_prompts/mech.md` | 40 | Minimal CTO system prompt (real persona work in Phase 3) |
| `agents/_prompts/sokka.md` | 40 | Minimal CFO system prompt |
| `clawos/db.py` | existing +80 | Verify schema: `sessions`, `messages`, `audit_log`. Add FTS5 triggers on messages. |
| `.env.example` | update | Add `CLAWOS_TOKENS`, `CLAUDECLAW_CONFIG`, `CLAWOS_TAILSCALE_ALLOWLIST` |

**Total Phase 1 scope:** ~1,600 Python + 600 HTML lines.

**Dependencies to add:** `fastapi`, `uvicorn[standard]`, `jinja2`, `pyyaml`, `cryptography`, `sse-starlette`, `claude-agent-sdk`.

**Acceptance test:**
1. `cd C:\Stellaris\clawos && uv sync`
2. `cp .env.example .env`, fill `CLAWOS_TOKENS` with carl-token + jarod-token
3. `uv run uvicorn clawos.main:app --host 0.0.0.0 --port 3141`
4. Basement: `http://localhost:3141/?token=<carl-token>` → dashboard, Mech + Sokka in agent list
5. Upstairs laptop: `http://<basement-pc-ip>:3141/?token=<carl-token>` → same dashboard
6. Send "hi, who are you?" → Mech replies within ~10s
7. Ctrl+C, restart, reload page → last message visible, conversation continues
8. Wrong token → 401. Jarod's token → 200, audit log records `initiated_by=jarod`

### Phase 2 — Discord gateway + Hermes bridge + Security

**New files:**
- `clawos/gateways/discord.py` — discord.py, one bot, multiple channels (one per agent), Jarod permissioning by channel role
- `clawos/bridges/hermes.py` — `sys.path.insert(0, ~/.hermes/venv/Lib/site-packages)`, `from hermes.sessions import run_turn`. Records turns in ClawOS messages table for dashboard visibility.
- `clawos/security.py` — PIN lock (salted SHA-256), idle auto-lock, kill phrase
- `clawos/exfiltration.py` — 15+ regex patterns (API keys, AWS, tokens, base64, URL-encoded). Redacts to `[REDACTED]`, logs to audit.
- `clawos/orchestrator.py` (v2) — `@agent` parsing, runtime field dispatch (claude vs hermes)

**Acceptance:**
- Discord message in #katara → routes through ClawOS → Hermes bridge → Gemini 3.1 reply
- Same path `@mech` from dashboard → Claude Max → Opus reply
- Send fake API key → exfiltration guard blocks outbound, logs entry

**Paperclip retired in Phase 2:** grep vault for `localhost:3100`, replace references with ClawOS pointer.

### Phase 3 — Memory v2 + Boardroom + Crystallization + Scheduler

**New files:**
- `clawos/memory.py` — 6-layer retrieval, formatted system context
- `clawos/memory_ingest.py` — Gemini extraction post-turn (fire-and-forget), 768-dim embeddings, dedup at 0.85 cosine
- `clawos/memory_consolidate.py` — 30-min background job, patterns + contradictions
- `clawos/crystallize.py` — see Section 7 above
- `clawos/boardroom.py` — `/board <topic>` fans out to 3 seats in parallel, returns perspectives, persists to `_decisions.md` + SQLite
- `clawos/scheduler.py` — 60s cron poll, priority ordering, auto-assignment
- `clawos/hive.py` — cross-agent activity log

**Vault changes (Phase 3 cleanup):**
- Rewrite `30-agents/_agent-roster.md` with correct 4-agent roster
- Update `CLAUDE.md` Board table
- Update `memory/project_board_structure.md` (Katara → Gemini 3.1, Nin → MiniMax 2.7)
- Update `memory/project_clawos.md` (slim roster to 4)
- Archive `30-agents/{sage,lyra,pax,zev,cass,ridge}/` → `80-archive/legacy-pp-agents/`
- Add `40-tools/clawos/README.md` and `40-tools/hermes/README.md`
- Mark `40-tools/paperclip/README.md` as superseded

**Acceptance:**
- `/board should we sign Client X?` → 3 perspectives within 60s, written to `_decisions.md` and dashboard
- Insert fact ("favorite beer is Spotted Cow") → 30 min later consolidation runs → fact surfaces in unrelated session
- Schedule cron "every Monday 9am, Sokka summarizes weekly spend" → fires Monday, posts to log
- High-importance memory ("client decision: Acme moving to monthly retainer Q2 2026") auto-appends to `clients/acme/brief.md` + commits to git

### Phase 4 — War Room voice + Windows service

**New files:**
- `warroom/server.py` — Pipecat on 7860, spawned from FastAPI lifespan, dual mode
- `warroom/router.py` — broadcast / name-prefix / pin routing
- `warroom/agent_bridge.py` — for `runtime: claude` shells claude-agent-sdk; for `runtime: hermes` Python imports. **Both speak in the same voice session.**
- `warroom/personas.py` — voice presets per agent (no GoT theme — use SR names)
- `warroom/config.py`, `warroom/voices.json`
- `templates/warroom.html` — cinematic UI at `/warroom`
- `scripts/install-nssm.ps1` — NSSM service installer

**Voice cascade:**
- STT: Groq Whisper → whisper-cpp fallback
- TTS: ElevenLabs → Kokoro (local) → Windows SAPI

**NOT in v1:** Meeting Bot. No Pika, no Recall.ai. Can be added as Phase 5 later.

**Acceptance:**
- Carl clicks War Room from dashboard → invites Katara + Mech → 3-way voice call (1 human + 1 Hermes agent + 1 Claude agent). Each speaks in a distinct voice. Transcript persists to SQLite.
- Reboot Windows → ClawOS auto-starts via NSSM → dashboard reachable within 60s

## 9. Open items (resolution plan)

| Item | When | How |
|---|---|---|
| Persona prompts for Katara/Mech/Sokka | Phase 3 kickoff | Co-drafting session w/ Carl. Use `boardroom-perspectives.md` template. ~200 words each. Save to `agents/_prompts/<name>.md` + `30-agents/<name>/profile.md`. |
| Telegram token strategy | Resolved — N/A | ClawOS uses Discord only. Existing Hermes-Telegram bots keep running untouched. |
| Tailscale setup | Phase 1 acceptance | Confirm 3 machines on same tailnet, get Jarod's node name, populate `CLAWOS_TAILSCALE_ALLOWLIST` |
| Paperclip shutdown | Phase 2 | Grep `localhost:3100` across vault, update references |
| Hermes venv import path | Phase 2 | Test `sys.path` insert from Python REPL before writing `bridges/hermes.py` |
| Mech's model | Anytime | One-line edit in `agents/mech.yaml` |

## 10. Estimated effort

- **Phase 1:** 1–2 focused sessions (ship working dashboard)
- **Phase 2:** ~1 week
- **Phase 3:** ~1 week (largest — memory engine + crystallization + boardroom)
- **Phase 4:** ~1 week (Pipecat integration is the unknown)

Total: ~4 weeks of focused work to full parity.

## 11. Decisions log (this session, 2026-04-14)

| Decision | Choice |
|---|---|
| Architecture approach | A — resume existing Python plan (not install ClaudeClawOS Node fresh) |
| Hermes integration | B — Hybrid (Hermes as first-class seat, not migration to Claude) |
| Deployment host | A — WSL was floated, then revised to **Windows-native** to match existing scaffold at `C:\Stellaris\clawos\` |
| Memory system | Hybrid — ClaudeClaw Memory v2 engine + vault crystallization for high-importance facts |
| Jarod access | C — full admin (will likely behave as B in practice) |
| Power Packs | B — all except Meeting Bot |
| Agents excluded | Sage, Lyra, Pax, Zev, Cass — legacy P&P agents removed |
| Roster runtimes | Katara=hermes/Gemini 3.1, Mech=claude/Opus 4.6, Sokka=claude/Haiku 4.5, Nin=hermes/MiniMax 2.7 |
| Phase 2 gateway | Discord only (skip Telegram + Slack) |
| Board vote composition | 3 seats: Katara + Mech + Sokka (Nin excluded as demoted) |
| Windows service | NSSM |
