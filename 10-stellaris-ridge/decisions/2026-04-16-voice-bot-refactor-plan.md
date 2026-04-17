---
date: 2026-04-16
type: implementation-plan
pairs-with: 2026-04-16-sr-voice-elevenlabs-pivot.md
branch: refactor/elevenlabs-source-of-truth
worktree: C:/Users/ctrep/Documents/Anti-Gravity/Website/voice-bot-refactor
repo: github.com/Blaez-13/vox
---

# Voice-Bot Refactor — Implementation Plan

> **For agents executing this plan:** Work only in the worktree at `C:/Users/ctrep/Documents/Anti-Gravity/Website/voice-bot-refactor`. Do NOT touch the main checkout at `Website/voice-bot`. All commits go to the `refactor/elevenlabs-source-of-truth` branch. Use the checkbox syntax `- [ ]` so progress is trackable. Commit frequently — every logical chunk.

**Goal:** Refactor the existing voice-bot platform so ElevenLabs is the source of truth for agent configuration, Firestore holds only SR business state, and the architects' behavioral recommendations (vertical templates in Git, fragment library, testing gate, monitoring) are wired in.

**Architecture:** Cloud Run + Firestore + ElevenLabs + Twilio + GCP Secret Manager + GitHub + Better Stack. No Supabase, no Vercel, no Retool, no n8n/Zapier. One webhook service.

**Tech stack:** Node.js (Express), Google Firestore, Firebase Admin SDK, Google Secret Manager, vanilla HTML/JS frontend.

---

## Phase 2 — Parallel Tasks (4 agents concurrent, ~30–45 min)

### Task A — Create `sr-voice-templates` repo structure

**Scope:** New Git repo with the vertical template + fragment library architecture. Plumber as the canonical v1 template. Other verticals are folder stubs (populated in later iterations).

**Location:** `C:/Users/ctrep/Documents/Anti-Gravity/sr-voice-templates/` (NEW — sibling of voice-bot-refactor worktree)

**Deliverables:**

- [ ] **A.1** Create directory structure:
  ```
  sr-voice-templates/
    .gitignore
    README.md
    fragments/
      00-datetime-anchor.md
      01-persona-warm-direct.md
      02-tool-discipline.md
      03-silence-handling.md
      04-off-topic.md
      05-escalation.md
      06-pii-discipline.md
      07-output-contract.md
      08-emergency-override.md
      09-no-pricing-policy.md
    tools/
      get_free_slots.json
      book_appointment.json
    templates/
      plumber/v1/
        workflow.yaml
        prompts/
          triage.md
          emergency.md
          booking.md
          support.md
          billing.md
        kb/
          01-service-area.md
          02-hours-and-after-hours.md
          03-pricing-policy.md
          04-emergency-criteria.md
          05-services-offered.md
          06-common-faqs.md
          07-escalation-contacts.md
        tests/
          simulation/
            01-flooding-emergency.yaml
            02-clog-booking.yaml
            03-billing-inquiry.yaml
            04-refuses-to-describe.yaml
            05-spanish-speaker.yaml
            06-upset-caller.yaml
            07-misframed-emergency.yaml
            08-empty-slots.yaml
            09-tool-failure.yaml
            10-pricing-question.yaml
          tool/ (empty stub)
          llm/ (empty stub)
        intake-form.json
        README.md
      dental/v1/README.md (stub: "TODO: populate in v2")
      hvac/v1/README.md (stub)
      legal/v1/README.md (stub)
      med-spa/v1/README.md (stub)
    deploy/
      stamp.py
      smoke.py
      monitor.py
      README.md
  ```

- [ ] **A.2** Seed `fragments/*.md` with concrete prompt components. Content per fragment comes from decision doc D7 + AI-engineer recommendations. Each fragment is a composable markdown block an assembler will concatenate.

- [ ] **A.3** Seed `tools/get_free_slots.json` and `tools/book_appointment.json` with the tool schemas ported from existing voice-bot code (`server.js` tool definitions around the `/tools/*` routes).

- [ ] **A.4** Seed `templates/plumber/v1/prompts/triage.md` using the structure from the AI engineer's Section 2 (identity/scope, datetime anchor, routing taxonomy, emergency overrides, qualifying questions, silence handling, off-topic, output contract).

