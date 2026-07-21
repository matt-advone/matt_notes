---
type: ProjectStatus
related_to: "[[tolaria]]"
status: Active
---

# Status Data Normalizer - Notes

## Last updated
2026-07-21

## Current session
Reviewed `/Users/mattperzel/src/engineHoursAudit/checker/compare-report.html` and `compare_audit.py` against `/Users/mattperzel/src/statusDataNormalizer`.

The report found 869 forward-time, same-diagnostic engine-hour decreases across 680 devices. Representative decreases are substantial, so they are not mainly floating-point jitter. However, the report was generated on 2026-07-20 from archived files and contains readings dated 2026-06-28 through 2026-07-06. The normalizer's `GetValidHistory` enforces a rolling 168-hour window relative to wall-clock time, so these rows could not have been returned by the live engine-hours API at report-generation time. The audit is therefore not a direct validation of the current API output.

## Findings
- Normal feed and adjustment paths validate before storing, rejecting any lower engine-hours value.
- `Poller.Bootstrap` clears `result.Data` before checking whether the page was shorter than `resultsLimit`, so it always exits after page one. This produces an incomplete bootstrap and can suppress historical data.
- `dedupAndSort` uses diagnostic ID, timestamp, and value but omits device ID. A fleet-wide result with two devices sharing those three values drops one device's row.
- Bootstrap directly seeds `DeviceStatusInfo` values with `AddValid` without validator checks. Together with the pagination defect, subsequent older replayed records are likely rejected against a newer seeded value.
- Valid engine-hour data is intentionally retained only for the configured time window and capped/downsampled; this explains expected row loss for historical queries.

## Verification
- `go test ./internal/feed -run '^(TestProcessRecords|TestPoller|TestCompactStores)' -count=1` passed.
- `go test ./internal/api ./internal/validator ./internal/store` passed.
- The four `TestServedSequence_*` tests currently fail because their fixed June/July 2026 fixtures are outside the 168-hour rolling retention window as of this session, yielding zero returned samples. Their dates should be based on `time.Now()` or the store should accept an injectable clock.

## Next up
- Add a multi-page bootstrap integration test using a mocked Geotab client.
- Add pagination support for bootstrap DeviceStatusInfo seeding; it currently makes one upstream request and can omit inactive devices beyond the first page.
- Export directly from the deployed normalizer's `Get StatusData` or `GetFeed` endpoint within the live retention window, retaining source/version metadata, then rerun the audit.
- To prove whether the remaining report drops are from a deployed normalizer version, export directly from its `Get StatusData` or `GetFeed` endpoint within the live retention window, retaining source/version metadata, then rerun the audit.

## 2026-07-19 Implementation update
- Fixed engine-hours acceptance so ECU and adjustment pollers validate and append under one `EngineHoursStore` lock. This prevents concurrent stale-history validation.
- Engine-hours validation now rejects records with an earlier timestamp, permits only flat values at an equal timestamp, and caps any configured rate multiplier at 1.0. An accepted sequence is therefore timestamp-ordered, nondecreasing, and bounded by elapsed wall-clock time.
- Sanitized restored snapshots by timestamp and the same physical invariant, rebuilding `lastValid` from retained samples. Existing legacy invalid samples cannot be served after a restart.
- Fixed bootstrap page termination by retaining the feed page count before freeing its buffer.
- Fixed API deduplication so different devices with equal diagnostic/timestamp/value records are not collapsed.
- Added regression coverage for late higher readings, strict same-timestamp handling, multiplier clamping, and cross-device deduplication. Updated served-sequence fixtures to use clock-relative timestamps so rolling retention does not invalidate them.
- Verified with `go test ./...` from `/Users/mattperzel/src/statusDataNormalizer/server`.
