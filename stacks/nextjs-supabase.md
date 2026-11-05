# Stack profile: Next.js + Supabase (+ Expo)

The **how** for the [App Playbook](../APP_PLAYBOOK.md) on this stack. Section numbers (P1–P9) are the same in every profile, so the playbook can point at them.

- **Stack:** Next.js (App Router, React 19) on Vercel; Supabase (Postgres, RLS, Auth, Storage, Realtime, Edge Functions); pnpm workspaces + Turborepo; TypeScript shared core; Expo (React Native) when a native app is added; PowerSync for local-first.
- **Fits platform options:** A (PWA), D (web + Expo), E (Expo everywhere, dropping Next.js). With B, wrap the same web app in Tauri or Electron.
- **Reference implementation:** Travel App (`yauwilson/travel-app`, private). Paths below are relative to `apps/web/src/` unless they say core (`packages/core/src/`).

---

## P1. Foundation

- **Repo:** pnpm workspaces + Turborepo. `apps/web` (Next.js), `packages/core` (`{{CORE_PKG}}`, no React or Next imports), `apps/mobile` (Expo) later.
- **Environments:** local Supabase in Docker (`pnpm db:start`), staging as a separate Supabase project or Supabase branching, production. Vercel preview deployments point at staging.
- **Region:** Vercel functions in the same region as the Supabase project.
- **CI** (`.github/workflows/ci.yml`), three checks:
  - "Lint, typecheck, unit tests": `next lint`, `tsc --noEmit`, vitest in core;
  - "Migrations, pgTAP, generated types": apply all migrations to a fresh local database, `supabase test db`, fail if `types.gen.ts` differs;
  - "End-to-end (Playwright)": production build, then Playwright against local Supabase.
- **E2E** in `apps/web/e2e/`. Phone profile (iPhone 13, with `browserName: "chromium"`) and desktop. Third-party stubs via e.g. `GOOGLE_PLACES_BASE_URL`.
- **Sentry:** `@sentry/nextjs` on server and client, source maps via `SENTRY_ORG`, `SENTRY_PROJECT`, `SENTRY_AUTH_TOKEN`. Browser SDK loaded lazily with a dynamic `import()` gated on `NEXT_PUBLIC_SENTRY_DSN` (`lib/report-error.ts`).
- **Env vars:** `.env.example` lists all, commented. Only `NEXT_PUBLIC_*` reaches the browser. Every server variable is also in `turbo.json` `globalEnv`, or Turbo's cache ignores changes to it.
- **Supabase auth settings:** asymmetric JWT signing keys on; TOTP MFA enabled (hosted dashboard and `supabase/config.toml`, see P8).

## P2. Data and security

- **RLS** on every table in the migration that creates it. Policies use helper functions such as `<resource>_role_for(<resource>_id)`.
- **pgTAP** tests in `supabase/tests/`, allow and deny cases. Run with `pnpm db:test`.
- **Locked columns:** `revoke update on profiles from authenticated; grant update (display_name, …) on profiles to authenticated;`. Only the service role writes `plan`, `role`, `comp_*`.
- **Entitlements:** tables `plan_features`, `entitlement_overrides`; columns `comp_plan`, `comp_plan_until`; function `entitlement()`; exposed as `my_entitlements()` and `<resource>_entitlements(<resource>)`; triggers and policies call `entitlement_limit` / `entitlement_allows`. Copy from migration `*_admin.sql`. Core keeps the fallback matrix in `entitlements/`, with a unit test that the seed matches.
- **Deletion:** author links `ON DELETE SET NULL`; a service-role-only delete function for a user's own resources.
- **Audit log:** append-only via a trigger that rejects update and delete, even for the service role.
- **Actor lookup in triggers:** `(select id from profiles where id = auth.uid())`, so a deleted user doesn't break inserts.
- **Types:** `timestamptz` for instants, `timestamp` for wall-clock, `numeric(12,2)` + currency code for money.
- **Tokens:** `gen_random_uuid()` (not `gen_random_bytes`, see P8).
- **Generated types:** `pnpm db:types` writes `packages/core/src/supabase/types.gen.ts`. Pin the Supabase CLI version in CI.
- **Local-first:** PowerSync (supports Supabase, web and React Native). Sync rules mirror RLS.
- **Realtime:** Supabase Realtime channels for live lists and feeds when server-first.

