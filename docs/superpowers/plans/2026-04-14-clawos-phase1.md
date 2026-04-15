# ClawOS Phase 1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a working end-to-end loop: open dashboard from upstairs laptop → send a message to Mech (Claude) → reply streams back → restart process → transcript reloads.

**Architecture:** FastAPI app binds `0.0.0.0:3141`. Per-user token auth (+ optional Tailscale hostname allowlist). Single-agent routing: dashboard → queue → orchestrator → Claude bridge (`claude-agent-sdk` subprocess) → DB-persisted reply → SSE to client. Hermes bridge, Discord gateway, memory engine, and voice all come in later phases.

**Tech Stack:** Python 3.11+, FastAPI, aiosqlite (WAL+FTS5+AES-GCM via existing `db.py`), `claude-agent-sdk`, Alpine.js + HTMX in a single-file embedded template.

**Spec reference:** `docs/superpowers/specs/2026-04-14-clawos-design.md` (Section 8 — Phase 1).

---

## Pre-flight context

**Working directory:** `C:\Stellaris\clawos\` (Windows-native path; Bash tool uses `/c/Stellaris/clawos/` or forward-slash equivalents).

**What already exists:**
- `pyproject.toml` — deps already include `fastapi`, `uvicorn[standard]`, `aiosqlite`, `claude-agent-sdk`, `cryptography`, `pyyaml`, `jinja2`, `pydantic-settings`. Dev deps include `pytest`, `pytest-asyncio`.
- `clawos/config.py` — `Settings` with `tokens` (parses `CLAWOS_TOKENS=name:token,name:token`), `tailscale_hosts` (parses `CLAWOS_TAILSCALE_HOSTS=host1,host2`), `db_path`, `ensure_dirs()`. Read this before writing auth — it is the source of truth for env-var names.
- `clawos/db.py` — `SCHEMA_SQL` covers `agents`, `sessions`, `messages` (+ `messages_fts`), `memories`, `audit_log`, `tasks`, `hive_activity`. `Database` class has `upsert_agent`, `list_agents`, `get_or_create_session`, `get_sdk_session_id`, `set_sdk_session_id`, `list_active_sessions`, `append_message`, `list_messages`, `recent_messages_global`, `add_memory`, `search_memory_fts`, `audit`, `recent_audit`. Do NOT re-write these; call them.
- `.env.example` — already lists `CLAWOS_TOKENS`, `CLAWOS_TAILSCALE_HOSTS`, `ANTHROPIC_API_KEY`, `CLAUDE_DEFAULT_MODEL`, `CLAUDE_BYPASS_PERMISSIONS`. Do not add duplicates.

**What Phase 1 adds (files listed by task):**
- `clawos/auth.py`, `clawos/queue.py`, `clawos/orchestrator.py`
- `clawos/agents/__init__.py`, `clawos/agents/claude_sdk.py`
- `clawos/routes/__init__.py`, `clawos/routes/health.py`, `clawos/routes/dashboard.py`
- `clawos/main.py`
- `templates/dashboard.html`
- `agents/mech.yaml`, `agents/sokka.yaml`, `agents/_prompts/mech.md`, `agents/_prompts/sokka.md`
- `tests/test_auth.py`, `tests/test_queue.py`, `tests/test_agents.py`, `tests/test_claude_sdk.py`, `tests/test_orchestrator.py`, `tests/test_routes.py`, `tests/conftest.py`

**File-layout decisions locked now:**
- Routes split by concern (health vs dashboard) so each file is small and the next task's additions don't bloat a single `routes.py`.
- Agent registry lives under `clawos/agents/` (package) — `__init__.py` loads YAML, bridges (`claude_sdk.py`, and later `hermes.py`) live alongside. This keeps "how we talk to an agent" colocated with "which agents exist."
- Tests mirror source layout (`tests/test_<module>.py`). No fancy structure.

**Style rules:**
- Line length 100 (per `pyproject.toml` ruff config).
- `from __future__ import annotations` at top of every new Python file.
- All I/O functions are `async`. Tests use `pytest-asyncio` auto-mode (already configured).
- No `Optional[X]` — use `X | None`.

---

## Task 1: Pre-flight — verify deps + initialize git baseline

**Files:**
- Modify: `c:\Stellaris\clawos\pyproject.toml` (add `httpx` to dev deps if needed for route testing)
- Verify: existing scaffold loads without error

- [ ] **Step 1: Sync deps with uv**

```bash
cd /c/Stellaris/clawos && uv sync --all-extras
```
Expected: deps install, no errors. If `uv` isn't installed, install it first: `pip install uv`.

- [ ] **Step 2: Verify existing config loads**

```bash
cd /c/Stellaris/clawos && uv run python -c "from clawos.config import get_settings; s = get_settings(); print('OK:', s.port, s.db_path)"
```
Expected output: `OK: 3141 <absolute-path>/data/clawos.db`

If it fails on `CLAWOS_SECRET_KEY`, that's fine — we don't need it until we open the DB.

- [ ] **Step 3: Add httpx to dev deps for route tests**

Open `pyproject.toml`. In the `[project.optional-dependencies]` `dev` list, ensure `httpx` is present (FastAPI's `TestClient` needs it). If missing, add:

```toml
dev = [
    "pytest>=8.3.0",
    "pytest-asyncio>=0.24.0",
    "ruff>=0.7.0",
    "httpx>=0.27.0",
]
```

Then: `uv sync --all-extras`.

- [ ] **Step 4: Confirm pytest runs (even with zero tests)**

```bash
cd /c/Stellaris/clawos && uv run pytest -q
```
Expected: `no tests ran in ...s` (not an error).

- [ ] **Step 5: Commit**

```bash
cd /c/Stellaris && git add clawos/pyproject.toml && git commit -m "clawos: pre-flight — add httpx dev dep for route tests"
```

(Skip the commit if `pyproject.toml` was unchanged.)

---

## Task 2: Test fixtures — shared conftest

**Files:**
- Create: `c:\Stellaris\clawos\tests\__init__.py` (empty)
- Create: `c:\Stellaris\clawos\tests\conftest.py`

- [ ] **Step 1: Create tests package marker**

Create empty file `tests/__init__.py`.

- [ ] **Step 2: Write conftest with DB + settings fixtures**

Create `tests/conftest.py`:

```python
"""Shared pytest fixtures — isolated settings + in-memory DB per test."""
from __future__ import annotations

import base64
import os
import secrets
from pathlib import Path

import pytest

from clawos.config import Settings, get_settings
from clawos.db import Database, FieldCrypto


def _fake_key() -> str:
    """Generate a urlsafe-base64 32-byte key for FieldCrypto."""
    return base64.urlsafe_b64encode(secrets.token_bytes(32)).decode("ascii")


@pytest.fixture
def test_settings(tmp_path: Path, monkeypatch: pytest.MonkeyPatch) -> Settings:
    """Return a Settings object pointing at a tmp data dir, with two test tokens."""
    monkeypatch.setenv("CLAWOS_DATA_DIR", str(tmp_path / "data"))
    monkeypatch.setenv("CLAWOS_AGENTS_DIR", str(tmp_path / "agents"))
    monkeypatch.setenv("CLAWOS_LOCK_FILE", str(tmp_path / "clawos.lock"))
    monkeypatch.setenv("CLAWOS_SECRET_KEY", _fake_key())
    monkeypatch.setenv("CLAWOS_TOKENS", "carl:carltoken123,jarod:jarodtoken456")
    monkeypatch.setenv("CLAWOS_TAILSCALE_HOSTS", "")  # disabled by default in tests
    get_settings.cache_clear()
    s = get_settings()
    s.ensure_dirs()
    return s


@pytest.fixture
async def db(test_settings: Settings) -> Database:
    """Open a fresh DB in the tmp data dir."""
    crypto = FieldCrypto(test_settings.secret_key)
    d = Database(test_settings.db_path, crypto)
    await d.open()
    yield d
    await d.close()
```

- [ ] **Step 3: Smoke-test the fixtures**

Create a throwaway test in `tests/test_fixtures_smoke.py`:

```python
"""Smoke test — conftest fixtures open a DB and load settings."""
from __future__ import annotations

from clawos.config import Settings
from clawos.db import Database


async def test_db_fixture_opens(db: Database) -> None:
    assert db.path.parent.exists()


def test_settings_tokens_parsed(test_settings: Settings) -> None:
    assert test_settings.tokens == {"carl": "carltoken123", "jarod": "jarodtoken456"}
```

- [ ] **Step 4: Run smoke tests**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_fixtures_smoke.py -v
```
Expected: `2 passed`.

