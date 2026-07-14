---
type: ProjectStatus
related_to: "[[geotab_shipping_parsing]]"
status: Active
---

# geotab-shipping-parsing — Handoff

## Last updated
2026-07-14 13:50 PDT

## Project location
/Users/mattperzel/src/geotab_shipping_parsing  (AWS region us-east-2, account 628338935433)

## Current session — Purchase Order audit (2026-07-14)
Task: reconcile 24 Simplot POs flagged "Shipped" but SO "Created" in
`~/Downloads/Purchase_Order_Audit - 2026-07-14.xlsx`.

Pipeline: SES email → S3 `advone-geotab-shipping` → parser Lambda →
`advone-geotab-shipping-parsed` → api-writer Lambda → Zoho Inventory
(PO Receive + SO Package + Shipment Order, orgId 702920318 / ADVA02).

### What was done
- Created tmp/audit-2026-07-14, synced both S3 buckets to tmp (input 1710 files,
  parsed 1162; 402 parsed within last month).
- Confirmed all 24 POs present in BOTH buckets (raw + parsed shipment JSON,
  created 2026-07-13 ~23:34–23:37 UTC).
  NOTE: audit uses `PO-7034-07-Jul-26` (mixed case) vs parsed `PO-7034-07-JUL-26`
  (uppercase) — casing only.
- Pulled `/aws/lambda/geotab-shipping-api-writer` logs (UTC window, not local!
  mktime bug — use calendar.timegm for UTC epochs).
- Root cause: on 2026-07-13 ~23:34 UTC a burst of 40 invocations all refreshed the
  Zoho OAuth token concurrently → Zoho returned 400 "too many requests continuously /
  Access Denied". 8 invocations succeeded (early), 30 failed incl. all 24 audit POs.
  No retries on July 14. CloudWatch Errors metric = 0 (app failure, returns 200).

### Key finding / current state
- Stages 1–2 (email→parse) OK for all 24. Stage 3 (api-writer→Zoho) failed for all 24
  due to Zoho OAuth refresh rate-limit under concurrent burst. SO Package/Shipment
  never created in Zoho → audit SO "Created" not "Shipped".
- Full report: tmp/audit-2026-07-14/SUMMARY.md
- Raw artifacts: tmp/audit-2026-07-14/{input,parsed,api-writer-logs.txt,api-writer-logs-jul14.txt}

## Reminders / gotchas
- AWS CLI auto-paginates `list-objects-v2`; `--query 'Contents | length(@)'` is wrong.
- Log time filters: use UTC epoch ms (calendar.timegm), NOT time.mktime (local/PDT).
- PO-number casing differs between audit export and parsed JSON.
- api-writer failures return statusCode 200 → Errors metric stays 0; rely on log content.

## Next up
- Re-drive the 24 failed parsed files through api-writer now that rate-limit cleared.
- Fix: shared/cached Zoho token + retry/backoff on OAuth "too many requests" + DLQ/errors
  so EventBridge retries app-level failures.
