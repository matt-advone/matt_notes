---
type: ProjectStatus
related_to: "[[tolaria]]"
status: Active
_organized: true
---

# OMSInt — Notes

## Last updated
2026-07-13 16:32 PDT

## Current session
Built a Tolaria vault operations guide for the OMSInt project under
`~/src/dev_documents/`. Created:
- `OMSInt Overview.md` — project explainer + index of all 8 ECS deployments.
- `OMSInt Downtime Playbook.md` — step-by-step incident runbook (restart → logs →
  IPsec → config → deploy).
- `OMSInt Deployments/*.md` — 8 per-customer pages with exact AWS console links,
  resource IDs, env vars, and copy-paste restart commands.

Two deployment generations discovered and documented:
- **infra-v2** (OpenTofu, ECS Managed Instances, ARM64, ASG, IPsec/SSM/Exec):
  cobbbeta + bputest in us-east-1 (both DRY_RUN=true).
- **legacy cdktf** (standalone EC2 container instances, x86, `:latest`, no
  SSM/Exec, plaintext secrets in task def): ilec (ue1), cobb/macon/dec/dawson/aandn (ue2).

All resource IDs, log groups, secrets, ECR URIs, EC2 instance IDs/IPs, ASGs,
EIPs, and VPC IDs captured live via `aws` CLI on 2026-07-13.

## Reminders / gotchas
- AWS SSO profile: `aws-oms-1-awsadministratoraccess` (account 793851606195).
  SSO token expired mid-session (~16:00) — had to skip fetching ilec's EC2
  instance type/IP; used the console link instead. Re-auth with `aws sso login`.
- Legacy deployments have **no SSM and no ECS Exec** — can't get a shell; debug
  via CloudWatch logs only. Documented this prominently.
- Legacy task defs store Geotab/MultiSpeak passwords in **plaintext** env vars
  (security note added to each legacy page).
- infra-v2 deploys use ARM64 (`linux/arm64`), legacy uses x86 `:latest`. The
  `full-deploy.sh` only allows 3 ECR repos by region (omsintecr ue1, oms ue2,
  advone-ca-central-1 ca-central-1).
- `cobbbeta` is the only deployment with IPsec enabled (gateway 74.231.9.5,
  remote subnet 192.168.77.0/24). PSK in Zoho Vault → "cobb network info".
- Worker auto-writes IPsec diagnostic Markdown to
  `s3://advone-<region>-logging/oms/<slug>/ipsectunnel/` when tunnel is down.
- Terraform state: `s3://omsdep1/customers/<slug>/<region>/terraform.tfstate`.

## Next up
- Re-auth AWS SSO and backfill ilec EC2 instance type/public IP in its deployment
  page (currently just a console link).
- Consider adding a CloudWatch alarm / dashboard section to the playbook.
- The legacy deployments are candidates for migration to infra-v2 (SSM/Exec +
  Secrets Manager would make ops much easier).

---

## Earlier session — 2026-07-13 12:15 PT

### Summary
Fixed the Geotab SDK `LocalStorageCredentialStore.clear()` crash in the worker by injecting an in-memory credential store, and added proactive 10-day session re-authentication to the main loop.

### Recently completed
- Added `lambda/packages/worker/src/inMemoryCredentialStore.ts` — a no-op credential store (`get`→false, `set`/`clear` no-ops) passed via `newCredentialStore` so the SDK never touches `localStorage`.
- Wired the store into the `CachingApi` construction in `worker/src/index.ts` (`rememberMe: false` + `newCredentialStore`). Cast to `any` because adv-mg-api-js's `GeotabApiOptions` type declares `newCredentialsStore?: boolean` but the runtime reads `newCredentialStore` (object) — type def is wrong.
- Added `REAUTH_INTERVAL_MS` (10 days) constant and a re-auth check at the top of the main loop: when elapsed, calls `await api.authenticate()` (published `Api.authenticate()` takes no args, re-auths with the original userName/password). Resets `lastAuthTime` only on success; failures fall through to the iteration catch + exponential backoff.
- Rebuilt `deploy/bundle.js` (build passes: `tsc --noEmit` + esbuild).

### Open questions
- The worker bundle is rebuilt locally; needs deploy to Lambda to take effect.
- `multiSpeakOutputter.test.ts` has 14 failing tests — pre-existing, caused by uncommitted edits to `multispeakOutput.ts`/`sendMultiSpeak.ts`/`config.ts` already in the working tree (NOT caused by this session's changes; confirmed by reverting `index.ts` to HEAD and re-running). These pre-existing mods may need their own follow-up.
- Consider patching `adv-mg-api-js` upstream so `clearLocalCredentials()`/store creation is gated behind `rememberMe` (currently `clear()` is called unconditionally on `InvalidUserException`).
- The `InvalidUserException` trigger itself (server-side session expiry mid-run) is what re-auth now mitigates proactively; may still want reactive re-auth on that error path.
