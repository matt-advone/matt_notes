---
type: ProjectStatus
related_to: "[[tolaria]]"
status: Active
---

# geotab_shipping_parsing — Status

## Last updated
2026-07-13 16:10 PT

## Current session summary
Analyzed 15 SES-stored emails from s3://advone-geotab-shipping to diagnose a "PO number not found" error. Found and fixed the root cause (regex/label mismatch on Geotab "delivered" notification emails) plus the duplicate-delivery concern (15 copies of one email to 15 recipients). ALL fixes applied, tested, and the bundle builds.

## Done this session (all verified)
- Config v1.2.0 → v1.3.0 in `packages/parser/config/parser-config.json`:
  - Added `.` to all PO regex char classes (`[A-Z0-9-]` → `[A-Z0-9.-]`) so dotted POs like `SANM03-05202026.2-KEY` extract (was truncating/missing at the dot)
  - Added bare `PO number` labelValue strategy (priority 8) for delivered-email template which uses `PO number` not `Your PO number`
  - Added `Delivery address` label alias to `shippingInformation` strategies (delivered emails label the address block `Delivery address`)
- Aligned inert legacy root `config/parser-config.json` to v1.3.0 (single source of truth; root `src/` is NOT built/deployed — build entry is `packages/parser/src/handler.ts`)
- New cross-invocation dedup: `packages/parser/src/modules/dedupChecker.ts` — S3 conditional-put marker (`dedup/<messageId>.marker`, `IfNoneMatch: '*'`) keyed by SES Message-Id. First invocation wins; the 14 duplicate recipients get 412 → skipped. Marker released on failure so retries can reprocess. Wired into `handler.ts` via `retrieveEmailWithHeaders()` (returns `{html, messageId}`) — `retrieveEmail()` kept as string wrapper for backward compat.
  - dedupChecker duck-types the 412 (`name==='PreconditionFailed' || $metadata.httpStatusCode===412`) instead of `instanceof S3ServiceException` for cross-realm robustness.
- Regression test added in `deliveryEmail.test.ts` (Test 6) using real delivered fixture `tests/fixtures/delivered-email-dot-po.html` → asserts `poNumber === 'SANM03-05202026.2-KEY'`, shippingInformation, items, no warnings.
- Unit tests for dedupChecker (`tests/unit/dedupChecker.test.ts`).
- Updated two configVersion assertions (1.2.0 → 1.3.0) in emailParser.test.ts / sesGeotabEmail.test.ts.
- Verified: `tsc --noEmit` clean; `jest` = 113 passed / 2 skipped / 0 failed; `pnpm build` bundles; end-to-end `run-parser.ts` on the S3 email now returns poNumber + 0 warnings/0 errors + shippingInformation extracted.

## Pre-existing (NOT touched, NOT caused by this work)
- `packages/api-writer/src/handler.ts` lines 247 & 390 fail `tsc --noEmit` (`string | undefined` not assignable to `string`, the `organizationId` assignment). Confirmed failing on clean tree via `git stash -u`. Separate issue — fix in a follow-up if desired.

## Next steps / open questions
- Deploy parser (config v1.3.0 + dedup). After deploy, the `dedup/*.marker` objects accumulate in the OUTPUT bucket (`advone-geotab-shipping-parsed`). Consider adding an S3 lifecycle rule to expire `dedup/` prefix after ~30 days (buckets are externally managed, so set the lifecycle rule out-of-band or via terraform if it manages them).
- Re-process the 15 emails already in S3 for this order once deployed (or rely on the dedup marker to handle future fan-out). The api-writer still needs a valid PO to mark the shipment delivered in Zoho.
- Optional: fix the pre-existing api-writer typecheck errors (lines 247/390) separately.
- Optional follow-up: remove the inert legacy root `src/` + `config/` if confirmed unused (currently only aligned, not deleted).

