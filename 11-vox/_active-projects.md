# Vox — Active Work

## Status: Production Live — Launch 2026-04-17

### Completed (2026-04-16 — launch-eve hardening)
- [x] **Phone migration Vapi → ElevenLabs** — `+12058318876` imported into ElevenLabs, bound to `agent_3201...` (Vox — Stellaris Ridge). Vapi phone released. Lower latency, one less hop.
- [x] **HTTP Basic Auth** deployed on sr-voice-bot rev 44 — /admin + /api/* now credentialed (user `admin`, password = `$ADMIN_TOKEN`).
- [x] **Web widget verified** on stellarisridge.com via Playwright — WebSocket connects, status "Ridge is listening…", no JS errors.
- [x] **Tool webhook end-to-end tested** — `/webhook/elevenlabs/tool/stellaris-ridge/check_availability` returns real GHL slots with correct `X-Vox-Tool-Secret`.
- [x] **6 vertical demo clients** provisioned on ElevenLabs (auto_repair, hvac, contractor, dental, salon, professional_services) — all diagnostics green.
- [x] **Broken demo-hvac archived** (had no services/FAQs/first-message).
- [x] Memory updated, vault notes refreshed.

### Completed (earlier — 2026-03-31 through 2026-04-12)
- [x] GHL calendar read/write middleware
- [x] Gemini Live web voice widget
- [x] Multi-tenant client registry (Firestore-backed)
- [x] Admin dashboard with client CRUD
- [x] GHL settings in dashboard, Secret Manager integration
- [x] Call routing config (desk phone, desk hours, transfer)
- [x] Embeddable widget.js with per-client routing
- [x] Auto-provision ElevenLabs agent + tools from dashboard
- [x] Embed code generator
- [x] Deployed to Cloud Run (project: internal-sr)
- [x] Stellaris Ridge website updated with voice widget
- [x] Photos added for Carl and Jarod
- [x] Migrated from JSON client files to Firestore (2026-04-09)
- [x] Vapi → ElevenLabs code migration (2026-04-09)
- [x] Client self-serve portal (Firebase Hosting)
- [x] Service request ticketing
- [x] Client website renderer (wildcard subdomain)

### Needs Testing (launch-eve smoke test by Carl before FB post)
- [ ] **Live phone call** to (205) 831-8876 — verify Ridge answers, books an appointment
- [ ] **Web widget on stellarisridge.com** — tap mic, speak, book an appointment
- [ ] **Admin login** — visit /admin, enter `admin` + `$ADMIN_TOKEN`
- [ ] **Multi-tenant isolation** — try one of the 6 vertical demos via widget with `?client=demo-birmingham-dental`

### Urgent Cleanup (do before next voice-bot commit)
- [ ] **Rotate GHL API key** — `pit-00b3bfd3...` is in voice-bot git history (commit `ce1c20d` in `clients/stellaris-ridge.json`). Rotate, update Secret Manager, redeploy.
- [ ] **Fix voice-bot git remote** — `https://github.com/Blaez-13/vox.git` returns 404. Verify org slug (Blaez-13 vs Blaez13) and repo existence.
- [ ] **Tighten `.gitignore`** in voice-bot: add `clients/*.json`, `.firebase/`, `.ridge_agent_ref.json`, `.tools_ref.json`, `portal/` (if served separately by Firebase Hosting).
- [ ] **Purge GHL key from git history** (git filter-branch or BFG Repo Cleaner) after rotation — any pre-rotation clone still has the key.

### Post-Launch Cleanup (no rush)
- [ ] Cancel Vapi subscription (phone already released, assistant dormant)
- [ ] Delete stale Cloud Run services: `sr-vox`, `sr-vox-api`, `sr-vox-agent`, `sr-vox-direct`
- [ ] Delete orphan ElevenLabs agents: `agent_8301` "Ridge" on `+12055263947`, `agent_5501` "Agent agent" on `+12058314421`
- [ ] Remove legacy `C:\Stellaris\10-stellaris-ridge\gemini-live-voice-agent\` (LiveKit Python attempt, dead)
- [ ] Fix `/webhook/elevenlabs/tool/:slug/check_availability` to accept "tomorrow"/"next Tuesday" natural phrases (currently ISO-only)
- [ ] Remove `/token/gemini` endpoint (legacy, widget uses `/ws/voice`)
- [ ] Clean stale `integrations.vapi` fields on Firestore stellaris-ridge doc
- [ ] ElevenLabs voice browser in admin UI (currently shows Gemini voices)
- [ ] Call recording + transcripts in dashboard
- [ ] Client billing / usage tracking
- [ ] Google Calendar / Calendly adapter (for non-GHL clients)

### Future
- [ ] Twilio call flow for business hours forwarding
- [ ] Per-user admin auth (currently single shared `admin` login) — Firebase Auth is already partially wired for the portal; extend to /admin
- [ ] Rate limiting on public endpoints (/ws/voice, /widget.js)

### Parked: Vox Direct (Twilio → Gemini 3.1 Live, no Vapi)
**Status:** Experiment parked 2026-04-05. Superseded by the ElevenLabs-direct architecture on 2026-04-16. Code at `c:\Users\ctrep\Documents\Anti-Gravity\Website\vox-direct`, Cloud Run service `sr-vox-direct`. Safe to delete.
