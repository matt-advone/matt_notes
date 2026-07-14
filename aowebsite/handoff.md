# Advantage One website — session handoff

## Project overview

- **What:** Advantage One marketing/website — a static-prerendered Vite + React app (`vite-react-ssg`). Every route in `src/App.tsx` is built into a standalone HTML file with real visible content, unique title, meta description, JSON-LD, and Open Graph tags at the static-HTML level. A postbuild validator (`scripts/validate-build.mjs`) enforces this; `npm run build` fails if any route is missing required metadata.
- **Where:** `/Users/mattperzel/src/aowebsite`
- **Build:** `npm run build` = `tsc -b && vite-react-ssg build && npm run sitemap && npm run validate`. Authoritative. Dev: `npm run dev` (CSR; per-route `<title>` does not update in dev due to a react-helmet-async + React 19 incompat — works in prod).
- **Rules:** See `CLAUDE.md` at the repo root (build commands, page checklist, validator contract) and `.kilo/rules/heroes.md` (standardized hero styles A/B/C).

## Session 2026-07-14 — RTA Fleet360 rename + integrations page updates

### What was done
1. **Renamed "RTA Fleet" → "RTA Fleet360"** across all user-facing and machine-readable references:
   - `src/data/mockData/nav.ts` — integrations dropdown label (already done in prior step, position kept).
   - `src/data/mockData/integrations.ts` — `fmisPartners[].name`, `rtaIntegrationData.headline`, and `rtaIntegrationData.subheadline` (now "RTA Fleet360 Management Software").
   - `src/pages/AboutPage.tsx` — FMIS partner listing name.
   - `src/pages/RTAPage.tsx` — `<PageHead title>` and JSON-LD `SoftwareApplication.name`.
   - `public/llms.txt` — integrations hub line and the RTA integration line.
   - Left standalone "RTA" references untouched (e.g. "Geotab + RTA", badge "Geotab + RTA Integration", "RTA's maintenance platform") since those aren't "RTA Fleet".
2. **Moved RTA to first in the integrations nav menu** — `integrationLinks` in `nav.ts` now reads: FMIS Overview, RTA Fleet360, Faster Web, Fleetio, AssetWorks.
3. **`/integrations` page "Supported FMIS Platforms" grid:**
   - Reordered `fmisPartners` so RTA Fleet360 is first (RTA Fleet360, Faster Web, Fleetio, AssetWorks). JSON-LD `ItemList` `position` auto-updates from array index.
   - Made each platform card `flex flex-col` and added `mt-auto` to the "Learn More" link so the CTAs align vertically across cards with differing description lengths.
4. **Committed** as `6a18a1d` "Reorder RTA Fleet360 to first on integrations grid and align card CTAs" (3 files: `src/data/mockData/integrations.ts`, `src/pages/IntegrationsPage.tsx`, `pnpm-workspace.yaml`). Branch `main` is 1 commit ahead of `origin/main`; NOT pushed.

### State / verification
- `npm run build` passes: 28 routes generated, 0 errors, 0 warnings. `integrations/rta.html` title = "RTA Fleet360 Integration — Geotab + RTA — Advantage One".
- No remaining "RTA Fleet" (without 360) references outside `dist/` (verified via ripgrep).
- A local code review (uncommitted) returned APPROVE with no issues.

### Next steps / open questions
- Push `main` to `origin/main` when ready (user has not asked).
- The `pnpm-workspace.yaml` change (`sharp: true`) rode along in the commit — intentional-ish (enables the sharp build); flag if it shouldn't have been included.
- Dev server note (pre-existing, not from this session): per-route `<title>` doesn't update in `npm run dev` due to react-helmet-async + React 19 incompat; fine in production builds.
