# Conclave Phase 2 — Hermes bridge + task coordination

**For agentic workers:** execute inline with user review between task groups. No subagent dispatch needed.

**Goal:** Katara + Ninimma reply via Hermes. Agents can @-mention each other to hand off tasks. Dashboard shows tasks and cross-agent activity. No Discord, no PIN lock.

**Status of scope decisions (settled 2026-04-15):**
- Discord: DROPPED (War Room will handle board discussions)
- Telegram: UNTOUCHED (Hermes gateway stays as-is)
- Roster: Mech/Sokka = Claude Max, Katara/Nin = Hermes
- Task creation: from @-mentions (auto-dispatch) + from UI (queue until user runs)
- PIN lock: DEFERRED
- Mark Kashef influence: specialist agents + visible task coordination, no video avatars

**Pre-phase chores (Carl runs before execution):**
1. `MSYS_NO_PATHCONV=1 wsl.exe -d Ubuntu -- bash -c '~/.hermes/hermes-agent/venv/bin/hermes login'` — Nous Portal token expired 2026-04-12
2. Decide Katara's Gemini 3.1 routing:
   - If `openrouter` (recommended, Hermes already has OpenRouter key): model string stays `gemini-3-pro`, Hermes auto-picks OpenRouter provider
   - If direct Google: add `GEMINI_API_KEY` to Hermes's `.env`
3. Decide Ninimma's MiniMax routing: same choice. OpenRouter covers `minimax/minimax-m2` too.

We'll test bridge with a simple model first (qwen-plus — already keyed) and iterate to the target models once auth confirmed.

---

## Task Group A — Fix Sokka's YAML + roster reality check

**Files:** `agents/sokka.yaml`

**Changes:** Already correct from Phase 1 — `runtime: claude, model: claude-haiku-4-5-20251001`. Verify nothing drifted. No edit needed if current file matches.

**Acceptance:** `curl /api/agents?token=...` shows Sokka with `runtime: claude, model: claude-haiku-4-5-20251001`.

**Commit:** Only if changed.

---

## Task Group B — Hermes subprocess bridge

**New file:** `conclave/bridges/__init__.py` (empty)
**New file:** `conclave/bridges/hermes.py` (~180 lines)
**New file:** `tests/test_hermes_bridge.py` (~120 lines)

**Design:**

```python
# conclave/bridges/hermes.py
@dataclass
class HermesReply:
    text: str
    tokens_in: int       # best-effort parse from --verbose output, else 0
    tokens_out: int
    cost_usd: float
    session_id: str | None

class HermesBridge:
    def __init__(self, settings: Settings) -> None:
        self.settings = settings
        # Hard-coded for now; expose via settings if other hosts ever
        self.wsl_distro = "Ubuntu"
        self.hermes_bin = "~/.hermes/hermes-agent/venv/bin/hermes"

    async def send(
        self, *, agent: AgentConfig, user_message: str, session_id: str | None
    ) -> HermesReply:
        # Build command
        cmd = ["wsl.exe", "-d", self.wsl_distro, "--", "bash", "-c",
               f"{self.hermes_bin} chat -q {shlex.quote(user_message)} "
               f"--quiet --source conclave --pass-session-id "
               + (f"--resume {shlex.quote(session_id)} " if session_id else "")
               + (f"-m {shlex.quote(agent.model)} " if agent.model else "")]
        # Spawn with asyncio.create_subprocess_exec
        # Capture stdout, parse for session ID + reply text
        # Return HermesReply
```

**Parsing strategy:**
- `--quiet --pass-session-id` outputs just the reply text, but emits a `Session: <uuid>` line we can grep. Confirm with a live test before writing the parser.
- If the output format is different from expectations, adapt the parse — log stdout on first run to eyeball it.

**Tests:**
- Monkeypatch `asyncio.create_subprocess_exec` with a fake that returns canned stdout
- Verify command args: WSL distro, hermes binary path, `--quiet`, `--source conclave`, `--resume` when session_id given, `-m <model>` when model given
- Verify reply text extraction + session_id capture

