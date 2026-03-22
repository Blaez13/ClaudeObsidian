---
type: checklist
updated: 2026-03-22
---

# API Keys Checklist

> What's configured, what's missing.
> DO NOT store actual key values here — this file is git-tracked.
> Keys live in environment variables or .env files.

## Configured (in Hermes / environment)

- [x] ANTHROPIC_API_KEY — Claude API
- [x] OPENROUTER_API_KEY — OpenRouter (free models)
- [x] GEMINI_API_KEY — Google Gemini (TTS + design)
- [x] GHL_API_KEY — GoHighLevel
- [x] CREATOMATE_API_KEY — Creatomate

## Not Yet Configured

- [ ] STRIPE_SECRET_KEY — needed for P&P membership ($4/mo billing)
- [ ] LATE_ZERNIO_API_KEY — needs documentation (see [[late-zernio]])
- [ ] Social platform tokens — Instagram, TikTok, Pinterest (for posting)

## Notes

- Keep .env in .gitignore — NEVER commit API keys
- If a key rotates, update environment variable AND note date here
