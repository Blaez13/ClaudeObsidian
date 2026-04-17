---
date: 2026-04-16
status: approved
decision-type: architecture
impact: high
owner: Carl
reviewers: [Jarod]
supersedes: Earlier "clean rebuild" draft (2026-04-16 AM)
---

# Decision: Refactor Voice-Bot Platform In Place (Not Rebuild)

## TL;DR

The existing voice-bot platform (`Website/voice-bot/`, deployed at `sr-voice-bot-925407339242.us-central1.run.app`) already implements multi-tenant client management, ElevenLabs provisioning, GHL integration, client portal with magic-link auth, service request routing, live-state event overrides, and industry templates. Migration from Vapi → ElevenLabs completed April 9. **Refactor this platform** — strip duplicate state, add the architects' behavioral recommendations (vertical templates, fragment library, testing gate, monitoring) — rather than rebuild from scratch. No paying clients yet, but rebuild would cost 4–6 weeks of re-solving solved problems for no functional gain.

**Stack stays:** Cloud Run + Firestore + ElevenLabs + Twilio + GCP Secret Manager + GitHub + Better Stack (new). No Supabase, no Vercel, no Retool, no n8n, no Zapier.

---

## Context — How This Decision Was Reached

### What I got wrong early

The initial research pointed at `Anti-Gravity/VOX/` — a ~1,200 LoC LiveKit experiment with no database, no admin panel. Based on that audit, I recommended a clean build on ElevenLabs + Supabase + Next.js + Retool over 6–8 weeks.

### What the follow-up audit revealed

Carl pointed out the admin panel exists. A second audit found the real SR platform lives at `Anti-Gravity/Website/voice-bot/` — ~8,200 LoC Express/Firestore multi-tenant system, on Cloud Run, actively maintained, Vapi→ElevenLabs migration completed one week ago.

### Carl's key insight

> "A lot of what was built and going to be stored in the database is already available in 11 labs."

Firestore is duplicating state that ElevenLabs already owns natively: system prompts, voice config, tool definitions, knowledge base content. Two sources of truth → drift → bugs. **The problem isn't the platform's size. It's the state-ownership boundary.**

---

## Decisions

### D1. Refactor in place (Option B). Reject rebuild (Option A) and hybrid (Option C)

- **Reject rebuild:** 4–6 weeks to re-solve solved problems. Adds Supabase + Vercel. Violates "fewer tools."
- **Reject hybrid:** Two codebases to maintain.
- **Accept refactor:** 1–2 weeks. Lower risk. No new platforms. Respects what works.

### D2. Architectural principle — ElevenLabs is source of truth for agent configuration

Firestore holds ONLY SR business state:

