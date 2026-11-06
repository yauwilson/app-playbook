# App Playbook

How we build apps, whatever the stack. It is ordered the way a strong team plans and builds:

1. Make the decisions that are expensive to change.
2. Lay the foundation that keeps every later change safe.
3. Model the data and security.
4. Design the system.
5. Build the app's shared patterns.
6. Build features on top.
7. Add money and operations.
8. Ship and run it.

This file holds the **principles**: what to decide, what to build and why. The **how** for a particular stack (commands, file paths, config, code to copy, stack gotchas) lives in a **stack profile** in `stacks/`. Each section here names the profile section that implements it, as *(profile §Pn)*.

| Stack profile | Covers |
|---|---|
| [`stacks/nextjs-supabase.md`](stacks/nextjs-supabase.md) | Next.js on Vercel, Supabase (Postgres, RLS, Auth), pnpm + Turborepo, Expo for native. Built from Travel App. |
| *(add more as you use them)* | Copy the section layout of an existing profile: P1–P9. |

**How to use it:**
- **Starting a new app:** create the repo from `app-starter` (see `SYNC.md`), make the Phase 0 decisions (§1) including the stack (§1.3), work through the phases in order, and finish with the checklist in §12. This file and the profile arrive in `docs/playbook/` by the sync.
- **Adding a feature to an existing app:** read the phase it touches here and in the profile, then the gotchas (§10 here, P8 in the profile) and §11.
- **When something bites:** in the same PR as the fix, add it under "To promote" in the app's `docs/APP_NOTES.md`, targeted at the profile's P8 (a stack lesson) or §10 here (any stack). It reaches this file through a PR to `app-playbook` (§13, `SYNC.md`).

### Conventions in this document

- **`<resource>`:** the app's main shared object, the thing users create and invite others into (a trip, a workspace, a project, a household). Read `<resource>` as that name.
- **`{{PLACEHOLDER}}`:** fill in per app (e.g. `{{APP_NAME}}`).
- **Example (Travel App):** a real case from the first app built this way, a collaborative trip planner used mostly on phones. Examples explain *why* a rule exists; they are not requirements.
- **"Shared core":** the package of logic, types, schemas and rules that every client (web, native, desktop, scripts) uses, with no UI framework imports.

---

## 1. Phase 0: decide first (the expensive-to-change decisions)

Top teams spend a day here because these decisions shape every screen. Changing one later means a rewrite, not a refactor. Record each answer in the app's decisions table (in `ROADMAP.md`), with a "revisit if" condition.

### 1.1 Who uses it, where, on what

- **Primary device and context.** Write down where and how people actually use it: phone or laptop, one hand or two, strong or weak connection, at a desk or on the move, and which OSes matter (iOS, iPadOS, macOS, Android, Windows, ChromeOS, watch, TV).
  - *Example (Travel App):* a phone held in one hand, on a weak connection, often outdoors. Desktop is for longer planning sessions.
- **Rule for a phone-primary app:**
  - Design mobile-first at 390pt wide.
  - Larger screens widen the same screens into columns.
  - Never design for desktop first and squeeze it down.
- If the app is genuinely desktop-primary (dense data, keyboard-heavy), say so here and still check every screen at phone width if phones are supported at all.

### 1.2 Platform strategy: how the app reaches each device

Decide this before the stack, because it largely decides the stack. Answer five questions first:

1. **Which platforms at launch, and which within a year?** (iOS, iPadOS, macOS, Android, Windows, Linux, web, watch/TV.)
2. **Which native surfaces are core to the product?** Push, background work, home-screen widgets, Live Activities, share extensions, App Intents/Siri, wallet passes, HealthKit, Bluetooth, camera/AR, audio in the background, menu bar or system tray, global shortcuts, file system access.
3. **How much does the web matter?** SEO, sharing links that open without an install, desktop browser use.
4. **Must data survive offline and never be cleared?** (Browsers can evict storage; iOS does for sites not added to the home screen.)
5. **What does the team already write well?** TypeScript, Swift, Kotlin, Dart, Rust.

#### The options

