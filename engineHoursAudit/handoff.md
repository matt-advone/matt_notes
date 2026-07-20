---
type: ProjectStatus
related_to: "[[engineHoursAudit]]"
status: Active
---

# engineHoursAudit — Notes

## Last updated
2026-07-19 20:35 PT

## Project location
`/Users/mattperzel/src/engineHoursAudit/checker/`

Data sources (outside the repo):
- Primary: `~/kiewitenginehouers/engine-hours/` (4045 gz files, ~59.8M records, 15,242 devices)
- Alternate: `~/kiewitenginehouers/AlternateData/engine-hours/` (1586 gz files, ~14.7M records, 9,383 devices)

## What this project does
Audits Geotab engine-hour `StatusData` records for anomalies:
- **Drops** (flat-or-increasing violation): `diff < 0 and curr_val != 0`
- **Jumps** (instability): `diff > elapsed + BUFFER_SECONDS` (BUFFER = 1200s / 20 min)

## Current session
Created a second audit script `compare_audit.py` that compares AlternateData against the
primary data and reports whether AlternateData meets the quality goal of being
"more stable AND always flat or increasing".

### Why
The original `audit.py` → `report.html` is **539 MB** (embedded all readings for every
device with issues) and can no longer render in a browser. We needed a small report that
just answers: did quality improve, and if not, what exactly is wrong with AlternateData?

### What compare_audit.py does
1. Loads both primary and alternate gz files (same loader as audit.py).
2. Per device, computes metrics: drops_count, drops_total_hours, jumps_count,
   jumps_excess_hours, readings_count.
3. Classifies each device verdict: `clean_both`, `improved`, `unchanged`, `regressed`.
4. Flags problem devices = alt still has drops (criterion violation) OR regressed OR
   unchanged-with-issues OR improved-but-not-clean.
5. Emits a SMALL HTML report (`compare-report.html`) — only problem devices get embedded
   detail (downsampled charts ≤800 pts each, raw readings windowed to ≤400 rows around
   anomalies, ≤200 issues per device).

### Result (2026-07-19 run)
- Report size: **11.47 MB** (49× smaller than 539 MB original — renders fine).
- Overall verdict: **PARTIAL** — alt reduced total issues but 680 devices still have
  drops, so it does NOT meet "always flat or increasing".
- Verdict tally (9,383 alt devices vs 15,242 primary; 9,170 overlap): improved 735,
  regressed 179, unchanged 115, clean rest.
- 1029 problem devices embedded with detailed drop/jump analysis + surrounding readings.

## Files
- `audit.py` — original full audit (generates 539 MB `report.html`, doesn't render)
- `compare_audit.py` — NEW comparison audit (generates `compare-report.html`, 11 MB)
- `report.html` — original 539 MB report (broken/too big)
- `compare-report.html` — NEW comparison report (11 MB, renders)
- `.env` — Geotab creds (mirror54 server, kiewit_mirrorpql54 db)

## Reminders / gotchas
- `.env` uses `mirror54.geotab.com` (not `my.geotab.com`).
- `BUFFER_SECONDS = 1200` (20 min) — matches audit.py; must stay in sync.
- Drop detection skips `curr_val == 0` (treated as reset, not corruption). This is the
  existing convention in audit.py; compare_audit.py keeps it.
- The `both` variable name collision bug (int count vs set) was fixed — `both_set` is the
  set, `both` is the count.
- Memory: loading 60M+ records into a defaultdict is fine on this machine but takes
  a few minutes. Non-problem device readings are freed before report build.

## Next up
- Investigate the 680 alt devices with drops — are these real data issues or a bug in
  the AlternateData generation pipeline?
- Consider streaming the load to reduce peak memory if the dataset grows.
- Possibly add a CSV export of just the problem-device alt drops for the data team.

## How to run
```bash
cd /Users/mattperzel/src/engineHoursAudit/checker
python3 compare_audit.py      # writes compare-report.html
open compare-report.html
```
