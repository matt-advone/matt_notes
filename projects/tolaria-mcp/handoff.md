---
type: ProjectStatus
status: Active
---

# Tolaria MCP — GitHub vault connector for Claude

Remote MCP server giving Claude org-wide, per-user-authorized read access to
Tolaria vaults (markdown repos on GitHub), with an S3 snapshot cache rebuilt
every 6 hours. Lives at `/Users/mattperzel/src/tolaria-mcp` (local git repo,
no remote yet).

## Session 2026-07-22 (initial build)

### What was built

- **Go 1.26 Lambda pair** (`provided.al2023`, arm64):
  - `cmd/server` — MCP Streamable HTTP endpoint (`/mcp`) behind a Lambda
    Function URL, plus a full OAuth 2.1 authorization server (RFC 9728/8414
    metadata, RFC 7591 dynamic client registration, PKCE) that proxies user
    sign-in to a GitHub OAuth app. Tools: `list_vaults`, `list_notes`,
    `read_note`, `search_notes`.
  - `cmd/sync` — every 6 h (EventBridge Scheduler): scans `github_owners`
    (default `AdvantageOne`), detects vaults (topic `tolaria-vault` OR root
    `AGENTS.md` containing "tolaria"), downloads tarballs, writes immutable
    SHA-addressed gzip packs + manifest to S3, flips `snapshot/current.json`
    pointer atomically, GCs old snapshots/orphan packs.
- **Cache design**: server holds packs in container memory with pre-lowered
  search text; revalidates the pointer via conditional GET ≤ every 30 s.
  GitHub is never on the request path.
- **Per-user authorization**: user's GitHub token (from the OAuth flow,
  AES-256-GCM encrypted in DynamoDB) checks `GET /repos/{o}/{r}` per private
  repo; verdict cached 10 min in DynamoDB + 60 s in-process. Public repos
  allowed for any connected user. "Not found" and "no access" are the same
  error string (no existence leak).
- **OpenTofu IaC** (`infra/`): cache bucket `tolaria-mcp-cache-628338935433`,
  DynamoDB `tolaria-mcp-oauth` (pk-only, TTL), Secrets Manager
  `tolaria-mcp/config`, IAM roles, both lambdas, Function URL, scheduler.
  Provider pins `allowed_account_ids = ["628338935433"]`, region us-east-2.
  Backend: `s3://advone-east-2-deploy`, key `tolaria-mcp/terraform.tfstate`,
  `use_lockfile = true`.
- **Scripts** (`scripts/`): `deploy.sh` (SSO identity check + explicit
  account confirmation, state-bucket bootstrap, cross-compile, init/apply,
  prints next steps; `TOLARIA_MCP_YES=1` for non-interactive),
  `configure-github.sh` (stores OAuth app creds + sync PAT + generated
  encryption key in the secret), `sync-now.sh`, `destroy.sh`.

### Key decisions

- **Go over Rust**: Rust wasn't installed and needs cargo-lambda+zig per
  deployer; Go cross-compiles with zero extra toolchain and cold starts are
  within ~30 ms of Rust. Chosen for the "nobody needs to know anything to
  deploy" requirement.
- **OAuth proxy pattern**: GitHub has no dynamic client registration, so the
  MCP server is itself the AS Claude talks to; GitHub OAuth app's callback
  URL is the server's `/callback`. Claude's tokens are opaque server-minted
  tokens; GitHub tokens never leave the server.
- Bearer tokens stored as SHA-256 hashes; refresh tokens single-use.

### State

- `go build`, `go vet`, unit tests (cachefmt/crypto/PKCE), `tofu validate`,
  and linux/arm64 cross-compile all pass. **Not yet deployed** — nothing has
  touched AWS.

### Next steps

1. `scripts/deploy.sh` (needs `aws sso login` to 628338935433).
2. Create the GitHub OAuth app (org-owned, callback = printed URL), then
   `scripts/configure-github.sh` (also needs a sync PAT with org repo read).
3. `scripts/sync-now.sh`, verify vault list in the output.
4. Claude admin: Settings → Connectors → Add custom connector with the
   printed `/mcp` URL.
5. Consider: push repo to GitHub (AdvantageOne org), custom domain for the
   Function URL if the `.on.aws` URL bothers anyone, tantivy-style index if
   corpus outgrows substring search.

## Session 2026-07-22 (later): sync found 0 vaults — diagnosed

Stack is deployed (Function URL
`https://jjoo2povxza4dv3jh2cjt2ylea0nvfua.lambda-url.us-east-2.on.aws`).
First sync returned 0 vaults. Root cause: the sync PAT is a fine-grained PAT
with **resource owner = matt-advone**, so it sees only AdvantageOne's public
repos (dev_documents is invisible), and matt-advone wasn't in
`github_owners`.

Fixed and redeployed: `github_owners` now `["AdvantageOne", "matt-advone"]`;
sync lambda now refuses to publish an empty snapshot over a non-empty one
(PAT-went-blind guard). Re-sync now caches `matt-advone/matt_notes`.

**Blocked on Matt**: replace the sync PAT with one whose resource owner is
the AdvantageOne org (fine-grained, All repositories, read-only
Contents+Metadata), store via `scripts/configure-github.sh`, rerun
`scripts/sync-now.sh` — should then find all 3 vaults.

### Open questions

- If the org enables OAuth App access restrictions, an owner must approve
  the OAuth app once.
- `matt_notes` (matt-advone/matt_notes) is only cached if its owner is added
  to `github_owners` in `infra/variables.tf`.
