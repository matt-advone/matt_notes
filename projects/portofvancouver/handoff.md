---
type: ProjectStatus
related_to: "[[portofvancouver]]"
status: Active
---

# Port of Vancouver — Notes

## Last updated
2026-07-20 08:40 PT

## Project location
`/Users/mattperzel/src/portofvancouver`

## What the project is
PoV drayage data (DLT + DPW terminals) — gate appointments and truck visits
CSVs, plus VFPA wait-time engine rules and the WTS BRD PDFs. Goal this
session: (1) analyze the CSV headers for common/differing fields, (2) build
a MyGeotab API client that pulls the full Device list with paging, (3)
match the local CSV asset identifiers against the Geotab device list.

## What was done this session

### 1. `analyze_headers.py`
- Scans every `*.csv` in its directory (or a given dir).
- Per-file header inventory + row counts.
- Common headers across ALL files (case-insensitive) and within file-type
  groups (GateAppointments / TruckVisits).
- Detects unique headers per file and candidate join fields for Geotab.
- Output: `headers_report.md`.
- Verified: 4 CSVs scanned (DLT/DPW × GateAppointments/TruckVisits).

### 2. `geotab_devices.py`
- JSON-RPC 2.0 client over `https://<server>/apiv1`.
- Authenticate (follows server redirect via `path`), re-auth on session
  expiry. Gzip/deflate handling.
- `Get<Device>` with **paging**: sort by `id` (unique field), `resultsLimit`
  page size, `offset` = last record's id, loop until empty page. `lastId`
  omitted when sorting by `id` (throws ArgumentException otherwise — per
  Geotab docs).
- Optional `--fields` PropertySelector to slim payload.
- Reads creds from env / `.env` (GEOTAB_USER, GEOTAB_PASSWORD,
  GEOTAB_DATABASE, GEOTAB_SERVER).
- **Ran successfully**: 2933 devices downloaded → `geotab_devices.json`
  (12.4 MB, full objects).

### 3. `geotab_groups.py`
- Pages `Group` entities (116 groups, all flat, no parents) →
  `geotab_groups.json`.
- Needed to resolve `Device.groups[].id` -> carrier group name.

### 4. `match_assets.py`
Two-pass matcher:
- **Pass 1 (naive):** direct field matching of CSV id columns vs Device
  name/licensePlate/VIN/serialNumber/comment. Confirms no native join key
  exists (the `comment` substring hits are false positives from short
  numeric plates matching inside longer comment strings).
- **Pass 2 (carrier-aware):** curated `TRKC_TO_CARRIER` table maps the 89
  CSV carrier codes to Geotab carrier group names; narrows candidates to a
  carrier, then matches the drayage-plate numeric suffix against
  `comment`/`licensePlate`/`name` digits.
- Outputs: `match_report.md`, `geotab_matches.csv`.

### Key data-shape findings
- CSV `TRUCK_LICENSE_NBR` = **drayage plate** = `<4-letter carrier><unit#>`
  (e.g. `WCOA1927`). Geotab stores NONE of this directly.
- Geotab `Device.name` = GO device serial (`G9DF2102A955`), sometimes with
  `_CarrierName` or `_VIN` suffix (938/2933 have a suffix).
- Geotab `Device.comment` = carrier unit number (`PW 78`, `R126`,
  `CVAP0019`) but only **6.6% populated** (193/2933).
- Geotab `Device.licensePlate` = actual government plate (`XX####` BC
  format), 48% populated — different identifier from the drayage plate.
- Geotab `Device.groups[].id` -> carrier name (resolved via Groups fetch).

### Match results
- 2933 devices, 1317 CSV drayage plates.
- Carrier mapping: 58/60 TRKC codes (in TruckVisits files) map to a
  Geotab carrier group with devices present.
- Per-vehicle matches: **29/1317 (2.2%)** via comment/licensePlate/name
  digits. Limited by `comment` population (6.6%).

## Current state / verified
- All 4 Python scripts compile (`python3 -m py_compile`).
- `analyze_headers.py` runs, produces `headers_report.md`.
- `geotab_devices.py` + `geotab_groups.py` ran live against
  `my.geotab.com` database `portofvancouver`.
- `match_assets.py` runs and produces `match_report.md` +
  `geotab_matches.csv`.
- `.env.example` and `.gitignore` created; real `.env` is gitignored.

## Next steps / open questions
- **Extend `TRKC_TO_CARRIER`** for the ~30 unmapped TRKC codes (some have
  no Geotab group at all — e.g. Forfar Enterprises `HUDD` has 78 CSV plates
  but 0 Geotab devices, suggesting that carrier isn't tracked in Geotab).
- **Populate `Device.comment`** with the drayage-plate unit number across
  all active devices to enable high-coverage per-vehicle joins. This is
  the single biggest lever — it would jump match coverage from ~2% toward
  ~100%.
- Alternatively: push the full drayage plate into a Geotab field
  (`vehicleIdentificationNumber`, a custom property, or `licensePlate`).
- **Manual alias table** (drayage plate -> Geotab device id) is the only
  option for carriers with no Geotab comment data and no shared identifier.
- Consider matching by VIN if PoV can provide the truck VINs (CSVs only
  have container `EQ_NBR`, not truck VINs).

## Reminders / gotchas
- Geotab `Get<Device>` paging: when sorting by `id` (unique), do NOT pass
  `lastId` or you get `ArgumentException`. Pass `offset` only.
- CSV headers are inconsistently cased across terminals (DLT lowercase,
  DPW uppercase) — matcher normalizes with `.lower()`.
- DLT `TruckVisits` has `stage_id`; DPW has `next_stage_id` — only header
  difference within the TruckVisits pair.
- `EROR` TRKC code (5 plates) appears to be an error bucket (plates like
  `PA178`, `WE938`) — not a real carrier.
