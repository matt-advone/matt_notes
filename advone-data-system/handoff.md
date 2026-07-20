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

Key decisions recommended in the plan:

- **One RDS PostgreSQL 16 instance** (db.t4g.medium, start single-AZ) in **ca-central-1**
  — chosen so Canadian data-residency asks are satisfied by default; US data in Canada is
  not a contractual problem. Two physical regional DBs rejected in favor of logical tenancy.
- **Two-level tenancy:** `region` ('US'/'CA') + `customer_id` columns on all core tables,
  enforced with Postgres RLS + role-per-access-tier (analyst_all, analyst_us/ca,
  customer_scoped, readonly_ai). Metabase sandboxing mirrors the same rules.
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