## P3. Design system

- **Tokens** as CSS custom properties on `:root` in `apps/web/src/app/globals.css`, with `light-dark()` for both themes from one token.
- **Theme cookie** rendered on the server (`parseThemeCookie`, `lib/theme.ts`), so there's no flash.
- **Shared values** in core `theme/`, so the Expo app uses the same palette.
- **Brand:** `lib/brand.ts`.
- **Tab bar:** floating glass bar above the safe area with a docked centre "+", positioned with `:has(.fab)` so screens without a "+" lay out normally. Desktop: the same tabs as a sticky row under the header.
- **Sheets:** native `<dialog>` (`components/ui/Sheet.tsx`): bottom sheet on phones, centred on desktop; trigger and controlled modes; `useSheet()`.
- **Mobile CSS:** inputs ≥16px; `touch-action: manipulation` (never `maximum-scale=1`); `--safe-b` / `--safe-t` from `env(safe-area-inset-*)`; scrollers with padding, `scroll-snap`, auto-scroll to today.

## P4. App shell

- **Session** (`lib/session.ts`):
  - `getSupabase = cache(createClient)`, one client per request;
  - `getAuthUser` verifies locally with `getClaims()` and returns `aal`; `getUser()` only to confirm the account exists;
  - `getViewer` loads profile, entitlements and admin role in one `Promise.all`; `requireViewer` / `requireUser` redirect.
- **Context helpers** (`get<Resource>Context`) batch in one `Promise.all`; pages start their own queries alongside.
- **Data layer:** data functions wrap Supabase queries and Server Actions (server-first) or PowerSync (local-first).
- **Server Actions** return an `ActionState`, never throw. Helpers in `lib/actions.ts`: `fromZod`, `fromDbError` (constraint/limit errors → friendly text + upgrade key), `formObject`. Each action re-checks entitlements and calls `revalidatePath(path, "layout")`.
- **`ActionForm`** (`components/ui/ActionForm.tsx`, with `SaveStatus.tsx`): optimistic close, "Saving… / Saved" pill, reopen with input on error, 12s watchdog, a 300 ms re-render while a save is in flight (P8), network errors to messages.
- **`ConfirmForm`:** hides the row instantly, restores on failure.
- **Loading and errors:** `loading.tsx` and `error.tsx` per route segment, plus `global-error.tsx`, sharing `components/ErrorScreen.tsx`. Header and tab bar render synchronously, never inside Suspense; a static `HeaderShell` in `loading.tsx`.
- **Caching:** `staleTimes: { dynamic: 30, static: 180 }`; app settings via `unstable_cache` (tag `app-settings`, 60s) and `revalidateTag` on save.
- **Offline:** `app/manifest.ts`, `public/sw.js` (versioned cache, network-first pages, cleared on sign-out, skips `/admin`; bump `VERSION` on change), `components/pwa/` offline banner.

## P5. Feature catalogue: where each lives

