---
name: design-kit
description: "Mandatory design protocol for ANY task that creates or changes what an interface looks like, feels like, or how it moves: pages, components, layouts, styling, themes, dashboards, forms, UI copy, HTML emails, app screens. Prevents generic AI-looking output; enforces direction before code, anti-slop rules, accessibility and verification."
license: MIT
---

# Design Kit

**Where the kit lives:** in this skill's own folder (the one containing this SKILL.md). Wherever the text below or any kit file says `design-kit/...`, read it as a path inside that folder: `design-kit/00-START-HERE.md` is `00-START-HERE.md` next to this file, and `design-kit/tools/slop-check.py` is `tools/slop-check.py` there. Project memory (`PRODUCT.md`, `DESIGN.md`) stays at the project root.

For ANY task that creates or changes what an interface looks like, feels like, or how it moves (pages, components, layouts, styling, themes, dashboards, forms, UI copy, HTML emails, app screens), you MUST first read `design-kit/00-START-HERE.md` and follow it. It points to the other files in `design-kit/` and says when to read each. This applies to small edits too: at minimum read `design-kit/01-anti-slop.md` and `design-kit/03-craft-floor.md`.

Non-negotiables (details in `design-kit/`):
1. **Precedence**: the user's request and pinned constraints win; then the project's existing design system (`DESIGN.md`, tokens); then the kit; your habits come last.
2. **Decide before you code**: for any new or redesigned surface, write a short Direction Contract (thesis, world, story, first viewport, signature move, risk) in your reply first. Commit fully; never ship the timid, category-default version.
3. **No AI slop**: do not default to indigo/violet gradients, gradient text, glow/glass decoration, equal icon-card grids, eyebrow labels above headings, nested cards, coloured side-stripe borders, emoji as icons, cream-plus-serif or neon-on-black by reflex, Inter/Space Grotesk/DM Sans/Playfair/Fraunces as an unexamined display voice, or generic AI copy ("Unlock the power of…", "Seamless", "Elevate your…"). A clear brief may earn any of these; habit may not.
4. **Content integrity**: write real, product-true copy and realistic data. Never invent prices, customers, testimonials, statistics or claims; label illustrative content and list what the user must supply.
5. **Craft floor**: WCAG AA contrast, visible keyboard focus, semantic HTML, reduced-motion support, all states designed (hover, focus, disabled, loading, error, empty), responsive from 390px to 1440px+, tokens instead of hard-coded values.
6. **Verify**: run `python3 design-kit/tools/slop-check.py <changed paths>` when a shell is available, and look at the rendered result at ~1440px and ~390px when you can. Say plainly what you could not verify.
7. Reply and write UI copy in the user's language.

Project memory lives in `PRODUCT.md` and `DESIGN.md` at the repo root (templates in `design-kit/templates/`). Read them if present.
