# Geotab Add-In Skills & Generator — handoff

## What this project is

A scaffolding kit for vibe-coding MyGeotab add-ins with Claude Code / Kilo Code.
Lives at **`/Users/mattperzel/src/geotab_addin_skills/`**.

- `createGeotabAddin.ts` — interactive generator, run with `node createGeotabAddin.ts`
  (Node 22.18+; older Node use `npx tsx`). Asks for directory (confirms creation
  with full absolute path), add-in name, menu label, support email, default
  theme (light/dark/system + whether the toggle shows), S3 bucket/prefix/region,
  and MyGeotab server/database/username for local dev. Then: copies `template/`,
  stamps `__PLACEHOLDER__` tokens, writes `.env`, mirrors `.claude/skills/*` →
  `.kilocode/rules/NN-*.md`, `git init` + initial commit.
- `template/` — the full generated project: React 18 + TS + Tailwind 3 + Zustand,
  Vite 5.

## Session 2026-07-20 — initial build

Key decisions:

- **Dual-mode Geotab client** (`template/src/geotab/api.ts`, singleton
  `geotabApi`): add-in mode wraps MyGeotab's callback `api` in promises;
  standalone mode does JSON-RPC to `https://{server}/apiv1` with `.env`
  credentials (`VITE_GEOTAB_*`), caches the session in sessionStorage, handles
  the federation redirect (`result.path`), and re-auths on
  `InvalidUserException`. Components never call it directly — Zustand store
  actions do.
- **Stores**: devices / groups / deviceStatus follow one template (`items`,
  `byId` Map, `loading`, `error` string, `lastFetched`, guarded `fetch()`),
  plus `startPolling` on deviceStatus and `fetchCoreData()` to load all three.
  Theme store: light/dark/system, persisted, applies `dark` class to the **app
  root div** (never `<html>` — MyGeotab injects the add-in into its own DOM;
  all CSS is scoped under `.ao-app`).
- **Design system** from advantageone.tech (sampled live via browser):
  dark navy `#061423` / surfaces `#0F1C2C` `#1E2B3B`, text `#D6E4F9`, blue
  `#0062A0` (light) / `#4E96D5` (dark), orange accent `#F15A22`, Inter +
  Plus Jakarta Sans, 8px radius. Tokens are `R G B` CSS-variable triplets in
  `src/index.css` mapped to semantic Tailwind colors (`bg-surface`,
  `text-muted`, `bg-primary/10`…) so one class works in both themes; `ao-*`
  component classes + React UI kit (Card/StatCard/Badge/Button/Spinner/
  EmptyState/Layout).
- **Build/deploy**: `ADDIN_BASE_URL` in `.env` becomes Vite's `base` so built
  asset URLs are absolute (required — MyGeotab fetches and injects the HTML).
  `scripts/deploy.sh` syncs to S3: hashed assets immutable, `index.html` +
  `addin.json` no-cache. `public/addin.json` is the manifest (also in `dist/`).
- **AI rules**: root `CLAUDE.md` + four skills (`geotab-api`, `geotab-stores`,
  `addin-ui-styling`, `addin-config-deploy`) under `.claude/skills/`; the
  generator copies them to `.kilocode/rules/` (single source of truth is the
  skills; CLAUDE.md notes to keep the mirror in sync).
- Generator prompting uses a hand-rolled line queue instead of
  `readline/promises` — `rl.question` drops lines that arrive between
  questions when stdin is piped.

## Current state — verified

- Two end-to-end generations tested into the session scratchpad (custom
  answers + all-defaults): zero leftover placeholders, git committed (`.env`
  correctly excluded), Kilo rules mirrored, `npm install && npm run build`
  passes, dist/index.html has absolute S3 URLs.
- Dev server run and screenshotted: dark theme renders (navy palette), theme
  toggle switches to light, standalone auth failure surfaces as a friendly
  error card with Retry (message pushed into devicesStore from main.tsx).
- NOT yet tested: against a real MyGeotab database (no password available),
  actual S3 deploy, or installation inside MyGeotab.

## Session 2026-07-20 (later) — testing stack

Added the 2026-standard testing stack to the template, per Matt's request:

- **Vitest + React Testing Library** for unit/component tests (shares the Vite
  config — `test` block in `vite.config.ts`, jsdom, setup in
  `src/test/setup.ts`), **Playwright** for E2E (`playwright.config.ts`,
  `e2e/`), **MSW** for API mocking in unit tests.
