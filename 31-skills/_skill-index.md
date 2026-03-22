---
type: index
updated: 2026-03-22
---

# Skill Index

> Quick reference for all available skills.
> Skills are tools that agents reach for — not part of their identity.

## Available Skills

| Skill | File | When to Use |
|-------|------|-------------|
| Boardroom Perspectives | [[boardroom-perspectives]] | Multi-stakeholder decision analysis. 5 angles, synthesis, next action. Trigger: "Boardroom this" / `/board [topic]` |
| Social Media Automation | [[social-media-automation]] | Content → graphic → post pipeline. No manual uploads. |
| Recipe Content Format | [[recipe-content-format]] | P&P recipe markdown standard. Use for all Lyra recipes. |
| Dog Food Safety | [[dog-food-safety]] | Ingredient safety reference for P&P. Use before any recipe. |

## Adding a New Skill

1. Create `31-skills/[skill-name].md`
2. Add entry to this index
3. Update CLAUDE.md skills section
4. If it's for a specific agent, note it in their profile

## Skill File Format

```markdown
# Skill: [Name]

> [One-line description of what it does]

## WHEN TO USE
[Trigger conditions]

## [SKILL CONTENT]
[The actual instructions]
```
