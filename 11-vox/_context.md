# Vox — White-Label AI Voice Receptionist

## What Is Vox
Premium, white-label AI voice receptionist product sold to local businesses. Handles phone calls and website voice chat — answers questions, checks calendar availability, books appointments, and transfers calls to the client's desk when needed.

## GitHub
https://github.com/Blaez-13/vox (private)
**⚠️ Remote may be misconfigured — push currently fails with 404. Verify the org slug (Blaez-13 vs Blaez13) and repo existence before next commit.**

## Source Code
`c:\Users\ctrep\Documents\Anti-Gravity\Website\voice-bot`

## Live URLs
- **Middleware + Admin:** https://sr-voice-bot-925407339242.us-central1.run.app
- **Admin Dashboard:** https://sr-voice-bot-925407339242.us-central1.run.app/admin/ *(now HTTP Basic Auth; user `admin`, pw = `$ADMIN_TOKEN` env var)*
- **Phone Receptionist:** (205) 831-8876 — now direct on ElevenLabs (Twilio → ElevenLabs, no Vapi)
- **Embeddable Widget:** `https://sr-voice-bot-925407339242.us-central1.run.app/widget.js`

## Stellaris Ridge Website
- **Live (Astro, Jarod's pick):** https://stellaris-ridge-925407339242.us-central1.run.app
- **V2 (alternative):** https://sr-website-v2-925407339242.us-central1.run.app
- **Astro source:** `c:\stellaris\astro`
- **V2 source:** `c:\Users\ctrep\Documents\Anti-Gravity\Website\site`

## Architecture (updated 2026-04-16 — Vapi removed)

```
Phone call → Twilio → ElevenLabs Conversational AI (Ridge persona, Gemini 2.5 Flash brain)
                         ↓ function calls (tool webhooks)
                    Vox Middleware (Cloud Run: sr-voice-bot)
                         ↓
                    GHL Calendar API (read slots, book appointments)

Website visitor → clicks mic → WebSocket /ws/voice → Vox Middleware
                                              ↓
                                         Gemini 3.1 Flash Live (native audio)
                                              ↓ function calls
                                         GHL Calendar API
```

**Migration note (2026-04-16):** Vapi was removed from the phone path. ElevenLabs now speaks directly to Twilio. Lower latency, one less hop, one less bill. Vapi assistant `849c4f0b...` still exists in Vapi account (no calls routed to it) — safe to delete when you cancel the Vapi subscription.

## Tech Stack
- **Server:** Node.js, Express, WebSocket (ws), Firestore, Secret Manager
- **Phone voice:** Twilio + ElevenLabs ConvAI (direct, no Vapi) + Gemini 2.5 Flash
- **Web voice:** Gemini 3.1 Flash Live (native audio) via Google GenAI SDK
- **Calendar:** GoHighLevel API v2 (2021-07-28)
- **Deployment:** Google Cloud Run (project: internal-sr, region: us-central1)
- **Admin:** Single-page HTML dashboard with REST API, HTTP Basic Auth
- **Auth model:** Express middleware — public allowlist for /health, /widget.js, /webhook/*, /tools/*, /ws/*, /sites/*, /api/portal/*; everything else requires Basic Auth using `ADMIN_USERS` or fallback `ADMIN_TOKEN` env

## Key Files (in voice-bot/)
| File | Purpose |
|------|---------|
| `server.js` | Express + WebSocket + ElevenLabs tool webhook + Gemini proxy + Admin API + Basic Auth middleware |
| `elevenlabs.js` | ElevenLabs ConvAI agent + tool provisioning (idempotent) |
| `ghl-calendar.js` | GHL read slots, create contacts, book appointments |
| `client-registry.js` | Multi-tenant Firestore-backed client registry |
| `prompt-builder.js` | Generates system prompts from client business data |
| `industries.js` | Industry templates (auto_repair, hvac, dental, contractor, salon, professional_services) |
| `secrets.js` | Secret Manager helpers |
| `frontend/index.html` | Admin dashboard |
| `frontend/widget.js` | Embeddable voice widget (one script tag) |
| `portal/index.html` | Client self-serve portal (Firebase Hosting) |

## Admin Dashboard Features
- HTTP Basic Auth gate (new 2026-04-16)
- Client CRUD (create, edit, archive) — Firestore-backed
- GHL Calendar Integration (API key stored in Secret Manager, refs on doc)
- Call Routing (desk phone, desk hours AM/PM, ring timeout, transfer)
- Voice Agent Settings (voice, personality, greetings, escalation)
- "Provision ElevenLabs" — auto-provisions ConvAI agent + 4 tools
- "Get Web Widget Code" — generates embed code
- "Preview Prompt" — view generated system prompt
- Apply industry template
- Service request ticketing
- Bulk client import

## Onboarding a New Client
1. Open admin dashboard (log in with Basic Auth)
2. Click "+ New Client"
3. Fill in business info, services, FAQs, greetings
4. Add GHL credentials (stored in Secret Manager — only the ref is saved on the doc)
5. Set voice and call routing
6. Save
7. Click "Provision Vox" → ElevenLabs agent + 4 tools auto-created (idempotent)
8. Click "Get Web Widget Code" → one-line embed for their website
9. Import their Twilio number into ElevenLabs + assign to their agent

## Accounts & Keys
**⚠️ Actual values are in Cloud Run env / Secret Manager / .env — NEVER commit in plaintext to this note or any repo.**
- **Vapi:** deprecated 2026-04-16. Key still in Secret Manager for rollback. Assistant `849c4f0b...` dormant.
- **Twilio:** SID in Cloud Run env as `TWILIO_ACCOUNT_SID`, number `+12058318876`. Auth token in Secret Manager (`twilio-auth-token-vox`). Not committed to any note.
- **ElevenLabs:** API key in `.env` locally + Secret Manager (`elevenlabs-api-key`). Tool webhook secret in Secret Manager (`elevenlabs-tool-secret`).
- **GHL (Stellaris Ridge):** API key in Secret Manager (`ghl-api-key-stellaris-ridge`). Location `80RqmUCVcLG00OAD9H5k`, Calendar `7WG0IOzKeJJdO3ZOYcT6`.
- **Gemini:** API key in `.env` locally + Cloud Run env.
- **Firebase/Firestore:** Uses Cloud Run service account (no API key needed).

## Client Roster (2026-04-16)
| Slug | Industry | ElevenLabs agent |
|------|----------|-------------------|
| `stellaris-ridge` | production | `agent_3201kpbjgjdhfm8tyj2aek5c23ne` ← phone `+12058318876` |
| `demo-bama-auto` | auto_repair | `agent_0601knwdr7zheqfrvefvyffq9631` |
| `demo-comfort-air` | hvac | `agent_2001kpbp2bj0fxst7hes6a8cwsxf` |
| `demo-mckinney-builders` | contractor | `agent_0301kpbp2d63en2bed62a1np1zgr` |
| `demo-birmingham-dental` | dental | `agent_7901kpbp3x3tedcvnqffthg1kxb3` |
| `demo-bellissima-salon` | salon | `agent_1601kpbp3z9ve68sz8sgt6d822hm` |
| `demo-summit-law` | professional_services | `agent_7801kpbp41c3ezm9ghv1yjc7s6k2` |

## Known Issues / Next Steps
- **CRITICAL: `clients/stellaris-ridge.json` has a plaintext GHL API key committed to voice-bot git history** (commit `ce1c20d`). Rotate the key and purge from history if the repo is public or ever was.
- **voice-bot git remote (`github.com/Blaez-13/vox`) returns 404** — push fails. Verify org slug and repo existence.
- `.gitignore` in voice-bot is minimal (`node_modules`, `.env`, `*.log`). Missing: `.firebase/`, `.ridge_agent_ref.json`, `.tools_ref.json`, `clients/*.json` (secrets).
- `~7,600 lines of uncommitted drift` in voice-bot that matches deployed production. Commit when the remote is fixed.
- `/webhook/elevenlabs/tool/:slug/check_availability` handler only accepts ISO dates (tool description says "tomorrow" works too — ElevenLabs resolves dates itself so this works in practice).
- Firestore `integrations.vapi` on stellaris-ridge doc still shows stale Vapi refs — cosmetic, no functional impact.
- Voice dropdown in admin UI still shows Gemini voices, not ElevenLabs voices.
- `/token/gemini` endpoint returns 404 (legacy, widget uses `/ws/voice` proxy instead — safe to remove).
- Orphan ElevenLabs assets: `agent_8301` "Ridge" on `+12055263947`, `agent_5501` "Agent agent" on `+12058314421` — old test artifacts, safe to delete.
- Gemini Live PCM audio input causes 1007 disconnect — widget uses Web Speech API STT as workaround. When Gemini 3.1 Flash Live supports direct PCM from browser, swap in.
- Vapi subscription should be cancelled to stop billing (assistant dormant, phone already removed).
