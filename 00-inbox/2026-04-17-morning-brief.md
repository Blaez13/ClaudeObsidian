---
date: 2026-04-17
type: morning-brief
author: claude-autonomous-review
session-reviewed: 2026-04-16-voice-bot-refactor
---

# Morning Brief — Voice-Bot Refactor Fresh-Eyes Review

Reading yesterday's session with no prior context. Here's what I see.

## The Pivot Is Correct

The ElevenLabs-as-source-of-truth decision is the right call for where SR is positioned. Eliminating Firestore-as-mirror removes the most predictable class of multi-tenant bugs — drift between what Firestore thinks an agent says and what ElevenLabs actually serves. For a phone-agent product where the client's brand voice is the product, drift is brand damage. The "fewer tools" constraint (no n8n, no Zapier, no Supabase, no Retool) is also sound for an agency at this stage. You don't want to debug five third-party integrations when a client's booking flow breaks at 11 PM.

The `sr-voice-templates` repo + `stamp.py` + fragment library approach directly serves the Facebook-group distribution angle and Jarod's pipeline: it lets you onboard a plumber in a morning instead of a sprint. That's the right competitive shape for where these clients are being sold.

## What Wasn't Verified

Three things in this plan are load-bearing assumptions that were never confirmed:

**Gemini 3.1 Flash Lite on emergency routing.** The LLM matrix puts Flash Lite on triage and emergency confirmation because it's fast and cheap. That's sensible reasoning, but Flash Lite is new enough that its tool-call reliability under voice latency pressure hasn't been documented publicly. Emergency routing is the highest-stakes node in the entire system — a false negative on "burst pipe flooding my basement" routes a homeowner to a booking flow while their kitchen floods. This assumption needs a concrete verification pass, even just reading ElevenLabs' own documentation on which models they've validated for Workflows tool calls.

**ElevenLabs post-call analysis API schema.** The session describes `conversation_analysis.evaluation_criteria` and `conversation_analysis.data_collection` as if these are verified field names, but the Task D deliverables clearly flag D.2 as "check ElevenLabs API docs for schema." The webhook handler and reconciliation job were built against this schema. If the actual ElevenLabs payload uses different field names, the analysis pipeline silently writes nothing to Firestore. This is a quiet failure mode — no errors, just missing data.

**The one-agent-per-client vs Workflows distinction.** The decision doc mentions "Workflows" and "Transfer-to-Number" but the implementation appears to use a single ElevenLabs agent with multiple conversation nodes, not ElevenLabs Workflows as a multi-agent graph. These are different product surfaces with different API shapes. The `workflow.yaml` in the templates repo suggests Workflows, but `stamp.py` was described as creating/updating a single agent. This needs to be disambiguated before Phase 5, because provisioning the wrong product tier is hard to debug under a test call.

## Where Bias Showed Up

The fragment library is the biggest complexity-bias flag. Ten composable prompt fragments, a YAML-driven assembler, a per-vertical intake form schema, 10 simulation scenarios — all for a product with zero paying clients. It's elegant architecture, and it will pay off at scale, but it's a lot of infrastructure to build before proving the first sale. The Opus code review focused on implementation correctness (good), but nobody stepped back and asked whether the fragment assembly layer was necessary for the first client.

There's also a mild leftover-LiveKit artifact: `vox-direct` is parked "to revisit when Gemini TTS matures" with no concrete trigger defined. That's fine, but it means two codebases exist in an ambiguous state. Worth a one-line decision: archive with a tag, or leave it. Leaving it in indefinite parking creates a false sense of optionality.

## Top Three to Action on Waking

**1. Implement the Layer 4 emergency-check route.** This is explicitly flagged in the handoff as unimplemented — schema staged, endpoint missing. The 10-scenario test suite includes emergency routing scenarios (#1, #7). Running Phase 5 verification against an incomplete 4-layer defense means your most important test scenarios will pass or fail against a system that isn't actually what you're shipping. Fifty lines of code, known scope, do this before Phase 5.

**2. Start the 911-disclaimer legal review.** This is a hard blocker before any client goes live, and legal reviews take time. Jarod's pipeline is described as "teed up" with first client onboarding as "the next hard deadline." If that's weeks, not months, the legal review clock should already be running. Don't wait until Phase 5 completes to initiate it.

**3. Verify Flash Lite tool-call behavior on ElevenLabs emergency node.** Quick targeted check: does ElevenLabs document which models support reliable tool calls within the emergency-routing latency budget (<10s)? If Flash Lite isn't validated for this, swap emergency confirmation to Flash before Phase 5 testing, not after.

The Twilio number ownership clause and LLM cost pass-through pricing are real items but not blockers today — they belong in the services contract iteration, which precedes the first client, not the first test call.

## Phase 5 Readiness

Not quite. The architecture is sound, the 44/44 tests pass, the Opus review was thorough, and the security hardening (HMAC, admin-auth middleware, tool-secret) is solid. But running Phase 5 with the Layer 4 emergency route missing means your verification suite doesn't actually test what you're planning to ship. Close that gap first — it's a focused task, not a rework.

**Recommendation: Implement Layer 4 emergency-check endpoint, then proceed to Phase 5. Address the legal review in parallel, not sequentially.**
