# Mastra Scripts Reference

> Every operational task for this repo is wrapped in a shell script under [`scripts/`](../scripts/). **Never run `tofu apply` directly** — use `deploy.sh`. This document lists each script, what it does, when to use it, and the AWS resources it touches.
>
> Shared helper libraries live in [`scripts/lib/`](../scripts/lib/) and are sourced by the scripts below (not meant to be run directly).

---

## Top-level scripts

### `deploy.sh` — Deployment orchestrator

**Path:** [`scripts/deploy.sh`](../scripts/deploy.sh)

**Purpose:** The single entry point for deploying either the **root** stack or an **instance** stack to AWS via OpenTofu. It walks through every prerequisite in order, detects what's already done, pauses for human action when needed, and confirms before applying.

**Usage:**
```bash
./scripts/deploy.sh                                    # interactive: pick a config from S3
./scripts/deploy.sh --config mastra/configs/<name>.yml  # pre-select a config
./scripts/deploy.sh --target root|instance             # override the stack target
./scripts/deploy.sh --skip-build                       # redeploy the existing ECR image
./scripts/deploy.sh --image-tag sha-abc1234            # override the image tag
./scripts/deploy.sh --auto-approve                     # skip the "yes" confirmation
./scripts/deploy.sh --help
```

**What it does (in order):**
1. Verifies required tools (`tofu`, `aws`, `yq`, `jq`, `docker`, `python3`) and AWS credentials.
2. Reads [`instances/instanceConfig.yml`](../instances/instanceConfig.yml) to find instances with `deploy: true`.
3. Selects a deployment config from S3 (`s3://advone-east-2-deploy/mastra/`) — interactive menu or `--config`.
4. Parses the config (instance name, stack target, tfvars, backend settings).
5. For instance deploys: gates on the instance's `deploy: true` flag.
6. One-time prerequisite checks: deploy bucket exists; (instance) root state exists; (root) Tailscale auth-key secret; (instance) instance secret populated.
7. For instance deploys (unless `--skip-build`): builds the Docker image and pushes it to ECR (`advone-mastra/<instance>`).
8. Runs `tofu init` → `tofu plan` → prompts `yes` (unless `--auto-approve`) → `tofu apply`.
9. For instance deploys: polls the ECS service until `RUNNING`/`HEALTHY` (or fails on circuit-breaker).
10. Approves Tailscale subnet routes via the Tailscale API and cleans up stale subnet-router devices; prints next steps and OpenTofu outputs.

**When to use:** Deploy or update infra/instances; redeploy after code changes; infra-only re-apply (`--skip-build`).

---

### `open-studio.sh` — Open Mastra Studio in Chrome

**Path:** [`scripts/open-studio.sh`](../scripts/open-studio.sh)

**Purpose:** Opens the Mastra Studio UI for an instance in a dedicated, SOCKS5-proxied Chrome window, routing through the Tailscale subnet router into the private VPC.

**Usage:**
```bash
./scripts/open-studio.sh                 # defaults to main-mastra
./scripts/open-studio.sh <instance-name>
```

**What it does:**
1. Auto-approves the `10.30.0.0/16` subnet route via the Tailscale API.
2. Finds the active Tailscale subnet-router IP from `tailscale status`.
3. Looks up the current ECS task private IP for `advone-mastra-<instance>`.
4. Tests `socks5://<router-ip>:1055` can reach `http://<task-ip>:4111/health`.
5. Launches Chrome with `--proxy-server=socks5://...` and a separate `--user-data-dir`.

**When to use:** Whenever you need to open Studio for an instance. **Re-run after every deployment** (task IP changes). Full details: [studio-access.md](studio-access.md).

---

### `set-instance-secrets.sh` — Populate an instance's Secrets Manager secret

**Path:** [`scripts/set-instance-secrets.sh`](../scripts/set-instance-secrets.sh)

**Purpose:** Interactively prompts for the non-Zoho secrets an instance needs, **merges** them into the existing `mastra/<instance>` secret (preserving keys managed by other scripts), and optionally restarts the ECS service.

**Usage:**
```bash
./scripts/set-instance-secrets.sh            # defaults to main-mastra
./scripts/set-instance-secrets.sh <instance>
```