- [ ] **Step 5: Commit**

```bash
cd /c/Stellaris && git add clawos/tests/ && git commit -m "clawos(tests): conftest with isolated settings + DB fixture"
```

---

## Task 3: Auth module — token + Tailscale host check

**Files:**
- Create: `c:\Stellaris\clawos\clawos\auth.py`
- Create: `c:\Stellaris\clawos\tests\test_auth.py`

- [ ] **Step 1: Write failing tests**

Create `tests/test_auth.py`:

```python
"""Auth tests — token check, hostname allowlist, audit on failure."""
from __future__ import annotations

import pytest
from fastapi import HTTPException
from starlette.requests import Request

from clawos.auth import User, verify_token
from clawos.config import Settings
from clawos.db import Database


def _req(query: str = "", host: str = "carl-basement") -> Request:
    """Build a minimal Starlette Request for direct dependency testing."""
    scope = {
        "type": "http",
        "method": "GET",
        "path": "/api/status",
        "query_string": query.encode("ascii"),
        "headers": [
            (b"host", host.encode("ascii")),
        ],
        "client": ("127.0.0.1", 1234),
    }
    return Request(scope)


async def test_valid_token_returns_user(test_settings: Settings, db: Database) -> None:
    req = _req(query="token=carltoken123")
    user = await verify_token(req, db)
    assert isinstance(user, User)
    assert user.name == "carl"
    assert user.role == "admin"


async def test_missing_token_raises_401(test_settings: Settings, db: Database) -> None:
    req = _req(query="")
    with pytest.raises(HTTPException) as exc:
        await verify_token(req, db)
    assert exc.value.status_code == 401


async def test_wrong_token_raises_401_and_audits(
    test_settings: Settings, db: Database
) -> None:
    req = _req(query="token=bogus")
    with pytest.raises(HTTPException) as exc:
        await verify_token(req, db)
    assert exc.value.status_code == 401
    audits = await db.recent_audit(limit=5)
    assert any(a["event"] == "auth_fail" for a in audits)


async def test_tailscale_allowlist_blocks_unknown_host(
    test_settings: Settings, db: Database, monkeypatch: pytest.MonkeyPatch
) -> None:
    monkeypatch.setenv("CLAWOS_TAILSCALE_HOSTS", "carl-basement,jarod-pc")
    from clawos.config import get_settings
    get_settings.cache_clear()

    req = _req(query="token=carltoken123", host="stranger-laptop")
    with pytest.raises(HTTPException) as exc:
        await verify_token(req, db)
    assert exc.value.status_code == 403


async def test_tailscale_allowlist_permits_known_host(
    test_settings: Settings, db: Database, monkeypatch: pytest.MonkeyPatch
) -> None:
    monkeypatch.setenv("CLAWOS_TAILSCALE_HOSTS", "carl-basement,jarod-pc")
    from clawos.config import get_settings
    get_settings.cache_clear()

    req = _req(query="token=jarodtoken456", host="jarod-pc")
    user = await verify_token(req, db)
    assert user.name == "jarod"
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_auth.py -v
```
Expected: fail with `ModuleNotFoundError: No module named 'clawos.auth'`.

- [ ] **Step 3: Implement `clawos/auth.py`**

Create `clawos/auth.py`:

```python
"""Token + Tailscale-host authentication.

`verify_token` is a FastAPI dependency. It resolves `?token=...` against
`CLAWOS_TOKENS`, optionally enforces `CLAWOS_TAILSCALE_HOSTS`, and writes
an `auth_fail` entry to the audit log on rejection.
"""
from __future__ import annotations

from dataclasses import dataclass

from fastapi import Depends, HTTPException, Request, status

from clawos.config import Settings, get_settings
from clawos.db import Database, get_db


@dataclass(frozen=True)
class User:
    name: str
    role: str  # "admin" for v1 (Carl + Jarod both admin)


def _host_from_request(req: Request) -> str:
    """Return the host header (lowercased, port stripped)."""
    raw = req.headers.get("host", "")
    return raw.split(":", 1)[0].strip().lower()


async def verify_token(
    request: Request,
    db: Database = Depends(get_db),
    settings: Settings = Depends(get_settings),
) -> User:
    """FastAPI dependency — returns User on success, raises HTTPException on failure."""
    token = request.query_params.get("token", "")
    tokens = settings.tokens  # {name: token}
    match = next((name for name, t in tokens.items() if t and token == t), None)

    if not match:
        await db.audit(
            event="auth_fail",
            actor=None,
            detail=f"invalid token from host={_host_from_request(request)}",
        )
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="invalid token")

    allow = settings.tailscale_hosts
    if allow:
        host = _host_from_request(request)
        if host not in allow:
            await db.audit(
                event="auth_fail",
                actor=match,
                detail=f"host not allowlisted: {host}",
            )
            raise HTTPException(status_code=status.HTTP_403_FORBIDDEN, detail="host not allowed")

    return User(name=match, role="admin")
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_auth.py -v
```
Expected: `5 passed`.

- [ ] **Step 5: Commit**

```bash
cd /c/Stellaris && git add clawos/clawos/auth.py clawos/tests/test_auth.py && \
  git commit -m "clawos: auth — token + Tailscale host check with audit on fail"
```

---

## Task 4: Agent registry — YAML loader

**Files:**
- Create: `c:\Stellaris\clawos\clawos\agents\__init__.py`
- Create: `c:\Stellaris\clawos\tests\test_agents.py`
- Create: `c:\Stellaris\clawos\agents\mech.yaml`
- Create: `c:\Stellaris\clawos\agents\sokka.yaml`
- Create: `c:\Stellaris\clawos\agents\_prompts\mech.md`
- Create: `c:\Stellaris\clawos\agents\_prompts\sokka.md`

- [ ] **Step 1: Write failing test**

Create `tests/test_agents.py`:

```python
"""Agent registry tests — YAML loader, registry operations."""
from __future__ import annotations

from pathlib import Path

import pytest

from clawos.agents import AgentDef, load_agents
from clawos.config import Settings


def _write_agent_yaml(agents_dir: Path, agent_id: str, body: str) -> None:
    agents_dir.mkdir(parents=True, exist_ok=True)
    (agents_dir / f"{agent_id}.yaml").write_text(body, encoding="utf-8")


def test_load_single_claude_agent(test_settings: Settings, tmp_path: Path) -> None:
    _write_agent_yaml(
        test_settings.agents_dir,
        "mech",
        """
id: mech
name: Mech
runtime: claude
model: claude-opus-4-6
role: CTO
is_board: true
cwd: .
system_prompt_path: _prompts/mech.md
mcp_servers: [all]
voice_preset: charon
color: "#4a9eff"
""",
    )
    (test_settings.agents_dir / "_prompts").mkdir(exist_ok=True)
    (test_settings.agents_dir / "_prompts" / "mech.md").write_text(
        "You are Mech, CTO.", encoding="utf-8"
    )

    agents = load_agents(test_settings.agents_dir)
    assert "mech" in agents
    assert isinstance(agents["mech"], AgentDef)
    assert agents["mech"].runtime == "claude"
    assert agents["mech"].model == "claude-opus-4-6"
    assert agents["mech"].is_board is True
    assert agents["mech"].system_prompt == "You are Mech, CTO."


def test_load_skips_files_with_underscore_prefix(
    test_settings: Settings,
) -> None:
    _write_agent_yaml(
        test_settings.agents_dir,
        "_template",
        "id: template\nname: Template\nruntime: claude\nmodel: x",
    )
    agents = load_agents(test_settings.agents_dir)
    assert "template" not in agents


def test_load_validates_runtime_field(test_settings: Settings) -> None:
    _write_agent_yaml(
        test_settings.agents_dir,
        "broken",
        "id: broken\nname: Broken\nruntime: openai\nmodel: x",
    )
    with pytest.raises(ValueError, match="runtime must be"):
        load_agents(test_settings.agents_dir)


def test_load_validates_id_matches_filename(test_settings: Settings) -> None:
    _write_agent_yaml(
        test_settings.agents_dir,
        "mismatched",
        "id: something_else\nname: X\nruntime: claude\nmodel: y",
    )
    with pytest.raises(ValueError, match="id.*must match filename"):
        load_agents(test_settings.agents_dir)
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_agents.py -v
```
Expected: fail with `ModuleNotFoundError`.

- [ ] **Step 3: Implement `clawos/agents/__init__.py`**

Create `clawos/agents/__init__.py`:

