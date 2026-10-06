# IT PMO Project Board (Demo)

A single-file Kanban board for tracking IT PMO work across a fictitious bank's project portfolio, written in vanilla HTML, CSS and JavaScript.

**Live demo:** https://joys0116.github.io/happylearning/

> **Demo / training project.** The bank, projects, people and tasks are all fictitious. This is not affiliated with, endorsed by, or an imitation of any real financial institution or its systems.

## Features

- **Four columns:** Backlog, In Progress, Blocked, Done. They sit side by side on desktop and stack on mobile (below 768px).
- **Add Task form** with inline validation: title, project, category, priority, assignee, due date and status. IDs are generated in the `UOB-ITPM-####` format.
- **Move cards** by drag and drop, or with the keyboard-friendly **Move ▸** menu on each card.
- **Delete** with an in-card "Delete? Yes / No" confirmation. There are no pop-up dialogs.
- **Filters** by project, assignee (text search) and priority, with a one-click **Clear filters**.
- **Header summary:** live counts per column and an **Overdue** count. Overdue cards are highlighted.
- **Email notification** for each new task via [FormSubmit](https://formsubmit.co) (optional, see below).
- **Accessible:** semantic markup, ARIA labels, and keyboard focus that's restored after every update.

Seed data uses due dates relative to today, so some tasks are always overdue.

## Run locally

No build, no install. Open the file:

```bash
open index.html
```

FormSubmit may reject requests from a `file://` page, so to test email, serve the folder instead:

```bash
python3 -m http.server 8000
```

Then visit http://localhost:8000.

## Email notifications (optional)

1. In `index.html`, set `FORMSUBMIT_ENDPOINT` at the top of the `<script>`:
   ```js
   const FORMSUBMIT_ENDPOINT = "https://formsubmit.co/ajax/you@example.com";
   ```
2. Add one task. FormSubmit sends a **one-time activation email**, and nothing is delivered until you click its link.
3. Each new task then sends an email with the subject `[UOB IT PMO]`.

While the placeholder `YOUR_EMAIL@example.com` is in place, the card is still added and a toast reads "Card added locally — email notification failed".

> ⚠️ Anything committed to this repo is public. If you put a real address in `FORMSUBMIT_ENDPOINT` and push it, that address will be visible to everyone.

## No persistence (by design)

Nothing is saved: there's no localStorage, cookies or backend. **Refreshing the page resets the board to the seed data.**

## Project structure

```
.
├── index.html                  # the whole app: markup + <style> + <script>
├── README.md
├── CLAUDE.md                   # notes for Claude Code
├── .gitignore
└── .github/workflows/ci-cd.yml # checks + GitHub Pages deploy
```

## CI/CD

[`.github/workflows/ci-cd.yml`](.github/workflows/ci-cd.yml) runs on every push and pull request to `main`:

1. **Secret scan** with [gitleaks](https://github.com/gitleaks/gitleaks).
2. **Project rules:** fails if `index.html` uses browser storage or cookies, `alert()`/`confirm()`, external scripts or stylesheets, `!important`, or any external host other than FormSubmit.
3. **HTML well-formedness** check (Python standard library, no npm).
4. **Deploy:** on pushes to `main`, `index.html` is published to GitHub Pages.

For the deploy step, the repo's **Settings → Pages → Source** must be set to **GitHub Actions**.

## Tech constraints

- Vanilla HTML/CSS/JS in a single `index.html`. No frameworks, libraries, CDNs, web fonts or image files.
- The only network call is FormSubmit's AJAX endpoint.
- CSS custom properties on `:root` for the palette and spacing; system font stack; Unicode/inline SVG icons.
