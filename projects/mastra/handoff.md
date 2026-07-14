---
_organized: true
---
# Mastra — Session Handoff

## Project
- **Path:** `/Users/mattperzel/src/mastra`
- **Main instance:** `instances/main-mastra/`
- **Goal:** Centralize Zoho integration in a shared `src/lib/zoho` library, replace MCP-based Zoho Desk with native Mastra tools, and establish automatic token refresh / health checks.

## What was done this session
- Deduplicated repeated `switch` cases in `instances/main-mastra/src/lib/zoho/src/services.ts` that were accidentally duplicated and preventing the build from completing.
- Created `instances/main-mastra/src/mastra/workflows/zoho-token-health-workflow.ts`, which wraps `check-zoho-tokens-tool`.
- Registered the new workflow in `instances/main-mastra/src/mastra/index.ts`.
- Added a declarative schedule to the workflow: daily at `08:00` in `America/Los_Angeles`.
- Enabled the Mastra scheduler in `src/mastra/index.ts` with `scheduler: { enabled: true, tickIntervalMs: 60_000 }`.
- Updated `instances/main-mastra/AGENTS.md` with token-health workflow details.
- Ran `pnpm install --frozen-lockfile` and `pnpm run build`; both completed successfully.

## Current state
- `pnpm run build` succeeds for `instances/main-mastra`.
- `services.ts` compiles and is not duplicated.
- `zoho-token-health-workflow` is registered and scheduled, but not yet executed end-to-end.

## What is not verified
- The scheduled workflow has not been run manually or observed to trigger automatically.
- No integration test covers the `check-zoho-tokens-tool` behavior with real Zoho tokens.
- DynamoDB has no persistent observability domain; token-health results are currently only logs.

## Next steps / open questions
1. Trigger a manual run of `zoho-token-health-workflow` to confirm the tool pings configured services correctly.
2. Decide if alerts or metrics should be produced when a service is unhealthy.
3. Update the shared `shared/zoho/required-scopes.md` if any new scopes were introduced by the tool paths.
4. Confirm the `ZOHO_REGION` and other per-service env vars are present in the instance `.env` and Terraform Secrets Manager before deploying.

---

## Session 2026-07-13 — Operations documentation

### What was done
- Created `docs/` in the mastra repo with three operations documents:
  - `docs/deployment.md` — full deployment guide for root + instance stacks, including **exact AWS resource IDs** read from OpenTofu state (`s3://advone-east-2-deploy/mastra/infra/root/terraform.tfstate` and `.../instance/main-mastra/terraform.tfstate`), plus a troubleshooting map for DevOps on-call.
  - `docs/studio-access.md` — how to reach Mastra Studio for each instance via `open-studio.sh` (Tailscale SOCKS5 proxy), with prerequisites, per-instance table, and troubleshooting.
  - `docs/scripts.md` — reference for every script in `scripts/` (deploy.sh, open-studio.sh, set-instance-secrets.sh, set-zoho-token.sh, set-tailscale-key.sh, new-instance.sh) plus the `scripts/lib/` helpers, with a quick decision table.
- Copied all three docs to `~/src/matt_notes/mastra/docs/` so the notes vault contains knowledge of this solution.

### Verified
- State pulled successfully from S3 for both root and `main-mastra` instance; resource IDs cross-checked against OpenTofu state.
- Account `628338935433`, region `us-east-2`, VPC `10.30.0.0/16`, cluster `advone-mastra-cluster`.

### Next steps / open questions
- The `finance` instance exists in `instanceConfig.yml` with `deploy: false`; no instance state file exists yet. Document its exact resources only after it is deployed.
- Consider adding a `docs/` entry to the repo README / AGENTS.md so it's discoverable.
- Resource IDs are point-in-time snapshots from state; regenerate these docs after major infra changes (the troubleshooting one-liners in `deployment.md` show how to re-pull state).