| Option | Reaches | UI codebases | Native feel & surfaces | Pick when | Watch out for |
|---|---|---|---|---|---|
| **A. Web app / PWA only** | Every browser; installable on Android, desktop and iOS (home screen) | 1 | Low. iOS push only once installed to the home screen; storage can be evicted; no widgets, Live Activities or share extensions | Validating an idea; desk-heavy use; SEO and links matter most | Hitting iOS limits after users depend on you. Build so a native client can be added (§5.2) |
| **B. Web + desktop shell** ("Chromium" route): **Electron** (bundles Chromium + Node) or **Tauri** (system webview + Rust) | Web, plus installable macOS, Windows and Linux apps. Tauri 2 also targets iOS and Android | 1 (the web UI) | Desktop surfaces: tray/menu bar, global shortcuts, file system, auto-update, offline. Still web UI inside | You need real desktop-OS integration or offline desktop use, and already have a web app | Electron: large installs (~100 MB+) and memory use. Tauri: each OS's webview differs (WebKit on macOS/iOS, WebView2 on Windows), so test each; native code is Rust. Neither fixes phone-native needs well |
| **C. Wrapped web app for phones** (**Capacitor**) | iOS and Android stores, from the existing web UI | 1 | Medium. Native plugins for push, camera, etc., but the UI is still web | You have a good mobile web app and need store presence and push quickly | App Store review can reject thin wrappers (minimum functionality rule). Scrolling, gestures and keyboards feel less native |
| **D. Web app + React Native via Expo**, sharing a TypeScript core and sync layer | Web (e.g. Next.js), iOS, Android; iPad apps; iPad apps run on Apple silicon Macs | 2 (web + native) | High. Native components, push, background tasks; widgets and Live Activities via native Swift/Kotlin targets added alongside | Phones are primary *and* a strong desktop/SEO web app matters, with a TypeScript team | Two UI codebases. Native-only features still need a little Swift/Kotlin. macOS/Windows via React Native's desktop ports is less mature than iOS/Android |
| **E. Expo everywhere** (Expo Router on iOS, Android and web) | Same as D | 1 | High on phones; web output is less "webby" | The phone is the product, web is secondary | Weaker SEO and desktop web polish |
| **F. Flutter** | iOS, Android, web, macOS, Windows, Linux | 1 (Dart) | Medium-high. Draws its own widgets, so pixel-identical everywhere but not native components; platform channels for native APIs | You want one UI on many platforms, including desktop, and web SEO doesn't matter; team knows or will learn Dart | Web output is a canvas app (poor SEO, larger downloads). New OS UI conventions arrive later than in native |
| **G. Kotlin Multiplatform** (shared logic; native UI, or Compose Multiplatform for shared UI) | Android, iOS, desktop JVM, web (Compose for web is newer) | 1–2 | High. Share logic, keep SwiftUI/Compose UIs, or share Compose UI | Android-first or Kotlin team; you want native UI per platform without duplicating logic | iOS developers must work with a Kotlin module; check the current maturity of the targets you need |
| **H. Full native per platform**: **SwiftUI** across iOS, iPadOS, macOS, watchOS, visionOS + **Kotlin/Jetpack Compose** on Android + a separate web app if needed | Every platform, each at its best | 2–3 (Swift, Kotlin, web) | Highest. New OS features on day one; best performance, accessibility and polish | Platform polish *is* the product; heavy device features (AR, audio, health, Bluetooth, watch); or an Apple-first app where SwiftUI covers iPhone, iPad and Mac from mostly one codebase | Highest cost: each feature built and tested 2–3 times, and no shared logic unless you add KMP or keep rules on the server. A small team ships much slower |