```python
"""Agent registry — loads `agents/*.yaml`, exposes in-memory definitions.

An AgentDef bundles the parsed YAML with the resolved system-prompt text so
bridges don't re-read files on every turn.
"""
from __future__ import annotations

import re
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any

import yaml

VALID_RUNTIMES = {"claude", "hermes"}
VALID_ID = re.compile(r"^[a-z][a-z0-9_-]{0,29}$")


@dataclass(frozen=True)
class AgentDef:
    id: str
    name: str
    runtime: str           # "claude" | "hermes"
    model: str
    role: str | None = None
    is_board: bool = False
    demoted: bool = False
    enabled: bool = True
    cwd: str = "."
    system_prompt: str = ""        # resolved text, not path
    mcp_servers: list[str] = field(default_factory=lambda: ["all"])
    voice_preset: str | None = None
    color: str = "#808080"
    raw: dict[str, Any] = field(default_factory=dict)


def _load_one(yaml_path: Path, agents_dir: Path) -> AgentDef:
    data = yaml.safe_load(yaml_path.read_text(encoding="utf-8")) or {}
    if not isinstance(data, dict):
        raise ValueError(f"{yaml_path.name}: root must be a mapping")

    aid = data.get("id", "")
    if aid != yaml_path.stem:
        raise ValueError(f"{yaml_path.name}: id={aid!r} must match filename stem")
    if not VALID_ID.match(aid):
        raise ValueError(f"{yaml_path.name}: id must match {VALID_ID.pattern}")

    runtime = data.get("runtime", "")
    if runtime not in VALID_RUNTIMES:
        raise ValueError(
            f"{yaml_path.name}: runtime must be one of {sorted(VALID_RUNTIMES)}, got {runtime!r}"
        )

    prompt_path = data.get("system_prompt_path")
    prompt_text = ""
    if prompt_path:
        resolved = (agents_dir / prompt_path).resolve()
        if resolved.exists():
            prompt_text = resolved.read_text(encoding="utf-8")

    return AgentDef(
        id=aid,
        name=data.get("name", aid),
        runtime=runtime,
        model=data.get("model", ""),
        role=data.get("role"),
        is_board=bool(data.get("is_board", False)),
        demoted=bool(data.get("demoted", False)),
        enabled=bool(data.get("enabled", True)),
        cwd=str(data.get("cwd", ".")),
        system_prompt=prompt_text,
        mcp_servers=list(data.get("mcp_servers", ["all"])),
        voice_preset=data.get("voice_preset"),
        color=str(data.get("color", "#808080")),
        raw=data,
    )


def load_agents(agents_dir: Path) -> dict[str, AgentDef]:
    """Load all `<id>.yaml` files under agents_dir (skip `_` prefixed)."""
    out: dict[str, AgentDef] = {}
    if not agents_dir.exists():
        return out
    for p in sorted(agents_dir.glob("*.yaml")):
        if p.stem.startswith("_"):
            continue
        agent = _load_one(p, agents_dir)
        if agent.enabled:
            out[agent.id] = agent
    return out


# --- Module singleton (set by main.py at startup) ---------------------------

_registry: dict[str, AgentDef] = {}


def set_registry(reg: dict[str, AgentDef]) -> None:
    global _registry
    _registry = reg


def get_registry() -> dict[str, AgentDef]:
    return _registry


def get_agent(agent_id: str) -> AgentDef:
    if agent_id not in _registry:
        raise KeyError(f"unknown agent: {agent_id}")
    return _registry[agent_id]
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_agents.py -v
```
Expected: `4 passed`.

- [ ] **Step 5: Create real Mech + Sokka agent YAMLs**

Create `agents/mech.yaml`:

```yaml
id: mech
name: Mech
runtime: claude
model: claude-opus-4-6
role: CTO — tech / infra / deployments
is_board: true
cwd: .
system_prompt_path: _prompts/mech.md
mcp_servers: [all]
voice_preset: charon
color: "#4a9eff"
```

Create `agents/sokka.yaml`:

```yaml
id: sokka
name: Sokka
runtime: claude
model: claude-haiku-4-5-20251001
role: CFO — finance / legal / budgets / ROI
is_board: true
cwd: .
system_prompt_path: _prompts/sokka.md
mcp_servers: [all]
voice_preset: alnilam
color: "#f5b642"
```

- [ ] **Step 6: Create minimal system prompts**

Create `agents/_prompts/mech.md`:

```markdown
You are Mech, CTO of Stellaris Ridge.

Scope: technology, infrastructure, deployments, SEO/AEO technicals, Hermes runtime, ClawOS, Stellaris vault tooling.

Voice: direct, technical, no fluff. When you don't know, say so. When a decision needs Carl, state it plainly.

Constraints:
- Do not handle finance, legal, pricing, or marketing strategy — defer to Sokka or Katara.
- When Carl mentions a client, read their brief at `10-stellaris-ridge/clients/<name>/brief.md` before answering.
- This is a Phase 1 minimal prompt; the full CTO persona will be co-drafted in Phase 3.
```

Create `agents/_prompts/sokka.md`:

```markdown
You are Sokka, CFO of Stellaris Ridge.

Scope: finance, legal, budgets, ROI, pricing, cost tracking, compliance, vendor contracts.

Voice: precise, numeric, risk-aware. Quote dollar figures. Flag assumptions.

Constraints:
- Do not handle technology implementation or marketing — defer to Mech or Katara.
- When Carl mentions a client, pull pricing from `10-stellaris-ridge/SR_Pricing_2026.xlsx` (or its notes) and the client brief before quoting.
- This is a Phase 1 minimal prompt; the full CFO persona will be co-drafted in Phase 3.
```

- [ ] **Step 7: Verify registry loads the real agents**

```bash
cd /c/Stellaris/clawos && uv run python -c "from clawos.agents import load_agents; from pathlib import Path; r = load_agents(Path('agents')); print(sorted(r)); print(r['mech'].model)"
```
Expected: `['mech', 'sokka']` and `claude-opus-4-6`.

- [ ] **Step 8: Commit**

```bash
cd /c/Stellaris && git add clawos/clawos/agents/ clawos/tests/test_agents.py clawos/agents/ && \
  git commit -m "clawos: agent registry — YAML loader + Mech/Sokka definitions"
```

---

## Task 5: Claude SDK bridge

**Files:**
- Create: `c:\Stellaris\clawos\clawos\agents\claude_sdk.py`
- Create: `c:\Stellaris\clawos\tests\test_claude_sdk.py`

**Note:** `claude-agent-sdk` spawns the `claude` CLI as a subprocess. For tests, we mock the SDK — actually invoking Claude during CI burns tokens and is slow. The implementation calls the real SDK; tests call it with `monkeypatch`.

- [ ] **Step 1: Write failing test**

Create `tests/test_claude_sdk.py`:

```python
"""Claude bridge tests — subprocess is mocked; we verify args + session mgmt."""
from __future__ import annotations

from pathlib import Path
from typing import Any

import pytest

from clawos.agents import AgentDef
from clawos.agents.claude_sdk import invoke_claude
from clawos.config import Settings
from clawos.db import Database


class _FakeMessage:
    def __init__(self, text: str) -> None:
        self.text = text


class _FakeResult:
    def __init__(self, session_id: str, text: str) -> None:
        self.session_id = session_id
        self.result = text
        self.num_turns = 1
        self.total_cost_usd = 0.0
        self.usage = {"input_tokens": 10, "output_tokens": 5}


async def _fake_query(**kwargs: Any):
    # Mimic the async iterator shape claude-agent-sdk yields.
    yield _FakeMessage("hello")
    yield _FakeResult(session_id="sess-abc", text="hello from mech")


@pytest.fixture
def mech_def(tmp_path: Path) -> AgentDef:
    return AgentDef(
        id="mech",
        name="Mech",
        runtime="claude",
        model="claude-opus-4-6",
        cwd=str(tmp_path),
        system_prompt="You are Mech.",
    )


async def test_invoke_claude_returns_text_and_persists_session(
    test_settings: Settings,
    db: Database,
    mech_def: AgentDef,
    monkeypatch: pytest.MonkeyPatch,
) -> None:
    import clawos.agents.claude_sdk as mod
    monkeypatch.setattr(mod, "query", _fake_query)

    session_pk = await db.get_or_create_session(
        chat_id="ui-carl", agent_id="mech", channel="dashboard"
    )
    await db.upsert_agent(
        agent_id="mech", display_name="Mech", runtime="claude", model="claude-opus-4-6",
        role="CTO", is_board=True, yaml_path=None, voice_preset=None, config={},
    )

    reply = await invoke_claude(
        agent=mech_def, message="hi", db=db, session_pk=session_pk
    )
    assert reply.text == "hello from mech"
    assert reply.tokens_in == 10
    assert reply.tokens_out == 5

    sdk_id = await db.get_sdk_session_id(session_pk)
    assert sdk_id == "sess-abc"


async def test_invoke_claude_resumes_existing_session(
    test_settings: Settings,
    db: Database,
    mech_def: AgentDef,
    monkeypatch: pytest.MonkeyPatch,
) -> None:
    """Second call should pass resume=<sdk_session_id> to query()."""
    captured: dict[str, Any] = {}

    async def _capturing_query(**kwargs: Any):
        captured.update(kwargs)
        yield _FakeResult(session_id="sess-abc-2", text="continuing")

    import clawos.agents.claude_sdk as mod
    monkeypatch.setattr(mod, "query", _capturing_query)

    session_pk = await db.get_or_create_session(
        chat_id="ui-carl", agent_id="mech", channel="dashboard"
    )
    await db.set_sdk_session_id(session_pk, "sess-abc")

    await invoke_claude(agent=mech_def, message="more", db=db, session_pk=session_pk)
    # claude-agent-sdk exposes `options.resume` on ClaudeAgentOptions.
    options = captured.get("options")
    assert options is not None
    assert getattr(options, "resume", None) == "sess-abc"
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_claude_sdk.py -v
```
Expected: fail (`ModuleNotFoundError`).

- [ ] **Step 3: Implement `clawos/agents/claude_sdk.py`**

Create `clawos/agents/claude_sdk.py`:

```python
"""Claude bridge — spawns the `claude` CLI via claude-agent-sdk.

The SDK wraps the CLI subprocess and exposes an async iterator of events.
We collect the final `ResultMessage` (which carries the session_id, token
counts, and the assistant's text) and persist state back to our DB.
"""
from __future__ import annotations

from dataclasses import dataclass

from claude_agent_sdk import ClaudeAgentOptions, query

from clawos.agents import AgentDef
from clawos.config import get_settings
from clawos.db import Database


@dataclass(frozen=True)
class ClaudeReply:
    text: str
    tokens_in: int
    tokens_out: int
    cost_usd: float
    sdk_session_id: str


async def invoke_claude(
    *,
    agent: AgentDef,
    message: str,
    db: Database,
    session_pk: int,
) -> ClaudeReply:
    """Send `message` to the Claude SDK for this agent; return reply + persist session."""
    settings = get_settings()
    prior_sdk_id = await db.get_sdk_session_id(session_pk)

    options = ClaudeAgentOptions(
        model=agent.model or settings.claude_default_model,
        cwd=agent.cwd,
        permission_mode="bypassPermissions" if settings.claude_bypass_permissions else "default",
        max_turns=settings.claude_max_turns,
        system_prompt=agent.system_prompt or None,
        resume=prior_sdk_id,
    )

    final_text = ""
    tokens_in = 0
    tokens_out = 0
    cost = 0.0
    new_session_id = prior_sdk_id or ""

    async for event in query(prompt=message, options=options):
        # SDK yields message events + a final ResultMessage. Use duck typing
        # so we don't need the SDK's internal type names.
        if getattr(event, "session_id", None) and hasattr(event, "result"):
            new_session_id = event.session_id
            final_text = getattr(event, "result", "") or ""
            usage = getattr(event, "usage", {}) or {}
            tokens_in = int(usage.get("input_tokens", 0))
            tokens_out = int(usage.get("output_tokens", 0))
            cost = float(getattr(event, "total_cost_usd", 0.0) or 0.0)

    if new_session_id and new_session_id != prior_sdk_id:
        await db.set_sdk_session_id(session_pk, new_session_id)

    return ClaudeReply(
        text=final_text,
        tokens_in=tokens_in,
        tokens_out=tokens_out,
        cost_usd=cost,
        sdk_session_id=new_session_id,
    )
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_claude_sdk.py -v
```
Expected: `2 passed`.

- [ ] **Step 5: Commit**

```bash
cd /c/Stellaris && git add clawos/clawos/agents/claude_sdk.py clawos/tests/test_claude_sdk.py && \
  git commit -m "clawos: claude bridge — SDK subprocess wrapper + session resume"
```

---

## Task 6: Message queue — per-chat async FIFO

**Files:**
- Create: `c:\Stellaris\clawos\clawos\queue.py`
- Create: `c:\Stellaris\clawos\tests\test_queue.py`

- [ ] **Step 1: Write failing test**

Create `tests/test_queue.py`:

```python
"""Message queue — one worker per chat_id serializes handling."""
from __future__ import annotations

import asyncio

import pytest

from clawos.queue import MessageQueue, QueuedMessage


async def test_messages_for_same_chat_process_serially() -> None:
    order: list[str] = []

    async def handler(msg: QueuedMessage) -> None:
        order.append(f"start-{msg.body}")
        await asyncio.sleep(0.05)
        order.append(f"end-{msg.body}")

    q = MessageQueue(handler=handler)
    await q.start()
    await q.enqueue(QueuedMessage(chat_id="A", agent_id="mech", body="1"))
    await q.enqueue(QueuedMessage(chat_id="A", agent_id="mech", body="2"))
    await q.drain()
    await q.stop()

    # Second message for same chat must not start until first ends.
    assert order == ["start-1", "end-1", "start-2", "end-2"]


async def test_messages_for_different_chats_process_concurrently() -> None:
    events: list[str] = []
    start_gate = asyncio.Event()

    async def handler(msg: QueuedMessage) -> None:
        events.append(f"start-{msg.chat_id}")
        if msg.chat_id == "A":
            await start_gate.wait()
        events.append(f"end-{msg.chat_id}")

    q = MessageQueue(handler=handler)
    await q.start()
    await q.enqueue(QueuedMessage(chat_id="A", agent_id="mech", body="a"))
    await q.enqueue(QueuedMessage(chat_id="B", agent_id="mech", body="b"))
    # Give the workers a moment to pick up.
    await asyncio.sleep(0.02)
    # B should have finished even though A is blocked.
    assert "end-B" in events
    start_gate.set()
    await q.drain()
    await q.stop()
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_queue.py -v
```
Expected: fail (`ModuleNotFoundError`).

- [ ] **Step 3: Implement `clawos/queue.py`**

Create `clawos/queue.py`:

```python
"""Per-chat_id asyncio FIFO queue.

One worker task per chat_id. Guarantees ordering within a chat while allowing
different chats to process concurrently. Handlers are awaited one at a time
per chat — they're expected to own side effects (DB write, SDK call, etc.).
"""
from __future__ import annotations

import asyncio
from collections.abc import Awaitable, Callable
from dataclasses import dataclass

Handler = Callable[["QueuedMessage"], Awaitable[None]]


@dataclass(frozen=True)
class QueuedMessage:
    chat_id: str
    agent_id: str
    body: str
    channel: str = "dashboard"


class MessageQueue:
    def __init__(self, *, handler: Handler) -> None:
        self._handler = handler
        self._queues: dict[str, asyncio.Queue[QueuedMessage]] = {}
        self._workers: dict[str, asyncio.Task[None]] = {}
        self._running = False

    async def start(self) -> None:
        self._running = True

    async def stop(self) -> None:
        self._running = False
        for t in list(self._workers.values()):
            t.cancel()
        for t in list(self._workers.values()):
            try:
                await t
            except asyncio.CancelledError:
                pass
        self._workers.clear()
        self._queues.clear()

    async def enqueue(self, msg: QueuedMessage) -> None:
        if not self._running:
            raise RuntimeError("queue not started")
        q = self._queues.get(msg.chat_id)
        if q is None:
            q = asyncio.Queue()
            self._queues[msg.chat_id] = q
            self._workers[msg.chat_id] = asyncio.create_task(self._worker(msg.chat_id, q))
        await q.put(msg)

    async def drain(self) -> None:
        """Wait until every per-chat queue is empty."""
        while self._queues:
            # Snapshot — workers may add/remove as they run.
            qs = list(self._queues.values())
            for q in qs:
                await q.join()
            # If queues weren't re-filled during the join, break.
            if all(q.empty() for q in self._queues.values()):
                break

    async def _worker(self, chat_id: str, q: asyncio.Queue[QueuedMessage]) -> None:
        try:
            while self._running:
                msg = await q.get()
                try:
                    await self._handler(msg)
                except Exception:
                    # Handler failures must not kill the worker; log in caller if needed.
                    pass
                finally:
                    q.task_done()
        except asyncio.CancelledError:
            raise
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_queue.py -v
```
Expected: `2 passed`.

- [ ] **Step 5: Commit**

```bash
cd /c/Stellaris && git add clawos/clawos/queue.py clawos/tests/test_queue.py && \
  git commit -m "clawos: per-chat asyncio FIFO message queue"
```

---

## Task 7: Orchestrator — route + persist + reply

**Files:**
- Create: `c:\Stellaris\clawos\clawos\orchestrator.py`
- Create: `c:\Stellaris\clawos\tests\test_orchestrator.py`

- [ ] **Step 1: Write failing test**

Create `tests/test_orchestrator.py`:

```python
"""Orchestrator tests — dispatches to Claude bridge, persists both sides of conversation."""
from __future__ import annotations

from pathlib import Path
from typing import Any

import pytest

from clawos.agents import AgentDef, set_registry
from clawos.agents.claude_sdk import ClaudeReply
from clawos.config import Settings
from clawos.db import Database
from clawos.orchestrator import Orchestrator


@pytest.fixture
def registry(tmp_path: Path) -> dict[str, AgentDef]:
    reg = {
        "mech": AgentDef(
            id="mech", name="Mech", runtime="claude", model="claude-opus-4-6",
            cwd=str(tmp_path), system_prompt="You are Mech.",
        ),
    }
    set_registry(reg)
    return reg


async def test_orchestrator_dispatches_claude_and_persists(
    test_settings: Settings, db: Database, registry: dict[str, AgentDef],
    monkeypatch: pytest.MonkeyPatch,
) -> None:
    await db.upsert_agent(
        agent_id="mech", display_name="Mech", runtime="claude", model="claude-opus-4-6",
        role="CTO", is_board=True, yaml_path=None, voice_preset=None, config={},
    )

    async def fake_invoke(**kwargs: Any) -> ClaudeReply:
        return ClaudeReply(
            text="reply from mech", tokens_in=4, tokens_out=7, cost_usd=0.0,
            sdk_session_id="sdk-123",
        )
    import clawos.orchestrator as mod
    monkeypatch.setattr(mod, "invoke_claude", fake_invoke)

    orch = Orchestrator(db=db)
    reply_text = await orch.handle(
        chat_id="ui-carl", agent_id="mech", body="hi mech", channel="dashboard"
    )

    assert reply_text == "reply from mech"

    # Both user msg and assistant msg persisted.
    session_pk = await db.get_or_create_session(
        chat_id="ui-carl", agent_id="mech", channel="dashboard"
    )
    msgs = await db.list_messages(session_pk=session_pk)
    assert [m.role for m in msgs] == ["user", "assistant"]
    assert msgs[0].content == "hi mech"
    assert msgs[1].content == "reply from mech"


async def test_orchestrator_rejects_unknown_agent(
    test_settings: Settings, db: Database, registry: dict[str, AgentDef],
) -> None:
    orch = Orchestrator(db=db)
    with pytest.raises(KeyError, match="unknown agent"):
        await orch.handle(chat_id="x", agent_id="ghost", body="hi", channel="dashboard")


async def test_orchestrator_raises_for_non_claude_runtime(
    test_settings: Settings, db: Database, tmp_path: Path,
) -> None:
    set_registry({
        "katara": AgentDef(
            id="katara", name="Katara", runtime="hermes", model="gemini-3.1",
            cwd=str(tmp_path),
        ),
    })
    orch = Orchestrator(db=db)
    with pytest.raises(NotImplementedError, match="hermes"):
        await orch.handle(
            chat_id="x", agent_id="katara", body="hi", channel="dashboard"
        )
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_orchestrator.py -v
```
Expected: fail (`ModuleNotFoundError`).

- [ ] **Step 3: Implement `clawos/orchestrator.py`**

Create `clawos/orchestrator.py`:

```python
"""Message orchestrator — picks a bridge based on agent.runtime.

Phase 1 handles `runtime: claude` only. `hermes` raises NotImplementedError
so the error is obvious if someone configures a Hermes agent before Phase 2
lands.
"""
from __future__ import annotations

from dataclasses import dataclass

from clawos.agents import AgentDef, get_agent
from clawos.agents.claude_sdk import ClaudeReply, invoke_claude
from clawos.db import Database


@dataclass
class Orchestrator:
    db: Database

    async def handle(
        self, *, chat_id: str, agent_id: str, body: str, channel: str
    ) -> str:
        """Persist the user message, dispatch, persist the reply, return reply text."""
        agent: AgentDef = get_agent(agent_id)

        session_pk = await self.db.get_or_create_session(
            chat_id=chat_id, agent_id=agent_id, channel=channel
        )
        await self.db.append_message(
            session_pk=session_pk, role="user", content=body
        )

        if agent.runtime == "claude":
            reply: ClaudeReply = await invoke_claude(
                agent=agent, message=body, db=self.db, session_pk=session_pk
            )
            await self.db.append_message(
                session_pk=session_pk,
                role="assistant",
                content=reply.text,
                tokens_in=reply.tokens_in,
                tokens_out=reply.tokens_out,
                cost_usd=reply.cost_usd,
            )
            return reply.text

        if agent.runtime == "hermes":
            raise NotImplementedError(
                "hermes bridge is a Phase 2 deliverable; set runtime=claude for now"
            )

        raise RuntimeError(f"unreachable runtime: {agent.runtime}")
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_orchestrator.py -v
```
Expected: `3 passed`.

- [ ] **Step 5: Commit**

```bash
cd /c/Stellaris && git add clawos/clawos/orchestrator.py clawos/tests/test_orchestrator.py && \
  git commit -m "clawos: orchestrator — dispatch to Claude bridge, persist both sides"
```

---

## Task 8: Routes package marker + health routes

**Files:**
- Create: `c:\Stellaris\clawos\clawos\routes\__init__.py` (empty)
- Create: `c:\Stellaris\clawos\clawos\routes\health.py`
- Create: `c:\Stellaris\clawos\tests\test_routes_health.py`

- [ ] **Step 1: Create routes package marker**

Create empty `clawos/routes/__init__.py`.

- [ ] **Step 2: Write failing test**

Create `tests/test_routes_health.py`:

```python
"""Health routes — no auth on /healthz, auth on /api/status."""
from __future__ import annotations

from fastapi import FastAPI
from fastapi.testclient import TestClient

from clawos.agents import AgentDef, set_registry
from clawos.config import Settings
from clawos.db import Database
from clawos.routes.health import router as health_router


def _build_app(db: Database) -> FastAPI:
    app = FastAPI()
    from clawos.db import get_db as real_get_db
    app.dependency_overrides[real_get_db] = lambda: db
    app.include_router(health_router)
    return app


def test_healthz_requires_no_auth(test_settings: Settings, db: Database) -> None:
    set_registry({})
    app = _build_app(db)
    client = TestClient(app)
    r = client.get("/healthz")
    assert r.status_code == 200
    assert r.json()["status"] == "ok"


def test_status_requires_token(test_settings: Settings, db: Database) -> None:
    set_registry({})
    app = _build_app(db)
    client = TestClient(app)
    r = client.get("/api/status")
    assert r.status_code == 401


def test_status_returns_agent_roster_with_valid_token(
    tmp_path, test_settings: Settings, db: Database,
) -> None:
    set_registry({
        "mech": AgentDef(id="mech", name="Mech", runtime="claude",
                         model="claude-opus-4-6", cwd=str(tmp_path)),
    })
    app = _build_app(db)
    client = TestClient(app)
    r = client.get("/api/status?token=carltoken123")
    assert r.status_code == 200
    data = r.json()
    assert data["user"]["name"] == "carl"
    assert any(a["id"] == "mech" for a in data["agents"])
```

- [ ] **Step 3: Run tests to verify they fail**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_routes_health.py -v
```
Expected: fail (`ModuleNotFoundError`).

- [ ] **Step 4: Implement `clawos/routes/health.py`**

Create `clawos/routes/health.py`:

```python
"""Health + status routes."""
from __future__ import annotations

from typing import Any

from fastapi import APIRouter, Depends

from clawos.agents import AgentDef, get_registry
from clawos.auth import User, verify_token

router = APIRouter()


@router.get("/healthz")
async def healthz() -> dict[str, str]:
    return {"status": "ok"}


@router.get("/api/status")
async def status(user: User = Depends(verify_token)) -> dict[str, Any]:
    agents: list[dict[str, Any]] = []
    for a in get_registry().values():
        a: AgentDef
        agents.append({
            "id": a.id, "name": a.name, "runtime": a.runtime, "model": a.model,
            "role": a.role, "is_board": a.is_board, "color": a.color,
        })
    return {
        "user": {"name": user.name, "role": user.role},
        "agents": agents,
    }
```

- [ ] **Step 5: Run tests to verify they pass**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_routes_health.py -v
```
Expected: `3 passed`.

- [ ] **Step 6: Commit**

```bash
cd /c/Stellaris && git add clawos/clawos/routes/ clawos/tests/test_routes_health.py && \
  git commit -m "clawos: routes — /healthz (unauth) + /api/status (agents + user)"
```

---

## Task 9: Dashboard routes — sessions, messages, send, SSE

**Files:**
- Create: `c:\Stellaris\clawos\clawos\routes\dashboard.py`
- Create: `c:\Stellaris\clawos\tests\test_routes_dashboard.py`

- [ ] **Step 1: Write failing test**

Create `tests/test_routes_dashboard.py`:

```python
"""Dashboard API tests — sessions list, messages fetch, send enqueue."""
from __future__ import annotations

import asyncio
from pathlib import Path

import pytest
from fastapi import FastAPI
from fastapi.testclient import TestClient

from clawos.agents import AgentDef, set_registry
from clawos.config import Settings
from clawos.db import Database
from clawos.queue import MessageQueue, QueuedMessage
from clawos.routes.dashboard import build_router


def _build_app(db: Database, queue: MessageQueue) -> FastAPI:
    app = FastAPI()
    from clawos.db import get_db as real_get_db
    app.dependency_overrides[real_get_db] = lambda: db
    app.include_router(build_router(queue))
    return app


@pytest.fixture
def registry(tmp_path: Path) -> dict[str, AgentDef]:
    reg = {
        "mech": AgentDef(id="mech", name="Mech", runtime="claude",
                         model="claude-opus-4-6", cwd=str(tmp_path)),
    }
    set_registry(reg)
    return reg


async def test_list_sessions_empty(
    test_settings: Settings, db: Database, registry: dict[str, AgentDef],
) -> None:
    q = MessageQueue(handler=lambda m: asyncio.sleep(0))
    await q.start()
    client = TestClient(_build_app(db, q))
    r = client.get("/api/sessions?token=carltoken123")
    assert r.status_code == 200
    assert r.json() == {"sessions": []}
    await q.stop()


async def test_send_enqueues_message(
    test_settings: Settings, db: Database, registry: dict[str, AgentDef],
) -> None:
    received: list[QueuedMessage] = []

    async def handler(m: QueuedMessage) -> None:
        received.append(m)

    q = MessageQueue(handler=handler)
    await q.start()
    client = TestClient(_build_app(db, q))
    r = client.post(
        "/api/send?token=carltoken123",
        json={"chat_id": "ui-carl", "agent_id": "mech", "body": "hi mech"},
    )
    assert r.status_code == 202
    await q.drain()
    assert len(received) == 1
    assert received[0].body == "hi mech"
    assert received[0].agent_id == "mech"
    await q.stop()


async def test_send_rejects_unknown_agent(
    test_settings: Settings, db: Database, registry: dict[str, AgentDef],
) -> None:
    q = MessageQueue(handler=lambda m: asyncio.sleep(0))
    await q.start()
    client = TestClient(_build_app(db, q))
    r = client.post(
        "/api/send?token=carltoken123",
        json={"chat_id": "x", "agent_id": "ghost", "body": "hi"},
    )
    assert r.status_code == 404
    await q.stop()


async def test_messages_fetch_returns_history(
    test_settings: Settings, db: Database, registry: dict[str, AgentDef],
) -> None:
    await db.upsert_agent(
        agent_id="mech", display_name="Mech", runtime="claude",
        model="claude-opus-4-6", role=None, is_board=False,
        yaml_path=None, voice_preset=None, config={},
    )
    session_pk = await db.get_or_create_session(
        chat_id="ui-carl", agent_id="mech", channel="dashboard",
    )
    await db.append_message(session_pk=session_pk, role="user", content="hi")
    await db.append_message(session_pk=session_pk, role="assistant", content="hello")

    q = MessageQueue(handler=lambda m: asyncio.sleep(0))
    await q.start()
    client = TestClient(_build_app(db, q))
    r = client.get(
        "/api/messages?token=carltoken123&chat_id=ui-carl&agent_id=mech&channel=dashboard"
    )
    assert r.status_code == 200
    msgs = r.json()["messages"]
    assert [m["role"] for m in msgs] == ["user", "assistant"]
    assert msgs[0]["content"] == "hi"
    await q.stop()
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_routes_dashboard.py -v
```
Expected: fail (`ModuleNotFoundError`).

- [ ] **Step 3: Implement `clawos/routes/dashboard.py`**

Create `clawos/routes/dashboard.py`:

```python
"""Dashboard API — sessions list, message history, send (enqueue), SSE stream."""
from __future__ import annotations

import asyncio
import json
from typing import Any

from fastapi import APIRouter, Depends, HTTPException, Request
from fastapi.responses import HTMLResponse, StreamingResponse
from pydantic import BaseModel, Field

from clawos.agents import get_registry
from clawos.auth import User, verify_token
from clawos.db import Database, get_db
from clawos.queue import MessageQueue, QueuedMessage


class SendBody(BaseModel):
    chat_id: str = Field(min_length=1, max_length=128)
    agent_id: str = Field(min_length=1, max_length=64)
    body: str = Field(min_length=1, max_length=8000)
    channel: str = Field(default="dashboard")


def build_router(queue: MessageQueue) -> APIRouter:
    """Factory — the queue singleton is created by main.py and injected here."""
    router = APIRouter()

    @router.get("/api/sessions")
    async def list_sessions(
        user: User = Depends(verify_token),
        db: Database = Depends(get_db),
    ) -> dict[str, Any]:
        sessions = await db.list_active_sessions(limit=50)
        return {"sessions": sessions}

    @router.get("/api/messages")
    async def list_messages(
        chat_id: str,
        agent_id: str,
        channel: str = "dashboard",
        user: User = Depends(verify_token),
        db: Database = Depends(get_db),
    ) -> dict[str, Any]:
        session_pk = await db.get_or_create_session(
            chat_id=chat_id, agent_id=agent_id, channel=channel,
        )
        msgs = await db.list_messages(session_pk=session_pk, limit=200)
        return {
            "messages": [
                {
                    "role": m.role,
                    "content": m.content,
                    "created_at": m.created_at,
                    "tokens_in": m.tokens_in,
                    "tokens_out": m.tokens_out,
                }
                for m in msgs
            ]
        }

    @router.post("/api/send", status_code=202)
    async def send(
        body: SendBody,
        user: User = Depends(verify_token),
        db: Database = Depends(get_db),
    ) -> dict[str, Any]:
        if body.agent_id not in get_registry():
            raise HTTPException(status_code=404, detail=f"unknown agent: {body.agent_id}")
        await db.audit(
            event="message",
            actor=user.name,
            chat_id=body.chat_id,
            detail=f"to={body.agent_id} channel={body.channel}",
        )
        await queue.enqueue(QueuedMessage(
            chat_id=body.chat_id,
            agent_id=body.agent_id,
            body=body.body,
            channel=body.channel,
        ))
        return {"accepted": True}

    @router.get("/api/events")
    async def events_stream(
        request: Request,
        user: User = Depends(verify_token),
        db: Database = Depends(get_db),
    ) -> StreamingResponse:
        async def _gen():
            last_pk = 0
            while True:
                if await request.is_disconnected():
                    return
                recent = await db.recent_messages_global(limit=50)
                new_msgs = [m for m in reversed(recent) if m["message_pk"] > last_pk]
                for m in new_msgs:
                    last_pk = max(last_pk, m["message_pk"])
                    yield f"data: {json.dumps(m, default=str)}\n\n"
                await asyncio.sleep(1.0)

        return StreamingResponse(_gen(), media_type="text/event-stream")

    return router
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_routes_dashboard.py -v
```
Expected: `4 passed`.

- [ ] **Step 5: Commit**

```bash
cd /c/Stellaris && git add clawos/clawos/routes/dashboard.py clawos/tests/test_routes_dashboard.py && \
  git commit -m "clawos: routes — sessions/messages/send/SSE with token auth"
```

---

## Task 10: Dashboard HTML template

**Files:**
- Create: `c:\Stellaris\clawos\templates\dashboard.html`

No test — this is the UI. We verify it manually in the acceptance test (Task 12).

- [ ] **Step 1: Write the template**

Create `templates/dashboard.html`:

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>ClawOS</title>
<script defer src="https://unpkg.com/alpinejs@3.14.1/dist/cdn.min.js"></script>
<style>
  :root {
    --bg: #0b0d12;
    --panel: #131722;
    --border: #1f2430;
    --text: #e3e6eb;
    --muted: #8a91a2;
    --accent: #4a9eff;
    --accent-2: #f5b642;
  }
  * { box-sizing: border-box; }
  html, body { margin: 0; height: 100%; background: var(--bg); color: var(--text);
               font: 14px/1.5 -apple-system,Segoe UI,Roboto,Inter,sans-serif; }
  .app { display: grid; grid-template-columns: 260px 1fr; height: 100vh; }
  .sidebar { background: var(--panel); border-right: 1px solid var(--border);
             display: flex; flex-direction: column; }
  .sidebar h1 { margin: 16px; font-size: 15px; letter-spacing: .2em;
                color: var(--muted); text-transform: uppercase; }
  .agent { padding: 10px 16px; cursor: pointer; border-left: 3px solid transparent; }
  .agent.active { border-left-color: var(--accent); background: #1a1f2c; }
  .agent .name { font-weight: 600; }
  .agent .role { color: var(--muted); font-size: 12px; }
  .main { display: flex; flex-direction: column; min-width: 0; }
  .transcript { flex: 1; overflow-y: auto; padding: 20px; }
  .msg { margin-bottom: 16px; max-width: 820px; }
  .msg .who { font-size: 12px; color: var(--muted); margin-bottom: 4px; }
  .msg .body { background: var(--panel); padding: 10px 14px; border-radius: 10px;
               white-space: pre-wrap; word-break: break-word; }
  .msg.user .body { background: #1c2437; }
  .composer { border-top: 1px solid var(--border); padding: 12px;
              display: flex; gap: 8px; background: var(--panel); }
  .composer textarea { flex: 1; background: #0d1018; color: var(--text);
                       border: 1px solid var(--border); border-radius: 8px;
                       padding: 10px; resize: vertical; min-height: 56px; }
  .composer button { background: var(--accent); color: #0b0d12; border: 0;
                     border-radius: 8px; padding: 10px 16px; font-weight: 600;
                     cursor: pointer; }
  .composer button:disabled { opacity: .5; cursor: not-allowed; }
  .topbar { padding: 10px 20px; border-bottom: 1px solid var(--border);
            display: flex; justify-content: space-between; align-items: center; }
  .topbar .status { color: var(--muted); font-size: 12px; }
  .err { color: #ff6b6b; font-size: 12px; }
</style>
</head>
<body>
<div class="app"
     x-data="clawos()"
     x-init="init()">

  <aside class="sidebar">
    <h1>ClawOS</h1>
    <template x-for="a in agents" :key="a.id">
      <div class="agent" :class="{active: selected===a.id}"
           @click="select(a.id)">
        <div class="name" x-text="a.name"></div>
        <div class="role" x-text="a.role || a.model"></div>
      </div>
    </template>
    <div style="flex:1"></div>
    <div class="agent" style="color: var(--muted);">
      <div class="role" x-text="user ? 'signed in: ' + user.name : ''"></div>
      <div class="err" x-text="error"></div>
    </div>
  </aside>

  <section class="main">
    <header class="topbar">
      <div x-text="selected ? agentName(selected) : 'pick an agent'"></div>
      <div class="status" x-text="statusLine"></div>
    </header>

    <div class="transcript" x-ref="scroll">
      <template x-for="(m, i) in messages" :key="i">
        <div class="msg" :class="m.role">
          <div class="who" x-text="m.role"></div>
          <div class="body" x-text="m.content"></div>
        </div>
      </template>
    </div>

    <form class="composer" @submit.prevent="send">
      <textarea x-model="draft" :disabled="!selected || sending"
                placeholder="message..." @keydown.enter.prevent="send"></textarea>
      <button type="submit" :disabled="!draft.trim() || sending">send</button>
    </form>
  </section>
</div>

<script>
function clawos() {
  const params = new URLSearchParams(location.search);
  const token = params.get("token") || "";
  const chatId = "ui-" + (params.get("who") || "carl");
  return {
    token, chatId, user: null, agents: [], selected: null,
    messages: [], draft: "", sending: false, statusLine: "",
    error: "",

    async init() {
      if (!token) { this.error = "missing ?token="; return; }
      try {
        const r = await fetch("/api/status?token=" + encodeURIComponent(token));
        if (!r.ok) { this.error = "auth failed: " + r.status; return; }
        const j = await r.json();
        this.user = j.user;
        this.agents = j.agents;
        if (this.agents.length) this.select(this.agents[0].id);
        this.openStream();
      } catch (e) { this.error = String(e); }
    },

    agentName(id) {
      const a = this.agents.find(x => x.id === id);
      return a ? a.name : id;
    },

    async select(id) {
      this.selected = id;
      this.messages = [];
      const r = await fetch(
        `/api/messages?token=${encodeURIComponent(this.token)}` +
        `&chat_id=${encodeURIComponent(this.chatId)}` +
        `&agent_id=${encodeURIComponent(id)}&channel=dashboard`
      );
      if (r.ok) {
        const j = await r.json();
        this.messages = j.messages;
        this.$nextTick(() => this.scrollToEnd());
      }
    },

    async send() {
      const body = this.draft.trim();
      if (!body || !this.selected) return;
      this.sending = true;
      this.statusLine = "sending...";
      this.messages.push({role: "user", content: body});
      this.draft = "";
      this.$nextTick(() => this.scrollToEnd());
      try {
        const r = await fetch("/api/send?token=" + encodeURIComponent(this.token), {
          method: "POST",
          headers: {"content-type": "application/json"},
          body: JSON.stringify({
            chat_id: this.chatId,
            agent_id: this.selected,
            body,
          }),
        });
        if (!r.ok) {
          const t = await r.text();
          this.error = "send failed: " + r.status + " " + t;
        } else {
          this.statusLine = "queued — waiting for reply";
        }
      } catch (e) { this.error = String(e); }
      this.sending = false;
    },

    openStream() {
      const es = new EventSource(
        "/api/events?token=" + encodeURIComponent(this.token)
      );
      es.onmessage = (ev) => {
        try {
          const m = JSON.parse(ev.data);
          // Only show messages for the currently-selected agent+chat.
          if (m.chat_id !== this.chatId) return;
          if (m.agent_id !== this.selected) return;
          // Avoid dup when we optimistically appended the user msg.
          if (m.role === "user" && this.messages.some(
              x => x.role === "user" && x.content === m.content)) return;
          this.messages.push({role: m.role, content: m.content});
          this.statusLine = m.role === "assistant" ? "" : this.statusLine;
          this.$nextTick(() => this.scrollToEnd());
        } catch (e) { /* ignore non-JSON keep-alives */ }
      };
      es.onerror = () => { this.statusLine = "stream disconnected"; };
    },

    scrollToEnd() {
      const el = this.$refs.scroll;
      if (el) el.scrollTop = el.scrollHeight;
    },
  };
}
</script>
</body>
</html>
```

- [ ] **Step 2: Commit**

```bash
cd /c/Stellaris && git add clawos/templates/dashboard.html && \
  git commit -m "clawos: embedded dashboard HTML (Alpine.js, SSE, 3-pane)"
