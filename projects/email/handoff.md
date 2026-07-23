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
- `advtracking.net` has a syntactically valid SPF record that evaluates normally. Both domains publish DMARC `p=reject`; AdvantageOne applies it only to 50 percent of mail.

## Reminders / gotchas
- Do not modify or depend on the Route 53 `advtracking.net` zone without first changing registrar delegation from `ns1/ns2.hurricanedatacentres.com` or removing the duplicate zone. Its Route 53 nameservers are publicly unused.
- Several old SES selector CNAMEs publish an empty `p=` key. These are disabled historical selectors; the currently configured SES selectors are valid.
- The SES subdomain is spelled `deliverabilty.advantageone.tech` in the current configuration.

## Next up
- Replace the root `advantageone.tech` SPF policy with a single, deliberately scoped policy that stays below ten DNS lookups and only authorizes active senders.
- Test representative mail from Zoho, Google, SES, Mailgun, LeadConnector, ZeptoMail, and Zoho services before removing their SPF authorizations. Prefer dedicated sending subdomains for bulk and vendor mail.
- After SPF/DKIM monitoring is clean, change AdvantageOne DMARC from `pct=50` to `pct=100`.
- Consolidate `advtracking.net` DNS administration at either Hurricane Electric or Route 53, then eliminate the inactive duplicate zone.
