# App Playbook

How I plan, build and ship apps: the decisions to make first, the foundation, data and security patterns, design system, app shell, and the gotchas learned along the way.

| File | What it is |
|---|---|
| [`APP_PLAYBOOK.md`](APP_PLAYBOOK.md) | The stack-neutral playbook: principles, platform choice, phases, gotchas, checklist |
| [`stacks/`](stacks/) | Stack profiles: the commands, paths, config and gotchas for one stack (sections P1–P9) |
| [`SYNC.md`](SYNC.md) | How apps receive updates from this repo, and how lessons flow back |
| [`AGENTS.md`](AGENTS.md) | Rules for editing this repo |

**Starting a new app?** Use [`app-starter`](https://github.com/yauwilson/app-starter) (a template repo). Each app gets read-only copies of these files in `docs/playbook/`, refreshed by a weekly sync PR.

**Improving the playbook?** Open a PR here. See [`SYNC.md`](SYNC.md) §2.
