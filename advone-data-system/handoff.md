# advone-data-system — handoff

## What this project is

A multi-tenant (US + Canada) business data platform on AWS RDS PostgreSQL for Advantage
One's Geotab reseller business. Unified query layer over both regions with row-level
security respecting customer data tenancy. Sources: Geotab MyAdmin, MyGeotab, Zoho CRM /
Books (×2 orgs) / Inventory / Desk, plus finance CSV drops.

- **Repo:** `/Users/mattperzel/src/advone-data-system` (greenfield — was empty before this session)
- **Plan:** `docs/plan.md` in the repo (the authoritative document)

## Session 2026-07-19 — architecture plan written

Matt asked for a plan for the system. No code yet — deliverable was the plan document.

**Rev 2 same day:** Matt confirmed a **strict bidirectional residency requirement** —
Canadian data stays in Canadian data centers, US data stays in US data centers. The
original single-instance-in-ca-central-1 design was replaced with two regional stacks.

Key decisions in the plan (rev 2):

- **Two RDS PostgreSQL 16 instances** (db.t4g.medium each, single-AZ to start):
  ca-central-1 for CA data, us-east-2 for US data. Each region gets its own full stack
  (S3 landing, Secrets, connector Lambdas, backups) — one IaC stamp deployed twice.
- **Query federation, not replication:** a hub in the **CA** database uses `postgres_fdw`
  (read-only, marts-schema-only `hub_fdw` role) to build `global.*` UNION ALL views over
  both regions. Nothing cross-border is ever at rest; hub is in Canada so CA data never
  leaves even in flight. Metabase caching disabled for global questions. Fallback if the
  obligation covers in-flight data too: rollup exchange (only pre-aggregated rollups cross).
- **Tenancy:** `region` enforced physically; `customer_id` enforced with RLS inside each
  regional DB. Metabase sandboxing mirrors the same rules. CI gates: cross-customer
  adversarial tests + a residency test (hub_fdw can't read outside marts; no dbt model
  persists foreign-table data).
- **Source routing:** Books/Billing/Inventory split naturally by org; CRM/Desk/MyAdmin
  (shared) route by the confirmed ERP rule (below); each region's connector persists only
  its own region's rows, filtered before write.

**Rev 3 same day:** Matt answered all open questions:

- Zoho topology: **1 CRM, 1 Desk, 2× Books, 2× Billing, 2× Inventory** — Zoho Billing
  added as a source (the MRR system of record).
- **One Geotab MyAdmin account** for both regions; rows routed by customer region.
- **CRM region rule: account ERP = 'ADVA01' → Canadian, everything else → US.** Already
  populated, deterministic, no backfill; Phase 0 just sanity-checks null-ERP accounts.
- Finance data arrives incrementally as CSV/Excel → plan §7a defines a **define-a-table
  framework**: YAML dataset spec in `datasets/` → generated DDL + dbt staging model →
  generic validating loader Lambda on S3 drop (Excel supported natively). New dataset
  ≈ 30 min; new months of existing data = drag-and-drop.
- No regional staff roles day 1, but **other systems will query this data** → per-system
  `svc_*` Postgres roles, read-only on `marts`/`global` (the published contract), private
  networking only; PostgREST as the future REST option.

**Rev 4 same day:** two changes at Matt's request:

- **OpenTofu instead of CDK** (plan §2, §11 Phase 0): single `regional-stack` module
  instantiated twice via provider aliases (aws.us/aws.ca) from one root — one `tofu plan`
  covers both regions. S3 state backend in ca-central-1, version pinned, plan-on-PR /
  apply-on-merge via GitHub Actions. Note: this diverges from FMIS's CDK choice.
- **Docker local dev environment** (new plan §7b): docker-compose with pg-us + pg-ca
  (real postgres_fdw federation between containers), MinIO for regional S3 buckets,
  WireMock replaying recorded/scrubbed Zoho/MyAdmin fixtures, optional Metabase.
  Connectors are handler/core split and ship as Lambda container images, so the artifact
  tested in compose is the deployed artifact. dbt + RLS/residency adversarial tests run
  locally; CI runs the same compose stack. LocalStack deliberately avoided (EventBridge/
  Secrets/IAM covered by tofu plan review + staging smoke).
- **ELT:** custom TypeScript Lambda connectors (EventBridge cron) → raw JSON to S3 →
  Postgres `raw` schemas → **dbt-core** transforms → staging/core/marts.
- **Customer master** (`mapping.customer_xref`, human-curated dbt seed, matcher-assisted)
  linking CRM ↔ Books ↔ Desk ↔ MyAdmin ↔ MyGeotab identities — flagged as the
  highest-value piece.
- **BI:** Metabase OSS on Fargate (ca-central-1, beside the hub). Cost target ~$200–265/mo
  (residency requirement adds ~$100/mo over the single-instance design).
- **Roadmap:** Phase 1 = infra + Zoho Books end-to-end slice; Phase 2 = customer master +
  CRM/Desk; Phase 3 = Geotab + the margin mart (billed revenue vs Geotab device cost);
  Phase 4 = hardening/insights.

