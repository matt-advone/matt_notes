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
