## Rules for this repo

This is the master copy of the app playbook. Apps copy `APP_PLAYBOOK.md`, `SYNC.md` and one file from `stacks/` into `docs/playbook/` through a weekly sync PR, so every change here reaches every app.

- **This repo is public.** No secrets, keys, customer data, private repo contents, or unreleased product plans. Examples from an app are one-line lessons, not internals.
- **Where things go:**
  - A rule or gotcha that applies on any stack → `APP_PLAYBOOK.md` (gotchas in §10).
  - Commands, paths, config and gotchas for one stack → that stack's file in `stacks/` (gotchas in P8).
  - Changes to the app templates (`ROADMAP.md`, `AGENTS.md`, `docs/APP_NOTES.md`, the sync workflow) → the `app-starter` repo, not here.
- **Write rules generically, then give the case.** State the rule in terms of `<resource>` (the app's main shared object), then add one line: *Example (App name):* what happened.
- **Keep section numbers stable.** Apps and their notes point at `§n` and `Pn`. If you must renumber, say so in the PR description.
- **Every stack profile uses the same P1–P9 sections.** Start a new profile by copying an existing one's headings.
- **Don't put anything in these files that changes on every sync** (dates, commit hashes), or every app gets an empty sync PR each week.
- **Branches:** `dev-<topic>` from `main` (`dev-lesson-<topic>` for promoted lessons). Small PRs; never commit to `main`.
- **Commits:** authored as the repo owner, with no `Co-Authored-By` lines. Short, imperative messages.
