---
name: cypress-nightlyreportaudit
description: Daily audit of the latest Jenkins Cypress nightly report - parses pass/fail data, classifies failures (broken vs flaky vs new regression vs shared-root-cause cluster), publishes a trend dashboard, then walks through fix candidates one by one with explicit confirmation before each edit.
---

Daily driver for the Jenkins Cypress nightly/weekly report. Point it at nothing and it finds
today's report itself, tells you what's actually broken vs what's just noisy, shows it as a
dashboard, and only touches test code when you say go — one fix at a time.

## Step 1 — Find the report

- If the user gave a path or pasted a report already, use that.
- Otherwise scan the Downloads folder for the newest `reports*` entry not already recorded in
  `.claude/cypress-audit/history/index.json`. Resolve Downloads dynamically
  (`$env:USERPROFILE\Downloads` in PowerShell, `$HOME/Downloads` in bash) — never hardcode a
  username.
- Jenkins archives the artifact as `reports.zip`; repeated browser downloads rename it to
  `reports - <timestamp>.zip` (and the browser may also leave an already-extracted
  `reports - <timestamp>/` folder next to it). Match `reports*`, prefer an already-extracted
  folder, otherwise extract the zip (Node's `adm-zip`, already a project dependency, or
  `Expand-Archive` in PowerShell).
- If nothing matches, ask the user for the path rather than guessing, or ask them to download
  `reports.zip` from the Jenkins build.
- A report folder holds `chrome_mochawesome/mochawesome.html`, and on Sundays also
  `firefox_mochawesome/mochawesome.html`. Parse whichever are present.

## Step 2 — Parse it

The pass/fail data is not a separate JSON file — `npm run chrome:mochawesome-merge` /
`firefox:mochawesome-merge` only emit the HTML report. The full dataset is HTML-entity-encoded
inside `<div id="report" data-raw="...">`. To read it:

1. Find `data-raw="` in the HTML, take everything up to the next unescaped `"`.
2. Decode `&quot;` `&#39;` `&amp;` `&lt;` `&gt;`.
3. `JSON.parse` the result — it has `.stats` (tests/passes/failures/pending/skipped/start/end)
   and `.results[].suites[]` (recurse through nested `.suites`). Only the top-level suite under
   each `results[]` entry carries `.file` — nested suites (the actual `describe` blocks) do not.
   Thread the parent's file name down through the recursion (pass it as an argument, falling
   back to it when a nested suite has no `.file` of its own) or every failure silently records an
   empty file name, which breaks cross-spec clustering (step 4) without erroring.
4. For every test with `state === "failed"`, record: spec file, full test title, and the
   **entire** `err.message` — not just its first line. Cypress error messages are multi-line,
   and the actually-identifying detail (which selector, "is being covered by another element",
   "is not visible because ... overflow: hidden", "Queried from:", etc.) is usually on line 2+,
   not line 1. Truncating to the first line collapses genuinely different bugs (e.g. a click
   blocked by an overlapping element vs. a click on a clipped element vs. a click during a page
   re-render) into one useless "cy.click() failed because this element:" bucket and breaks both
   clustering (step 4) and root-cause tracing (step 6). Keep the full message in the history
   file; truncate only when _displaying_ it in the dashboard.

Write a small throwaway Node script for this each time rather than trying to grep/sed it out.

## Step 3 — Persist to history

This is what makes the tool "daily" instead of a one-off:

- Save today's parsed result to `.claude/cypress-audit/history/<run-start-date>-<browser>.json`,
  keyed by the report's own `stats.start` date (not today's date — a nightly build spans
  midnight).
- Maintain `.claude/cypress-audit/history/index.json`: an ordered list of
  `{date, browser, sourceFolder, stats}` so a later run can tell what's already been ingested
  (skip re-parsing something already logged) and pull prior runs for comparison without
  re-opening every HTML file.
- Pull however many prior runs exist per browser for the classification step below — don't
  assume a fixed history depth.

## Step 4 — Classify every currently-failing test

For each test failing in the freshly parsed report, compare against that browser's history:

- **Broken (deterministic)** — same test, near-identical error message, in every run on record,
  not just today. `cy.task('...')` failures and byte-identical assertion messages across runs are
  the tell. Rank these highest: they are not timing flakiness, they are just unaddressed.