```

---

## Task 11: Main FastAPI app — wire everything

**Files:**
- Create: `c:\Stellaris\clawos\clawos\main.py`
- Create: `c:\Stellaris\clawos\tests\test_main.py`

- [ ] **Step 1: Write failing test**

Create `tests/test_main.py`:

```python
"""Main app smoke test — /healthz works, dashboard HTML served at /."""
from __future__ import annotations

from fastapi.testclient import TestClient

from clawos.config import Settings
from clawos.main import create_app


def test_root_serves_dashboard_html(test_settings: Settings) -> None:
    app = create_app()
    # TestClient runs startup + shutdown (FastAPI lifespan).
    with TestClient(app) as client:
        r = client.get("/")
        assert r.status_code == 200
        assert "ClawOS" in r.text
        assert "alpinejs" in r.text.lower()


def test_healthz_works_end_to_end(test_settings: Settings) -> None:
    app = create_app()
    with TestClient(app) as client:
        r = client.get("/healthz")
        assert r.status_code == 200
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_main.py -v
```
Expected: fail (`ModuleNotFoundError: No module named 'clawos.main'`).

- [ ] **Step 3: Implement `clawos/main.py`**

Create `clawos/main.py`:

```python
"""FastAPI app factory + uvicorn entry.

Wires: DB open → load agents → register in DB → build queue → mount routes.
"""
from __future__ import annotations