- [ ] **A.5** Seed `templates/plumber/v1/kb/*.md` with `{{placeholder}}` markers for client-specific content (service area, hours, etc).

- [ ] **A.6** Seed `templates/plumber/v1/tests/simulation/*.yaml` — one YAML file per scenario from decision doc D5. Format: simulated caller turns + expected agent behavior (triage label, tool invoked, transfer fired, etc).

- [ ] **A.7** Write `templates/plumber/v1/intake-form.json` — JSON schema of fields the client fills during onboarding (service area zips, hours, emergency phone, pricing policy, etc).

- [ ] **A.8** Write stubs for `deploy/stamp.py`, `smoke.py`, `monitor.py` with docstrings describing purpose. Implementation stays stubs for now — Phase 3 Task H wires them up.

- [ ] **A.9** Write `README.md` documenting: repo purpose, layout, how to add a new vertical, how to compose a per-client agent from template + intake-form.

- [ ] **A.10** `git init`, initial commit, add remote (user will create GitHub repo later — leave `git remote add origin ...` as a TODO in README).

**Does not modify the voice-bot-refactor worktree. Pure new-repo scaffolding.**

**Success criteria:**
- Directory structure matches spec exactly
- Every fragment file has real content (no TBD)
- Plumber v1 is complete enough that someone could read it and understand what a deployed plumber agent would do
- README explains the "compose + stamp" onboarding flow

---

### Task B — Remove Vapi legacy code

**Scope:** Migration to ElevenLabs completed 2026-04-09. Vapi fallback is no longer needed. Remove code, retain a git tag for rollback safety.

**Location:** `C:/Users/ctrep/Documents/Anti-Gravity/Website/voice-bot-refactor/`

