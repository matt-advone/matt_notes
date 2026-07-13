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
