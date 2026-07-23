---
type: ProjectStatus
related_to: "[[email]]"
status: Active
---

# Email - DNS Deliverability Audit

## Last updated
2026-07-23

## Project
Email infrastructure and DNS for `/Users/mattperzel/src/email`, specifically `advantageone.tech` and `advtracking.net`.

## Current session
- Audited the Route 53 hosted zones and their publicly authoritative DNS answers using the AWS SSO administrator role in account `628338935433`.
- Confirmed `advantageone.tech` is delegated to its Route 53 nameservers. `advtracking.net` is instead delegated to Hurricane Electric nameservers, so the Route 53 hosted zone is not authoritative.
- Confirmed the active AWS SES identities in `us-east-2` for both domains are verified, sending-enabled, and use successful 2048-bit Easy DKIM.
- Found a critical SPF defect at `advantageone.tech`: its root SPF record has a second `v=spf1` declaration and evaluates to `permerror` because it also exceeds the SPF ten-DNS-lookup limit.
- Repaired the `advantageone.tech` SPF record in Route 53. Change `C0846621NY2BRGCHOJOX` is `INSYNC`; the authoritative AWS nameserver returns one valid policy with nine recursive DNS lookups.
- The new policy preserves Zoho, Zoho Subscriptions, ZeptoMail, Zoho Campaigns, Google Workspace, Zoho Analytics, LeadConnector, and `206.55.134.0/24`. The root Mailgun authorization was removed because Mailgun is configured on dedicated `ao` and `av` subdomains and retaining it would exceed the SPF limit.
- `advtracking.net` has a syntactically valid SPF record that evaluates normally. Both domains publish DMARC `p=reject`; AdvantageOne applies it only to 50 percent of mail.

## Reminders / gotchas
- Do not modify or depend on the Route 53 `advtracking.net` zone without first changing registrar delegation from `ns1/ns2.hurricanedatacentres.com` or removing the duplicate zone. Its Route 53 nameservers are publicly unused.
- Several old SES selector CNAMEs publish an empty `p=` key. These are disabled historical selectors; the currently configured SES selectors are valid.
- The SES subdomain is spelled `deliverabilty.advantageone.tech` in the current configuration.

## Next up
- Confirm delivery from each active root-domain sender after recursive DNS caches expire (up to one hour from the 2026-07-23 change), especially any legacy Mailgun sender that might still use `@advantageone.tech`.
- Prefer dedicated sending subdomains for bulk and vendor mail.
- After SPF/DKIM monitoring is clean, change AdvantageOne DMARC from `pct=50` to `pct=100`.
- Consolidate `advtracking.net` DNS administration at either Hurricane Electric or Route 53, then eliminate the inactive duplicate zone.
