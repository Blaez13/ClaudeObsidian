---
type: decisions-log
company: Stellaris Ridge
updated: 2026-04-16
---

# Stellaris Ridge — Decisions Log

> Log key decisions here with date and rationale.
> Never delete — archive old decisions to [[80-archive/old-decisions/]].

---

## 2026-03-22 — Vault Architecture

**Decision:** Consolidate all memory into a single Obsidian vault at C:\Stellaris\
rather than maintaining separate C:\StellarisRidge\ and C:\PawAndPantry\ directories.

**Rationale:** Claude Code needs to see everything in one place. Cross-referencing
between SR and P&P work is frequent. One vault, two folders.

**Made by:** Carl

---

## 2026-03-28 — SR Vox Tech Stack

**Decision:** Use LiveKit Cloud + Gemini 3.1 Flash Live (speech-to-speech) for the
SR Vox voice agent product. Not using LiveKit's inference marketplace (pipeline approach).

**Rationale:** Speech-to-speech gives lower latency (~0.5s vs ~1-2s). Direct Google API
billing is cheaper than LiveKit's inference markup. Simpler architecture, fewer failure points.
LiveKit is transport only — WebRTC, phone numbers, SIP bridging, agent dispatch.

**Made by:** Carl

---

## 2026-03-28 — SR Vox Model: Gemini 3.1 Flash Live

**Decision:** Use `gemini-3.1-flash-live-preview` instead of the older 2.5 native audio models.

**Rationale:** Carl confirmed after watching demo video — 3.1 has significant improvements
in voice quality and conversation handling. Tested and confirmed working.

**Made by:** Carl

---

## 2026-03-28 — SR Business Location

**Decision:** Stellaris Ridge is based in Greater Birmingham, Alabama (not Wisconsin).
Carl is in Wisconsin but the business operates out of Alabama where Jarod is based.

**Rationale:** Jarod handles sales and client relationships. Face-to-face meetings with
local Alabama clients is a key differentiator. Carl handles tech remotely.

**Made by:** Carl

---

## 2026-03-30 — SR Vox V2: Split Architecture + GHL-Only Calendar

**Decision:** Rebuild Vox with split Cloud Run services (sr-vox-api + sr-vox-agent) and remove Google Calendar integration. GHL is the single calendar system. Model is swappable via one env var.

**Rationale:** V1 had two problems:
1. Running token server + agent worker in one Cloud Run container caused crashes — Cloud Run kills idle containers but the agent needs a persistent connection to LiveKit.
2. Function calling didn't work — blamed on Gemini 3.1 but root cause was outdated `livekit-plugins-google` plugin. The GCal/Apps Script workaround added fragile complexity for no benefit.

Split: sr-vox-api (scales to 0, cheap) handles HTTP. sr-vox-agent (min-instances=1, always on) maintains LiveKit connection.

**Made by:** Carl

---

## 2026-03-28 — SR Vox Client Management

**Decision:** Clients stored as JSON files in `clients/` directory with a web admin
dashboard at `/admin`. Not using a database.

**Rationale:** Simple, version-controllable, easy to backup. Sufficient for current
scale (5-10 clients). Can migrate to DB later if needed.

**Made by:** Carl
**Superseded:** 2026-03-31 — migrated to Firestore (see below).

---

## 2026-03-31 — Firestore Migration

**Decision:** Move client storage from ephemeral JSON files to Google Cloud Firestore
(Native mode) with real-time `onSnapshot` listeners for cache sync across Cloud Run instances.

**Rationale:** Cloud Run's ephemeral filesystem wiped 5 test clients on redeploy.
Firestore gives persistence, real-time sync, subcollections for events/leads/call_logs,
and prepares for the client portal (Firebase Auth + Firestore security rules).

**Made by:** Carl

---

## 2026-04-05 — Client Portal (Firebase Auth + Postmark)

**Decision:** Build a client-facing portal at internal-sr.web.app using Firebase Auth
with passwordless email-link sign-in. Bypass Firebase's SMTP (broken with Postmark)
by generating sign-in links server-side via Admin SDK and sending through Postmark HTTP API.

**Rationale:** Clients need self-service for services, hours, FAQs, and live state.
Firebase Auth's SMTP integration failed across ports 587/2525/465 with both STARTTLS
and SSL. Server-side Postmark is 100% reliable.

**Made by:** Carl

---

## 2026-04-09 — Vapi → ElevenLabs Migration

**Decision:** Replace Vapi with ElevenLabs Conversational AI as the voice platform for Vox.
Per-client agents with custom webhook tools. Lucy voice (EQu48Nbp4OqDxsnYh27f), eleven_turbo_v2 TTS,
Gemini 2.5 Flash LLM. Vapi code retained for rollback only.

**Rationale:** Carl evaluated ElevenLabs offerings and found: (1) native silence handling
beats Vapi's idle messages, (2) premium voice quality aligns with product positioning,
(3) single platform for TTS + ConvAI reduces latency vs Vapi proxying through ElevenLabs,
(4) ElevenLabs feature velocity is faster (Creator tier at $22/mo vs Vapi metering).

**Made by:** Carl

---

## 2026-04-09 — Service Call Handling (Premium Feature)

**Decision:** Add complaint/emergency/urgent callback handling for trade clients (HVAC,
plumbing, contractors). Vox classifies calls early, captures details, logs tickets to
Firestore + GHL pipeline mirror, pages owner via Postmark email + Twilio SMS.
Emergency live transfer is configurable per-client (default off).

**Rationale:** Competitive audit showed no provider in SR's target market offers
AI-powered service call triage. This is the differentiator that makes Vox premium.
Existing customers (HVAC/plumber) get panic calls post-job — currently goes to
voicemail. Vox handling this is worth 2-3x the base price.

**Made by:** Carl

---
## 2026-04-16 — Phone cutover: Vapi removed from production path

**Decision:** (205) 831-8876 migrated off Vapi; now routes Twilio → ElevenLabs
Conversational AI directly → voice-bot webhooks. Vapi phone-number record deleted
(`c2dfc2d9...`). Vapi assistant (`849c4f0b...`) remains in the account but receives
no traffic and can be deleted when the Vapi subscription is cancelled.

**Rationale:** ElevenLabs Conversational AI handles TTS + ASR + LLM routing
natively, so Vapi was an unnecessary middle hop adding latency and a second
bill. The ElevenLabs agent calls our tool webhooks (`/webhook/elevenlabs/tool/...`)
directly — faster path, single vendor for voice.

**Made by:** Carl (executed during launch-eve hardening session)

---

## 2026-04-16 — HTTP Basic Auth on sr-voice-bot admin endpoints

**Decision:** Wrap `/admin` and `/api/*` routes (except the public allowlist —
/health, /widget.js, /webhook/*, /tools/*, /ws/*, /sites/*, /api/portal/*) in
HTTP Basic Auth middleware. Credentials come from `ADMIN_USERS` env
(format: `user:pass,user:pass`) OR fallback to username `admin` with password
from the existing `ADMIN_TOKEN` env. Rev 44 live 2026-04-16.

**Rationale:** Pre-launch audit surfaced that the admin dashboard and every
`/api/*` endpoint (including Firestore CRUD on clients, Secret Manager writes,
ElevenLabs agent provisioning, call logs, leads) was publicly reachable with
zero auth. Once Jarod advertised the widget URL in Facebook groups, the
Cloud Run URL would be trivially discoverable and the full client roster
exposed. HTTP Basic Auth was picked over Firebase Auth for speed of launch;
upgrade to per-user Firebase Auth post-launch.

**Made by:** Carl

---