**Manual verification (outside tests):**
```bash
MSYS_NO_PATHCONV=1 wsl.exe -d Ubuntu -- bash -c '~/.hermes/hermes-agent/venv/bin/hermes chat -q "say hello in 3 words" --quiet --source conclave --pass-session-id'
```
Capture the exact stdout shape before finalizing the parser.

**Commit:** `conclave: Hermes subprocess bridge — WSL forwarding with session resume`

---

## Task Group C — Orchestrator wires Hermes bridge

**Edit:** `conclave/orchestrator.py` — replace the NotImplementedError stub for `runtime: hermes`.

**Change:**
```python
async def _run_hermes(self, *, session_pk: int, agent: AgentConfig, msg: QueuedMessage) -> ClaudeReply:
    sid = await self.db.get_sdk_session_id(session_pk)
    reply = await self._hermes.send(
        agent=agent, user_message=msg.content, session_id=sid
    )
    if reply.session_id and reply.session_id != sid:
        await self.db.set_sdk_session_id(session_pk, reply.session_id)
    # Return shape matches ClaudeReply so orchestrator doesn't care which runtime.
    return ClaudeReply(
        text=reply.text, tokens_in=reply.tokens_in, tokens_out=reply.tokens_out,
        cost_usd=reply.cost_usd, sdk_session_id=reply.session_id,
    )
```

Also adjust `Orchestrator.__init__` to accept/construct `HermesBridge`.

**Test:** extend `test_orchestrator.py` — hermes agent case now dispatches (with mocked bridge) instead of raising.

**Acceptance (live):**
- Start server, open Katara's page, send "hi"
- Get a real reply from Gemini via Hermes (10-20s cold)
- Send second message, confirm session resumes (Katara remembers)

**Commit:** `conclave: orchestrator routes Hermes agents through WSL bridge`

---

## Task Group D — Exfiltration guard

**New file:** `conclave/exfiltration.py` (~130 lines)
**New file:** `tests/test_exfiltration.py` (~60 lines)

**Patterns to scan in outbound replies:**
- Anthropic API keys: `sk-ant-[A-Za-z0-9_\-]{32,}`
- OpenAI keys: `sk-[A-Za-z0-9]{20,}` (also `sk-proj-…`)
- OpenRouter: `sk-or-v1-[a-f0-9]{64}`
- Google: `AIza[0-9A-Za-z\-_]{35}`
- AWS: `AKIA[0-9A-Z]{16}` and secret-key regex
- Generic Bearer token with long suffix: `Bearer\s+[A-Za-z0-9\-_\.]{40,}`
- JWT: `eyJ[A-Za-z0-9_\-]{10,}\.[A-Za-z0-9_\-]{10,}\.[A-Za-z0-9_\-]{10,}`
- Generic password= assignment
- Base64 blob > 80 chars (soft check, flags only)
- URL-encoded secret markers (`%3D%3D` at end suggests base64 payload)

**API:**
```python
@dataclass
class SecretMatch:
    kind: str     # "anthropic_key" | "aws_access" | etc.
    start: int
    length: int
    preview: str  # first 8 chars + "…"

def scan(text: str) -> list[SecretMatch]
def redact(text: str) -> tuple[str, list[SecretMatch]]
```

**Integration point:** in orchestrator, after agent reply returns but before `append_message(role="assistant")`, call `redact(reply.text)`. If matches found, replace with `[REDACTED:<kind>]`, append message with redacted text, write audit `event="exfil_blocked"` with count.

**Tests:** unit tests with known-fake keys, verify they're caught + redacted.

**Commit:** `conclave: exfiltration guard — regex scan + redact outbound agent replies`

---

## Task Group E — Task coordination (the meat)

**New files:**
- `conclave/tasks.py` (~200 lines — task CRUD + @-mention parser + auto-dispatch)
- `tests/test_tasks.py` (~130 lines)

**Schema:** Already exists in `db.py` — `tasks` table with `task_id, title, description, status, priority, assigned_agent, payload_json, result_json, cron_expr, next_run_at`. No migration needed.

**Task lifecycle:**
```
queued → running → done / failed
         ↑
         manual "run" from UI, OR auto-dispatch on @-mention
```