- **Shared fixtures** are the key design: `src/test/fixtures/geotabData.ts`
  holds a fake fleet (3 devices/groups/statuses — one driving, one offline)
  plus `dispatchGeotabCall()`, a fake MyGeotab JSON-RPC server. The MSW
  handlers (unit) and the Playwright `page.route` mock (E2E) both answer from
  it, so no test layer ever touches a real database. Playwright's webServer
  also injects fake `VITE_GEOTAB_*` creds pointing at `e2e.geotab.invalid` as
  a second safety net.
- Sample tests double as copy-paste patterns: `devicesStore.test.ts` (happy +
  error path via per-test `server.use` override), `themeStore.test.ts` (DOM
  class + persistence), `Dashboard.test.tsx` (RTL: data/loading/error states),
  `e2e/app.spec.ts` (dashboard + theme toggle w/ reload persistence).
- New **`testing` skill** (`.claude/skills/testing/SKILL.md`, mirrored to
  `.kilocode/rules/50-testing.md`) mandates automatic test creation — a table
  of "you just did X → you must also write Y", definition of done
  (`lint && test && build`, + e2e when flows change), and "never weaken a test
  to pass". CLAUDE.md hard rules 7–8 updated to match.
- Verified on a fresh generation: 7/7 Vitest tests, 2/2 Playwright tests,
  lint + build all pass. Note: `npx playwright install chromium` needed once
  per machine (~95 MB).

## Session 2026-07-21 — MockGeotab fleet simulator

Replaced the static test fixtures with **MockGeotab**
(`template/src/test/mockGeotab/`), a deterministic fleet simulator that
impersonates the MyGeotab API for all test layers:

- `random.ts` (seeded mulberry32 PRNG), `geo.ts` (haversine/interpolation/
  jitter/hexagons), `regions.ts` (real zones: 12 GTA places incl. Oakville HQ,
  Pearson Cargo, Hamilton Port; 9 Chicagoland places), `MockGeotab.ts` (the
  simulator), plus its own 17-test suite (`mockGeotab.test.ts`).
- World model: vehicle archetypes (trucks/vans/service) based at real depots,
  per-day trip planning within work hours, physically consistent everything —
  trip LogRecords lie on the route inside the trip window with a trapezoid
  speed profile, odometer StatusData grows by exactly the trip distance,
  ignition on/off at trip boundaries, engine-hours accumulate driving+idle.
  ExceptionEvents (speeding >100 km/h, harsh braking, idling >10 min,
  after-hours) reference Rule entities; FaultData from a 7-fault J1939/OBD
  library. Entity types added to `src/geotab/types.ts` (Zone, Trip, LogRecord,
  StatusData, FaultData, ExceptionEvent, Rule, Diagnostic, FeedResult).
- API surface via `mock.dispatch()`: Authenticate, Get (deviceSearch /
  date-range / name-wildcard / groups / resultsLimit filtering), GetCountOf,
  GetFeed (versioned, incremental), ExecuteMultiCall, Add/Set/Remove.
- **`mock.next({minutes})`** advances the live world (vehicles move, trips
  complete, new trips start, DSI + feeds update) and returns a TickSummary —
  the update-reaction test pattern is proven in mockGeotab.test.ts
  (fetch store -> next(30) -> fetch -> assert change).
- Deterministic mode: same seed => byte-identical world AND evolution
  (tested). `mode:"random"` = fuzz with the seed logged/exposed for exact
  reproduction. Data never runs out — generation is procedural.
- `fixtures/geotabData.ts` is now a thin shim exposing a 3-vehicle default
  world (Truck 101 driving, Van 201 parked online, Service 301 offline —
  guaranteed by ensureDrivingNow + offlineCount). Existing exports kept.
  `geotabHandlersFor(mock)` in mocks/server.ts points MSW at a private
  instance for live tests (default shared world is read-only by rule).
- Perf: 100 vehicles x 180 days = ~47k trips / 187k StatusData / 47k events
  in ~400ms; next(60 min) ~13ms.
- Docs updated: testing skill gained a "MockGeotab simulator" section,
  geotab-api skill, CLAUDE.md, both READMEs. Verified on a fresh generation:
  lint, 24/24 unit tests, build, 2/2 e2e all pass.

Note: this handoff file was moved by the vault convention to
`projects/geotab-addin-skills/handoff.md` (was `geotab-addin-skills/`).

## Next steps / ideas

- Test with real credentials (`VITE_GEOTAB_PASSWORD` in `.env`) and a real
  deploy + install in MyGeotab.
- Planned growth path (per Matt): add reusable components to
  `template/src/components/ui/` — first a **DataTable** (sortable/filterable)
  — and document each in `addin-ui-styling/SKILL.md`, so prompts like "show
  all offline vehicles in a data table" work out of the box.
- Possibly `git init` the kit repo itself and/or publish as an npm create-*
  package.
- The kit directory itself is not a git repo yet.
