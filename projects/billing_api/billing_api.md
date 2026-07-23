---
type: ProjectStatus
related_to: "[[tolaria]]"
status: Active
_organized: true
---

# billing_api — Status

## Last updated
2026-07-23 11:45 (Pacific)

## Current session summary
Built `tools/zoho-token-rotator/` — an AWS Secrets Manager rotation Lambda that refreshes Zoho OAuth self-client tokens for the two production secrets (`billings-zoho-VJBcmz`, `billing-crm-zoho-Kk1OtO`). Includes OpenTofu (Lambda + scoped IAM role + log group + per-secret invoke perms) and a zero-input `deploy.sh` guarded to account 628338935433 / us-east-2 with S3 backend `s3://advone-east-2-deploy/billing_api_rotator/`.

## In progress
- (nothing)

## Recently completed
- Added `tools/zoho-token-rotator/src/index.ts` — rotation Lambda implementing all four Secrets Manager steps; refreshes via Zoho `grant_type=refresh_token`, preserves the long-lived refresh_token, recomputes `expiry_date`. Reads the secret to rotate from event `SecretId`, so the same Lambda handles both secrets.
- `build.mjs` bundles with esbuild to `dist/index.mjs` (AWS SDK externalised — nodejs22.x runtime provides it; avoids the ESM "dynamic require of node:https" issue). Bundle is ~5.6 KB.
- `deploy/` OpenTofu: `main.tf`, `variables.tf`, `outputs.tf`, `guards.tf` (data sources forcing provider to contact account/region). Provider uses `allowed_account_ids`.
- `deploy.sh`: verifies STS account == 628338935433 and bucket region == us-east-2 (aborts otherwise), installs deps, typechecks, bundles, zips, runs `tofu init/validate/fmt/plan/apply` with `-backend-config` to the S3 backend. Modes: `./deploy.sh` (apply) or `./deploy.sh plan`.
- Verified: typecheck clean, esbuild bundle OK, `tofu plan` shows 7 to add, account/region guard aborts on mismatch, and a **read-only** live Zoho refresh test confirmed a new access_token + 1h expiry with refresh_token preserved (nothing written back to the secret).

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
- **Zoho token rotator**: `~/src/billing_api/tools/zoho-token-rotator/` — `deploy.sh` to deploy; `deploy/main.tf` for infra; rotation Lambda ARN emitted as tofu output `lambda_function_arn`.

## Next steps / open questions
- The attach scripts require AWS credentials (Secrets Manager for Zoho + S3 read) — run with AWS SSO configured. In a shell without creds the handler fails at "Region is missing" (expected).
- Consider adding the two scripts to the billing_api README (under the existing bootstrap section near `src/bootstrap/addAttachment*.ts`).
- `runCAMonthly.sh` uses `node dist/src/main.js` so the data-lake project must be built first; `runUSMonthly.sh` uses `bun src/main.ts` directly. Documented in the runbook.
- **Rotator not yet applied** — `./deploy.sh` was only run in `plan` mode (7 to add). Run `./deploy.sh` to create the Lambda, then manually attach it as the rotation function on each of the two Zoho secrets in the console (IAM + invoke perms are pre-wired). Suggest rotation schedule `rate(1 hour)` since Zoho access tokens expire hourly.

## Local review fixes applied (2026-07-23)
After a `/local-review-uncommitted` pass, fixed all 9 findings on the rotator:
- CRITICAL: repo-root `.gitignore` rule `deploy/` was silently excluding `tools/zoho-token-rotator/deploy/*.tf` — anchored it to `/deploy/` so nested dirs track (verified with `git check-ignore`).
- IAM: removed unused account-wide `secretsmanager:ListSecrets`; scoped `kms:Decrypt` with a `kms:ViaService = secretsmanager.us-east-2.amazonaws.com` condition.
- `deploy.sh`: bucket-location lookup now `|| abort` (was silently aborting under `set -e`); `npm install` → `npm ci`; `tofu fmt -check` made non-blocking; added DynamoDB state-lock table `billing-api-rotator-tofu-locks` (auto-created, PAY_PER_REQUEST) + `-backend-config="dynamodb_table=..."` + `-reconfigure` on init.
- Removed dead `state_bucket`/`state_key` terraform vars (replaced with a real `state_lock_table` var) and the unused `__testables` export.
- Verified: typecheck clean, build clean, `tofu validate` ok, `tofu plan` = 7 to add, lock table ACTIVE, handler still exports correctly.