from collections.abc import AsyncIterator
from contextlib import asynccontextmanager
from pathlib import Path

from fastapi import FastAPI
from fastapi.responses import HTMLResponse

from clawos.agents import load_agents, set_registry
from clawos.config import get_settings
from clawos.db import Database, FieldCrypto, get_db
from clawos.orchestrator import Orchestrator
from clawos.queue import MessageQueue, QueuedMessage
from clawos.routes.dashboard import build_router as build_dashboard_router
from clawos.routes.health import router as health_router

TEMPLATES_DIR = Path(__file__).resolve().parent.parent / "templates"


@asynccontextmanager
async def _lifespan(app: FastAPI) -> AsyncIterator[None]:
    settings = get_settings()
    settings.ensure_dirs()

    # DB
    crypto = FieldCrypto(settings.secret_key)
    db = Database(settings.db_path, crypto)
    await db.open()

    # Expose db to FastAPI deps (override the module-level `get_db`)
    import clawos.db as db_mod
    db_mod._db = db  # lifespan_db isn't used here because we own the lifecycle

    # Agents
    registry = load_agents(settings.agents_dir)
    set_registry(registry)
    for a in registry.values():
        await db.upsert_agent(
            agent_id=a.id, display_name=a.name, runtime=a.runtime,
            model=a.model, role=a.role, is_board=a.is_board,
            yaml_path=None, voice_preset=a.voice_preset, config=a.raw,
        )

    # Orchestrator + queue
    orch = Orchestrator(db=db)

    async def handle(msg: QueuedMessage) -> None:
        await orch.handle(
            chat_id=msg.chat_id, agent_id=msg.agent_id,
            body=msg.body, channel=msg.channel,
        )

    queue = MessageQueue(handler=handle)
    await queue.start()
    app.state.queue = queue

    # Mount routes that need the queue
    app.include_router(build_dashboard_router(queue))

    try:
        yield
    finally:
        await queue.stop()
        await db.close()
        db_mod._db = None