| Feature | Where it lives |
|---|---|
| Auth flows | `app/(auth)/*`, `app/auth/callback` |
| Session helpers | `lib/session.ts` |
| Theming | `globals.css`, `lib/theme.ts`, core `theme/` |
| Entitlements & plans | core `entitlements/` (`FeatureKey`, `can()`, `limitOf()`, `planHighlights`, `FEATURE_UPSELL`), migration `*_admin.sql` |
| Upsell UI | `components/ui/Upsell.tsx` |
| Admin portal | `app/admin/*`, `lib/admin.ts`, core `admin/roles.ts` |
| Settings / kill switches | `lib/settings.ts` |
| Action forms | `components/ui/ActionForm.tsx`, `SaveStatus.tsx`, `lib/actions.ts` |
| Sheets | `components/ui/Sheet.tsx` |
| Confirm / optimistic delete | `components/ui/ConfirmForm.tsx` |
| Error screens | `app/error.tsx`, `app/global-error.tsx`, `components/ErrorScreen.tsx` |
| Skeletons / empty states | `components/ui/Skeleton.tsx`, `EmptyState.tsx` |
| PWA / offline | `app/manifest.ts`, `public/sw.js`, `components/pwa/` |
| Invite links | `app/invite/*`, `components/members/InviteForm.tsx` |
| Activity feed | migration `*_expenses_activity.sql` |
| Calendar export | `app/trips/[tripId]/calendar.ics/` |
| Place autocomplete | `app/api/places/*`, `lib/places.ts`, `components/places/*`: Google Places (New) proxy, session tokens, location bias, `(regions)` for cities, field mask `id,location` |
| Time zones / money / templates | core `time/`, `budget/`, `lists/templates` |

## P6. Money and operations

- **Billing:** Stripe Checkout + Customer Portal + a webhook running as a Supabase Edge Function with the service role; the only writer of `profiles.plan`.
- **In-app purchases:** RevenueCat, its webhook feeding the same plan tables.
- **Admin:** roles in `admin_users`; pages require `aal2` (TOTP); first owner from `ADMIN_BOOTSTRAP_EMAIL`, bootstrapped by one function every caller uses.
- **Admin writes** use the service-role client, only behind `requireAdmin`.

## P7. Ship and operate

| Layer | Tool | Where |
|---|---|---|
| Unit | vitest | `packages/core` |
| Database | pgTAP | `supabase/tests/` |
| End to end | Playwright | `apps/web/e2e/` |
| Static | `tsc --noEmit`, `next lint`, `next build` | every push |

**Deploy a feature with a migration or new env vars:**
1. CI green; migrations applied on staging.
2. Merge.
3. `npx supabase db push` on production.
4. Add env vars in Vercel (and `turbo.json` `globalEnv`).
5. Redeploy.
6. Smoke-test on a real phone.
7. Watch Sentry for an hour.

**Local development commands:**

```bash
pnpm install
pnpm db:start            # Supabase in Docker; applies migrations
cp .env.example apps/web/.env.local   # fill in the URL + anon key printed by db:start
pnpm dev                 # http://localhost:3000
pnpm db:test             # pgTAP tests for RLS + entitlements
pnpm --filter {{CORE_PKG}} test   # unit tests
pnpm --filter {{WEB_PKG}} build && pnpm e2e   # Playwright e2e (needs db:start)
pnpm db:types            # regenerate types.gen.ts after a migration
```

## P8. Gotchas, with fixes

**Next.js / React**
- **React 19 resets a form after an action runs.** To keep input when the action fails, submit via `onSubmit` + `startTransition(() => action(formData))`.
- **A Suspense-streamed header left `useActionState` stuck** ("Saving…" forever). Render the chrome synchronously, and use a static `HeaderShell` in `loading.tsx`.
- **After a Server Action's response arrives, React can leave Next's router update parked.** The page keeps showing old data and `useActionState` stays pending until something else re-renders (a tap on a menu does it; a scroll or click on empty space doesn't).
  - Evidence: in the stuck state the React root's `pendingLanes`, `suspendedLanes` and `entangledLanes` are all the same transition lane, with `pingedLanes` 0 and no scheduled callback. Any new update clears `suspendedLanes`, which is why anything that re-renders frees it.
  - Under load it was about 1 save in 7; a lone save on an idle machine almost never.
  - Fix: while a save is in flight, update a bit of the form's own state every 300 ms (`ActionForm`'s `retry` interval). A single nudge on response doesn't work, because it can land before the render suspends.
  - Keep the 12s watchdog as the backstop.
