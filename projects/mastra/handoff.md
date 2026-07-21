---
type: ProjectStatus
related_to: "[[mastra]]"
status: Active
---

# Mastra — Notes

## Last updated
2026-07-20 (session 2)

## Current session
Created a standalone Zoho Desk diagnostic-ticket report script at
`scripts/desk/diagnostic-ticket-report.sh` (+ `scripts/desk/reports/` output dir).
No Mastra/agent changes — pure ops/analysis script. Reworked it in session 2 to
use a self-hosted gitignored token file instead of re-prompting each run.

## Reminders / gotchas
- The script reads required scopes from `shared/zoho/required-scopes.json` (desk entry).
- **Self-hosted OAuth token file** (session 2 change): on first run it asks for
  region, client id, client secret, AND a one-time authorization code (from the
  Zoho API Console Self Client). It exchanges the code and saves the FULL token
  JSON response to `.tokens/desk-diagnostic-report.json` (gitignored, chmod 600).
  The file also stores client_id/client_secret/region/obtained_at so it can
  self-refresh later. On every subsequent run it loads the file silently and
  refreshes the access token if expired — no prompts, no AWS Secrets Manager, no
  instance .env writes. Pass `--reset-token` to re-authenticate.
- `.tokens/` added to `.gitignore`.
- Does NOT touch `scripts/set-zoho-token.sh`, AWS secrets, or `instances/<i>/.env`.
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
- 401 mid-run → refresh token once, persist, retry.

## Next up
- Run it live against Zoho Desk to validate the serial regex false-positive rate
  against real ticket text; tighten patterns if needed.
- Optionally add a `--department`/departmentId filter and an orgId override flag.

## Session 3 (2026-07-20) — code-review fixes
Local review flagged 3 CRITICAL + 5 WARNING issues; all fixed:
- **Pagination** (CRITICAL): was breaking after page 1 because it relied on a
  `more` field the Desk API doesn't reliably return (confirmed via
  desk-client.ts which uses length-based detection). Now requests
  `sortBy=createdTime&sortOrder=desc` and terminates on `len(batch) < limit`
  or empty page, with a window-bound early-stop that only fires AFTER collecting
  an in-window ticket (guards against non-descending sort dropping tickets).
- **XSS** (CRITICAL): `json.dumps(rows)` was embedded in a `<script>` block
  unescaped → `</script>` in a ticket subject = stored XSS. Now HTML-safe via
  `safe_json()` (escapes `<`,`>`,`&`,U+2028/2029 to JS unicode escapes).
- **PII gitignore** (CRITICAL): added `scripts/desk/reports/` to `.gitignore`
  (reports contain ticket subjects/serials).
- **save_token perms** (WARNING): now creates the file atomically with
  `os.open(..., 0o600)` instead of write-then-chmod (no world-readable window,
  no swallowed OSError).
- **Serial regex** (WARNING): labelled "serial:/SN:" patterns now only match
  when the ticket mentions "geotab"; bare G/GO/GA-prefix formats always match.
- **Diagnostic regex** (WARNING): narrowed to diagnostic-REQUEST intent
  (run/pull/send logs, run diagnostic, troubleshoot, fault code, DTC) — dropped
  pure symptom words (offline, not reporting, no data) that over-counted.
- **api_get/api_get_raw duplication + 429** (WARNING): unified into one core
  `_request(path, params, header_builder)` with 401-refresh + 429/Retry-After
  backoff retry. Thread fetch now pre-filtered (skip when subject+description
  already yield both serial+diag), with a small inter-request delay and a
  thread-error count in the summary.
- **Dead code** (SUGGESTION): removed unused Python `esc()` + `html` import.

Verified: bash -n OK, python -W error compile OK, logic tests (serial gating,
diag narrowing, XSS-safe JSON, pagination guard) all pass, save_token perms
0o600, git check-ignore confirms .tokens/ + reports/ ignored while the script
itself stays trackable.

---

## Earlier session — 2026-07-13 (and prior)

### Project
- **Path:** `/Users/mattperzel/src/mastra`
- **Main instance:** `instances/main-mastra/`
- **Goal:** Centralize Zoho integration in a shared `src/lib/zoho` library, replace MCP-based Zoho Desk with native Mastra tools, and establish automatic token refresh / health checks.

### What was done this session
- Deduplicated repeated `switch` cases in `instances/main-mastra/src/lib/zoho/src/services.ts` that were accidentally duplicated and preventing the build from completing.
- Created `instances/main-mastra/src/mastra/workflows/zoho-token-health-workflow.ts`, which wraps `check-zoho-tokens-tool`.
- Registered the new workflow in `instances/main-mastra/src/mastra/index.ts`.
- Added a declarative schedule to the workflow: daily at `08:00` in `America/Los_Angeles`.
- Enabled the Mastra scheduler in `src/mastra/index.ts` with `scheduler: { enabled: true, tickIntervalMs: 60_000 }`.
- Updated `instances/main-mastra/AGENTS.md` with token-health workflow details.
- Ran `pnpm install --frozen-lockfile` and `pnpm run build`; both completed successfully.

### Current state
- `pnpm run build` succeeds for `instances/main-mastra`.
- `services.ts` compiles and is not duplicated.
- `zoho-token-health-workflow` is registered and scheduled, but not yet executed end-to-end.

### What is not verified
- The scheduled workflow has not been run manually or observed to trigger automatically.
- No integration test covers the `check-zoho-tokens-tool` behavior with real Zoho tokens.
- DynamoDB has no persistent observability domain; token-health results are currently only logs.

### Next steps / open questions (from that session)
1. Trigger a manual run of `zoho-token-health-workflow` to confirm the tool pings configured services correctly.
2. Decide if alerts or metrics should be produced when a service is unhealthy.
3. Update the shared `shared/zoho/required-scopes.md` if any new scopes were introduced by the tool paths.
4. Confirm the `ZOHO_REGION` and other per-service env vars are present in the instance `.env` and Terraform Secrets Manager before deploying.

### Session 2026-07-13 — Operations documentation

#### What was done
- Created `docs/` in the mastra repo with three operations documents:
  - `docs/deployment.md` — full deployment guide for root + instance stacks, including **exact AWS resource IDs** read from OpenTofu state (`s3://advone-east-2-deploy/mastra/infra/root/terraform.tfstate` and `.../instance/main-mastra/terraform.tfstate`), plus a troubleshooting map for DevOps on-call.
  - `docs/studio-access.md` — how to reach Mastra Studio for each instance via `open-studio.sh` (Tailscale SOCKS5 proxy), with prerequisites, per-instance table, and troubleshooting.
  - `docs/scripts.md` — reference for every script in `scripts/` (deploy.sh, open-studio.sh, set-instance-secrets.sh, set-zoho-token.sh, set-tailscale-key.sh, new-instance.sh) plus the `scripts/lib/` helpers, with a quick decision table.
- Copied all three docs to the mastra notes subfolder so the notes vault contains knowledge of this solution.

#### Verified
- State pulled successfully from S3 for both root and `main-mastra` instance; resource IDs cross-checked against OpenTofu state.
- Account `628338935433`, region `us-east-2`, VPC `10.30.0.0/16`, cluster `advone-mastra-cluster`.

#### Next steps / open questions
- The `finance` instance exists in `instanceConfig.yml` with `deploy: false`; no instance state file exists yet. Document its exact resources only after it is deployed.
- Consider adding a `docs/` entry to the repo README / AGENTS.md so it's discoverable.
- Resource IDs are point-in-time snapshots from state; regenerate these docs after major infra changes (the troubleshooting one-liners in `deployment.md` show how to re-pull state).