Context that informed this: Matt's FMIS project (`~/FMIS/docs/plan/`) locked in
CDK/TypeScript/Postgres-with-RLS patterns — this plan deliberately reuses them. Zoho Desk
org is 702896317. Zoho MCP connectors for CRM/Books/Inventory/Desk are already wired in
Matt's Claude Code setup.

## Session 2026-07-19 (later) — infra implemented (OpenTofu + advone CLI)

Matt asked for the plan to be made deployable: OpenTofu + foolproof wizard-driven
scripts, config saved to S3 for easy redeploys, backend/tfvars managed by CLI (never
raw tofu). Built and validated (`tofu validate` passes, `bash -n` passes, help/check
exercised; NOT yet deployed to AWS — no creds on this machine during the session):

- `scripts/advone` — single bash CLI (macOS bash-3.2 safe). Subcommands:
  `check` (preflight, reports all missing tools incl. session-manager-plugin),
  `bootstrap` (idempotent admin bucket `advone-ds-admin-<acct>` in ca-central-1:
  versioned, KMS, public-blocked; holds `config/<env>.json` + `tfstate/<env>.tfstate`),
  `configure` (wizard with validated prompts — residency-guarded region choices,
  CIDR/email/instance-class validation, defaults prefilled from existing config;
  pushes config to S3), `config show|pull|push`, `deploy` (pulls S3 config →
  generates backend.hcl + tfvars.json into gitignored `.advone/<env>/` → init
  -reconfigure → plan → confirm → apply), `plan`, `output`, `destroy` (typed-name
  guard; auto-disables RDS deletion protection first), and `db tunnel|psql|init`
  (SSM port-forwarding via bastion; `db init` applies sql/ files to both DBs,
  generates role passwords via Secrets Manager get-random-password, upserts one
  secret per role per region, builds the FDW hub; idempotent, re-run rotates passwords).
- `infra/modules/regional-stack/` — per-region stamp: VPC (2 AZ, private DB subnets,
  public bastion subnet, NO NAT), RDS Postgres 16 (gp3, encrypted, force_ssl param
  group, 7-day backups, deletion protection, final snapshot, master creds in Secrets
  Manager), regional data bucket (landing/ + drops/ prefixes, versioned, KMS,
  lifecycle), t4g.nano SSM bastion (no ingress, IMDSv2, installs postgresql16 client).
- `infra/modules/peering/` — cross-region peering + routes + THE one cross-border SG
  rule (CA VPC → US DB :5432); deliberately no US→CA mirror rule.
- `infra/root/` — both stacks from the same module via provider aliases + peering +
  account budget (80% actual / 100% forecast alerts). Residency guards as variable
  validations (ca_region must be ca-*, us_region must be us-*). `backend "s3" {}`
  partial config, filled by generated backend.hcl (uses OpenTofu ≥1.10 use_lockfile —
  no DynamoDB).
- `sql/` — 00_regional_init (schemas staging/core/marts/meta/mapping, meta.sync_state,
  roles dbt_transform/readonly_ai/analyst_local via \gexec create-if-missing, default
  privileges), 10_us_hub_user (hub_fdw: marts-only, conn limit, statement timeout),
  20_ca_federation_hub (postgres_fdw, drop+recreate us_marts server, user mappings,
  IMPORT FOREIGN SCHEMA marts INTO us_fdw, global schema).
- `README.md` quickstart + `.gitignore` (.advone/, .terraform/, tfstate/tfplan).

Design notes: no NAT gateways (nothing private needs egress yet; revisit when
connector Lambdas land — VPC endpoints or NAT then). Local tofu provider cache
exists under infra/root/.terraform from validation. Repo is still not a git repo —
Matt hasn't asked to init/commit.

## Current state

- Plan (docs/plan.md rev 4) + full deployable infra implemented as above.
- Verified: tofu validate ✓, bash syntax ✓, CLI help/check ✓. Not verified: actual
  AWS deploy (no credentials in session), db init against real RDS, wizard end-to-end
  (interactive tty required).
- First real deploy: `scripts/advone check` → `configure` → `deploy` → `db init`.

## Next steps

All §12 questions answered (see rev 3 notes above); §12 is now a decisions log. One
remaining sub-question: does the residency obligation cover data **in flight** (query-time
federation through the CA hub) or only **at rest**? Plan defaults to at-rest; rollup-exchange
fallback documented in plan §2.

Ready to start **Phase 0**: sanity-check the ERP=ADVA01 rule against all CRM accounts,
gather API credentials (Zoho self-clients ×5 services, MyAdmin API user, MyGeotab service
accounts), scaffold the repo (`infra/` OpenTofu, `connectors/`, `dbt/`, `datasets/`,
`fixtures/`, `docker-compose.yml`, `docs/`). Then Phase 1: compose stack + Books connector
proven locally first, then the dual-region OpenTofu stamp + federation hub + the same
Books slice deployed to AWS.