Other routes worth knowing: **.NET MAUI** (C# teams, strong on Windows), **Mac Catalyst / "Designed for iPad"** (a cheap way to get an iPad app onto macOS), **React Native for Windows/macOS** (Microsoft's ports, for RN teams needing desktop).

#### How to choose

1. **Start from where the product lives.**
   - Mostly a browser or desk → **A**, and add **B** when desktop-OS integration or offline desktop becomes a need.
   - The phone is the product → **D** or **E** with a TypeScript team; **H** (SwiftUI first) if it's Apple-first and polish or device features are the point; **F** or **G** if the team's language points there.
   - Many platforms with one small team and SEO doesn't matter → **F**.
2. **Count the native surfaces from question 2.** If more than one or two are core, rule out **A**, and lean away from **B** and **C**.
3. **Count UI codebases you can afford to maintain.** A solo developer or a pair can usually sustain one, maybe two. Full native (**H**) for iOS + Android + web is three.
4. **Whatever you pick, keep the escape hatch open:** business rules in the database or API (§3), logic in the shared core, data through a data layer (§5.2). Then adding a client later is new UI, not a rewrite.

**Default for most new apps:** **D**, starting web-first: ship the web app, built so the Expo app can be added without a rewrite, and add it when a §1.2a trigger fires. Choose differently when the questions above say so, and record why.

**§1.2a When to add a native app (from A, B or C).** Add it when any of these becomes a top user need:
- reliable push notifications or reminders;
- offline data that must never be cleared;
- native surfaces: home-screen widgets, Live Activities, a share sheet to "send to app", wallet passes;
- app-store discovery.

Build the sync layer (§1.4) first, so the native app reuses it instead of building data access twice.

**§1.2b What a second client changes.** As soon as any client isn't the web server itself (a native app, a desktop shell talking to a server, a script), every rule must live where all clients pass through: the database's access rules or a shared API. Logic that only exists in web server code (e.g. a Next.js Server Action) is skipped by the other clients. This is why §3 puts rules in the database.

### 1.3 Stack

Pick the backend, hosting and client framework that fit §1.2, then pick (or write) the stack profile.

- **Backend:** prefer one that enforces access rules next to the data (e.g. Postgres row-level security, or security rules in a document store), so §3 holds for every client. Note what it gives you: auth, file storage, realtime, server functions, and a local copy for development and CI.
- **Hosting:** the app server runs in the same region as the database.
- **Client:** follows from §1.2.
- **Shared core language:** the language most clients can import (TypeScript for web + React Native; Kotlin for KMP; Dart for Flutter). With full native (H), the shared core is the server and database; keep the client logic thin.
- **Requirements every stack must meet** (each profile says how):
  - a local environment and a staging environment separate from production;
  - migrations as code, applied in CI to a fresh database;
  - tests for access rules (allowed and denied);
  - types generated from the real schema;
  - a way to stub third-party APIs in tests;
  - error monitoring with readable stack traces.

*Example (Travel App):* option D web-first, on the [Next.js + Supabase profile](stacks/nextjs-supabase.md).

### 1.4 Data and sync strategy: server-first or local-first

This is the most expensive decision to change after the platform.

- **Server-first.**
  - The server renders or serves data; writes go to the server.
  - Optimistic UI hides the wait.
  - It's simple and fine when users are online.
- **Local-first.**
  - Data lives in a database on the device (usually SQLite).
  - Reads are instant and work offline.
  - Writes queue and sync in the background.
  - Use a sync engine on top of your backend (the profile names one, P2).

**Choose local-first from day one if any of these is true:**
- users need to *edit* while offline;
- the app is used on unreliable networks;
- several people edit the same data;
- a native app is likely.

Retrofitting local-first means rewriting how every screen reads and writes. *Example (Travel App):* it started server-first and paid that cost later.

**When local-first, also decide:**
- **Sync scope.** What each user downloads, e.g. every `<resource>` they're a member of. This becomes the sync rules, which mirror the access rules.
- **Conflict policy.** Last-write-wins per field is usually enough.
- **Rejected offline writes.** For example, a plan limit is hit while offline. Users see "couldn't sync: …" with a fix, never silent data loss.

### 1.5 Money model

- **Free vs paid tiers,** and which features are gated.
- **Who pays for a shared resource.** Decide whether a `<resource>`'s features follow its creator's plan or each member's plan.
  - *Example (Travel App):* a trip's creator's plan unlocks trip features for everyone on the trip. One subscriber upgrades the group, which is a good growth loop.
- **The billing channel:**
  - A payment processor on the web (e.g. Stripe).
  - App-store purchases inside native apps: unify them with a service like RevenueCat, and check the current store rules for linking out to web checkout.
  - Make this decision before building a native app.

### 1.6 Third parties and their costs

List every paid API (maps, AI, SMS, email) with:
- its free allowance;
- its caching terms;
- a kill switch plan (§3.6).

*Example (Travel App):* Google Places, with place IDs kept indefinitely, coordinates refreshed after the caching limit, and an admin switch to turn it off.

### 1.7 Decisions table

Record these in `ROADMAP.md`, each with a "revisit if":

| Decision | Write down |
|---|---|
| Primary device & context | §1.1 |
| Platform strategy | Option A–H (§1.2) and the platforms at launch |
| Stack & profile | Backend, hosting, client framework, shared-core language (§1.3) |
| Sync | Server-first or local-first, and the sync engine (§1.4) |
| Money model & billing | Tiers, who pays for shared resources, billing channel (§1.5) |
| Third-party APIs | Each one's allowance, caching terms, kill switch (§1.6) |
| Admin auth | Roles + two-factor; SSO when staff grows |
| Limits | Plan defaults as data + per-user overrides (§3.3) |

---

## 2. Phase 1: foundation (first week, before features)

Everything here is cheap on day one and expensive later. Each item was missed or done late in an earlier app and cost real time. *(profile §P1)*

1. **Repo:** one repo for every client and the shared core (a monorepo), with the shared core free of UI framework imports.
2. **Environments:** local, **staging** and production. Preview builds point at staging.
3. **CI on every PR:**
   - lint, typecheck and unit tests;
   - apply **all migrations to a fresh database** and run the access-rule tests;
   - check the generated types are up to date;
   - build every client;
   - run end-to-end tests against a local stack.
   - This catches the "works locally, fails on hosted" class of bug before you deploy.
4. **E2E tests live in the repo.**
   - Each spec seeds and cleans up its own data.
   - Test on a phone-sized profile and on desktop (and on each native target you ship).
   - Stub third-party APIs: a `<PROVIDER>_BASE_URL` setting points at a stub server in tests.
5. **Error monitoring** on every client and the server, with source maps or symbol files.
   - Without it, a user's "something went wrong" can't be diagnosed.
   - Load it in a way that doesn't slow the first screen. *Example (Travel App):* a static import of the browser SDK added about 65 kB to every page.
   - Off until its key is set.
6. **Product analytics:** a few key events only: the core create action, invite, upgrade.
7. **Config discipline:**
   - An example env file lists every variable, with a comment.
   - Only variables explicitly marked public reach a client; secrets stay on the server.
   - The build cache knows about every variable that changes the build.
8. **Agent and project docs** (from the templates):
   - `AGENTS.md`: short rules plus a pointer to this playbook and the profile.
   - `ROADMAP.md`, with `[x]` / `[~]` / `[ ]` status and an action plan at the top.
   - `docs/playbook/`: this file and the stack profile, synced from `app-playbook` and never edited in the app.
   - `docs/APP_NOTES.md`: app-only quirks, and lessons to promote to the master.
   - `CHANGELOG.md`: what shipped and when (§9).
9. **Workflow rules from the first commit** (§9): small PRs, `dev-<topic>` branches, migration naming.
10. **Branch protection** once more than one person pushes: require a PR and the CI checks on `main`. (GitHub doesn't enforce rules on private repos on the free plan; until you upgrade, merge only when every check is green.)

---

## 3. Phase 2: data model and security

Model the data with security built in. Every rule here is enforced by the database or the one API every client goes through, so it holds for web, native, desktop, scripts and future integrations (§1.2b). *(profile §P2)*

### 3.1 Access rules on everything

- Every table (or collection) gets access rules **in the migration that creates it**.
- Write rules through helper functions, e.g. `<resource>_role_for(<resource>_id)`, rather than repeating joins.
- Every rule gets a test: allowed cases *and* denied cases.

### 3.2 Lock privileged fields

- Users must never be able to write their own `plan`, `role`, `comp_*` or similar fields.
- Grant writes only on the fields users may edit.
  - *Example (Travel App):* it shipped with this hole at first: any user could set their own plan to paid.
- Only a privileged server process writes privileged fields, e.g. a billing webhook.

### 3.3 Plan limits and features as data, from the start

- **Plan defaults** in a table (`plan_features`).
- **Per-user overrides** with optional expiry and a reason (`entitlement_overrides`).
- **Free upgrades** granted by an admin, with an end date (`comp_plan`, `comp_plan_until`).
- **Resolution** in one place (a database function, or one server function), exposed as "my entitlements" and resource-level entitlements.
- **Enforcement** in the database or API calls that same resolution.
- **Shared core** keeps a fallback matrix and a `FeatureKey` registry, with a unit test that the seed data matches it.
- **Why from the start:** hard-coding limits means rewriting every check when the admin portal arrives. *(Travel App did exactly that.)*

### 3.4 Shared entities are normalized

- **Make things that can appear in several places their own table.** Examples: a place, a person, a file.
- **Don't copy their fields onto every table.** A single shared table referenced by each makes cross-cutting views (a map, a gallery, "near here") and refreshes simple.
  - *Example (Travel App):* it copied place ID and coordinates onto plans, bookings and destinations, and later had to migrate them into a `places` table.
- **Store a third party's ID plus a `*_fetched_at` timestamp** when the provider limits how long you may cache its data.

### 3.5 Lifecycle: deletion, audit and history

- **Account deletion is designed up front:**
  - links from shared content to its author are cleared on delete, so content in other people's `<resource>`s survives without attribution;
  - the user's own resources are removed by an explicit, privileged-only function.
- **The audit log is append-only, enforced by the database,** so even privileged code can't edit history.
  - Store who, what, before and after, and the reason.
  - Store a snapshot of anything deleted.
- **Activity and audit writers survive edge cases:**
  - Skip bookkeeping-only updates, e.g. only `updated_at` changed.
  - Look up the actor safely, so a deleted user doesn't break inserts.

### 3.6 Settings and kill switches

- An `app_settings` table, readable by the app and writable only by admins.
- Every per-request paid API gets a switch, and the feature falls back gracefully when it's off.
- Also: pause sign-ups, and an announcement banner.

### 3.7 Time and money

- **Times:**
  - Store instants with a time zone (UTC instants).
  - Show them in the **resource's** time zone when it has one (e.g. a trip's or an event's), not the device's. Keep DST-safe helpers in the shared core.
  - Store what's printed on a document as local wall-clock time, e.g. "departs 10:35".
- **Money:**
  - Exact decimals (never floats) with a currency code.
  - Total per currency, and never guess exchange rates.
  - Splits are cent-exact, with minimal settle-up transfers.

### 3.8 Tokens and links

- Tokens are random and unguessable (UUIDv4 or better).
- Invite links are single-use, expire (default 14 days) and can be revoked.
- Only follow `?next=` style redirects to same-origin paths. For native apps, use universal links / app links, not custom schemes, for anything sensitive.

### 3.9 Generated types

- Regenerate types from the real schema after every migration, with one script.
- Never hand-write row types. CI fails when the committed types differ.

---

## 4. Phase 3: design system (before building screens)

Agree the system with mock-ups of the navigation, a form sheet, a card, an empty state and an error state *before* feature work. *Example (Travel App):* it designed after building, and the navigation went through several rounds. *(profile §P3)*

### 4.1 UX principles

1. **It should feel instant even when the network isn't.**
   - Forms close on submit and save in the background.
   - Deleted rows disappear straight away.
   - Show skeletons, not spinners, plus a progress indicator on navigation.
2. **Never trap the user.**
   - Every error state has Retry, Back and Home.
   - A failed save gives the form back with the input intact.
   - No full-screen dead ends.
3. **Thumb reach.** Primary actions sit at the bottom: the tab bar, the docked "+", bottom sheets. Destructive actions are small, out of the way, and confirmed.
4. **One "+" per screen, and it means the obvious thing on that screen.**
   - A list of X → add an X.
   - Use a quick-add menu only where no single action is obvious (e.g. an overview screen).
   - *Example (Travel App):* Plan → add a plan; People → add a person; Overview → quick-add menu.
5. **Offline is a feature.**
   - An offline banner.
   - Visited screens stay readable.
   - Failed saves explain why.
6. **Premium gates sell.**
   - Show the locked feature with a preview and one upgrade prompt.
   - Never show an error, and never hide the feature.
7. **Respect each platform.** Shared design tokens, but native navigation patterns, gestures and system controls on each OS (back gestures on Android, sheets and swipe actions on iOS, menus and keyboard shortcuts on desktop).

### 4.2 Tokens and theming

- **Every colour, radius, spacing and shadow is a named token,** defined once.
- **Light, dark and auto** from the same tokens.
- **Palettes are swappable,** optionally with a custom accent for paid users. *(Travel App ships 9.)*
- **The theme is applied before the first paint,** so there's no flash on load.
- **Token values live in the shared core,** so every client uses the same palette.
- **Brand in one place:** name, logo, colours, copy.

### 4.3 Layout and navigation

- **Phones:** a tab bar above the safe area, with a docked centre "+" on screens that have one primary action.
- **Larger screens:** the same destinations as a sidebar or a top row, with multi-column content.
- **Translucent or blurred surfaces** only for floating chrome. Otherwise generous radius and soft elevation.
- **Category colours** drive card accents.

### 4.4 Components

- **Sheets for forms:** a bottom sheet on phones and a centred dialog on larger screens, with focus trapping and dismiss gestures. Content mounts only while open, so every form starts fresh.
- **Empty states:** every list has one, with a single next action.
- **Copy:** plain and friendly ("Saving… / Saved"). Errors say what happened and what to do, with no codes.

### 4.5 Mobile details

- **Text inputs are large enough not to trigger zoom** (16px on iOS Safari).
- **Never disable pinch-zoom** or dynamic type; it breaks accessibility.
- **Respect safe areas** for every fixed bar.
- **Horizontal scrollers:** side padding, snapping, and auto-scroll to the current item (e.g. "today").

---

## 5. Phase 4: app shell and shared patterns

*(profile §P4 has the implementation for each)*

### 5.1 Session and request context

- **One backend client per request** (server) or per signed-in session (native).
- **Verify sessions cheaply** (e.g. verify a signed token locally) and only make a network call when you must confirm the account still exists.
- **One "viewer" loader** returns the profile, entitlements and admin role together. Screens that need a user redirect when there's no session.
- **Load each screen's data in one parallel batch,** so a screen costs one round trip.

### 5.2 Data access layer

- **Screens never talk to the database directly.** They call data functions, e.g. `<resource>s.list()`, `items.save(item)`.
- **Server-first:** these wrap backend queries and server mutations.
- **Local-first:** the same functions read and write the on-device database.
- **Why:** this layer is what lets you switch sync strategy or add a client without rewriting screens. *Example (Travel App):* its screens queried the database directly, so moving to local-first touched every screen.

### 5.3 Mutations and forms

- **One form wrapper** used by every form:
  - closes optimistically;
  - shows a "Saving… / Saved" status;
  - on error, reopens the form with the input intact;
  - has a **watchdog** (e.g. 12s) that turns a stuck save into a retryable error;
  - turns network failures into a message.
- **Mutations always return a result object** (success, field errors, or an error plus an optional upgrade key) and never throw to the UI.
- **Every mutation re-checks entitlements** server-side and returns `{ error, upgrade: featureKey }` so the UI can show the upgrade prompt.
- **Every mutation refreshes every view that shows the changed data** (other tabs, feeds, counts), not just the current screen.
- **Deletes hide the row instantly and restore it if the delete fails.**

### 5.4 Loading and errors

- **Every screen has a loading state (skeletons) and an error state,** plus an app-wide fallback. They share one error screen with Retry, Back and Home.
- **Navigation chrome renders without waiting on data.**

### 5.5 Performance

- **Visited screens are cached on the client,** so moving between tabs is instant; your own edits always refresh them.
- **App-wide settings are cached briefly** and invalidated when an admin saves them.
- **Server code runs in the database's region.**

### 5.6 Offline baseline (server-first)

- Web: a manifest and a service worker with a versioned cache, network-first pages, the cache cleared on sign-out, and admin pages excluded.
- Native: cache the last-loaded data on the device.
- An offline banner everywhere.
- Local-first apps replace most of this with the sync engine (§1.4).

### 5.7 Live updates

- When several people edit the same data, design for realtime updates (or the sync engine) from the start. Lists and feeds update in place, without a refresh.

---

## 6. Phase 5: feature catalogue (reusable modules)

These are worth bringing to almost any app. The profile's P5 lists where each lives in its reference implementation, so you can copy the code.

### 6.1 Generic modules (bring to every app)

| Feature | What it gives you |
|---|---|
| **Auth flows** | Sign-up/login with inline validation, forgot/reset password, email-link callback, safe redirects; social sign-in (Sign in with Apple is required on iOS if you offer other social logins) |
| **Session helpers** | Auth user, viewer (profile + entitlements + role), require-user guards |
| **Theming** | Light/dark/auto, palettes, custom accent, applied before paint, live preview, premium themes gated |
| **Entitlements & plans** | `FeatureKey` registry, `can()`, `limitOf()`, server-resolved entitlements; pricing copy generated from live values; upsell copy per feature |
| **Upsell UI** | One `<Upsell feature=…>` component, paid-tier badges, preview-before-buy |
| **Admin portal** | Owner/admin/support roles checked server-side, two-factor required, bootstrap owner via config, user search, free upgrades, per-user overrides, resets, suspend, audited delete |
| **Settings / kill switches** | Pause sign-ups, announcement banner, turn paid APIs off |
| **Action forms** | Optimistic save, status pill, watchdog, error recovery |
| **Sheets** | Bottom sheet / dialog, controlled or trigger mode |
| **Confirm / optimistic delete** | Instant hide, restore on failure |
| **Error screens** | Retry / Back / Home everywhere |
| **Skeletons / empty states** | Consistent loading and empty UI |
| **Offline** | Manifest, icons, service worker or on-device cache, offline banner |
| **Invite links** | Token links, preview page, sign-up round trip, collaborator limits |
| **Activity feed** | Written by the database, readable when entitled |

### 6.2 Domain modules (bring when the app needs them)

Built for Travel App, reusable wherever the same need appears: calendar export (`.ics`), place autocomplete through a server proxy (session tokens, location bias, plain-text fallback, test stub), DST-safe time-zone helpers, multi-currency totals and cent-exact splits, seeded starter templates.

---

## 7. Phase 6: money and operations

*(profile §P6)*

- **Gates in three layers:**
  - the UI, with `can()`;
  - the server mutation;
  - the database or API, which is the real gate.
- **Locked features stay visible,** with an upgrade path.
- **Pricing copy is generated from the live plan values,** so it's never stale.
- **Billing:** checkout, a customer portal, and a webhook running with privileged access. The webhook is the only writer of `plan`. In-app purchases feed the same plan tables.
- **The admin portal** (from §6) ships before launch. It provides:
  - roles;
  - two-factor login;
  - free upgrades;
  - per-user overrides;
  - audited deletion;
  - kill switches.
- **Watch the cost** of every paid API. Each one has a switch and a budget alert.

---

## 8. Phase 7: ship and operate

*(profile §P7)*

### 8.1 Test layers

| Layer | Covers |
|---|---|
| Unit (shared core) | Maths, time zones, entitlements, schemas |
| Database / API rules | Access allow and deny, limits, triggers, audit |
| End to end | Real flows on a phone size and on desktop (and each native target), with third-party stubs |
| Static | Typecheck, lint, build of every client, on every push |

### 8.2 Deploy checklist (a feature with a migration or new config)

1. CI is green on the PR, and migrations have been applied on staging.
2. Merge.
3. Apply migrations to production.
4. Add new config and secrets to the hosting provider.
5. Redeploy (and for native: submit the build; remember store review time, and keep the server backwards-compatible with app versions still in use).
6. Smoke-test on a real phone.
7. Watch error monitoring for an hour.
8. Release the changelog: rename `## Unreleased` in `CHANGELOG.md` to today's date (§9).

Any step a person has to do by hand goes into the ROADMAP's "Waiting on you" list until it's done.

### 8.3 Monitoring

- **Error monitoring** alerts on new errors.
- **Analytics** cover the key funnel.
- **The admin audit log** reviews sensitive actions.

---

## 9. Workflow conventions

- **Branches:**
  - `main` is the default.
  - One `dev-<topic>` branch per feature, cut from `main`.
  - Never add unrelated work to an open PR.
- **PRs:**
  - Small and focused.
  - Each one updates `ROADMAP.md` (and this playbook or the profile, if it taught something).
  - Each one that changes something a user would notice adds a line under `## Unreleased` in `CHANGELOG.md`.
  - The description is only the content (what changed, how it was checked). **No attribution lines:** no "Requested by" line or link back to the chat or project thread that asked for it, and no "Generated with" tool footer or session link. Leave them out even when a tool's default template adds them.
- **Commits:**
  - Authored as `{{GIT_AUTHOR}}` (set in each app's `AGENTS.md`).
  - **No `Co-Authored-By` lines.**
  - Short, imperative messages.
- **Changelog** (`CHANGELOG.md` in each app, newest first):
  - What shipped and when, in words a user would understand. The ROADMAP holds plans and status; git history and PRs hold developer detail.
  - Entries start with **Added**, **Changed**, **Fixed** or **Removed**, e.g. "Added: CSV export for Pro". Leave out refactors, tests, docs and dependency bumps unless users notice them.
  - PRs add lines under `## Unreleased`. At a deploy (§8.2), or at least weekly when every merge deploys, rename it to the date (`## 2026-11-05`) and start a new `## Unreleased`.
  - Use dates while the app ships continuously on the web. Add version numbers (`## 1.4.0 — 2026-11-05`) once there are store builds, which need them.
- **Migrations:**
  - One per feature.
  - Named with a sortable timestamp prefix, e.g. `YYYYMMDDHHMMSS_topic`.
  - Re-dated to sort last if another branch merged first (many migration tools skip files older than the latest applied).
- **Features that need a key or account nobody has yet:**
  - The feature falls back gracefully when the key is missing.
  - Otherwise, keep the PR open.

---

## 10. Gotchas that apply on any stack

Stack-specific gotchas are in the profile's P8.

- **A side effect hidden in one entry point.** *Example (Travel App):* the first-owner bootstrap ran only when `/admin` was opened, while the menu read the table directly, so the Admin link never showed. Anything that decides "is this user X?" goes through one function that every caller uses.
- **Don't share one "which sheet is open" state across forms whose saves run in the background.** A save that finishes late closes the sheet you've since opened. Give each open form its own id, and remove only that one when it closes.
- **A summary screen must compute with the same rule as the detail screen.** If a `<resource>` has a setting that changes how its numbers are worked out, a home or overview screen that recomputes them has to read the same setting through the same function. *Example (Splitly):* Home always simplified debts while the group page respected the group's "simplify debts" setting, so the two named different people as owing you.
- **Refresh every view of changed data, not just the current screen,** or other tabs and feeds show stale data until their cache expires.
- **A cleared required text field:** store an empty string, not null, and save a whole settings form in one write; a null once caused a partial save.
- **"Works locally, fails on hosted"** is usually an unapplied migration, a missing config value, or a function that exists locally but not in the hosted database. CI on a fresh database catches the first and third.
- **Hunt a flaky bug by repeating the test, not by retrying it.** Run the one test dozens of times in parallel; it reproduced a 1-in-20 hang in minutes. Add temporary logging around the failing step.
- **Don't allow test retries in CI.** They turn a real bug into a green run marked "flaky".
- **Before "fixing" the app for an e2e failure, check the test.** Common causes: native validation blocking submit, a key press with nothing open, a desktop-only label, data left over from another spec.
- **Tests own their data.** Seed users through the admin API and delete them afterwards. Tests that change app-wide settings run last and put everything back.
- **`pgrep -f <pattern>` matches its own shell.** Use a PID file or a restart script.

---

## 11. Dos and don'ts

**Do**
- Make the Phase 0 decisions, including the platform (§1.2), and write them down with a "revisit if".
- Set up CI, staging, error monitoring and repo-tracked e2e tests before the first feature.
- Put access rules and a test on every table.
- Put every gate behind `can()` and a database or API check, with every limit in the feature registry.
- Normalize shared entities, and store third-party IDs with a `*_fetched_at`.
- Go through the data access layer; keep the shared core free of UI framework imports.
- Use optimistic UI with a rollback and a watchdog. Return a result object from every mutation.
- Design at phone width first. Mock up the system before building screens.
- Add a kill switch for every paid API, and keep secrets on the server.
- Record a reason for destructive admin actions, and write the audit entry with them.
- Add a user-facing changelog line in the same PR as the change.

**Don't**
- Don't start server-first if offline editing or a native app is likely.
- Don't pick more UI codebases than the team can maintain.
- Don't put rules only in web server code if any other client exists or is likely (§1.2b).
- Don't let users write `plan`, `role` or `comp_*` fields.
- Don't use privileged database access on a path a user can reach without an admin check.
- Don't hard-code limits, or pricing copy, in code paths.
- Don't hide premium features; show them locked with an upgrade path.
- Don't apply the theme after first paint, or make navigation chrome wait on data.
- Don't disable pinch-zoom or dynamic type.
- Don't bundle unrelated work into one PR.
- Don't merge a feature that depends on a key nobody has yet, unless it falls back gracefully.

---

## 12. New app checklist (in order)

Each step has a matching stack-specific step in the profile's P9.

**Phase 0: decide**
1. Name the app and its `<resource>`.
2. Primary device and context.
3. Platform strategy (§1.2).
4. Stack and profile (§1.3).
5. Server-first vs local-first.
6. Money model, who pays for shared resources, and billing channel.
7. Paid third parties, with caching terms and kill switches.
8. Write the decisions table in `ROADMAP.md` (§1.7).

**Phase 1: foundation**
9. Create the repo from `app-starter`, run the playbook sync once, and fill in the placeholders in `ROADMAP.md` and `AGENTS.md`. Set up the monorepo with the shared core.
10. Local, staging and production environments, with the database and app server in the same region, and two-factor auth available.
11. CI: lint, typecheck, unit tests, migrations on a fresh database, access-rule tests, generated-types check, builds and e2e. Error monitoring and analytics wired up.
12. Hosting configured, with config mirrored where the build cache needs it.

**Phase 2: data**
13. First migration: profiles with a locked `plan`; core tables with access rules and role helpers; shared entities normalized.
14. Entitlements tables and resolution, `app_settings`, the audit log, and the deletion functions.
15. Tests for all of it. Generate types.

**Phase 3: design**
16. Mock up the navigation, sheet, card, empty state and error state.
17. Design tokens and theme applied before paint. Rename the palettes and brand.

**Phase 4: shell**
18. Session helpers, mutation helpers, and the shared components (form wrapper, sheet, confirm, error screen, skeleton, empty state, upsell).
19. The data access layer, or the sync engine if local-first.
20. Navigation (one "+" per screen), plus loading and error states for every screen.
21. Offline baseline.

**Phases 5–7**
22. Features from the catalogue (§6).
23. The admin portal, with a bootstrap owner.
24. Billing.
25. Deploy checklist and monitoring.

---

## 13. Keeping this playbook general

- **One master copy.** The master playbook and every profile live in [`yauwilson/app-playbook`](https://github.com/yauwilson/app-playbook). The app templates live in `yauwilson/app-starter`. Each app has read-only copies in `docs/playbook/`, refreshed by a weekly sync PR. `SYNC.md` has the full process.
- **Lessons flow back.** Apps mark lessons under "To promote" in `docs/APP_NOTES.md`, and an agent turns them into a PR here: stack lessons to the profile, general lessons to this file, template changes to `app-starter`.
- **Write rules generically, then give the case.** State the rule in terms of `<resource>`, then add a one-line *Example (App name):* with what happened. App-only details (screen names, product copy, pricing) belong in that app's `ROADMAP.md`.
- **New stack, new profile.** Copy an existing profile's P1–P9 layout and fill each section as you learn it. Don't put stack details in this file.
- **Change a recommendation only with a reason.** If an app picks differently from §1.2's default and it works out better, update the default and say why.
