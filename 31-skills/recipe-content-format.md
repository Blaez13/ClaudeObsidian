---
type: skill
name: Recipe Content Format
for: Paw & Pantry / Lyra
updated: 2026-03-22
---

# Skill: Recipe Content Format

> Standard format for all Paw & Pantry recipe files.
> Every recipe Lyra creates uses this template.

## WHEN TO USE

- Any time a new recipe is created for P&P
- When converting an existing recipe into a P&P-branded file
- Before any recipe is reviewed by Sage

---

## RECIPE FILE FORMAT

```markdown
---
title: "[Recipe Name]"
type: recipe
dog-size: "[small/medium/large/all]"
prep-time: "[X minutes]"
cook-time: "[X minutes]"
servings: "[X days for a [size] dog]"
safety-checked: [true/false]
reviewed-by: Sage
approved: [true/false]
tags: [chicken, rice, beginner, etc.]
---

# [Recipe Name]

> [One warm, inviting sentence about this recipe. Barefoot Contessa energy.]

## Why Dogs Love This

[2-3 sentences. Benefits. Real food angle. Not clinical.]

## Ingredients

| Ingredient | Amount | Notes |
|-----------|--------|-------|
| [ingredient] | [amount] | [note if needed] |

## Instructions

1. [Step one]
2. [Step two]
3. [Continue...]

## Storage

[How to store. How long it keeps. Freezer tips.]

## Vet Note

> [Any important nutrition or safety note. Keep it brief and warm, not alarming.]

## Variations

- [Optional: swap X for Y]
- [Optional: add Z for extra protein]
```

---

## SAFETY REQUIREMENTS

Before any recipe is published:
1. Verify NO toxic ingredients (see [[dog-food-safety]])
2. Mark `safety-checked: true` in frontmatter
3. Sage reviews and sets `approved: true`

Common toxic ingredients to NEVER include:
- Onions, garlic, chives (all forms)
- Grapes, raisins
- Macadamia nuts
- Xylitol (sweetener)
- Chocolate, caffeine
- Raw yeast dough
- Alcohol
