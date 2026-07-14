---
type: ProjectStatus
related_to: "[[tolaria]]"
status: Active
---

# geotab_shipping_parsing — Notes

## Last updated
2026-07-14

## Current session
Built out the Tolaria vault (`~/src/dev_documents`) documentation for the
geotab_shipping_parsing project. Created a new `projects/geotab_shipping_parsing/`
directory with an overview README, a runbook with exact AWS resource IDs, and a
per-package doc for all 5 monorepo packages.

Resources were enumerated live from AWS (account 628338935433, us-east-2, profile
`internal-prod-awsadministratoraccess`) so the runbook has exact ARNs for both
Lambdas, the S3 buckets, EventBridge rules, IAM roles, Secrets Manager secrets,
SES receipt rule, CloudWatch log groups + alarms, and the Terraform state backend.

## Reminders / gotchas
- The `api-writer` code is **not** shipped by Terraform (bundle ~200 MB due to
  @sparticuz/chromium). Terraform uses a placeholder zip; real code goes via
  `pnpm run deploy:api-writer` → `s3://advone-east-2-deploy/api-writer/lambda.zip`
  → `aws lambda update-function-code`.
- The `parser` code IS shipped by Terraform (`data.archive_file` zips
  `packages/parser/dist` on `apply`).
- There's a third adjacent Lambda `geotab-shipping-notification-lambda` that also
  triggers off `advone-geotab-shipping-parsed` (separate EventBridge rule). It is
  NOT in this monorepo (deployed from S3, handler `bundle.handler`). Documented in
  the runbook for awareness only.
- Zoho org routing is hard-coded in `packages/api-writer/src/zoho-client.ts`:
  ADVA02 → 702920318, ADVA01 → 851438275, unmapped → ADVA02.
- Tolaria wikilinks use the npm package names as note titles
  (`[[@geotab-shipping-types]]` etc.) to avoid collisions in the large vault.

## Files created
- `dev_documents/projects/geotab_shipping_parsing/README.md` (overview)
- `dev_documents/projects/geotab_shipping_parsing/Geotab Shipping Parser Runbook.md`
- `dev_documents/projects/geotab_shipping_parsing/packages/{types,parser,api-writer,email-viewer,setup-secrets}/@geotab-shipping-*.md`

## Next up
- (Optional) Add a deployment page per environment if a dev/staging env is created.
- (Optional) Wire CloudWatch alarms to an SNS topic for email/pager alerts
  (currently alarms exist but have no SNS action).
- Update `dev_documents/aws/lambdas.md` if any of these functions' metadata changes.
