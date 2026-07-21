---
type: ProjectStatus
related_to: "[[tolaria]]"
status: Active
---

# wheels-transactions — Notes

## Last updated
2026-07-21 16:02 (PT)

## Where it lives
- Repo: `/Users/mattperzel/src/wheels_transactions`
- Formal team doc: `~/src/dev_documents/projects/wheels-transactions/README.md`

## Current session
Built the whole thing from scratch from `example.csv` + the MyGeotab FuelTransaction
API reference (developers.geotab.com). Lambda ingests SES→S3 emails, extracts the CSV
attachment, maps rows to `FuelTransaction`, and calls MyGeotab `Add`.

## What was done this session
- Fetched the Geotab `FuelTransaction` object ref + `FuelTransactionProductType` /
  `FuelTransactionProvider` enums. No "Wheels" provider exists → used `Unknown`.
- Confirmed 3 assumptions with Matt: CSV is an **attachment**; odometer is **miles**;
  transaction time is **US Eastern** (DST-aware → UTC).
- TypeScript Lambda: `src/` split into config, logger, s3, emailParser (mailparser),
  csvParser (papaparse), mapper, geotab client/types, credentials (SSM), index handler.
- Interactive CLI `cli/setup-credentials.ts` → SSM (`/…/server|database|username`,
  `password` as SecureString). Optionally tests Geotab auth before saving.
- OpenTofu `infra/`: S3 backend on `advone-east-2-deploy` /
  `simplot-wheels-fuel-transactions/terraform.tfstate`; `wheels-import` bucket; SES
  domain identity + DKIM + receipt rule set writing to `simplot/` prefix; Lambda
  (nodejs20.x) with IAM (S3 GetObject on prefix, ssm:GetParameters + kms:Decrypt on
  alias/aws/ssm); S3 bucket notification → Lambda. `tofu validate` passes.
- Tests (vitest): 22 passing over csvParser + mapper (incl. miles→km, gal→L, ET→UTC).
- `npm run package` produces `infra/lambda.zip` (index.js at root).
- Wrote repo `README.md` + `AGENTS.md`.

## Reminders / gotchas
- **luxon ships no `.d.ts` in its npm tarball** (3.4.4 and 3.7.2 both) — needed
  `@types/luxon` from DefinitelyTyped. Also `@types/mailparser` for MIME types.
- esbuild `--log-level` accepts `info`/`warning`/etc, **not** `summary`.
- `aws_ses_receipt_rule` action blocks require `position = 1` in AWS provider v5.
- SES **receiving** requires the domain verified + MX → region's inbound SMTP. us-east-2
  receiving support — verify in the target region; if unsupported, change `aws_region`.
- Odometer column header says "Mi/Km" — we assume miles (US fleet). If a Canadian/metric
  batch arrives, set `ODOMETER_UNIT=km`.
- Dedup is NOT implemented — reprocessing the same email creates duplicate
  FuelTransactions. Deterministic `externalReference` (`WHL-<sha1>`) is set so a
  future Get-based skip could be added.

## Verified
- `npm run typecheck` ✅  `npm run lint` ✅  `npm test` ✅ (22/22)
- `npm run build` ✅  `npm run package` ✅  `tofu init -backend=false && tofu validate` ✅

## Next up
- Deploy: `npm run package` then `cd infra && tofu init/plan/apply` (set tfvars).
- Run `npm run setup-credentials` to populate SSM, then add SES DNS records from
  `tofu output` (TXT, 3× DKIM CNAME, MX).
- Send a real Wheels email to the SES recipient and watch CloudWatch logs.
- Consider implementing dedup (Get FuelTransaction by externalReference, skip if exists).