- **Use `revalidatePath(path, "layout")` in every action that changes `<resource>` data.** Page-only revalidation leaves other tabs and feeds stale for the `staleTimes` window.
- **CSS comments containing `**/`** (e.g. `app/**/loading.tsx`) end the comment early and break cssnano in production builds.
- **Vercel `STATIC_BUILD_NO_OUT_DIR`:** set the Framework Preset to **Next.js** (not "Other"), with Root Directory `apps/web`.
- **`.next/cache/fetch-cache` keeps `unstable_cache` data across dev restarts.** Delete it when testing settings.

**Supabase / Postgres**
- **"Could not find the table … in the schema cache"** means the migrations weren't pushed.
- **Migrations dated before the latest applied one are skipped.** Re-date the file, or `db push --include-all` is needed.
- **`gen_random_bytes` is missing from the search path in hosted Supabase.** Use `gen_random_uuid()`.
- **Local MFA is off by default.** Enable it in `supabase/config.toml`, and check TOTP is on in the hosted dashboard:
  ```toml
  [auth.mfa]
  max_enrolled_factors = 10

  [auth.mfa.totp]
  enroll_enabled = true
  verify_enabled = true
  ```
- **Local emails need the mail server.** Don't start with `-x mailpit`.
- **Check the generated types file** for duplicated exports (`Json`) if you add a header to it.
- **`supabase gen types` output varies by CLI version and formatting.** One script (`pnpm db:types`), pinned CLI in CI, fail on diff.

**Sentry**
- **Since `@sentry/nextjs` v11, `withSentryConfig` is imported from `@sentry/nextjs/config`.**
- **A static SDK import costs about 65 kB of first-load JS.** Use a dynamic `import()` gated on the DSN.

**Tooling**
- **Turbo "Exec format error" in some sandboxes.** Run `tsc` / `next lint` per package.
- **Turbo adds a managed agent-rules block to `AGENTS.md`.** Leave it and commit it, or set `"agentGuidance": false` in `turbo.json`.

**Playwright**
- **iPhone device profiles default to WebKit.** Set `browserName: "chromium"` if CI only installs Chromium.
- **Wait for the page to load before `context.setOffline(true)`,** or the test sees a blank page.
- **Seed users with `auth.admin.createUser`** (service role) and delete them afterwards.
- **Settings-changing specs run last as a separate project** with `dependencies`, restoring everything in `afterAll`.
- **Repeat, don't retry:** `playwright test -g "name" --repeat-each 60 --workers 4`. For hangs, log `app:save` events, dialog mounts and `fetch` timings, and read React's root lanes.

## P9. New app checklist: stack steps

Matches the playbook's §12 numbering.

- **9.** `apps/web` + `packages/core`, pnpm + Turbo.
- **10.** Supabase projects for local, staging and prod in Vercel's region; TOTP MFA on; asymmetric JWT signing keys.
- **11.** The three CI checks (P1). Sentry and analytics.
- **12.** Vercel: Framework Preset Next.js, Root Directory `apps/web`, env vars mirrored in `turbo.json` `globalEnv`.
- **13–15.** First migration with column-locked `plan`, RLS and helper role functions; copy the entitlements migration, `app_settings`, audit log and deletion functions; pgTAP for all of it; `pnpm db:types`.
- **17.** Copy the design tokens and theme cookie; rename palettes and `lib/brand.ts`.
- **18.** Copy `lib/session.ts`, `lib/actions.ts`, `ActionForm`, `Sheet`, `ConfirmForm`, `ErrorScreen`, `Skeleton`, `EmptyState`, `Upsell`.
- **19.** Data layer over Supabase, or PowerSync if local-first.
- **20.** Tab bar, `loading.tsx` and `error.tsx` per segment.
- **21.** Manifest and `sw.js`: bump `VERSION`, set excluded paths.
- **23.** Admin portal with `ADMIN_BOOTSTRAP_EMAIL`.
- **24.** Stripe via an Edge Function; RevenueCat when native.
