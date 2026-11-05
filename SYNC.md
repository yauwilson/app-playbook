# Keeping the playbook in sync

How lessons move between the master repo (`app-playbook`) and each app repo.

## The two kinds of files

| In each app | Owned by | Edited where |
|---|---|---|
| `docs/playbook/APP_PLAYBOOK.md` | Master | **Only in `app-playbook`.** The app's copy is replaced by every sync |
| `docs/playbook/stacks/<profile>.md` | Master | **Only in `app-playbook`** |
| `ROADMAP.md`, `AGENTS.md` | App | In the app. Copied once from `app-starter` when the app is created, never synced |
| `docs/APP_NOTES.md` | App | In the app. App-only quirks, plus the "To promote" list |

## 1. Marking something to send back to the master

Use the test: **"Would I want this in my next app?"** If yes, add it to the **To promote** list at the top of the app's `docs/APP_NOTES.md`, in the same PR as the fix:

```markdown
## To promote

- [ ] **→ profile nextjs-supabase P8:** React can leave the router update parked after a Server Action; re-render every 300 ms while saving. (#14, 2026-11-05)
- [ ] **→ playbook §10:** A side effect hidden in one entry point. (#9, 2026-10-28)
- [ ] **→ starter ROADMAP:** Add a "Decided against" section. (2026-11-02)
```

- **Target** comes first: `playbook §n`, `profile <name> Pn`, or `starter ROADMAP / AGENTS` (these PRs go to `app-starter`, not `app-playbook`).
- **Then the lesson** in one or two lines, plus the PR it came from and the date.
- Don't edit the files in `docs/playbook/`. The next sync would overwrite the edit.

What stays in the app instead: features, bugs, decisions (ROADMAP), app rules (AGENTS), quirks only this app has (APP_NOTES, outside the To promote list), code, secrets, customer details, product plans.

## 2. Sending it up (app → master)

Whichever way, it ends as a PR to `app-playbook` that you review and merge.

- **Right away:** if the session has both repos attached, the agent writes the lesson in general terms (with a one-line *Example (App):*), opens a `dev-lesson-<topic>` PR on `app-playbook`, and ticks the item: `- [x] … → app-playbook#7`.
- **In a sweep:** every few weeks, or before starting a new app, ask an agent: *"Promote the To promote items from `<app>` to app-playbook."* It opens one PR with all of them and ticks each one with the PR link.

Remember `app-playbook` is public: lessons only, no secrets or private product details.

## 3. Bringing it down (master → apps)

Each app has a GitHub Action, `.github/workflows/sync-playbook.yml`, that copies the master files into `docs/playbook/` and opens a PR titled "Sync playbook from app-playbook" only if something changed. You review and merge it.

- **Automatically:** weekly (Monday morning).
- **On demand:** GitHub → the app repo → Actions → "Sync playbook" → **Run workflow**. Or from a terminal: `gh workflow run sync-playbook.yml -R <owner>/<app>`.
- **Which files:** `APP_PLAYBOOK.md`, `SYNC.md`, and the one stack profile named in the workflow (`PROFILE: nextjs-supabase`). Each copy starts with a "managed copy, don't edit" line.
- **`app-starter` runs it too,** so a new app starts with the current playbook.
- **If the PR undoes a local edit** to a managed file, that edit was in the wrong place: move it to the master or to APP_NOTES before merging.

One-time setup per app repo: Settings → Actions → General → turn on "Allow GitHub Actions to create and approve pull requests". (Sync PRs don't trigger CI; that's fine for docs.)

## 4. Starting a new app

1. On GitHub, open `app-starter` → **Use this template** → create a private repo.
2. Turn on the Actions setting above in the new repo.
3. If the app uses a different stack, change `PROFILE` in `.github/workflows/sync-playbook.yml`, then run the workflow once.
4. Fill in the `{{placeholders}}` in `ROADMAP.md`, `AGENTS.md` and `docs/APP_NOTES.md` (find them with `grep -rn '{{' --include='*.md' .`); from then on they're the app's own.
5. Replace `README.md` with the app's own, and start on the ROADMAP's first item.

## Cheat sheet

| I want to… | Do this |
|---|---|
| Save a general lesson | Add it under **To promote** in `docs/APP_NOTES.md` |
| Save an app-only quirk | Add it to `docs/APP_NOTES.md` (not under To promote) |
| Send lessons to the master | Ask an agent to promote them; merge the PR on `app-playbook` |
| Get the latest playbook now | Actions → Sync playbook → Run workflow; merge the PR |
| Change a playbook rule | PR to `app-playbook`, never the app's copy |
