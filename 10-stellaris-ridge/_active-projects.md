---
type: tracker
company: Stellaris Ridge
updated: 2026-04-16
---

# Stellaris Ridge — Active Projects

> Update this file whenever a project status changes.
> Archive completed projects to [[80-archive/completed-projects/]].

## Currently Active

| Client | Service | Status | Next Action | File |
|--------|---------|--------|-------------|------|
| Stellaris Ridge | SR Vox (voice agent) | **Launch-ready** — phone on ElevenLabs, widget live, admin auth deployed | Jarod FB launch 2026-04-17. Live smoke test before posting. | [[11-vox/_active-projects]] |
| Stellaris Ridge | FB Group Launch | Jarod starts posting 2026-04-17 | Carl: test call to (205) 831-8876 + widget on stellarisridge.com first | — |
| All demo clients | Service call handling | Built, ready to enable | Enable per-client in admin Service Calls tab | — |
| Vox demos | 6 vertical test agents | Provisioned on ElevenLabs | Use for Jarod sales demos (dental, hvac, contractor, auto, salon, legal) | [[11-vox/_active-projects]] |

## In Progress (Internal)

| Task | Owner | Status | Notes |
|------|-------|--------|-------|
| Vault setup | Carl | Done | Phase 1 complete |
| SR context file | Carl | Done | Populated 2026-03-22 |
| P&P context file | Carl | In progress | Needs brand brief |
| Git sync | Carl | Pending | Set up after vault done |
| **SR Vox — Phase 1** | Carl | Done | Voice agent + web widget + admin dashboard working locally |
| **SR Vox — V2 Rebuild** | Carl | Done | Stripped GCal, GHL-only calendar, model swappable via env var |
| **SR Vox — Firestore Migration** | Carl | Done | Moved from ephemeral filesystem to Firestore with real-time listeners |
| **SR Vox — Portal (Phase D)** | Carl | Done | Firebase Auth magic link, tabbed SPA at internal-sr.web.app |
| **SR Vox — ElevenLabs Migration** | Carl | Done | Replaced Vapi with ElevenLabs Conversational AI (2026-04-09) |
| **SR Vox — Service Call Handling** | Carl | Done | Complaints/emergencies for HVAC/plumber/contractor clients |
| **SR Vox — Phone Cutover** | Carl | **Done (2026-04-16)** | +12058318876 imported to ElevenLabs, bound to agent_3201. Vapi released. |
| **SR Vox — Admin Auth** | Carl | **Done (2026-04-16)** | HTTP Basic Auth middleware deployed on sr-voice-bot rev 44. user=`admin`, pw=`$ADMIN_TOKEN`. |
| **SR Vox — 6 Vertical Demos** | Carl | **Done (2026-04-16)** | auto/hvac/contractor/dental/salon/legal clients provisioned on ElevenLabs. |
| **GHL API Key Rotation** | Carl | **Urgent** | `pit-00b3bfd3...` committed to voice-bot git history. Rotate + purge history. |
| **voice-bot GitHub remote fix** | Carl | **Urgent** | `github.com/Blaez-13/vox` returns 404 — org/user not set up yet. Create or repoint. |

## Tech Stack (Current as of 2026-04-12)

| Layer | Technology | Notes |
|-------|-----------|-------|
| Voice platform | **ElevenLabs Conversational AI** | Replaced Vapi (2026-04-09). Per-client agents with custom tools. |
| TTS | **ElevenLabs** (eleven_turbo_v2) | Voice: Lucy (EQu48Nbp4OqDxsnYh27f) default |
| LLM | **Gemini 2.5 Flash** | Via ElevenLabs agent config. Also supports GPT-4o, Claude. |
| Telephony | **Twilio** | Number imported into ElevenLabs for inbound calls |
| Calendar/CRM | **GoHighLevel (GHL)** | Read availability, book appointments, mirror service tickets |
| Data store | **Firestore** (Native mode) | Clients, service_requests, leads, call_logs, events |
| Hosting | **Cloud Run** (sr-voice-bot) | Express server, us-central1, min-instances=1 |
| Portal | **Firebase Hosting** | internal-sr.web.app (client portal), vox-client-sites (marketing sites) |
| Auth | **Firebase Auth** | Email-link passwordless via Postmark |
| Secrets | **Google Cloud Secret Manager** | GHL keys, Twilio token, ElevenLabs key, Postmark token |
| SMS | **Twilio** | Service call alerts to business owners |
| Email | **Postmark** | Portal sign-in links + service ticket alert emails |
| Web voice | **Gemini 3.1 Flash Live** | Browser-based voice widget (separate from phone agent) |

## Completed

- 2026-03-22: Obsidian vault created at C:\Stellaris\
- 2026-03-22: Full folder structure built
- 2026-03-22: CLAUDE.md populated
- 2026-03-22: SR and agent context files created
- 2026-03-28: SR Vox Phase 1 built — voice agent, web widget, admin dashboard
- 2026-03-28: Tested Gemini 3.1 Flash Live voice conversation (working)
- 2026-03-28: Multi-client routing + admin dashboard built
- 2026-03-30: V2 rebuild — removed GCal/Apps Script, GHL-only calendar, function calling
- 2026-03-30: Split architecture — token server and agent worker are separate Cloud Run services
- 2026-03-31: Unified schema, Firestore migration, Secret Manager, audit logs
- 2026-04-05: Client portal (Firebase Auth, tabbed SPA, services/hours/FAQ edit)
- 2026-04-06: Agency admin support, client picker, new client quick-create
- 2026-04-07: Website renderer, per-industry templates, Firebase Hosting
- 2026-04-08: Firestore security rules, field-level write restrictions
- 2026-04-09: ElevenLabs migration — replaced Vapi with ElevenLabs Conversational AI
- 2026-04-09: Service call handling — classify/capture/notify flow for HVAC/plumber/contractor
- 2026-04-09: Notifications module — Postmark email + Twilio SMS + webhook fan-out
- 2026-04-09: GHL Opportunities API — service tickets mirrored to CRM pipeline
- 2026-04-10: Deployed all changes to Cloud Run, Firebase Hosting, Firestore rules
- 2026-04-10: First ElevenLabs agent provisioned successfully via admin dashboard
- 2026-04-16: Full code review completed (see session handoff in 11-vox/)
- 2026-04-16: **HTTP Basic Auth deployed** — sr-voice-bot rev 44, closes public /admin + /api/* leak
- 2026-04-16: **Phone cutover complete** — +12058318876 migrated Twilio→Vapi→(removed)→ElevenLabs direct
- 2026-04-16: **6 vertical demo clients** provisioned on ElevenLabs (auto_repair, hvac, contractor, dental, salon, professional_services)
- 2026-04-16: Web widget verified live on stellarisridge.com via Playwright