**Secrets it manages (merged, never wipes):**
- `OPENROUTER_API_KEY` — required for LLM calls (https://openrouter.ai/keys).
- `MASTRA_EMAIL_FROM` — optional SES sender for the send-email tool.
- `MASTRA_PLATFORM_ACCESS_TOKEN` — optional, enables Mastra Platform observability.

**What it also does:** backfills any missing `ZOHO_*` keys with empty strings so the ECS task definition (which mounts all of them) doesn't fail to start with `ResourceInitializationError`.

**When to use:** First deploy of an instance; rotating API keys; after `deploy.sh` complains the secret still has `REPLACE_ME`.

---

### `set-zoho-token.sh` — Set Zoho OAuth credentials for an instance

**Path:** [`scripts/set-zoho-token.sh`](../scripts/set-zoho-token.sh)

**Purpose:** Interactive helper to obtain and store Zoho OAuth credentials (self-client) for one Zoho service on one Mastra instance. It exchanges an authorization code for access/refresh tokens and writes them both locally and to AWS.

**Usage:**
```bash
./scripts/set-zoho-token.sh                          # prompts for service + instance
./scripts/set-zoho-token.sh <service> <instance>     # service ∈ crm|desk|billings|inventory|books
# example:
./scripts/set-zoho-token.sh desk main-mastra
```

**What it does:**
1. Reads required scopes from [`shared/zoho/required-scopes.json`](../shared/zoho/required-scopes.json) and prints the comma-separated scope string to paste into the Zoho API Console.
2. Prompts for region, self-client ID, client secret, and one-time authorization code.
3. Exchanges the code for an access + refresh token via `https://accounts.zoho.<region>/oauth/v2/token`.
4. Updates the dev `.env` at `instances/<instance>/.env` with `ZOHO_<SERVICE>_{CLIENT_ID,CLIENT_SECRET,REFRESH_TOKEN}` and `ZOHO_REGION`.
5. Merges the credentials into the AWS secret `mastra/<instance>` (preserving other keys).
6. Optionally restarts `advone-mastra-<instance>` to pick up the new secrets.

**When to use:** Adding/configuring a Zoho integration (Desk, CRM, Books, Inventory, Billings) for an instance; rotating Zoho credentials.

---

### `set-tailscale-key.sh` — Store the Tailscale auth key and restart the router

**Path:** [`scripts/set-tailscale-key.sh`](../scripts/set-tailscale-key.sh)

**Purpose:** Writes a Tailscale reusable auth key to Secrets Manager and forces the subnet-router ECS service to restart and pick it up.

**Usage:**
```bash
./scripts/set-tailscale-key.sh
```

**What it does:**
1. Prints instructions for generating a Tailscale key (reusable, pre-authorized, tagged `tag:mastra`, ≥90-day expiry) at <https://login.tailscale.com/admin/settings/keys>.
2. Reads the key silently, validates it starts with `tskey-auth-`.
3. Writes it to `mastra/tailscale-authkey` (`arn:aws:secretsmanager:us-east-2:628338935433:secret:mastra/tailscale-authkey-sCTQ1G`).
4. Forces a new deployment of ECS service `advone-mastra-tailscale`.
5. Waits, then greps the recent logs to confirm auth succeeded or failed.

**When to use:** The subnet router can't authenticate (key expired/rejected); first-time Tailscale setup; the `open-studio.sh` "No active Tailscale subnet router found" error points to an auth failure.

---

### `new-instance.sh` — Scaffold a new Mastra instance

**Path:** [`scripts/new-instance.sh`](../scripts/new-instance.sh)

**Purpose:** Creates a new instance directory from the `main-mastra` template and registers it (with `deploy: false`) in the instance registry plus a starter deployment config.

**Usage:**
```bash
./scripts/new-instance.sh <instance-name>   # lowercase, hyphenated
# example:
./scripts/new-instance.sh finance
```

**What it does:**
1. Copies `instances/main-mastra` → `instances/<name>` (excluding `node_modules`, `.mastra`, `.env`, lockfiles).
2. Updates the new `package.json` `name` field.
3. Appends an entry to `instances/instanceConfig.yml` (`deploy: false`, default CPU/mem, routing host `<name>.mastra.internal`).
4. Writes a starter deployment config to `scripts/examples/<name>-instance.yml`.
5. Prints the remaining steps: customize the instance, add `<name>` to root `instance_names` + re-apply root, set `deploy: true`, upload the config to S3, populate secrets, deploy.

**When to use:** Standing up a brand-new, independent Mastra server/agent deployment.

---

## Shared helper libraries (`scripts/lib/`)

These are sourced by the scripts above; not run directly.

| File | Used by | Purpose |
|------|---------|---------|
| `require.sh` | `deploy.sh`, `open-studio.sh` | Tool/credential checks, colored output helpers (`header`, `step_ok`, `step_fail`, `cmd_block`, `url`, etc.). |
| `select_config.sh` | `deploy.sh` | Interactive S3 config picker — lists objects under `s3://advone-east-2-deploy/mastra/`, downloads the chosen YAML to a temp dir. |
| `parse_config.sh` | `deploy.sh` | Parses the selected config YAML into shell vars (`INSTANCE_NAME`, `TARGET_STACK`, `BACKEND_HCL`, `TFVARS_JSON`, `ROUTING_HOST`, etc.). |
| `build_push.sh` | `deploy.sh` | Builds the instance Docker image and pushes it to its ECR repo; computes the image tag (git SHA by default). |
| `tailscale.sh` | `deploy.sh`, `open-studio.sh` | Tailscale helpers: `get_active_subnet_router_ip`, `approve_subnet_routes`, `cleanup_stale_subnet_routers` (uses the Tailscale API key from the `mastra/tailscale-authkey` secret). |

---

## Quick decision table

| You want to… | Run |
|--------------|-----|
| Deploy or update infra/an instance | `./scripts/deploy.sh` |
| Redeploy the same image (infra-only change) | `./scripts/deploy.sh --config <cfg> --skip-build` |
| Open Studio for an instance | `./scripts/open-studio.sh <instance>` |
| Set an instance's API keys / platform token | `./scripts/set-instance-secrets.sh <instance>` |
| Set up Zoho OAuth for an instance | `./scripts/set-zoho-token.sh <service> <instance>` |
| Fix/rotate the Tailscale auth key | `./scripts/set-tailscale-key.sh` |
| Create a new instance from the template | `./scripts/new-instance.sh <name>` |

---

## Reference

- Deployment details & exact AWS resources: [deployment.md](deployment.md)
- Studio access: [studio-access.md](studio-access.md)
- Instance registry: [`instances/instanceConfig.yml`](../instances/instanceConfig.yml)
- Deployment rule: [`.kilo/rules/deploy.md`](../.kilo/rules/deploy.md)
- Studio access rule: [`.kilo/rules/studio-access.md`](../.kilo/rules/studio-access.md)