**Known Vapi references from audit:**
- `schema.js` lines 178–183 — `integrations.vapi` object
- `server.js` line ~1270 — `/api/admin/sync-all-vapi` endpoint
- `server.js` line ~1304 — `/api/admin/bulk-provision-vapi` endpoint
- `server.js` line ~2247 — `POST /webhook/vapi` handler (~90 lines)
- `client-registry.js` — Vapi assistant ID lookup `byAssistant`
- `merge.js` — cloning logic includes Vapi config
- `migrate-to-firestore.js` — historical migration script (leave alone — it's historical)
- `.env.example` — any `VAPI_*` keys

**Deliverables:**

- [ ] **B.1** Before touching anything: `git tag pre-vapi-removal` on main (in main repo, not worktree) so rollback is one command.

- [ ] **B.2** `grep -rn -i "vapi" --include="*.js" --include="*.json" --include="*.html" --include="*.env.example"` — produce complete reference list. Cross-check against known list above.

- [ ] **B.3** Remove `integrations.vapi` from `schema.js`. Update any schema validators.

- [ ] **B.4** Remove the two `/api/admin/*-vapi` endpoints from `server.js`.

- [ ] **B.5** Remove `POST /webhook/vapi` handler and helper functions from `server.js`.

- [ ] **B.6** Remove `byAssistant` (or Vapi-specific half) from `client-registry.js`.

- [ ] **B.7** Remove Vapi cloning keys from `merge.js`.

- [ ] **B.8** Remove `VAPI_*` env vars from `.env.example`.

- [ ] **B.9** Leave `migrate-to-firestore.js` alone — historical artifact, not runtime.

- [ ] **B.10** Verify app still starts: `cd voice-bot-refactor && node -e "require('./server.js')"` should not throw. (If server.js auto-starts on require, check for a syntax/import error instead via `node --check server.js`.)

- [ ] **B.11** Commit: `git commit -m "Remove Vapi legacy code (migration completed 2026-04-09)"`

**Success criteria:**
- `grep -rni "vapi" .` returns only `migrate-to-firestore.js` matches and any README/docs
- `node --check server.js` passes
- Existing ElevenLabs tests (if any) still pass
- Single clean commit

---

### Task C — Monitoring & correlation IDs

**Scope:** Add structured logging with correlation IDs throughout the request lifecycle. Prepare Better Stack integration (Carl completes SaaS signup separately). Establish the alerting threshold config in code so it's versioned.

**Location:** `C:/Users/ctrep/Documents/Anti-Gravity/Website/voice-bot-refactor/`

**Deliverables:**

- [ ] **C.1** Add `pino` or `winston` to `package.json` for structured JSON logging. Prefer `pino` (faster, simpler).

- [ ] **C.2** Create `lib/logger.js` — exports a logger instance configured for JSON output. Log level from `LOG_LEVEL` env var (default `info`).

- [ ] **C.3** Create `middleware/correlation.js` — Express middleware that:
  - Reads `X-SR-Call-ID` from incoming requests, or generates a new UUID if missing
  - Attaches to `req.correlationId`
  - Adds to every log line via logger child context
  - Echoes back as response header

- [ ] **C.4** Wire `correlation.js` middleware into `server.js` early in the middleware chain.

- [ ] **C.5** Replace `console.log` / `console.error` with logger calls in the HOT PATHS only (tool webhooks, ElevenLabs webhooks, GHL calls). Leave other `console.log` for Phase 3 cleanup. Scope: `server.js` handlers for `/webhook/elevenlabs/*`, `/tools/*`, `/api/webhooks/*`.

- [ ] **C.6** Create `config/alert-thresholds.js` — exports the threshold matrix from decision doc D6:
  ```javascript
  module.exports = {
    triage_accuracy: { warn: 0.92, page: 0.85 },
    emergency_misroute: { warn: 1, page: 2 },
    tool_success_rate: { warn: 0.95, page: 0.90 },
    booking_completion_rate: { warn: 0.70, page: 0.50 },
    negative_sentiment_rate: { warn: 0.15, page: 0.25 }
  };
  ```

- [ ] **C.7** Create `docs/MONITORING.md` — documents: Better Stack signup steps (Carl runs), env vars needed (`BETTERSTACK_SOURCE_TOKEN`), log shipping setup, alert rule definitions matching thresholds, on-call escalation flow.

- [ ] **C.8** Add placeholder for Better Stack log shipping in `lib/logger.js` (commented out, activated when env var is set).

- [ ] **C.9** Commit: `git commit -m "Add structured logging, correlation IDs, and Better Stack config"`

**Success criteria:**
- Server starts clean (`node --check server.js`)
- Every hot-path request produces JSON log lines with correlation ID
- `MONITORING.md` gives Carl a clear runbook
- Alert thresholds are code, not prose

---

### Task D — ElevenLabs post-call analysis integration

**Scope:** Configure per-agent post-call analysis on provisioning. Add webhook endpoint to receive analysis results. Write to Firestore `call_logs` subcollection enriched with analysis fields.

**Location:** `C:/Users/ctrep/Documents/Anti-Gravity/Website/voice-bot-refactor/`

**Background:** ElevenLabs supports per-agent `conversation_analysis` config in agent creation/update. On call end, ElevenLabs runs analysis, then fires post-call webhook with results. Current code has a webhook handler but may not capture analysis fields. This task: ensure analysis is configured AND captured.

**Deliverables:**

- [ ] **D.1** Read existing ElevenLabs provisioning code in `server.js` (route: `/api/clients/:id/provision-elevenlabs`). Understand current `agent.conversation_config` structure.

- [ ] **D.2** Check ElevenLabs API docs for `conversation_analysis.evaluation_criteria` and `conversation_analysis.data_collection` schema. Build a reference structure matching decision doc D7 metrics:
  - triage_intent (data collection, string)
  - triage_correct (LLM judge, boolean)
  - tool_calls_attempted / tool_calls_succeeded (counts)
  - escalated_to_human (boolean + reason string)
  - emergency_keywords_present (boolean)
  - emergency_routed_correctly (boolean)
  - caller_sentiment (categorical)
  - call_completed_successfully (boolean)

- [ ] **D.3** Create `lib/elevenlabs-analysis-config.js` — exports the standard analysis config object that all SR agents share. Parameterize by vertical where criteria differ (dental uses different emergency keywords than plumber).

- [ ] **D.4** Update provisioning function to include `conversation_analysis` in the agent config.

- [ ] **D.5** Locate existing post-call webhook handler (`/webhook/elevenlabs/post-call` or similar). Update to extract analysis fields from the payload and write to Firestore:
  - New subcollection: `clients/{slug}/call_analysis/{conversation_id}`
  - Or new fields on existing `call_logs/{call_id}` document

- [ ] **D.6** Add HMAC signature verification on the webhook if not already present. ElevenLabs signs post-call webhooks — verify before writing.

- [ ] **D.7** Create nightly reconciliation stub `lib/analysis-reconciliation.js` — reads last 24 hours of ElevenLabs conversations via REST API, compares against Firestore writes, fills gaps. Scheduled run via Cloud Scheduler (Carl wires later).

- [ ] **D.8** Add unit tests for the analysis config builder and the webhook handler (verify signature check, field extraction).

- [ ] **D.9** Update `docs/ELEVENLABS-ANALYSIS.md` with schema, webhook payload example, reconciliation procedure.

- [ ] **D.10** Commit: `git commit -m "Add ElevenLabs post-call analysis config and webhook capture"`

**Success criteria:**
- Provisioning creates agents with analysis config included
- Post-call webhook writes analysis fields to Firestore
- Signature verification works
- Documentation covers schema + reconciliation

---

## Phase 3 — Sequential Tasks (4 agents ordered, ~2 hrs)

### Task E — Split `server.js` into modules

**Scope:** 2,699 lines in one file. Split into focused modules without changing runtime behavior. Smoke test after split.

**Depends on:** B (Vapi removal) complete — don't split code that's about to be deleted.

**Deliverables:**

- [ ] **E.1** Read `server.js`. Identify logical groupings: routes (admin, tools, webhooks, portal, public), integrations (ghl, elevenlabs, twilio, postmark, gemini), middleware (auth, correlation, error), helpers (schema builders, validators, prompt generators).

- [ ] **E.2** Create directory structure:
  ```
  voice-bot-refactor/
    server.js  (now thin — bootstrap only, ~100 lines)
    routes/
      admin/
        clients.js
        agencies.js
        events.js
        members.js
        diagnostics.js
      tools/
        calendar.js
      webhooks/
        elevenlabs.js
        gemini.js
      portal/
        auth.js
        clients.js
      public/
        sites.js
        widget.js
    integrations/
      ghl.js
      elevenlabs.js
      twilio.js
      postmark.js
      gemini.js
    middleware/
      auth.js
      correlation.js  (from Task C)
      error.js
    lib/
      logger.js  (from Task C)
      schema.js  (move existing schema.js here)
      client-registry.js  (existing, move)
      prompts.js
  ```

- [ ] **E.3** Move code in groups, one commit per logical group. Each commit = working server.

- [ ] **E.4** After each group move: `node --check server.js` + run existing tests (if any). If tests fail, fix before next move.

- [ ] **E.5** Final `server.js` is the Express app bootstrap only — imports routes, middleware, starts listener.

- [ ] **E.6** Verify: `node server.js` starts without error. Hit `/health` — should return 200.

- [ ] **E.7** Final commit: `git commit -m "Complete server.js split into focused modules"`

**Success criteria:**
- `server.js` is under ~150 lines
- No file in `routes/` exceeds ~400 lines
- All existing endpoints respond exactly as before
- No behavioral changes (this is a pure restructure)

---

### Task F — Refactor Firestore schema (strip duplicates)

**Scope:** Remove fields from Firestore that duplicate ElevenLabs state. Write migration script. Update all read-sites to fetch from ElevenLabs instead of Firestore for those fields.

**Depends on:** E (split) complete — easier to modify code that's in focused modules.

**Deliverables:**

- [ ] **F.1** Read `lib/schema.js` (moved from schema.js in Task E). Identify fields to remove:
  - `brand` → **remove voice-specific tone fields** (expressed as EL persona). Keep high-level brand data (tagline, colors) for site renderer.
  - `vox` → **remove: system_prompt, first_message, voice_id, tools_config, llm_config, greeting**. Keep: `elevenlabs_agent_id`, `elevenlabs_phone_number_id`, `last_synced_at`, `industry_template_ref`.
  - `knowledge` → **remove** the KB content (FAQs, policies, certifications, etc). Keep only `elevenlabs_knowledge_base_id` reference.
  - Keep everything else as-is: identity, hours, services, media (logo is used by site renderer), social_proof, integrations (credentials refs), live events, team members, audit log.

- [ ] **F.2** Write `scripts/migrate-2026-04-16-strip-duplicates.js`:
  - Reads every client doc from Firestore
  - For each, snapshots current schema to `clients/{slug}/backup/{timestamp}`
  - Removes deprecated fields
  - Writes migrated doc
  - Logs what was removed per client
  - Dry-run mode by default; `--execute` flag to actually write

- [ ] **F.3** Update `lib/schema.js` to reflect new reality — remove deprecated fields, add JSDoc pointing to ElevenLabs for removed data.

- [ ] **F.4** Update all read-sites in routes/ — grep for references to removed fields. Replace with ElevenLabs API reads or remove entirely.

- [ ] **F.5** Update `integrations/elevenlabs.js` with helper functions: `getAgentConfig(agentId)`, `updateSystemPrompt(agentId, prompt)`, `getKnowledgeBaseContent(kbId)`, etc.

- [ ] **F.6** Update the provisioning route — new clients now have their prompt/KB/voice configured directly on ElevenLabs, not written to Firestore first then synced.

- [ ] **F.7** Run migration in dry-run mode against dev environment. Review output. Do NOT run against production in this task — Phase 5 verification handles that.

- [ ] **F.8** Commit: `git commit -m "Refactor Firestore schema — ElevenLabs is source of truth for agent config"`

**Success criteria:**
- Migration script is dry-run-safe and logs what it would change
- Schema deprecations documented in JSDoc
- No production data touched (migration deferred to Phase 5)
- Server still starts and responds to `/health`

---

### Task G — Rewrite admin "vox" tab

**Scope:** The admin dashboard's `vox` tab in `frontend/index.html` currently edits Firestore-mirrored state. Rewrite to hit ElevenLabs REST API directly. Single source of truth = what the admin sees = what the agent actually uses.

**Depends on:** F (schema refactor) complete — duplicate fields gone, routes know where to fetch.

**Deliverables:**

- [ ] **G.1** Read the existing `vox` tab in `frontend/index.html`. Inventory all fields and their current data source (Firestore paths).

- [ ] **G.2** Create new backend routes in `routes/admin/elevenlabs-passthrough.js`:
  - `GET /api/clients/:slug/elevenlabs/agent` — fetches agent from EL by stored `agent_id`
  - `PATCH /api/clients/:slug/elevenlabs/agent` — applies updates via EL API
  - `GET /api/clients/:slug/elevenlabs/knowledge-base` — fetch KB content
  - `PATCH /api/clients/:slug/elevenlabs/knowledge-base` — update KB content
  - `POST /api/clients/:slug/elevenlabs/sync-from-template` — apply a vertical template (from sr-voice-templates repo)

- [ ] **G.3** All routes auth-gated by existing admin middleware.

- [ ] **G.4** Update the `vox` tab JS to call new routes. Remove references to Firestore paths for vox data.

- [ ] **G.5** Add a "Last synced with ElevenLabs: <timestamp>" indicator in the UI — since we're reading direct, this is the sync-check replacement.

- [ ] **G.6** Keep the "Preview Prompt" button working — it now fetches from EL instead of building from Firestore.

- [ ] **G.7** Update the "Apply Industry Template" button: pulls YAML from `sr-voice-templates/templates/{vertical}/v1/` (via a new endpoint that reads that repo), applies to EL agent.

- [ ] **G.8** Smoke test in dev: load admin, open vox tab for a test client, verify all fields populate from EL, edit a prompt, save, re-load, verify change persisted on EL.

- [ ] **G.9** Commit: `git commit -m "Rewrite admin vox tab to use ElevenLabs as source of truth"`

**Success criteria:**
- Vox tab loads data from EL, not Firestore
- Saves write to EL, not Firestore
- Industry template application works end-to-end
- No regressions in other admin tabs

---

### Task H — Wire 10-scenario test suite

**Scope:** Connect `sr-voice-templates/templates/plumber/v1/tests/simulation/*.yaml` to ElevenLabs' test API. Build the `deploy/smoke.py` stub into a real runner. Block agent go-live until ≥9/10 pass.

**Depends on:** A (templates repo) complete, G (vox tab) complete.

**Deliverables:**

- [ ] **H.1** Read ElevenLabs test API docs — understand `llm` test, `tool` test, `simulation` test payload formats. Identify authentication pattern (API key header).

- [ ] **H.2** Implement `deploy/smoke.py` in `sr-voice-templates`:
  - Args: `--agent-id <id> --template <path> [--min-pass 9]`
  - Loads test YAMLs from the template folder
  - Calls ElevenLabs test API for each
  - Aggregates pass/fail
  - Exits 0 if ≥ min-pass threshold, 1 otherwise
  - Writes a JSON report to `./reports/smoke-{timestamp}.json`

- [ ] **H.3** Implement `deploy/stamp.py`:
  - Args: `--template <vertical/v1> --intake <intake.json> --client-slug <slug>`
  - Composes prompts (assembles fragments per template config)
  - Renders KB content (substitutes `{{placeholders}}` from intake data)
  - Calls ElevenLabs API to create/update the agent
  - Calls `smoke.py` with the new agent ID
  - Reports pass/fail to stdout

- [ ] **H.4** Add admin dashboard button "Run Test Suite" on vox tab (hits a new endpoint that invokes `smoke.py` on the client's agent).

- [ ] **H.5** Create `routes/admin/test-runner.js` in voice-bot-refactor that shells out to smoke.py and streams results back.

- [ ] **H.6** Populate one YAML test fully as a reference (01-flooding-emergency.yaml). Other 9 can be populated iteratively — acceptable to have them as skeletons with pending content, as long as the runner ignores pending tests.

- [ ] **H.7** Document in `sr-voice-templates/README.md`: how to add a new scenario, how to run smoke.py manually, how the go-live gate works.

- [ ] **H.8** Commits in both repos:
  - In sr-voice-templates: `git commit -m "Add smoke.py test runner and stamp.py deployment"`
  - In voice-bot-refactor: `git commit -m "Add admin endpoint to invoke 10-scenario test suite"`

**Success criteria:**
- `smoke.py --agent-id X --template plumber/v1` runs against a real EL agent
- Exit code reflects pass/fail correctly
- Admin UI can trigger the suite
- At least one fully-populated test scenario demonstrating the format

---

## Phase 4 — Code Review (Opus)

- [ ] **R.1** Dispatch `voltagent-qa-sec:code-reviewer` against the full diff on the `refactor/elevenlabs-source-of-truth` branch
- [ ] **R.2** Synthesize review output — flag real issues (security, correctness, performance) vs style opinions
- [ ] **R.3** Apply fixes with one more refactoring agent pass if needed

---

## Phase 5 — Verification

- [ ] **V.1** Run all existing tests
- [ ] **V.2** Run `smoke.py` against a test ElevenLabs agent
- [ ] **V.3** Deploy refactor branch to a staging Cloud Run service
- [ ] **V.4** Manually provision a test plumber client end-to-end
- [ ] **V.5** Place a real test call through the provisioned agent
- [ ] **V.6** Verify post-call analysis wrote to Firestore
- [ ] **V.7** Verify Better Stack logs appear with correlation IDs
- [ ] **V.8** Only when all pass: merge `refactor/elevenlabs-source-of-truth` → `main`, deploy to prod Cloud Run

---

## Execution Notes

- **Commits frequently** — every logical chunk within a task gets its own commit so git history is legible
- **Don't cross task boundaries** — each agent works only on its assigned task
- **No shared state between parallel agents** — tasks A/B/C/D touch different files and different repos
- **If a task reveals a dependency on another task's work**, stop and report — don't proceed with guesswork
- **The main voice-bot checkout at `Website/voice-bot/` is off-limits** — work only in `voice-bot-refactor/`
- **Carl's 4 uncommitted files in main stay in main** — refactor does not need them

## Self-Review Complete

Plan reviewed against the decision doc (D1–D11). Scope coverage verified. Type consistency across tasks checked. No placeholders in specified code. File paths are exact. Ready for execution.
