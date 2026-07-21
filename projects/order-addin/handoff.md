---
type: ProjectStatus
related_to: "[[order-addin]]"
status: Active
---

# order-addin — Notes

## Last updated
2026-07-14 13:50

## Current session
Aligned Zoho Inventory refresh logic in the app with the setup script (`backend/scripts/secrets/setup_zoho_inventory_secret.ts`).

## Context / findings
- CRM side already uses `@advantageone/zoho_lib` `ZSDK` + `SecretManagerStore`. The library's `ZohoAuthClient.getToken()` refreshes against `https://accounts.zoho.com/oauth/v2/token`, persists new token + absolute `expiry_date` back to the secret via `setContext`. CRM setup writes the `ZohoAuthContext` shape `{clientId, clientSecret, tokens:{...}}` — matches. No change needed.
- Inventory side (`backend/src/utils/zoho/inventory.ts`) was a hand-rolled `ZohoInventoryClient` that did NOT match the setup script:
  1. Setup writes `expires_in`, but constructor hardcoded `Date.now() + 3600*1000` → treated a days-old stored access_token as fresh for an hour every cold start.
  2. Setup writes `api_domain`, but client hardcoded `https://www.zohoapis.com/inventory/v1`.
  3. Client refreshed only in-memory; never persisted the refreshed token → secret stayed stale forever; relied on 401-retry.

## What changed (backend/src/utils/zoho/inventory.ts)
- Added `expires_at?: string` (absolute ISO) to `ZohoInventorySecrets`; added `PersistSecretFn` type.
- Constructor: derive `baseUrl` from `secrets.api_domain` (fallback `https://www.zohoapis.com`); set `tokenExpiresAt` from `secrets.expires_at` only; if absent (fresh setup), leave undefined → forces refresh on first use (mirrors zoho_lib when `expiry_date` missing).
- `refreshAccessToken`: uses `data.expires_in` (fallback stored lifetime, default 3600), computes absolute expiry with 300s skew, and persists `{access_token, api_domain, token_type, expires_in, expires_at}` back via the persist callback (non-fatal on failure).
- `getAccessToken`: refreshes when expiry is unknown (`tokenExpiresAt === undefined`), not only when a token exists.
- `getZohoInventoryClient`: wires a `persistSecret` callback that calls `secretManager.setSecret(arn, updated)`.

## Verified
- `npx tsc --noEmit` → no errors in inventory.ts (only pre-existing unrelated `@smithy/node-http-handler` error in src/utils/cache/s3.ts).
- `esbuild` bundle of inventory.ts → BUILD OK.

## Notes / gotchas
- `backend/scripts/secrets/package.json` was created and `backend/scripts/secrets` added to `pnpm-workspace.yaml` so `bun ./setup_zoho_crm_secret.ts` resolves modules from local `node_modules`.
- zoho_lib refresh response sets `expiry_date` as a Dayjs; our `expires_at` is a plain ISO string — different field, only read by our own client.

## Next up
- Deploy + observe logs for `[ZohoInventoryClient] Access token refreshed successfully` and confirm `expires_at` lands in the secret.
- Confirm IAM role for the Lambda has `secretsmanager:PutSecretValue` on the inventory secret ARN (needed for the new persistence path).
