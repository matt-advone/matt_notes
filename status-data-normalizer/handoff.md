---
type: ProjectStatus
related_to: "[[tolaria]]"
status: Active
---

# Status Data Normalizer - Notes

## Last updated
2026-07-19 20:46 PDT

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
- Fix bootstrap pagination by saving the page length before clearing `result.Data`, and add a multi-page bootstrap test.
- Include `Device.ID` in `dedupAndSort` keys and add a two-device collision test.
- Make served-sequence tests independent of wall-clock time.
- To prove whether the remaining report drops are from a deployed normalizer version, export directly from its `Get StatusData` or `GetFeed` endpoint within the live retention window, retaining source/version metadata, then rerun the audit.
