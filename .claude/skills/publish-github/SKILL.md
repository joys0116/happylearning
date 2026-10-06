---
name: publish-github
description: Security-scan, document and publish this project to a GitHub repo with Pages, CI/CD and About section
argument-hint: <github-repo-url>
disable-model-invocation: true
allowed-tools: Bash(git:*), Bash(gh:*), Bash(grep:*), Bash(ls:*), Bash(cat:*), Bash(find:*), Bash(du:*), Bash(file:*), Bash(python3:*), Bash(curl:*), Bash(pbcopy:*), Bash(pbpaste:*), Read, Write, Edit, Glob, Grep
---

# Publish this project to GitHub

Repo URL supplied by the user: `$ARGUMENTS`

Work through the steps below **in order**. The security scan runs first because scanning after a push is too late. Stop and ask the user whenever a step says so. Never force-push, never install software, and never print a full secret value (mask everything except the first 4 characters).

---

## Step 0: Preconditions

1. If `$ARGUMENTS` is empty, ask the user for the repo link. Accept any of these and parse `OWNER` and `REPO` from them, stripping any `.git` suffix:
   - `https://github.com/<owner>/<repo>[.git]`
   - `git@github.com:<owner>/<repo>.git`
   - a Pages URL, `https://<owner>.github.io/<repo>[/]`
2. Use `https://github.com/OWNER/REPO.git` as the remote URL, unless the user gave an SSH URL; in that case keep the SSH URL.
3. **Choose the mode:**
   - Run `git --version`. On macOS, `/usr/bin/git` is a stub until the Command Line Tools are installed, so a failure or `xcode-select: note: No developer tools were found` means git is unusable.
   - Run `gh --version` and `gh auth status`.
   - If all of these work, use **CLI mode**: Steps 1–7 below.
   - Otherwise, use **Web mode** (the section at the end of this file). Don't suggest installing anything; the user can't install software on this Mac. Web mode is the normal path here, not a fallback.
4. Read `CLAUDE.md` and `index.html` so the README and CI checks reflect the project's real behaviour and its hard constraints.

## Step 1: Security scan (this step blocks the push)

1. **`.gitignore`**: create it or extend it so it contains at least the entries below. Keep any existing entries.
   ```
   .DS_Store
   .env
   .env.*
   *.pem
   *.key
   *.p12
   *.pfx
   *.log
   node_modules/
   .claude/settings.local.json
   ```
2. **File list**: inside a git repo, use `git ls-files --cached --others --exclude-standard`. Otherwise, use `find . -type f -not -path './.git/*'` and filter it against `.gitignore`.
3. **Grep** every listed file (`grep -nIE`) for:
   | Kind | Pattern |
   |---|---|
   | Private key | `-----BEGIN [A-Z ]*PRIVATE KEY-----` |
   | AWS access key | `AKIA[0-9A-Z]{16}` |
   | GitHub token | `gh[pousr]_[A-Za-z0-9]{36,}` or `github_pat_[A-Za-z0-9_]{20,}` |
   | Slack token | `xox[abprs]-[A-Za-z0-9-]{10,}` |
   | Google API key | `AIza[0-9A-Za-z_-]{35}` |
   | Stripe live key | `(sk\|rk)_live_[A-Za-z0-9]{10,}` |
   | OpenAI/Anthropic key | `sk-(ant-)?[A-Za-z0-9_-]{20,}` |
   | JWT | `eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}` |
   | Generic secret | `(api[_-]?key\|secret\|token\|passw(or)?d)["']?\s*[:=]\s*["'][^"']{8,}["']` (case-insensitive) |
   | Connection string | `[a-z]+://[^/\s:]+:[^@\s]+@` |