| Firestore keeps | ElevenLabs owns (remove from Firestore) |
|---|---|
| Client metadata (slug, agent_id, phone, billing tier) | System prompts |
| GHL / Calendar credentials (Secret Manager refs) | Voice selection |
| Call log aggregations (beyond EL's retention) | Tool definitions |
| Service requests (SR workflow escalations) | Knowledge base content |
| Audit log | Workflow graphs |
| Team members / access | First message / greeting |
| Industry template references | Dynamic variables |
| Live-state event overrides (vacation, urgent, promo) | |

Admin dashboard's `vox` tab reads/writes ElevenLabs directly via REST API, not through Firestore.

### D3. Integration middleware — FastAPI-style webhook only (Carl: "fewer tools")

Flow: `ElevenLabs agent → SR webhook endpoint → GHL / Google Calendar`. No n8n. No Zapier. No Make. The existing Express service already implements this pattern.

### D4. Vertical templates as code — Git is source of truth

Create `sr-voice-templates` repo structure:

```
sr-voice-templates/
  fragments/              (reusable prompt components, versioned)
  tools/                  (tool schemas as JSON)
  templates/
    plumber/v1/
      workflow.yaml
      prompts/ (composed from fragments)
      kb/ (templates with {{placeholders}})
      tests/ (10 simulation scenarios)
      intake-form.json
    dental/v1/
    hvac/v1/
    legal/v1/
    med-spa/v1/
  deploy/
    stamp.py              (intake + template → ElevenLabs API)
    smoke.py              (run 5 scenarios post-deploy)
    monitor.py            (pull analysis, check thresholds, alert)
```

Admin dashboard's "industry templates" feature pulls from this repo instead of hardcoded templates in server.js.

### D5. Testing gate — 10 scenarios before client go-live

Must pass ≥9/10 before handoff:

1. Clear emergency → transfers in <10s
2. Routine booking → books successfully
3. Billing lookup → tool call succeeds
4. Caller refuses problem description → transfers after 2 clarifiers
5. Non-English caller → handoff or Spanish routing
6. Upset caller → de-escalates within 2 turns
7. Mis-framed emergency ("toilet running, water on floor") → emergency branch fires
8. Empty tool result → offers callback, doesn't fabricate
9. Tool failure → acknowledges, doesn't claim success
10. Pricing question → refuses, reads policy, offers dispatch

### D6. Production monitoring — Better Stack + alert thresholds

$30/mo. Uptime, log search, alerting, on-call rotation.

Per-client rolling 7-day thresholds:

| Metric | Warn | Page |
|---|---|---|
| Triage accuracy | <92% | <85% |
| Emergency mis-route | 1 event | 2+ events |
| Tool success rate | <95% | <90% |
| Booking completion | <70% | <50% |
| Negative sentiment end-of-call | >15% | >25% |

Post-call analysis: 20% sample + 100% of flagged calls. Run async (off hot path).

### D7. LLM selection per agent node

| Node | LLM | Rationale |
|---|---|---|
| Triage | Gemini 3.1 Flash Lite | Speed, cost, classification |
| Emergency confirmation | Gemini 3.1 Flash Lite | Speed over nuance |
| Support | Gemini 3.1 Flash Lite | KB lookups, short answers |
| Booking | Gemini 3.1 Flash | Tool-call reliability |
| Billing | Gemini 3.1 Flash | Tool + PII-adjacent |
| Post-call analysis | Claude Sonnet 4-6 | Quality > latency, async |

Temperature 0.7. Reasoning effort low. TTS: Eleven Flash v2.5.

### D8. Vapi removal — safe to delete

Migration completed April 9, 2026. Vapi fallback code (~90 lines across `server.js`, `schema.js`, `client-registry.js`, `merge.js`) is rollback insurance no longer needed. Remove in Phase 2.

### D9. server.js split

2,699 lines in a single file. Split into focused modules — `routes/`, `integrations/` (ghl, elevenlabs, twilio, postmark), `webhooks/`, `middleware/`. Easier for Claude + Carl to work in going forward.

### D10. HIPAA deferred

BAA requires ElevenLabs Enterprise tier — significant cost step. Defer dental and med spa verticals until pricing is modeled. Plumber, HVAC, legal (intake only), home services launch first.

### D11. vox-direct experiment — park, don't kill

`Website/vox-direct/` is a 306-line Gemini-Live-via-Twilio experiment. Not part of this refactor. Leave as-is; revisit when Google's Gemini TTS stack matures and ElevenLabs pricing pressure shifts.

---

## Deliverables (what this refactor produces)

1. **Voice-bot platform refactored** — Firestore schema stripped of duplicates, admin `vox` tab reads ElevenLabs directly, Vapi legacy removed, server.js modularized
2. **`sr-voice-templates` Git repo** — fragment library, per-vertical templates, test scenarios, intake forms
3. **Testing gate wired** — 10-scenario suite runs against ElevenLabs test API before client go-live
4. **Better Stack monitoring** — uptime, logs, alerts against threshold matrix
5. **Post-call analysis config** — triage accuracy, emergency routing, tool success, sentiment extraction on every call
6. **Updated documentation** — architecture overview, template usage guide, deployment/onboarding runbook

---

## Execution Strategy (Superpowers Pattern)

**Phase 0 — Plan** (Opus): Bite-sized implementation plan, exact file paths, TDD where applicable.

**Phase 1 — Worktree**: Git worktree of `voice-bot` repo for isolated refactor.

**Phase 2 — Parallel Sonnet agents** (fan-out, ~30–45 min wall-clock):
- A. Create `sr-voice-templates` repo
- B. Remove Vapi legacy code
- C. Add Better Stack monitoring + alerts
- D. Add ElevenLabs post-call analysis config

**Phase 3 — Sequential Sonnet agents** (ordered, ~2 hrs):
- E. Split server.js into modules
- F. Refactor Firestore schema (strip duplicates)
- G. Rewrite admin `vox` tab → direct ElevenLabs API
- H. Wire 10-scenario test suite

**Phase 4 — Opus review** (code-reviewer agent + my synthesis)

**Phase 5 — Verification** (run tests, smoke admin, verify monitoring)

**Phase 6 — Merge + deploy** to Cloud Run

**Total active time:** ~4 hours across one or more sessions.

---

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Existing uncommitted changes in voice-bot (.env.example, client-registry.js, frontend/index.html, ghl-calendar.js) | Carl reviews/commits these before refactor touches files |
| State-ownership refactor breaks in-flight provisioning calls | Phase 5 verification includes "provision a test agent end-to-end" |
| Removing Vapi code breaks some overlooked dependency | Search-and-audit in Phase 2B; retain fallback branch for 90 days |
| Emergency false-negative liability at launch | Every client contract includes "911 is the only emergency line" disclaimer read at call start. Legal review before first live deploy. |
| LLM cost pass-through from ElevenLabs (coming) | Price client contracts at $0.12–0.15/min to absorb future pass-through |
| Twilio number ownership on client churn | Services contract clarifies SR retains number |
| ElevenLabs post-call webhook loss | Nightly reconciliation job vs EL REST API — build day 1 (included in Phase 2D) |

---

## Open Items Carl Must Resolve (non-blocking)

1. **Uncommitted changes in voice-bot repo** — 4 files. Commit, stash, or roll into refactor?
2. **Vertical priority** — Plumber first (default)? Or Jarod has a specific pipeline?
3. **HIPAA timeline** — skip dental/med spa for 6 months, or model Enterprise tier pricing now?
4. **`vox-direct` long-term fate** — archive or keep for Gemini-TTS-GA experiments?

---

## Approval

- [x] Carl approved refactor path (Option B)
- [ ] Jarod briefed on new service offering + timeline
- [ ] Decision committed to git
