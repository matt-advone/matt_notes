# fuel-algorithm — handoff

Project: `/Users/mattperzel/src/fuel_algorithm`
TypeScript library for a fleet fuel information system built on Geotab StatusData.

## 2026-07-11 (later) — paired lifetime counters documented

Matt clarified the semantics of two related diagnostics:
`DiagnosticTotalFuelUsedId` (ECU-native lifetime total) and
`DiagnosticDeviceTotalFuelId` (same measurement, but accumulated by Geotab
server-side since telematics device install, for vehicles whose ECU only
reports ongoing fuel used). Same pairing for the idle variants. Preference:
ECU-native first, since-install as fallback — which the resolver already did;
the *why* is now documented in `docs/PLAN.md` §3.1, plus comments in
`src/diagnostics.ts` and `src/normalize.ts`. Key insight recorded: algorithms
only consume window deltas, so the differing baselines don't matter, and when
both report they cross-validate each other (agreement ratio / quality score).
Tests still 18/18, typecheck clean.

## 2026-07-11 — initial design + implementation

### What it is
Ingests the 15 Geotab fuel diagnostics (levels, lifetime/trip counters, rates,
capacity), normalizes heterogeneous per-vehicle reporting into canonical metrics
with provenance, detects refuels and drains/theft, and maintains mergeable
fixed-size aggregates (map-reduce style) so per-device and fleet trends work
without storing raw readings. Storage target is S3 (small JSON only).

**Read `docs/PLAN.md` first** — full architecture, algorithms, normalization
priority table, metrics catalog, S3 key layout, retention, roadmap.

### Key decisions
- Every statistic is a commutative monoid (Welford stats, trendline-fit sums,
  first/last, P² quantile): fleet = merge(devices), month = merge(days),
  constant memory, idempotent late-data merges.
- Canonical metrics resolve from a fixed source-priority order (ECU lifetime
  counter → device counter → trip counters → level integration) and always
  carry provenance; raw per-diagnostic aggregates are kept alongside.
- Fleet rollups resolve per device *first*, then sum — each vehicle contributes
  its best available signal.
- Counter resets: trip counters credit fuel-since-reset on falling edges;
  lifetime counters skip the gap (never invent fuel). Spikes above a burn-rate
  cap are rejected and counted as glitches.
- Refuel/drain detection: median-of-3 filter → hysteresis plateau segmentation
  → classify steps (fast rise = refuel; fast fall not explainable by plausible
  burn = drain candidate).
- Cross-source agreement ratio + 0–100 quality score make missing/disagreeing
  signals visible.

### Current state
- All source in `src/` (diagnostics, ingest, sketches, counters, levels,
  aggregate, normalize, metrics, storage, pipeline). Module map in README.
- 18 tests pass (`npm test`), typecheck clean (`npx tsc --noEmit`).
- `npm run example` runs a 7-day synthetic 3-vehicle fleet end-to-end and
  prints a fleet report + idle-fraction trendline.
- Storage is interface-based (`FuelStore`); only `MemoryFuelStore` exists.

### Next steps
- Phase 2 (per PLAN.md): real S3 `FuelStore` adapter (~30 lines with
  `@aws-sdk/client-s3`) + Geotab `GetFeed` poller runner (ECS/Fargate or
  scheduled Lambda), using the persisted `fromVersion` cursor.
- Join Trips/odometer for distance-based economy (pipeline.addDistance exists).
- Later: fuel-card reconciliation, cost price tables, driver attribution.