4. **Sensitive files**: flag any listed file named `.env*`, `id_rsa*`, `*.pem`, `*.key`, `*.p12`, `*.pfx`, `credentials*`, or `*.sqlite`/`*.db`.
5. **Email addresses** (`[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}`): allow `*@example.com`, `*@example.org` and `noreply@anthropic.com`. Flag **every other address**, especially one in `FORMSUBMIT_ENDPOINT` in `index.html`: once pushed, that address is public and becomes a spam target.
6. **Large or binary files**: flag files over 5 MB (`find . -size +5M`) and any binary files (`file`) that the project doesn't obviously need.
7. **App security review**: follow the project's `cybersecurity-analyst` skill (`.claude/skills/cybersecurity-analyst/SKILL.md`) against `index.html`, limited to what it's worth catching before the page goes public:
   - Run its **Quick audit commands**.
   - For every `innerHTML` (or similar) hit, trace each interpolated `${...}` and flag any user-supplied value not wrapped in `escapeHtml()`, including attribute values. Toasts must use `textContent`.
   - Flag any network destination other than `FORMSUBMIT_ENDPOINT`, any persistence API, and any `eval`/`new Function`/string `setTimeout`.
   - Flag anything that makes the page look like a real bank's system (real logos, login or credential fields). The `UOB-ITPM-####` IDs and `[UOB IT PMO]` subject are required spec strings and are not findings.
   - Note, as **recommendations only** (they don't block the push): a missing Content-Security-Policy / `no-referrer` meta, and `_captcha: "false"` without a `_honey` field.
   Rate each finding Critical/High/Medium/Low with its CIA property. XSS and unexpected egress block the push; recommendations don't.
8. Show a findings table (file:line, kind, severity, masked value).
   - **Any finding (other than item 7's recommendations): stop and ask** the user how to proceed (remove it, add it to `.gitignore`, or confirm it's safe). Don't continue until they answer.
   - No findings: say "Security scan: clean" and continue.

## Step 2: README.md (create or update)

- If `README.md` exists, read it first. Update the sections listed below in place and keep any content the user wrote.
- Write it from the actual code and `CLAUDE.md`, not from assumptions. Sections:
  1. Title and a one-line description. This same line is used later for the About section.
  2. **Live demo**: `https://OWNER.github.io/REPO/`
  3. Disclaimer: a demo/training project for a **fictitious** bank, not affiliated with any real institution.
  4. Features: the columns, add/move/delete, drag & drop, filters, summary counts, overdue highlighting, keyboard support, and the email notification.
  5. Run locally: `open index.html`, or `python3 -m http.server 8000` (needed for FormSubmit).
  6. Email notifications: set `FORMSUBMIT_ENDPOINT` at the top of the script. Cover the one-time FormSubmit activation email, and warn that the address becomes public if it's committed.
  7. No persistence: a refresh resets to seed data, by design.
  8. Project structure: the file tree.
  9. CI/CD: what the workflow checks and how it deploys to GitHub Pages.
  10. Tech constraints: vanilla HTML/CSS/JS, a single file, no dependencies.

## Step 3: CI/CD workflow

Create or update `.github/workflows/ci-cd.yml`. If the file already exists, merge in these jobs instead of overwriting the user's custom steps. Base it on:

```yaml
name: CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: pages-${{ github.ref }}
  cancel-in-progress: false

jobs:
  checks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Secret scan (gitleaks)
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Project constraint checks
        run: |
          set -e
          test -f index.html || { echo "index.html missing"; exit 1; }
          fail=0
          check() { if grep -nE "$1" index.html; then echo "::error::$2"; fail=1; fi; }
          check 'localStorage|sessionStorage|indexedDB|document\.cookie' 'Persistence APIs are not allowed'
          check '(^|[^.a-zA-Z_])(alert|confirm)\(' 'Native alert()/confirm() are not allowed'
          check '<script[^>]+src=' 'External scripts are not allowed'
          check '<link[^>]+href="?https?:' 'External stylesheets/fonts are not allowed'
          check '!important' '!important is not allowed'
          hosts=$(grep -oE 'https?://[A-Za-z0-9.-]+' index.html | sed -E 's#https?://##' | sort -u | grep -vE '^(formsubmit\.co|www\.w3\.org)$' || true)
          if [ -n "$hosts" ]; then echo "::error::Unexpected external hosts: $hosts"; fail=1; fi
          exit $fail

      - name: HTML well-formedness
        run: |
          python3 - <<'PY'
          from html.parser import HTMLParser
          VOID = {"area","base","br","col","embed","hr","img","input","link","meta","source","track","wbr"}
          class P(HTMLParser):
              def __init__(self):
                  super().__init__(); self.stack = []; self.errors = []
              def handle_starttag(self, tag, attrs):
                  if tag not in VOID: self.stack.append((tag, self.getpos()))
              def handle_endtag(self, tag):
                  if tag in VOID: return
                  if not self.stack or self.stack[-1][0] != tag:
                      self.errors.append(f"Unexpected </{tag}> at line {self.getpos()[0]}")
                      while self.stack and self.stack[-1][0] != tag: self.stack.pop()
                  if self.stack: self.stack.pop()
          p = P(); p.feed(open("index.html", encoding="utf-8").read())
          unclosed = [f"<{t}> line {l}" for t, (l, _) in p.stack if t not in ("html","head","body","p","li")]
          for e in p.errors + [f"Unclosed {u}" for u in unclosed]: print(f"::error::{e}")
          raise SystemExit(1 if p.errors or unclosed else 0)
          PY

  deploy:
    needs: checks
    if: github.event_name != 'pull_request' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - uses: actions/checkout@v4
      - name: Assemble site
        run: |
          mkdir _site
          cp index.html _site/
          touch _site/.nojekyll
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: _site
      - id: deployment
        uses: actions/deploy-pages@v4
```

Before committing, run the "Project constraint checks" and "HTML well-formedness" logic locally against `index.html`. If they fail, stop and report the failures. Don't weaken the checks to make them pass. If `python3` isn't usable locally (the macOS stub), skip only the well-formedness check, say so, and leave it to CI.

## Step 4: Push to GitHub

1. If this isn't a git repo, run `git init -b main`. If it is, make sure you're on `main`; if you aren't, ask before switching or renaming.
2. Remote:
   - No `origin`: `git remote add origin <url>`.
   - `origin` already points at a different repo: **ask** before changing it.
3. Run `gh repo view OWNER/REPO`. If the repo doesn't exist, **ask** whether to create it as public or private (`gh repo create OWNER/REPO --public|--private`). Pages on a private repo needs a paid plan.
4. Run `git add -A`, then **re-run the Step 1 grep against `git diff --cached --name-only`**. On any new finding, stop and ask.
5. Commit with a descriptive message, e.g. `Publish IT PMO board: README, CI/CD workflow, Pages deploy`, ending with:
   ```
   Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
   ```
6. Run `git fetch origin`. If `origin/main` exists and isn't an ancestor of `HEAD` (unrelated or diverged history), **stop and ask**: rebase, merge, or abort. Never use `--force`.
7. `git push -u origin main`.

## Step 5: GitHub Pages (create or update)

1. Run `gh api repos/OWNER/REPO/pages`.
   - 404: `gh api -X POST repos/OWNER/REPO/pages -f build_type=workflow`
   - It exists but `build_type` isn't `workflow`: `gh api -X PUT repos/OWNER/REPO/pages -f build_type=workflow`
2. If the API refuses (for example a private repo on a free plan), report the error and continue to the next step, using `https://OWNER.github.io/REPO/` as the expected URL.
3. Read `html_url` from `gh api repos/OWNER/REPO/pages --jq .html_url`. That's `PAGES_URL`.
4. If Pages was enabled only after the push, re-run the deploy with `gh workflow run ci-cd.yml --ref main`.

## Step 6: About section, with the Pages link

1. Run `gh repo view OWNER/REPO --json description,homepageUrl,repositoryTopics`.
2. If there's already a description that differs from the README one-liner, **ask** before overwriting it.
3. Run:
   ```
   gh repo edit OWNER/REPO \
     --description "<README one-liner>" \
     --homepage "PAGES_URL" \
     --add-topic kanban,project-management,pmo,vanilla-js,html,css,github-pages,demo
   ```

## Step 7: Verify and report

1. Run `gh run list --workflow ci-cd.yml --limit 1` once. Don't poll in a loop. If the run is still in progress, tell the user they can watch it with `gh run watch`.
2. Re-check the About section with `gh repo view OWNER/REPO --json description,homepageUrl,repositoryTopics`.
3. End with a summary table:

| Item | Result |
|---|---|
| Security scan | clean / findings resolved |
| Files created/updated | … |
| Commit | short SHA |
| Repo | https://github.com/OWNER/REPO |
| GitHub Pages | PAGES_URL |
| Actions run | run URL + status |
| About section | description / homepage / topics set |

Then list any manual follow-ups, for example: "the first Pages deploy can take 1–2 minutes" or "click the FormSubmit activation email".

---

# Web mode (no working `git` or `gh`)

Use this when Step 0 found that `git` or `gh` can't be used. Claude does the local work and the checks. The user does the clicks on github.com, one numbered step at a time, and Claude waits for them to say "done" before checking and moving on.

Lessons from earlier runs: pasted file contents can silently fail to save, and topics are only kept once they turn into tags. Always verify each step instead of assuming it worked.

## W1: Inspect the repo
- Run `curl -s https://api.github.com/repos/OWNER/REPO`. If it returns 404, ask the user to create the repo at https://github.com/new (Public; no README, .gitignore or licence), then re-check.
- List what's already there: `curl -s https://api.github.com/repos/OWNER/REPO/contents/` (look at the `name` and `size` fields), plus `.../contents/.github/workflows`. Note any file that shouldn't be public, such as saved preview pages, configs, or anything flagged by the scan.
- Check Pages: `has_pages` in the repo JSON, and whether `https://OWNER.github.io/REPO/` loads (`curl -s -o /dev/null -w '%{http_code}'`).

## W2: Security scan
- Run Step 1 against the **local** files. **Quote filenames:** names with spaces broke unquoted `grep $(find ...)` before, so use `find ... -print0 | xargs -0 grep ...` or a loop over `"$f"`.
- Also scan files that are in the repo but not local: fetch each one from `https://raw.githubusercontent.com/OWNER/REPO/main/<path>` into the scratchpad and run the same patterns.
- For anything flagged in the repo, give the user the exact delete or edit steps (open the file, **⋯ → Delete file**, then commit). Note that a deleted file stays in the git history, so a real secret has to be **revoked or rotated** as well.
- Stop and ask on any finding, as in Step 1.

## W3: Write the local files
Write `README.md` (Step 2), `.gitignore` (Step 1) and `.github/workflows/ci-cd.yml` (Step 3) into the project folder. Use the Pages URL `https://OWNER.github.io/REPO/` in the README.

## W4: Guide the uploads, one at a time
For each file, copy its contents to the clipboard with `pbcopy < <file>` right before that step, and tell the user it's on the clipboard.
1. **README.md:** **Add file → Upload files** (drag it in) or **Add file → Create new file**, name it `README.md`, paste, then **Commit changes**.
2. **.gitignore:** **Add file → Create new file**, name it `.gitignore`, paste, then commit. Dot-files are hidden in Finder, so the user needs to create this one rather than upload it.
3. **Pages source first:** go to `https://github.com/OWNER/REPO/settings/pages` → **Build and deployment → Source: GitHub Actions**. This must happen **before** the workflow commit, or the deploy job fails.
4. **Workflow:** **Add file → Create new file**, name it `.github/workflows/ci-cd.yml` (typing `/` makes folders), paste with Cmd+V, and check that line 1 reads `name: CI/CD`. If the file already exists, open it, click the ✏️ pencil, press Cmd+A, then Cmd+V.

After each "done", verify the step:
- Fetch the raw file and `diff` it against the local copy. An empty or mangled workflow shows up on GitHub as "No event triggers defined in `on`".
- Then check the latest run: open `https://github.com/OWNER/REPO/actions` in the browser pane, or use `curl -s 'https://api.github.com/repos/OWNER/REPO/actions/runs?per_page=1'` and look at `status` and `conclusion`. If a run failed, open it and read the annotations to find the cause.

## W5: About section
Tell the user where it is: the right-hand column on the repo page, ⚙️ next to **About**. On a narrow window it moves to the top, and it's only editable on their own repo while signed in. They should set:
- **Description:** the README's one-line description.
- **Website:** tick **Use your GitHub Pages website**.
- **Topics:** suggest topics that fit the project. Each one must be followed by **Space or Enter** so it becomes a tag, or it won't be saved.

Verify with `curl -s https://api.github.com/repos/OWNER/REPO` (fields `description`, `homepage`, `topics`) and check the topics for typos.

## W6: Report
Use the same summary table as Step 7, built from what you verified: the API, the raw files, the latest Actions run, and whether the Pages URL loads.