- **New regression** — passed in the immediately-prior run for that browser, fails today.
  Cross-reference `git log --since <prev run's end> --until <this run's end> -- cypress/` and
  list candidate commits as a lead, not a verdict.
- **Resolved** — failed in the prior run, passes today. Surface these as wins in the dashboard —
  this is the actual proof a past fix stuck, which is the thing worth tracking.
- **Intermittent** — fails in some runs, passes in others, no dominant pattern. Lowest individual
  priority unless it also shows up in the next bucket.
- **Cross-spec cluster** — regardless of the above buckets, group all of today's failures by a
  normalized error signature and flag any signature shared across tests in ≥2 different spec
  files. These matter disproportionately: one broken shared command can look like a dozen
  unrelated failures  . Normalize
  conservatively — strip only long hex/uuid-style ids (e.g. `dx-<hash>` DOM ids) and ISO
  timestamps.   — those usually distinguish genuinely different assertions, and
  collapsing them manufactures false clusters. When in doubt, under-cluster: a missed grouping
  costs a duplicate row in the dashboard; a false grouping sends the user chasing a shared root
  cause that doesn't exist. For each real cluster, grep `cypress/support/*.js` for the shared
  command/selector and cite the file:line when you can pin it down — say plainly when you can't.

## Step 5 — Publish the dashboard

- Load the `artifact-design` and `dataviz` skills before building it.
- Show: pass-rate trend across all history for each browser (not just today); today's raw stats;
  ranked fix candidates — cross-spec clusters first, then broken/deterministic, then new
  regressions, then intermittent — each with affected tests/specs, evidence, confidence, and a
  suggested fix direction; a "resolved since last run" section.
- Keep a permanent local copy per date, never overwritten: save the dashboard's HTML to
  `.claude/cypress-audit/dashboards/<run-start-date>-<browser>.html`, using the same date key as
  the history file it was built from. If a file already exists for that exact date+browser, this
  is a same-day re-run — overwriting _that_ file is fine. Never touch a different date's file to
  do it, and never collapse multiple days into one file the way a single recurring dashboard
  would.
- Publish each day's dashboard as its own Artifact — a new artifact per date, not an update to a
  previous day's. Record it in `.claude/cypress-audit/dashboards/index.json` as an ordered list of
  `{date, browser, path, url}` so a later run (or the user) can find any prior day's dashboard
  without re-publishing or guessing which link belongs to which night.
- Follow the Artifact link with a short spoken summary in the terminal, and mention the local file
  path too; don't let the dashboard be the only place the headline finding shows up.

## Step 6 — Ask before fixing, then go one at a time

Never edit test code automatically. After the dashboard:

1. Ask the user which candidate to start with (default suggestion: the top-ranked one).
2. For the chosen candidate, confirm the root cause for real — read the actual support file/spec,
   and, if the fix touches how a test waits on the app, the relevant frontend service/component
   backing that request or selector. Never guess a `cy.intercept` endpoint or a selector; verify
   from code, the same bar already used in this repo's `remove-wait-*` OpenSpec changes.
3. Show the specific, minimal edit before applying it. Everything under `cypress/e2e`,
   `cypress/support`, `cypress/fixtures` must follow this repo's CLAUDE.md Cypress rules: flat
   `cy` command sequences matching the target file's existing style, no loops/conditionals/
   helpers/classes/promise chaining unless that exact pattern already exists for that purpose, no
   hardcoded `cy.wait(ms)`, reuse existing selectors/commands/fixtures/intercepts before inventing
   new ones, and never change frontend behavior/styles/services/business logic to make a test
   pass — only `data-testid` attributes may be added to the frontend, and only with the user's
   go-ahead.
4. Apply only after explicit confirmation. Then move to the next candidate; stop whenever the
   user wants to stop.
5. If a candidate looks bigger than a quick fix (real behavior question, needs a Jira ticket, not
   just a test-code bug), say so  

## Notes

- Manually invoked (`/cypress-nightlyreportaudit`), meant to run once per morning after the nightly Chrome job
  . It is not scheduled on its own — offer the user a
  recurring `/loop` or a scheduled run if they'd rather not call it by hand.
- Stay inside `.claude/cypress-audit/` for bookkeeping and the specific Cypress files being fixed
  with explicit confirmation. Don't touch anything else.