def create_app() -> FastAPI:
    app = FastAPI(title="ClawOS", version="0.1.0", lifespan=_lifespan)

    # Static routes that don't need lifespan-built deps
    app.include_router(health_router)

    @app.get("/", response_class=HTMLResponse)
    async def root() -> HTMLResponse:
        html_path = TEMPLATES_DIR / "dashboard.html"
        return HTMLResponse(html_path.read_text(encoding="utf-8"))

    return app


app = create_app()


def main() -> None:
    """Entry for `uv run clawos` if we add a script; else use uvicorn directly."""
    import uvicorn
    settings = get_settings()
    uvicorn.run(
        "clawos.main:app",
        host=settings.host, port=settings.port,
        reload=settings.env != "production",
    )
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
cd /c/Stellaris/clawos && uv run pytest tests/test_main.py -v
```
Expected: `2 passed`.

- [ ] **Step 5: Run the full test suite — everything must pass**

```bash
cd /c/Stellaris/clawos && uv run pytest -v
```
Expected: all tests from Tasks 2–11 pass.

- [ ] **Step 6: Commit**

```bash
cd /c/Stellaris && git add clawos/clawos/main.py clawos/tests/test_main.py && \
  git commit -m "clawos: main FastAPI app — lifespan wires DB/agents/queue/routes"
