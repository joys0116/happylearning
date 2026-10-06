# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A demo/training IT PMO Kanban board for a **fictitious** bank, delivered as a single file: `index.html` (markup + one `<style>` + one `<script>`). There is no build, no package manager, no tests, and no linter.

## Running

Open the file directly — no server needed:

```bash
open index.html
```

FormSubmit may reject requests from a `file://` page (null origin). If email delivery needs testing, serve the folder instead:

```bash
python3 -m http.server 8000
```

## Hard constraints (from the original spec — do not violate)

- Vanilla HTML/CSS/JS only. No frameworks, libraries, CDNs, web fonts, image files, bundlers or npm. Icons are Unicode/inline SVG; fonts are the system stack.
- Everything stays in the one `index.html`.
- **No persistence**: no localStorage/sessionStorage/IndexedDB/cookies. A refresh resets to seed data on purpose, and the header note saying so must stay.
- The only network call is FormSubmit's AJAX endpoint. Never send data anywhere else.
- No native `alert()`/`confirm()`. Validation errors are shown inline; delete uses the in-card "Delete? Yes / No" toggle.
- Branding: neutral "IT PMO" text wordmark and a corporate blue palette. No real bank logos or imitation of official systems. The `UOB-ITPM-####` ID format and the `[UOB IT PMO]` email subject are plain strings the spec requires.
- CSS uses custom properties on `:root` for palette and spacing; no `!important`. Columns sit side by side on desktop and stack below 768px.

## Architecture (inside the `<script>`)

- **Single source of truth**: `state = { tasks, filters, nextId, pendingDeleteId, openMoveId, focusAfterRender, isSending }`. UI toggles (open Move menu, pending delete confirm) live in `state`, not in the DOM.
- **Render from state**: `renderBoard()` rebuilds all four columns via `renderCard()` and calls `renderSummary()` (header counts, always over the full task list) and `restoreFocus()`. Don't edit card DOM anywhere else. Change `state`, then call `renderBoard()`.
- **Mutations**: `addTask()`, `moveTask()`, `deleteTask()` change `state.tasks` only and don't render; callers render. `applyFilters()` is a pure filter over `state.tasks`.
- **Escaping**: every user-supplied string inserted via template strings must go through `escapeHtml()`. Toasts use `textContent`.
- **Events are delegated** on `#board` (clicks dispatched by `data-action` + `data-id`, plus native HTML5 drag/drop). Because the board's HTML is replaced on each render, never bind listeners to individual cards.
- **Keyboard focus**: since re-rendering destroys the focused element, set `state.focusAfterRender` (`{id, action}` or `{selector}`) before `renderBoard()` to restore focus.
- **Dates**: handled as local `YYYY-MM-DD` strings (`toISODate`, `todayISO`) and compared as strings. Don't use `toISOString()` for dates — it shifts the day across UTC. Seed due dates are relative to today (`daysFromToday`), so some tasks are always overdue.
- **Add Task flow** (`handleSubmit`): validate → add the card optimistically → render → success toast → `notifyNewTask()` in try/catch while the button shows "Sending…" (`setSending`). On failure the card stays and a warning toast reads "Card added locally — email notification failed".
- **FormSubmit**: `FORMSUBMIT_ENDPOINT` at the top of the script is the one place to set the email address. `notifyNewTask()` deliberately throws while it still contains the `YOUR_EMAIL@example.com` placeholder. It also treats a `{success: "false"}` response body as a failure and times out after `FORMSUBMIT_TIMEOUT_MS`. A new address needs one-time activation: the first submission sends a confirmation email whose link must be clicked before anything is delivered.
- Dropdown options (projects, categories, priorities, statuses) come from the constants `PROJECTS`, `CATEGORIES`, `PRIORITIES`, `STATUSES` and are filled into the form and filter selects at init. Change the lists there, not in the markup.
