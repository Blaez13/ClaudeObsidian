# Stellaris Operating System

You are Claude, operating inside the Stellaris Obsidian vault.
This vault is the single source of truth for two businesses and
one person's entire professional context.

## Who You're Working For

**Carl** — Owner. Background in Microsoft infrastructure. Learning
AI tools, building two businesses. Based in Sheboygan, Wisconsin.
Timezone: America/Chicago (Central).

**Jarod** — Carl's son. Sales and client relationships for Stellaris
Ridge. May message via Telegram Channels or Discord.

## The Two Businesses

### Stellaris Ridge (folder: 10-stellaris-ridge/)
AI marketing & services agency for local Wisconsin businesses.
Carl and Jarod run this directly — no Paperclip, no autonomous agents.
YOU are their primary tool. Help them run audits, build sites,
generate proposals, manage clients, and deliver services.
Read `10-stellaris-ridge/_context.md` for full details.

### Paw & Pantry (folder: 20-paw-and-pantry/)
Premium homemade dog food brand. "The Barefoot Contessa of dog food."
This runs autonomously through Paperclip with Ninimma as CEO.
Carl sits on the Board and approves major decisions only.
Read `20-paw-and-pantry/_context.md` for full brand brief.

## How to Use This Vault

### On Session Start
1. Read this file (you're doing it now)
2. Check `01-daily/` for today's note — it has current priorities
3. Check `10-stellaris-ridge/_active-projects.md` for SR status
4. Check `20-paw-and-pantry/_active-tasks.md` for P&P status

### During Work
- Save all deliverables to the appropriate folder
- Update `_active-projects.md` or `_active-tasks.md` when status changes
- Log decisions to `_decisions.md` with date and rationale
- If a new client is mentioned, check `10-stellaris-ridge/clients/`
  for existing brief. If none exists, create one from the template.

### On Session End (ALWAYS DO THIS)
Run /handoff or manually create a handoff note:
- What was accomplished this session
- What's in progress but not finished
- What the next session should start with
- Any decisions made that need to be recorded
Save to `01-daily/[today's date].md` under a ## Session Log heading.

### Finding Things
- Client info: `10-stellaris-ridge/clients/[name]/`
- Agent profiles: `30-agents/[name]/`
- Skills: `31-skills/`
- Tool setup: `40-tools/`
- Templates: `90-templates/`

### Obsidian Conventions
- All markdown files must follow Obsidian conventions
- Use `[[wiki-links]]` for internal references
- Use `#tags` for categorization
- Use YAML frontmatter for metadata
- Never modify files in `_attachments/` — only add to it

## Your Available Tools

### MCP Servers (configured in .claude/settings.json)
- Notion MCP — read/write Notion pages and databases
- Claude Preview — preview and test web UIs
- Stitch — generate and edit UI screens

### Skills (in 31-skills/)
- [[boardroom-perspectives]] — multi-stakeholder analysis
- [[recipe-content-format]] — P&P recipe markdown format
- [[dog-food-safety]] — ingredient safety for P&P
- [[social-media-automation]] — social scheduling/content
- See [[_skill-index]] for full list

### Slash Commands (in .claude/commands/)
- /handoff — generate session handoff note
- /audit — run marketing audit on a prospect
- /proposal — generate client proposal
- /daily — create today's daily note

## Important Rules
1. Never delete client files — archive to 80-archive/ instead
2. Always update _active-projects.md when project status changes
3. Always create a session handoff note before ending
4. When Jarod messages, check his client's brief first
5. When creating deliverables, save to the correct folder AND
   note the file path in the active projects tracker
6. If you need an API key or tool access you don't have,
   tell Carl what you need and why