```

---

## Task 12: Acceptance test (manual, end-to-end)

No new files — this exercises the full stack against a real Claude subprocess. The run consumes real Claude Max tokens.

**Pre-reqs:**
- `CLAWOS_SECRET_KEY` populated (gen with `python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"`)
- `CLAWOS_TOKENS=carl:<long-random>,jarod:<long-random>`
- `claude` CLI installed and logged in (verify with `claude --version` from Bash; the SDK subprocess uses the same auth)

- [ ] **Step 1: Fill `.env`**

Copy `.env.example` → `.env`, set `CLAWOS_SECRET_KEY` and `CLAWOS_TOKENS`. Leave Tailscale empty for local test.

- [ ] **Step 2: Launch**

```bash
cd /c/Stellaris/clawos && uv run uvicorn clawos.main:app --host 0.0.0.0 --port 3141
```

Expected: uvicorn logs `Application startup complete.` — no stack trace.

- [ ] **Step 3: Basement browser check**

Open `http://localhost:3141/?token=<carl-token>` in any browser on the same machine.

Expected:
- Dashboard loads, dark theme.
- Sidebar shows Mech + Sokka.
- Topbar shows "signed in: carl".
- No red error text.

- [ ] **Step 4: Upstairs laptop check**

From upstairs laptop on the same LAN: `http://<basement-ip>:3141/?token=<carl-token>`.

