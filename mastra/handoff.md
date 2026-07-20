---
type: ProjectStatus
related_to: "[[mastra]]"
status: Active
---

# Mastra — Notes

## Last updated
2026-07-20

## Current session
Created a standalone Zoho Desk diagnostic-ticket report script at
`scripts/desk/diagnostic-ticket-report.sh` (+ `scripts/desk/reports/` output dir).
No Mastra/agent changes — pure ops/analysis script.

## Reminders / gotchas
- The script reads required scopes from `shared/zoho/required-scopes.json` (desk entry).
- It prompts for region, client id, client secret, and refresh token (or one-time
  auth code). It refreshes the access token itself (mirrors set-zoho-token.sh +
  oauth-client.ts refresh flow) and auto-retries once on 401.
- Ticket fetching paginates `GET /api/v1/tickets?from=&limit=100` using the `more`
  flag, plus a descending-sort heuristic to stop once a whole page is older than
  the window, plus a 200-page safety cap. Sort order from Desk is NOT guaranteed,
  so it keeps in-window tickets across pages regardless.
- For each ticket it fetches threads (`/tickets/{id}/threads`, capped 100) and
  scans subject + description + thread content for:
  - Geotab serial: labelled "serial no/number/num/#/SN:" patterns + bare GO/GA/G
    prefixes, 6-15 alphanumerics.
  - Diagnostic: regex of diagnostic/diag/fault code/DTC/log/snapshot/
    troubleshooting/not reporting/offline keywords.
- Output: self-contained HTML in `scripts/desk/reports/desk-diagnostic-report-<ts>.html`
  with summary stats, filter box, sortable table, and `open`/`xdg-open` auto-open.

## Next up
- Run it live against Zoho Desk to validate the serial regex false-positive rate
  against real ticket text; tighten patterns if needed.
- Optionally add a `--department`/departmentId filter and an orgId override flag.
