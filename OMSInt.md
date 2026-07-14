---
type: ProjectStatus
related_to: "[[tolaria]]"
status: Active
---

# OMSInt — Status

## Last updated
2026-07-13 12:15 PT

## Current session summary
Fixed the Geotab SDK `LocalStorageCredentialStore.clear()` crash in the worker by injecting an in-memory credential store, and added proactive 10-day session re-authentication to the main loop.

## In progress
- (none)

## Recently completed
- Added `lambda/packages/worker/src/inMemoryCredentialStore.ts` — a no-op credential store (`get`→false, `set`/`clear` no-ops) passed via `newCredentialStore` so the SDK never touches `localStorage`.
- Wired the store into the `CachingApi` construction in `worker/src/index.ts` (`rememberMe: false` + `newCredentialStore`). Cast to `any` because adv-mg-api-js's `GeotabApiOptions` type declares `newCredentialsStore?: boolean` but the runtime reads `newCredentialStore` (object) — type def is wrong.
- Added `REAUTH_INTERVAL_MS` (10 days) constant and a re-auth check at the top of the main loop: when elapsed, calls `await api.authenticate()` (published `Api.authenticate()` takes no args, re-auths with the original userName/password). Resets `lastAuthTime` only on success; failures fall through to the iteration catch + exponential backoff.
- Rebuilt `deploy/bundle.js` (build passes: `tsc --noEmit` + esbuild).

## Next steps / open questions
- The worker bundle is rebuilt locally; needs deploy to Lambda to take effect.
- `multiSpeakOutputter.test.ts` has 14 failing tests — pre-existing, caused by uncommitted edits to `multispeakOutput.ts`/`sendMultiSpeak.ts`/`config.ts` already in the working tree (NOT caused by this session's changes; confirmed by reverting `index.ts` to HEAD and re-running). These pre-existing mods may need their own follow-up.
- Consider patching `adv-mg-api-js` upstream so `clearLocalCredentials()`/store creation is gated behind `rememberMe` (currently `clear()` is called unconditionally on `InvalidUserException`).
- The `InvalidUserException` trigger itself (server-side session expiry mid-run) is what re-auth now mitigates proactively; may still want reactive re-auth on that error path.
