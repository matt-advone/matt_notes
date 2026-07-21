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
