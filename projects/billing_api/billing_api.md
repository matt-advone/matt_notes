---
type: ProjectStatus
related_to: "[[tolaria]]"
status: Active
_organized: true
---

# billing_api — Status

## Last updated
2026-07-13 15:55 (Pacific)

## Current session summary
Created a monthly billing runbook and two parameterised shell scripts (`attachUSDeviceLists.sh` / `attachCADeviceLists.sh`) that wrap the existing `addAttachmentUS`/`addAttachmentCA` bootstrap handlers, replacing the manual VS Code `launch.json` workflow for attaching device lists to Zoho invoices.

## In progress
- (nothing)

## Recently completed
- Added `scripts/_attachDeviceLists.sh` (shared helper) + thin wrappers `scripts/attachUSDeviceLists.sh` (ADVA02) / `scripts/attachCADeviceLists.sh` (ADVA01) — interactive wrappers for the `addAttachment*` bootstrap handlers. Prompts for `fromDate` (defaults to **last day of previous month**, `YYYY-MM-DD` only), `bucket`, `prefix` (defaults to previous-month `deviceLists/<YYYY-MM>/{US,CA}`), `dryRun`, `customerNameFilter`, `skip`, `apiKey`. Builds via `tsc`, generates a temp ESM bootstrap in `dist/src/bootstrap/` importing `handler` from `../main.js`, runs it, cleans up. `--skip-build` reuses existing `dist/`.
- Created runbook `~/src/dev_documents/billing_api/monthly-billing-runbook.md` documenting the full monthly flow: `runUSMonthly.sh`/`runCAMonthly.sh` (upload Philip's Excels to S3 + split per customer) and the new attach scripts.
- Verified all paths end-to-end (syntax, all-defaults, custom params, bad-date validation, abort, temp-file cleanup).
- Local review fixes applied: (a) removed hardcoded Zoho OAuth clientSecret/authCode from `src/bootstrap/refreshToken.ts` (reverted to empty — use `tools/zoho-token-cli/` to refresh tokens); (b) changed `fromDate` default from today to end-of-previous-month to match the handler's forward-only `[fromDate, +10d]` invoice window; (c) deduplicated the two attach scripts into a shared `_attachDeviceLists.sh` helper.

## Key references
- Data-lake monthly scripts: `~/src/advone-sdk/packages/data_lake/helpers/runUSMonthly.sh`, `runCAMonthly.sh`, `setupUSMonthly.sh`, `setupCAMonthly.sh`
- Billing attach bootstrap handlers: `~/src/billing_api/src/bootstrap/addAttachmentUS.ts` (ADVA02), `addAttachmentCA.ts` (ADVA01)
- **New attach scripts** (use these instead of editing launch.json / bootstrap files):
  - `~/src/billing_api/scripts/attachUSDeviceLists.sh` (thin wrapper → ADVA02)
  - `~/src/billing_api/scripts/attachCADeviceLists.sh` (thin wrapper → ADVA01)
  - `~/src/billing_api/scripts/_attachDeviceLists.sh` (shared body sourced by both)
- Monthly runbook: `~/src/dev_documents/billing_api/monthly-billing-runbook.md`
- Handler: `runAddAttachmentHandler` in `src/addAttachmentHandler.ts`; action wired in `src/main.ts` (`addAttachments`).
- Attach event shape: `AddAttachments` in `src/types.ts`.

## Next steps / open questions
- The attach scripts require AWS credentials (Secrets Manager for Zoho + S3 read) — run with AWS SSO configured. In a shell without creds the handler fails at "Region is missing" (expected).
- Consider adding the two scripts to the billing_api README (under the existing bootstrap section near `src/bootstrap/addAttachment*.ts`).
- `runCAMonthly.sh` uses `node dist/src/main.js` so the data-lake project must be built first; `runUSMonthly.sh` uses `bun src/main.ts` directly. Documented in the runbook.