Expected: same dashboard state as basement.

(If the request times out, it's Windows Firewall. Add an inbound rule for port 3141 or run `New-NetFirewallRule -DisplayName "ClawOS" -Direction Inbound -Protocol TCP -LocalPort 3141 -Action Allow` in an elevated PowerShell.)

- [ ] **Step 5: Round-trip with Mech**

1. Pick Mech in the sidebar.
2. Type: `Hi Mech — what's your role?`
3. Press send.
4. Within ~10s a reply appears.

Expected: the reply is context-appropriate (mentions CTO / technology). No error banner.

- [ ] **Step 6: Persistence across restart**

1. Ctrl+C the uvicorn process.
2. `uv run uvicorn clawos.main:app --host 0.0.0.0 --port 3141` again.
3. Reload the dashboard in the browser.

Expected: prior transcript with Mech is still visible. Send one more message — the reply references the earlier conversation (proves session resumption worked).

- [ ] **Step 7: Auth fail audit**

Open `http://localhost:3141/?token=wrong` — expect a blank dashboard with "auth failed: 401" in the sidebar error line.

Verify the audit record was written:

```bash
cd /c/Stellaris/clawos && uv run python -c "import asyncio; from clawos.config import get_settings; from clawos.db import Database, FieldCrypto; \
s = get_settings(); \
async def _r(): \
    d = Database(s.db_path, FieldCrypto(s.secret_key)); await d.open(); \
    rows = await d.recent_audit(limit=5); \
    for r in rows: print(r['event'], r['actor'], r['detail']); \
    await d.close(); \
asyncio.run(_r())"
```

Expected: at least one row with `event=auth_fail`.

- [ ] **Step 8: Jarod token works, audit records Jarod**

Open `http://localhost:3141/?token=<jarod-token>` → dashboard loads. Send a message → reply arrives.

Re-run the audit dump from Step 7. Expected: most recent `message` audit row has `actor=jarod`.

- [ ] **Step 9: Update README**

Open `clawos/README.md`. Flip:

```markdown
- [x] Phase 1 — Core pipeline & dashboard
```

This should already be checked in the existing README — confirm no new checkbox edits needed. If the README's acceptance claim was aspirational (it was), add a one-line note: `*Verified end-to-end on 2026-04-XX — basement + upstairs laptop.*` under the checklist.

- [ ] **Step 10: Commit acceptance note**

```bash
cd /c/Stellaris && git add clawos/README.md && \
  git commit -m "clawos: Phase 1 acceptance test passed — dashboard reachable, sessions persist"
```

---

## Self-review

Spec coverage (Section 8 of the design spec, Phase 1 row):
- [x] `main.py` — Task 11
- [x] `auth.py` — Task 3
- [x] `queue.py` — Task 6
- [x] `orchestrator.py` — Task 7
- [x] `agents/__init__.py` — Task 4
- [x] `agents/claude_sdk.py` — Task 5
- [x] `routes/dashboard.py` — Task 9
- [x] `routes/health.py` — Task 8
- [x] `templates/dashboard.html` — Task 10
- [x] `agents/mech.yaml`, `sokka.yaml`, `_prompts/*.md` — Task 4
- [x] `db.py` — already existed, verified adequate in Task 1 pre-flight
- [x] `.env.example` — already lists required keys; Task 12 consumes them

Spec acceptance test (Section 8):
- [x] Steps 1–8 mapped to Task 12 Steps 2–8. Tailscale test deferred to Phase 2 kickoff (spec says "NOT tested in Phase 1 acceptance").

Type consistency: `ClaudeReply`, `AgentDef`, `QueuedMessage`, `User` all defined once, consumed consistently. `get_registry`/`set_registry`/`get_agent` used everywhere the registry is referenced.

No placeholders, no "similar to Task N", every code step ships complete code.

---

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-04-14-clawos-phase1.md`. Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints.

**Which approach?**