**Add to `db.py`:**
- `create_task(title, description, assigned_agent, payload, priority=5) -> task_id`
- `list_tasks(agent_id: str | None, status: str | None, limit=50) -> list[dict]`
- `get_task(task_id) -> dict | None`
- `update_task_status(task_id, status, result=None)`
- `list_hive_activity(limit=50) -> list[dict]` + `insert_hive_event(agent_id, activity, summary, ref_message_pk=None)`

**Mention parsing:**
- Pattern: `@([a-z][a-z0-9_\-]{0,29})[:\s]+(.+?)(?=@[a-z]|\Z)` (regex, DOTALL)
- Given an assistant reply, extract all `@agent: instruction` delegations
- For each: verify agent exists in registry; if yes, create a task + fire-and-forget dispatch

**Gemini auto-assign (added per Pack 4 spec + 2026-04-15 confirmation):**

When a task is created via UI with `assigned_agent=null`, call a cheap Gemini model via Hermes (`hermes chat -q "<classification prompt>" --provider openrouter -m gemini/gemini-3-flash --quiet`) with a classifier prompt that includes each agent's role + the task description. Returns an agent_id; update the task. Fail-soft: if Gemini errors, task stays unassigned until user picks.

**Stuck task recovery (added from Pack 4 spec):**

In `_lifespan` on startup, `UPDATE tasks SET status='queued' WHERE status='running'`. Any task mid-flight when the process died comes back to the queue. One line, trivially worth it.

**Auto-dispatch flow:**
```python
# In orchestrator.handle(), AFTER persisting assistant reply:
delegations = parse_delegations(reply.text)  # list of (agent_id, instruction)
for to_agent, instruction in delegations:
    if to_agent not in registry:
        continue
    task_id = await db.create_task(
        title=instruction[:80],
        description=instruction,
        assigned_agent=to_agent,
        priority=5,
    )
    await db.insert_hive_event(
        agent_id=msg.agent_id, activity="delegated",
        summary=f"→ @{to_agent}: {instruction[:60]}",
    )
    # Dispatch async — don't block the current reply
    asyncio.create_task(self._run_task(task_id, to_agent, instruction))

async def _run_task(self, task_id: str, agent_id: str, instruction: str) -> None:
    await self.db.update_task_status(task_id, "running")
    await self.db.insert_hive_event(agent_id=agent_id, activity="started",
                                     summary=f"task {task_id[:8]}: {instruction[:60]}")
    try:
        # Synthesize a QueuedMessage for this task
        synth_msg = QueuedMessage(
            chat_id=f"task:{task_id}", agent_id=agent_id,
            channel="task", content=instruction, actor="orchestrator",
            meta={"task_id": task_id, "from_delegation": True},
        )
        reply = await self.handle(synth_msg)
        await self.db.update_task_status(task_id, "done", result=reply.text)
        await self.db.insert_hive_event(agent_id=agent_id, activity="finished",
                                         summary=f"task {task_id[:8]} done")
    except Exception as e:
        await self.db.update_task_status(task_id, "failed", result=str(e))
        await self.db.insert_hive_event(agent_id=agent_id, activity="failed",
                                         summary=f"task {task_id[:8]}: {e}")
```

