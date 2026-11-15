# Stack profile: Next.js + Prisma + Neon (+ Expo)

The **how** for the [App Playbook](../APP_PLAYBOOK.md) on this stack. Section numbers (P1–P9) are the same in every profile, so the playbook can point at them.

- **Stack:** Next.js (App Router, React 19) on Vercel; PostgreSQL on Neon through Prisma (schema, migrations, client types); the app's own email/password auth with database sessions; TypeScript; Tailwind; zod; vitest and Playwright; Sentry. Expo (React Native) when a native app is added.
- **What it doesn't give you, compared with a backend-as-a-service:** no row-level security, hosted auth, storage, realtime or edge functions. Access rules live in the data layer (P2), auth is in the app (P4), and anything else is a deliberate third-party choice (§1.6).
- **Fits platform options:** A (PWA), D (web + Expo; the Expo app calls the same Next.js API routes). With B, wrap the same web app in Tauri or Electron.
- **Keep the database portable.** Neon is plain Postgres: use its pooled connection, keep Neon-only features (branches, the serverless driver's HTTP mode) out of app code, make every schema change a Prisma migration, and keep backups outside Neon. Moving host is `pg_dump` / `pg_restore` (or logical replication) and a new `DATABASE_URL`.
- **Reference implementation:** Splitly (`yauwilson/cash-split-app`, private), a single Next.js package. Paths below are relative to its `src/` unless they say otherwise. An app with a native client splits this into `apps/web` + `packages/core` (P1).

---

## P1. Foundation

- **Repo:**
  - Web only: one Next.js package (npm), with framework-free logic in `lib/` (money, splits, plans, parsers) and its tests next to it.
  - With a native client (option D): pnpm workspaces + Turborepo; `apps/web` (Next.js, owns `prisma/`), `packages/core` (`{{CORE_PKG}}`, no React, Next or Prisma imports), `apps/mobile` (Expo) later. Core can import Prisma's *types* only through its own re-exports, so the Expo bundle never pulls in the client.
- **Environments:**
  - Local: Postgres 16 in Docker (`docker-compose.yml`), or a personal Neon branch.
  - Staging: Vercel preview deployments, each on its own Neon branch via the Neon integration (P8).
  - Production: the Neon project's main branch.
- **Region:** Vercel functions in the same region as the Neon project.
- **Connection strings** (both set by Vercel's Neon integration):
  - `DATABASE_URL`: pooled (PgBouncer, `-pooler` host). The app uses it, so serverless functions don't exhaust connections.
  - `DATABASE_URL_UNPOOLED`: direct. Migrations use it, since they need a session-level connection. Either set `directUrl = env("DATABASE_URL_UNPOOLED")` in the datasource, or swap the variable in the build script as Splitly does (P7). Use `directUrl` for any script that also runs locally (P8).
- **CI** (`.github/workflows/ci.yml`), three checks, each with a Postgres 16 service container:
  - "Lint, typecheck, unit tests": `prisma generate`, `eslint .`, `tsc --noEmit`, `vitest run`;
  - "Migrations on a fresh database": `prisma migrate deploy` on an empty database, the production seed, then the drift check (P2);
  - "End-to-end (Playwright)": migrate, seed, `next build`, Playwright against `next start`.
- **E2E** in `e2e/` (or `apps/web/e2e/`). Phone profile (iPhone 13, with `browserName: "chromium"`) and desktop; then a `settings` project and an `exclusive` project for specs that change app-wide settings (P8). No retries. Run against a production build, because the service worker only runs there.
- **Sentry:** `@sentry/nextjs`, server errors from `instrumentation.ts` (`register` + `onRequestError`, both returning early without a DSN), browser errors from `lib/report-error.ts` with a dynamic `import()` gated on `NEXT_PUBLIC_SENTRY_DSN`. Source maps only when `SENTRY_AUTH_TOKEN` is set (`withSentryConfig` in `next.config.ts`).
- **Env vars:** `.env.example` lists all, commented. Only `NEXT_PUBLIC_*` reaches the browser. In a Turborepo, every server variable is also in `turbo.json` `globalEnv`.

## P2. Data and security

- **Access rules in the data layer** (no RLS). Every query on `<resource>` data is scoped by membership:
  - reads: `findFirst({ where: { id, members: { some: { userId } } } })`, else `notFound()` (`getGroupForUser` in `lib/data.ts`);
  - writes: `assertMember(<resource>Id, userId)` before any mutation, and child rows are looked up *through* the `<resource>` they belong to, never by id alone;
  - screens, Server Actions and API routes call these functions, never `db` directly; `db` and the data layer import `server-only`.
- **Access-rule tests:** with no database-level rules, test them end to end. A second user calls every `<resource>`-scoped page, Server Action and API route with valid payloads, on the paid plan (so a plan gate can't hide a missing membership check), and expects 404/403 (`e2e/access.spec.ts`). Add a case for every new table or route.
- **Data-layer access tests:** also call every membership-scoped service directly as a non-member, against a real migrated database, and check it throws and nothing changed (a `*.db.test.ts` vitest config, run after `prisma migrate deploy` in the migrations CI job). They're fast, cover writes a page can't reach, and fail on the exact function that lost its check.
  *Example (Finance Tracker):* `access.db.test.ts` covers create, update, close, reopen and delete of accounts, plus a member whose membership was removed.
- **Locked columns:** there is no column-level grant, so lock them in code. Profile updates go through a zod schema that lists only editable fields (never spread form data into `update`). Only billing and admin code write `plan`, `compPlan*`, `adminRole`, `suspendedAt`.
- **Entitlements:** core `plans.ts`: `planOf(user)` resolves billing plan and admin comp (`compPlan`, `compPlanUntil`); `can(plan, feature)` and `limit(plan, key)` for every gate, in the UI and again in the Server Action or API route. Move the matrix to `plan_features` / `entitlement_overrides` tables when plans need changing without a deploy, with a unit test that the seed matches the code fallback.
- **App settings:** an `AppSetting` table (`key`, JSON `value`, `description`). Default rows are inserted by the migration that introduces them (`INSERT … ON CONFLICT ("key") DO NOTHING`), so production gets them without a seed step. Readers parse with a typed defaults map and fall back to it if a row is missing (`lib/settings-parse.ts`), wrapped in React `cache()`.
- **Audit log:** a table with no foreign keys (copy the actor's email in, so entries outlive deleted users), made append-only in the same migration by a trigger that raises on `UPDATE`, `DELETE` and `TRUNCATE`. Never write one-time secrets into it.
- **Deletion and sync-ready rows:** synced or user-visible data is soft-deleted (`deletedAt`), never hard-deleted, and every query filters `deletedAt: null` (index `[<resource>Id, deletedAt]`). For local-first later, also give every table a `version` (or `updatedAt` you trust) from the first migration, so "what changed since" works without a backfill. Author links are `onDelete: SetNull`; a user's own data cascades from `User`.
- **Types:** `DateTime` is `timestamp(3)` (UTC instant). Store wall-clock values (a date with no time) as `@db.Date` or a `YYYY-MM-DD` string. **Money is `Int` minor units plus a currency code**, never `Float` or `Decimal` in app code (`lib/money.ts`: `toCents`, `formatMoney`, `allocate` for remainders).
- **Enums:** plain `String` columns with the allowed values in code (`lib/constants.ts`) and zod; adding a value then needs no migration.
- **Ids and tokens:** `@default(cuid())` for ids; `crypto.randomBytes(32).toString("base64url")` for tokens, storing only a SHA-256 hash.
- **Idempotent uploads:** tables written from a device outbox get `clientId String? @unique`. The service looks it up first and also catches `P2002` on create, returning the existing row (`createExpense` in `lib/expense-service.ts`).
- **Migrations:** edit `prisma/schema.prisma`, then `prisma migrate dev --name <topic>` (folder `YYYYMMDDHHMMSS_topic`). Hand-edit the generated SQL for triggers, seeds and partial indexes before committing. Never `prisma db push` against a shared database.
- **Generated types:** Prisma Client, generated by `prisma generate` (in `build` and in CI), not committed. The CI check that replaces "types differ" is the drift check: `prisma migrate diff --from-migrations prisma/migrations --to-schema-datamodel prisma/schema.prisma --shadow-database-url … --exit-code` fails when the schema changed with no migration.
- **Local-first:**
  - Stage 1 (Splitly today): saved pages plus an on-device outbox for new items (P4).
  - Stage 2: a full on-device copy synced by "what changed since" (`version` / `updatedAt` + `deletedAt`), either in-house or with a sync engine that works on plain Postgres (PowerSync, ElectricSQL). Sync scope mirrors the membership checks.
- **Live updates:** no realtime channel. Refresh on focus and on an interval (Splitly: on return to the app, and every 45 s while open, never on a route with a form in progress). Add a realtime provider or the sync engine when several people edit the same data at once.

## P3. Design system

- **Tokens** as CSS custom properties in `app/globals.css`, mapped into Tailwind (`tailwind.config.ts`), with `.dark` overrides. User accent and corner radius are set as `--accent` / `--radius` on `<html>` from the user's preferences.
- **Theme before paint:** the server renders the stored theme as a class and `data-theme` on `<html>`; a tiny inline script handles "system" (`matchMedia`) before first paint. Preferences are one validated JSON column (`lib/preferences.ts`, zod with defaults).
- **Shared values:** with a native client, palettes and spacing move to core `theme/`.
- **Font:** `next/font` (Inter) as a CSS variable. Self-sized inputs must use the same `font-variant-numeric` as their sizer (§10).
- **Sheets:** `components/sheet.tsx`, rendered into `document.body` with `createPortal` (P8): bottom sheet on phones, centred on desktop.
- **Navigation:** bottom tab bar on phones, sidebar on desktop (`components/app-shell.tsx`), tabs ordered and hidden from preferences.
- **Mobile CSS:** inputs ≥16px; `touch-action: manipulation` (never `maximum-scale=1`); `viewportFit: "cover"` and `env(safe-area-inset-*)` padding.

## P4. App shell

- **Auth** (`lib/auth.ts`), no library:
  - passwords with `bcryptjs` (cost 12), and a dummy hash compare for unknown emails so timing doesn't reveal accounts;
  - sessions in a `Session` table storing only the SHA-256 of a random token; the cookie is `httpOnly`, `sameSite: "lax"`, `secure` in production; `lastSeenAt` touched at most hourly; a list of devices and "sign out other devices";
  - `getSession` / `getCurrentUser` wrapped in React `cache()`; `requireUser()` redirects to `/login`; suspended users get no session;
  - `middleware.ts` (`proxy.ts` from Next.js 16) only checks the cookie *exists* (redirects to `/login?next=…`); real validation happens in `requireUser()`.
- **Two-factor (admin):** TOTP from `node:crypto` (RFC 6238, ~40 lines, `lib/totp.ts`) tested against the RFC vectors; QR as SVG from `qrcode` on the server; per-session `adminVerifiedAt` and a failure counter.
- **Data layer:** `lib/data.ts` (reads) and `lib/*-service.ts` (writes shared by Server Actions and API routes), all membership-scoped (P2). Pages call these, start independent queries with `Promise.all`.
- **Server Actions** return a `FormState` (`{ error?, ok?, message? }`), never throw for user errors; validate with zod; re-check `can()` / `limit()`; call `revalidatePath("/", "layout")` (or the `<resource>` layout) after changing `<resource>` data.
- **API routes** for anything a device or native client calls (the outbox upload, exports). Check `Origin` matches `Host` on state-changing routes; return 401 (not a redirect) for an expired session from the route itself, and exclude those paths from the middleware redirect if the client needs the 401.
- **Forms:** submit buttons with `useFormStatus` pending text (`components/submit-button.tsx`); confirm-then-act buttons for destructive actions (`components/confirm-button.tsx`). Port `ActionForm`'s "Saving… / Saved" pill and 300 ms nudge (profile `nextjs-supabase` P4, P8) when saves happen in sheets.
- **Loading and errors:** `error.tsx` per route group plus `global-error.tsx`; app chrome rendered synchronously in the layout.
- **Offline, stage 1:**
  - `public/manifest.webmanifest`; `public/sw.js` with a versioned pages cache: hashed `/_next/static` cache-first and kept across deploys, signed-in pages network-first with a 4 s timeout falling back to the saved copy, API and RSC requests never cached, auth and `/admin` pages never cached, everything wiped on logout and on a different user signing in. Serve `sw.js` with `Cache-Control: no-cache` (`next.config.ts` `headers()`). Bump `VERSION` when changing it.
  - An outbox in IndexedDB (`lib/offline/outbox.ts`): items keyed by a client-generated UUID, rendered as "Waiting to sync" from precomputed display fields, uploaded in order by a single-flight sync (Web Locks where available) to an idempotent API route, retried on network errors and 5xx/408/429, marked failed with the server's message on 4xx, stopped on 401 with a "log in again" prompt, wiped on logout. Tabs stay in step via `BroadcastChannel`.
  - `components/offline/sync-manager.tsx` registers the worker, runs the sync on load / online / focus, refreshes data, and shows the offline or "saved copy from …" banner.
  - Edits, deletes and settle-up stay online-only until stage 2.

## P5. Feature catalogue: where each lives

| Feature | Where it lives (Splitly) |
|---|---|
| Auth and sessions | `lib/auth.ts`, `middleware.ts`, `app/(auth)/*`, `app/actions/auth.ts` |
| Data layer | `lib/data.ts`, `lib/expense-service.ts` |
| Money | `lib/money.ts` |
| Theming and preferences | `app/globals.css`, `tailwind.config.ts`, `lib/preferences.ts`, `app/layout.tsx` |
| Entitlements & plans | `lib/plans.ts` (`planOf`, `can`, `limit`, `FEATURES`), `app/(app)/settings/plan` |
| App settings / kill switches | `lib/settings.ts`, `lib/settings-parse.ts`, `AppSetting` rows in migrations |
| Admin portal | `app/(app)/admin/*`, `lib/admin.ts`, `lib/admin-roles.ts`, `lib/totp.ts`, `app/actions/admin.ts` |
| Audit log | `AdminAudit` + trigger in migration `*_admin_portal` |
| Activity feed | `lib/activity.ts`, `ActivityLog` |
| Sheets | `components/sheet.tsx` |
| Error screens | `app/(app)/error.tsx`, `app/global-error.tsx` |
| PWA / offline | `public/manifest.webmanifest`, `public/sw.js`, `lib/offline/*`, `components/offline/*`, `app/api/expenses/route.ts` |
| Invite links | `app/join/[code]` |
| CSV import / export | `lib/import.ts`, `app/actions/import.ts`, `app/api/groups/[id]/export` (§10: neutralize formula cells) |
| Error monitoring | `instrumentation.ts`, `lib/report-error.ts`, `components/error-reporting.tsx` |
| Health check | `app/api/health` (Playwright's `webServer.url`) |

## P6. Money and operations

- **Billing:** Stripe Checkout + Customer Portal; the webhook is a Next.js route handler (`app/api/stripe/webhook`) that verifies the signature on the raw body and is the only writer of `plan` and `planRenewsAt`. Until billing exists, `plan` is set by SQL or an admin comp.
- **In-app purchases:** RevenueCat, its webhook feeding the same columns.
- **Admin:** `adminRole` on `User` (OWNER, ADMIN, SUPPORT), permissions in `lib/admin-roles.ts`; owners bootstrapped from `ADMIN_OWNER_EMAIL` (comma-separated); pages need a TOTP check within 12 h; non-admins get a 404. Every admin action takes a reason and writes the audit log.
- **Admin writes** go through `adminActor(permission)`, which re-checks role and two-factor on each Server Action.
- **Backups outside Neon:** a scheduled `pg_dump` (e.g. a GitHub Actions cron with the unpooled URL as a secret) to storage you control, plus Neon's own point-in-time restore. Test a restore before you need one.

## P7. Ship and operate

| Layer | Tool | Where |
|---|---|---|
| Unit | vitest | next to the code (`lib/*.test.ts`) or `packages/core` |
| Migrations | `prisma migrate deploy` on an empty database + drift check | CI |
| Access rules | Playwright, second-user deny cases | `e2e/access.spec.ts` |
| Data-layer access | vitest against a migrated database | `*.db.test.ts`, CI migrations job |
| End to end | Playwright | `e2e/` |
| Static | `tsc --noEmit`, `eslint`, `next build` | every push |

**Build on Vercel** (`package.json`):

```json
"build": "prisma generate && next build",
"vercel-build": "prisma generate && DATABASE_URL=\"${DATABASE_URL_UNPOOLED:-$DATABASE_URL}\" prisma migrate deploy && SEED_DEMO=0 tsx prisma/seed.ts && next build"
```

Migrations apply on every deploy, previews included, so previews must have their own database branch (P8).

**Deploy a feature with a migration or new env vars:**
1. CI green; the preview deployment (its own Neon branch) applied the migration.
2. Add env vars in Vercel (and `turbo.json` `globalEnv` in a monorepo).
3. Merge. The production build runs `prisma migrate deploy`.
4. Smoke-test on a real phone.
5. Watch Sentry for an hour.

A migration that drops or renames a column ships in two deploys: first stop using it, then remove it.

**Local development commands:**

```bash
docker compose up -d     # Postgres 16 matching .env.example
cp .env.example .env
npm install
npm run setup            # prisma migrate deploy + seed (demo data unless SEED_DEMO=0)
npm run dev              # http://localhost:3000
npm run db:migrate -- --name <topic>   # after editing schema.prisma
npm test                 # vitest
npm run build && npm run e2e           # Playwright against a production build
npm run db:reset         # wipe the local database and re-seed
```

## P8. Gotchas, with fixes

**Next.js / React:** the Next.js gotchas in profile `nextjs-supabase` P8 apply here too (form reset after actions, the parked router update, `revalidatePath(…, "layout")`, portals for overlays, `**/` in CSS comments, Vercel's Framework Preset).

- **Next.js 16 renamed `middleware.ts` to `proxy.ts`,** exporting `proxy` instead of `middleware`. The old name still runs but is deprecated; the build lists it as "Proxy (Middleware)".
- **The middleware redirects signed-out API calls to `/login` (307) before the route runs**, so a client expecting 401 gets HTML. Either exclude those routes from the matcher and return 401 in the route, or treat a redirect as "signed out". In tests, use `maxRedirects: 0`.
- **The service worker only runs in production builds.** Add an opt-in flag (e.g. `NEXT_PUBLIC_ENABLE_SW=1`) to test it under `next dev`, and run e2e against `next start`.

**Prisma**
- **Migrations through the pooled URL fail or hang** (advisory locks need a session). Use the unpooled URL for `migrate deploy` / `migrate dev`, the pooled one for the app.
- **One `PrismaClient` per process:** keep it on `globalThis` in development (`lib/db.ts`), or hot reload opens a new pool each edit and exhausts connections.
- **`prisma generate` must run before `next build` and before `tsc` in CI**, since the client isn't committed.
- **`migrate dev` needs a shadow database.** Locally it creates one; in CI create it yourself for the drift check (`CREATE DATABASE <name>_shadow`).
- **Unique-constraint races surface as `P2002`.** Catch it where a retry is expected (idempotent `clientId` uploads) and return the existing row.
- **The `DATABASE_URL="${DATABASE_URL_UNPOOLED:-$DATABASE_URL}"` swap only works where the variables are in the shell** (Vercel, CI). Locally they live in `.env`, which the shell never reads, so the swap sets `DATABASE_URL` to an empty string and Prisma fails. Keep it in `vercel-build` only, or use `directUrl`.
  *Example (Finance Tracker):* `db:setup` failed locally with the swap and worked once the datasource had `directUrl`.
- **pnpm 10 skips dependency install scripts unless allowed,** so Prisma's engines and `@sentry/cli` don't set themselves up. List `@prisma/client`, `@prisma/engines`, `prisma`, `esbuild` and `@sentry/cli` under `onlyBuiltDependencies` in `pnpm-workspace.yaml`.
- **Triggers, partial indexes and seed rows aren't in `schema.prisma`.** Add them by hand to the generated `migration.sql`; the drift check ignores them, so cover them with a test.

**Neon / Vercel**
- **Preview deployments run migrations.** Without a separate database they change the production schema from an unmerged branch. Turn on the Neon integration's "create a database branch for each preview deployment", and check Preview's `DATABASE_URL` is managed by it, not a copy of production's.
- **Neon's free plan caps a project at 10 branches**, one per previewed git branch. With several PRs a day previews start failing in seconds ("database branch … limit reached"). Turn on "automatically delete obsolete preview branches" in the integration and GitHub's "Automatically delete head branches".
- **Neon computes scale to zero.** The first request after idle waits for a cold start; keep Prisma's connect timeout generous and don't treat the first slow query as an outage.

**Sentry**
- **Since `@sentry/nextjs` v11, `withSentryConfig` is imported from `@sentry/nextjs/config`.**
- **A static SDK import costs about 65 kB of first-load JS.** Use a dynamic `import()` gated on the DSN.

**Playwright**
- **iPhone device profiles default to WebKit.** Set `browserName: "chromium"` if CI only installs Chromium.
- **Before `context.setOffline(true)`, wait until the service worker has saved the page and its assets** (`caches.match` on the page, then each script and style it references). Waiting for load alone failed about 1 run in 6 under parallel load.
- **Seed users through the app or Prisma with a test-only email domain** (e.g. `@e2e.<app>.test`) and delete them in `globalTeardown`.
- **Specs that change app-wide settings run last** in a `settings` project with `dependencies`; settings that break every other spec (pausing sign-ups, a banner on every page) go in an `exclusive` project after it, `mode: "serial"`, restoring values in `afterAll`.
- **Saves through the outbox finish in the background.** Wait for the server-rendered row, not the optimistic one.
- **To forge a hidden form field in an access test** (another `<resource>`'s id in your own edit form), set it right before the click that submits. React writes a controlled input's `value` back on any re-render, so a value set before opening a confirmation is silently restored and the test acts on your own row.
- **Repeat, don't retry:** `playwright test -g "name" --repeat-each 60 --workers 4`.

## P9. New app checklist: stack steps

Matches the playbook's §12 numbering.

- **9.** One Next.js package, or `apps/web` + `packages/core` with pnpm + Turbo if a native client is planned.
- **10.** Docker Postgres locally; a Neon project in Vercel's region, connected through Vercel's Neon integration with a branch per preview; TOTP for admins (P4).
- **11.** The three CI checks (P1), including the drift check. Sentry gated on its DSN.
- **12.** Vercel: Framework Preset Next.js, the `vercel-build` script (P7), auto-delete of preview branches in Neon and of head branches in GitHub.
- **13–15.** First migration: `User` with `plan`, `compPlan*`, `adminRole`; `Session`; `<resource>` and its members; `deletedAt` (and `version`) on synced tables. Then `AppSetting` with default rows, the append-only audit table, `plans.ts`, and the second-user access spec.
- **17.** Copy `globals.css` tokens, `tailwind.config.ts`, `lib/preferences.ts` and the theme script in `app/layout.tsx`; rename palettes and brand.
- **18.** Copy `lib/auth.ts`, `lib/db.ts`, `middleware.ts` (`proxy.ts` on Next.js 16), `lib/money.ts`, `Sheet`, `SubmitButton`, `ConfirmButton`, error screens.
- **19.** `lib/data.ts` and services with membership checks; an idempotent upload route and the outbox if offline edits are needed.
- **20.** App shell with tab bar and sidebar, `error.tsx` per route group.
- **21.** Manifest, `sw.js` (bump `VERSION`, set never-cached paths), `SyncManager` and banners.
- **23.** Admin portal with `ADMIN_OWNER_EMAIL`, TOTP and the audit log.
- **24.** Stripe webhook route; RevenueCat when native.
