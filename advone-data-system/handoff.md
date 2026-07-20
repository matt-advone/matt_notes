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
  (S3 landing, Secrets, connector Lambdas, backups) — one CDK stamp deployed twice.
- **Query federation, not replication:** a hub in the **CA** database uses `postgres_fdw`
  (read-only, marts-schema-only `hub_fdw` role) to build `global.*` UNION ALL views over
  both regions. Nothing cross-border is ever at rest; hub is in Canada so CA data never
  leaves even in flight. Metabase caching disabled for global questions. Fallback if the
  obligation covers in-flight data too: rollup exchange (only pre-aggregated rollups cross).
- **Tenancy:** `region` enforced physically; `customer_id` enforced with RLS inside each
  regional DB. Metabase sandboxing mirrors the same rules. CI gates: cross-customer
  adversarial tests + a residency test (hub_fdw can't read outside marts; no dbt model
  persists foreign-table data).
- **Source routing:** Books/Inventory split naturally by org; CRM/Desk (shared orgs) route
  by a region field on each account, unclassified records quarantined in CA; Phase 0
  includes a CRM region-classification pass.
- **ELT:** custom TypeScript Lambda connectors (EventBridge cron) → raw JSON to S3 →
  Postgres `raw` schemas → **dbt-core** transforms → staging/core/marts.
- **Customer master** (`mapping.customer_xref`, human-curated dbt seed, matcher-assisted)
  linking CRM ↔ Books ↔ Desk ↔ MyAdmin ↔ MyGeotab identities — flagged as the
  highest-value piece.
- **BI:** Metabase OSS on Fargate. Cost target ~$100–150/mo.
- **Roadmap:** Phase 1 = infra + Zoho Books end-to-end slice; Phase 2 = customer master +
  CRM/Desk; Phase 3 = Geotab + the margin mart (billed revenue vs Geotab device cost);
  Phase 4 = hardening/insights.

Context that informed this: Matt's FMIS project (`~/FMIS/docs/plan/`) locked in
CDK/TypeScript/Postgres-with-RLS patterns — this plan deliberately reuses them. Zoho Desk
org is 702896317. Zoho MCP connectors for CRM/Books/Inventory/Desk are already wired in
Matt's Claude Code setup.

## Current state

- Plan document written; nothing built, nothing deployed. Repo has only `docs/plan.md`.

## Next steps / open questions (answers needed from Matt — §12 of plan)

1. Data-residency requirements in any Canadian customer contract?
2. Confirm Zoho topology: two Books orgs (US/CA), one CRM, one Desk?
3. One Geotab MyAdmin account or separate US/CA?
4. Finance beyond Books: payroll/banking as connectors or CSV drops?
5. Who queries this besides Matt — do region-scoped roles matter day 1?

Then: Phase 0 (credentials inventory, repo scaffold) → Phase 1 (CDK infra + Books slice).