Note: recursive handoffs (Katara delegates → Mech's reply delegates to Sokka) are naturally supported because `self.handle()` re-runs the delegation parser.

**Cycle protection:** `meta["delegation_depth"]` incremented per hop; bail at 5. Stops ping-pong loops.

**API endpoints (add to `main.py`):**
- `GET /api/tasks?agent_id=&status=` — list
- `POST /api/tasks` — create (body: `{title, description, assigned_agent, priority}`)
- `POST /api/tasks/{task_id}/run` — manually dispatch a queued task
- `POST /api/tasks/{task_id}/cancel` — mark failed with reason "cancelled"
- `GET /api/hive?limit=50` — activity feed

**Tests:**
- Mention parser unit tests — extract 0/1/multiple @-mentions, handle trailing punctuation
- Task lifecycle test — create → run (mocked orchestrator) → done
- Delegation depth cap test — construct chain that would exceed, verify bails

**Commit:** `conclave: task coordination — @mention auto-dispatch, hive activity, task CRUD`

---

## Task Group F — Dashboard UI for tasks + hive

**Edit:** `templates/dashboard.html`

**Adds:**
1. **New "Tasks" tab** in nav
   - Columns: Agent | Title | Status | Age
   - Status filter: All / Queued / Running / Done / Failed
   - Click row → expand to show full description + result
   - "New task" button opens inline form (title, assignee, priority)
   - Queued tasks have "Run" button; running tasks show a spinner

2. **On agent detail page** — right column, below Profile:
   - "Tasks for <Agent>" card with count badges (Q/R/D) and clickable list
   - Link to Tasks tab filtered to this agent

3. **Activity tab** — add hive-mind events to the existing feed
   - Render rows like: `[timestamp] · <agent> · <activity> · <summary>`
   - Agent name badges colorized by runtime

4. **@mention hint** — small helper text under the chat composer: *"Tip: reply with @katara, @mech, @sokka, @ninimma to delegate a follow-up task"*

**No tests for UI** (it's Alpine.js in one file; eyeball-verify during live acceptance).

**Commit:** `conclave: dashboard — Tasks tab, per-agent task panel, hive activity feed`

---

## Task Group G — Nin triage prompt + retire Paperclip

**Nin as the Main/triage agent (confirmed 2026-04-15):**

Update `agents/ninimma.yaml` system_prompt to a "delegate, don't execute" frame. She reads incoming requests, picks a specialist (Katara/Mech/Sokka), and `@-mentions` them with a concrete ask. She only answers directly when the question is genuinely hers (calendar, personal triage, quick yes/no). This matches Mark Kashef's "Main" pattern — the manager doesn't do the work, she routes it.

**Prompt additions to ninimma.yaml:**
```
ROUTING RULES — when Carl sends a message, follow this hierarchy:
1. If it's a marketing / sales / ads / reputation / GEO / content question → @katara: <full instruction>
2. If it's a tech / infra / deployment / architecture question → @mech: <full instruction>
3. If it's a finance / legal / pricing / ROI / budget question → @sokka: <full instruction>
4. If it's calendar / reminder / personal triage / "where's the X document" → answer yourself
5. If ambiguous → ask Carl one clarifying question before routing

NEVER execute specialist work yourself. Your value is triage speed, not breadth.
When you delegate, write the @mention + instruction in your reply. Conclave's
orchestrator auto-dispatches and tracks the task for you.
```

**Paperclip retirement chores:**

**Chores:**
- `grep -rn "localhost:3100" c:/Stellaris/` — find references
- Update each reference to "Conclave (dashboard at `http://localhost:3141/`)" or similar
- Add `c:/Stellaris/40-tools/paperclip/README.md`:
  ```markdown
  # Paperclip — superseded

  Paperclip at localhost:3100 was never built. Superseded by **Celestial Conclave** at `C:\Stellaris\conclave\` (port 3141), which provides the agent orchestration, task queue, and dashboard the Paperclip concept described.
  ```
- Archive `20-paw-and-pantry/_active-tasks.md` references if they point at localhost:3100

**Commit** (in the VAULT repo, not Conclave): `vault: mark Paperclip superseded by Conclave`

---

## Acceptance (end of Phase 2)

1. Open Katara's page → send "hi, one short sentence" → get a real Gemini reply (10-20s first time, faster after). 
2. Open Mech's page → send "Mech, ask Katara to draft a 50-word LinkedIn post about agentic marketing. @katara: please draft 50 words on agentic marketing for LinkedIn." Watch: Mech's reply completes → a task auto-appears on Katara's tasks list with status `running` → Katara replies → task goes `done`, result visible.
3. Tasks tab shows both runs.
4. Activity tab shows the delegation chain in the hive feed.
5. Try to send an assistant reply containing a fake API key (e.g., hand-craft by asking "echo back this string: sk-ant-api03-deadbeef…") → outbound reply shows `[REDACTED:anthropic_key]`, audit log has `exfil_blocked` entry.
6. Paperclip references in vault updated, `40-tools/paperclip/README.md` exists.

---

## Deferred (post-Phase 2)

- Cron-scheduled tasks (Phase 3 scheduler)
- PIN lock (add if/when we expose anything publicly)
- Telegram-for-Mech decision (revisit after living with the dashboard a few days)
- Memory v2 + crystallization (Phase 3 — or now Phase 4 if War Room jumps ahead)
