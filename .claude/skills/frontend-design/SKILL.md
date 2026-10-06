---
name: frontend-design
description: Visual design direction for the IT PMO Kanban board (index.html). Use when restyling the board, adding or reshaping UI (cards, columns, header summary, Add Task form, filters, toasts), or making the page look less templated, while staying inside the project's hard constraints (single-file vanilla HTML/CSS/JS, system fonts, corporate blue, no persistence).
license: Adapted from anthropics/skills frontend-design. Complete terms in LICENSE.txt
---

# Frontend Design — IT PMO Kanban Board

This is the Anthropic `frontend-design` skill adapted to this repository. The general philosophy is in `UPSTREAM.md`. Read it when you need the full process. **When the upstream guidance and this file disagree, this file and `CLAUDE.md` win.**

## The brief (already decided, do not re-ask)

- **Subject**: An internal IT Project Management Office board for a *fictitious* bank, used for demos and training.
- **Audience**: PMO leads, project managers, and IT delivery staff triaging work across four statuses. They scan many cards, act quickly, and use the keyboard a lot.
- **Primary job**: Show at a glance what is blocked, overdue, or high priority, and let people add, move, and delete tasks with as little friction as possible.
- **Tone**: Calm, trustworthy, and dense with information. It should feel like a well-made internal operations tool, not a marketing page. There is no "hero". The board itself is the first thing people see.

## Hard constraints (from CLAUDE.md, never trade these for aesthetics)

- Everything stays in the one `index.html`: one `<style>`, one `<script>`. No frameworks, CDNs, **web fonts**, image files, or npm.
- Typography uses the **system font stack only**. Distinctiveness comes from scale, weight, spacing, and tabular figures, not from a new typeface. (This overrides the upstream advice to "choose typefaces deliberately".)
- Icons are Unicode or inline SVG.
- Palette is **corporate blue** and must be defined as custom properties on `:root` (`--blue-*`, `--ink*`, `--line`, `--surface`, `--bg`, `--prio-*`, `--status-*`, `--ok/warn/danger(-bg)`, `--focus`, `--sp-1..6`). Extend tokens there. Never hard-code hex values in rules, and never use `!important`.
- Neutral "IT PMO" text wordmark. No real bank logos, colors that imitate a real bank's brand, or imitation of official systems.
- The header note saying data resets on refresh must stay visible.
- Columns sit side by side on desktop and stack below **768px**.
- No `alert()`/`confirm()`. Errors appear inline, and delete uses the in-card "Delete? Yes / No" toggle.

## Where boldness is allowed

Spend it in **one** place. Good candidates for this board:
1. **Status and urgency encoding.** Make blocked, overdue, and critical cards unmistakable at a glance using more than color alone: edge bars, icons, `is-overdue` treatment, and the header summary counts.
2. **The header summary.** A crisp, glanceable read of the whole board (it always counts the full task list, not the filtered view).

Everything else (form, filters, toasts) stays quiet and disciplined.

## Avoid these tells (project-specific)

- The generic SaaS-card kit: identical soft-shadow cards with one radius everywhere. Vary emphasis by status and priority instead.
- All-caps eyebrow labels above every heading, `A · B · C` meta strings, `→` appended to buttons.
- Decorative gradients, entrance animations on every card, and hover lifts on every element. Motion should only answer user actions (move, drag-over, delete confirm, toast) and must respect `prefers-reduced-motion`.
- Emoji as icons.

## Copy rules

- Buttons say what happens: "Add task", "Move to In Progress", "Delete". Keep a verb the same throughout a flow, so "Add task" leads to an "Task added" toast.
- Errors are specific and tell the user how to fix the problem ("Due date is required"), without apologizing.
- Empty columns invite action ("No tasks here yet. Drag a card in or add one.").
- Keep the spec strings exactly as they are: `UOB-ITPM-####` IDs and the `[UOB IT PMO]` email subject.

## Process for any visual change

1. **Plan**: Write down the token changes (new or changed `:root` variables), the affected components, and an ASCII sketch if the layout changes.
2. **Check against the brief and constraints**: Is anything here a generic default? Does it break a hard constraint? Revise before coding.
3. **Build** through the existing architecture. Styles go in the `<style>` block. Card markup changes go **only** in `renderCard()` / `renderBoard()`. Never patch card DOM elsewhere.
4. **Check the result**: Open `index.html` in the browser pane and test at desktop width and at under 768px. Tab through a card's actions to confirm focus is still visible and is restored after re-render (`state.focusAfterRender`). Turn on reduced motion. Check contrast of `--ink-muted` and the priority pills on `--surface`.
5. Before finishing, remove one decorative element you don't need.

## Quality floor (non-negotiable)

- WCAG AA contrast (4.5:1 for text, 3:1 for UI and large text). Status must never be conveyed by color alone.
- Visible `--focus` ring on every interactive element.
- Touch targets of at least 44×44px when stacked on mobile.
- No horizontal page scroll at 360px width.
