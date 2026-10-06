---
name: cybersecurity-analyst
description: Security review and threat modeling for the IT PMO Kanban board (index.html). Use before publishing/sharing the board, when touching rendering (innerHTML/template strings), the Add Task form, or the FormSubmit email call, or when asked for a security assessment. Applies STRIDE, OWASP, CIA triad and defense-in-depth to this single-file client-side app.
---

# Cybersecurity Analyst — IT PMO Kanban Board

This is a project-scoped version of the `cybersecurity-analyst` skill from `rysweet/amplihack`. The full general frameworks (CIA triad, STRIDE, MITRE ATT&CK, NIST, incident response, and so on) are in `UPSTREAM.md`, with a cheat sheet in `QUICK_REFERENCE.md`. Use them for depth. This file captures what actually matters for this repo.

Scope is **defensive review of this project only**. Do not run scanners against third-party hosts, including formsubmit.co.

## System model (already established)

- **What it is**: A single static `index.html` demo board for a *fictitious* bank. It has no backend, no auth, and no persistence (refreshing resets it to seed data).
- **Trust boundary**: The browser. All task data lives in the in-memory `state`.
- **Only egress**: `fetch` POST to `FORMSUBMIT_ENDPOINT` (`https://formsubmit.co/ajax/<email>`) when a task is added, with a timeout of `FORMSUBMIT_TIMEOUT_MS`.
- **Inputs**: Add Task form (title ≤80, description ≤500, assignee ≤60, selects, due date), filter controls, drag/drop events, and `data-*` attributes on board elements.
- **Hosting**: Opened as `file://` locally, or published through GitHub Pages (`.github/workflows/ci-cd.yml`, `/publish-github` command).

## Assets and CIA focus

| Asset | C | I | A | Notes |
|---|---|---|---|---|
| Recipient email in `FORMSUBMIT_ENDPOINT` | **High** | Med | — | Becomes public in page source once published. It is spam/abuse bait, and anyone can POST to it. |
| Rendered board (DOM) | — | **High** | Med | XSS risk from user text in template strings |
| Notification content | Med | Med | Low | Task text is sent to a third party (FormSubmit) |
| Demo realism/branding | — | **High** | — | Must not imitate a real bank's systems (phishing look-alike risk) |

## STRIDE checklist for this app

- **Spoofing**: Is the "fictitious bank" status still clear? Are there no real logos, official-looking login prompts, or credential fields? The `[UOB IT PMO]` subject and `UOB-ITPM-####` IDs are required spec strings. Flag anything that goes further toward impersonation.
- **Tampering / XSS (highest priority)**: Every user-supplied value inserted through template strings into `innerHTML` must go through `escapeHtml()`. This includes values in attributes (`data-id`, `datetime`, `data-status`). Toasts must use `textContent`. Look for `innerHTML`, `insertAdjacentHTML`, `outerHTML`, `document.write`, `eval`, `new Function`, `setTimeout(string)`, inline `on*=` handlers built from data, and `href`/`src` built from user input.
- **Repudiation**: Not applicable (demo, no accounts). Don't add logging that sends data anywhere.
- **Information disclosure**: No secrets, tokens, or real personal data in the source or seed data. The FormSubmit email is visible to anyone, so recommend a dedicated alias address and FormSubmit's hashed-endpoint option instead of a raw personal address. Make sure the `YOUR_EMAIL@example.com` placeholder guard in `notifyNewTask()` stays. Check `.gitignore` coverage (`.env`, keys, `.mcp.json`, `settings.local.json`).
- **Denial of service**: The client-side impact is small. Confirm the length limits are enforced in JS validation, not only through `maxlength`. Confirm the board can't be wedged by huge input, the fetch timeout aborts, and `isSending` prevents double-submits.
- **Elevation of privilege**: Not applicable (no roles). Make sure no `eval`-like paths exist.

## Defense-in-depth checks (allowed within the constraints)

- Escaping at every sink, plus server-independent validation in `handleSubmit()`.
- Optional `<meta http-equiv="Content-Security-Policy">` that keeps everything in the single file. For example: `default-src 'none'; style-src 'unsafe-inline'; script-src 'unsafe-inline'; connect-src https://formsubmit.co; img-src data:; base-uri 'none'; form-action 'none'`. Inline script and style require `'unsafe-inline'` (or hashes), so treat CSP as a supplementary control. `connect-src` locks down egress, which enforces the "only network call is FormSubmit" rule.
- `<meta name="referrer" content="no-referrer">` to avoid leaking the page URL to FormSubmit.
- FormSubmit fields: `_captcha: "false"` disables FormSubmit's captcha. Note the spam trade-off, and consider the `_honey` honeypot field.
- No persistence APIs (`localStorage`/`sessionStorage`/IndexedDB/cookies). Their absence is a project requirement, and it also reduces data-at-rest exposure.
- Supply chain: no third-party scripts or CDNs (already required). Pin GitHub Actions versions in `ci-cd.yml` and keep their permissions minimal.

## Quick audit commands

```bash
grep -nE "innerHTML|insertAdjacentHTML|outerHTML|document\.write|eval\(|new Function" index.html
grep -nE "fetch\(|XMLHttpRequest|sendBeacon|WebSocket|EventSource|<script src|<link " index.html
grep -nE "localStorage|sessionStorage|indexedDB|document\.cookie" index.html
grep -nE "FORMSUBMIT_ENDPOINT|_captcha|_honey|_subject" index.html
grep -rnE "(api[_-]?key|secret|token|password)\s*[:=]" --include=*.html --include=*.yml --include=*.json .
```

For each `innerHTML` hit, trace every interpolated `${...}` back to its source, and confirm it is either a constant or wrapped in `escapeHtml()`.

## Analysis process

1. **Scope**: Identify which change, file, or release is being reviewed.
2. **Attack surface**: Map the inputs and sinks listed above for the code being reviewed.
3. **Threat model**: Walk through STRIDE using the checklist above.
4. **Verify**: Run the audit greps, read the code paths, and optionally confirm in the browser pane with a payload such as `<img src=x onerror=alert(1)>` in title, description, and assignee. It should render as literal text.
5. **Risk-rate** each finding (likelihood × impact: Critical/High/Medium/Low) and give the CIA property affected.
6. **Recommend** fixes that respect `CLAUDE.md`: no new dependencies, no new network destinations, and no persistence.
7. **Report**: Rank findings by severity. For each finding give the file:line, the issue, a concrete exploit scenario, the fix, and the residual risk. Say explicitly when something was checked and found clean.

## Out of scope

Pentesting FormSubmit or GitHub, social engineering, and anything offensive beyond a harmless local XSS proof inside this page.
