---
type: ProjectStatus
related_to: "[[tolaria]]"
status: Active
---

# aws-stuff — Notes

## Last updated
2026-07-23

## Current session
Added CORS support to CloudFront distribution `EGU78B1OKLWGB` (add-in bucket
`andrew-b-add-in-bucket`) for origins `https://my.geotab.com` and
`https://gov.geotab.com`.

## What was done
- Created a CloudFront response headers policy `geotab-addin-cors-policy`
  (ID `bf5355ca-707c-4872-add7-5eb3892d8b6d`) with a CORS config allowing
  `https://my.geotab.com` and `https://gov.geotab.com`, OriginOverride=true.
- Attached it to the default cache behavior of distribution `EGU78B1OKLWGB`
  (`ResponseHeadersPolicyId`).
- Added `OPTIONS` to AllowedMethods/CachedMethods so CloudFront answers CORS
  preflight directly without hitting the origin.
- Distribution redeployed and verified with curl.

## Why a response headers policy (not S3 bucket CORS)
- Origin is S3 behind OAC (private bucket) and the cache policy is the managed
  `CachingDisabled` policy, which does NOT forward the `Origin` header to S3.
  So S3-side CORS alone wouldn't work reliably. A response headers policy makes
  CloudFront add the CORS headers per-request based on the request `Origin`,
  independent of the origin/cache key.

## Verified
- GET with `Origin: https://my.geotab.com` → `access-control-allow-origin: https://my.geotab.com`
- GET with `Origin: https://gov.geotab.com` → `access-control-allow-origin: https://gov.geotab.com`
- GET with disallowed origin → no `access-control-allow-origin` header
- OPTIONS preflight from my.geotab.com → returns allow-methods/allow-headers/max-age
- 200 GET on a real object (`esri-map-browser-addin/`) carries the CORS header

## Notes / gotchas
- AccessControlAllowCredentials is `false`. If the add-in ever needs cookies/auth
  sent cross-origin, flip this to true in the policy (cannot use `*` for origins
  then; specific origins are already set so that's fine).
- The root path `/` returns 403 because there is no default root object and no
  object at `/` — pre-existing, unrelated to CORS.
- Distribution domain: `d209vxufgoircg.cloudfront.net`

## Next up
- None unless credentials-based CORS or additional origins are needed.
