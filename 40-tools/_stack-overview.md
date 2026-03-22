---
type: reference
updated: 2026-03-22
---

# Tech Stack Overview

> Current tools in use. See individual tool folders for setup details.

## CRM & Automation

| Tool | Purpose | Status | Notes |
|------|---------|--------|-------|
| **GoHighLevel (GHL)** | CRM, automation, client sub-accounts | Active | Carl is admin |

## Website & Design

| Tool | Purpose | Status | Notes |
|------|---------|--------|-------|
| **Google Stitch** | Website builder (MCP-connected) | Active | SR client sites |
| **Webflow** | P&P brand site | Pending | Zev's domain |
| **Creatomate** | Video/graphic generation | Configured | API key set |

## AI & Agents

| Tool | Purpose | Status | Notes |
|------|---------|--------|-------|
| **Hermes** | Agent runtime (Ninimma) | Active | Localhost |
| **Paperclip** | P&P agent orchestration | Active | http://127.0.0.1:3100 |
| **Claude Code** | SR primary tool (this) | Active | Reads C:\Stellaris\ |
| **OpenRouter** | Model routing | Active | Free models for most tasks |

## APIs & Models

| Service | Key Env Var | Status | What It Does |
|---------|------------|--------|-------------|
| **Claude** | ANTHROPIC_API_KEY | Active | Premium output, client-facing work |
| **OpenRouter** | OPENROUTER_API_KEY | Active | Free models (Nemotron, Kimi K2.5) |
| **Gemini** | GEMINI_API_KEY | Active | Ninimma TTS (Capella), design gen |
| **GHL** | GHL_API_KEY | Active | Contacts, workflows, booking |
| **Creatomate** | CREATOMATE_API_KEY | Active | Video/graphic generation |
| **Late Zernio** | — | TBD | See [[apis/late-zernio]] |

## Payments

| Tool | Purpose | Status |
|------|---------|--------|
| **Stripe** | P&P membership billing ($4/mo) | Pending setup |

## Communication

| Tool | Purpose |
|------|---------|
| **Telegram** | Carl + Jarod communicate with Ninimma |
| **Claude Channels** | Additional channel integration |

## API Keys Checklist

See [[apis/_api-keys-checklist]] for what's configured and what's missing.
