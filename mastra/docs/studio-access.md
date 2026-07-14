# Mastra Studio Access Guide

> Mastra Studio (the agent authoring/management UI) runs **inside the private VPC** on each Mastra instance ECS task and is **not publicly exposed**. This document explains how to reach it for each instance.
>
> Studio is served on **port `4111`** of each ECS task's private IP. Access from a laptop is routed through a **Tailscale SOCKS5 proxy** running on an ECS subnet-router task.

---

## The only supported method: `open-studio.sh`

```bash
./scripts/open-studio.sh                 # defaults to main-mastra
./scripts/open-studio.sh main-mastra     # explicit
./scripts/open-studio.sh finance         # another instance
```

**Never** hand-craft a Chrome command or hardcode IPs. The ECS task IP changes on every deployment and the active Tailscale subnet router IP can also change; the script detects both automatically.

### What the script does

1. **Auto-approves the VPC subnet route** (`10.30.0.0/16`) via the Tailscale API using the key in `mastra/tailscale-authkey` — no manual admin-console step needed.
2. **Finds the active Tailscale subnet router IP** from `tailscale status` (the device tagged as the subnet router on the tailnet).
3. **Looks up the current ECS task private IP** for the instance's service via the ECS API.
4. **Tests the SOCKS5 proxy** (`socks5://<router-ip>:1055`) can reach `http://<task-ip>:4111/health`.
5. **Launches a dedicated Chrome instance** with `--proxy-server=socks5://<router-ip>:1055` and a separate `--user-data-dir` (so the proxy does not affect your normal browser session), pointed at `http://<task-ip>:4111`.

---

## How the network path works

```
Your laptop
   │  Chrome  --proxy-server=socks5://<router-ip>:1055
   ▼
Tailscale subnet router ECS task  (service advone-mastra-tailscale, port 1055 SOCKS5)
   │  private VPC 10.30.0.0/16
   ▼
Mastra instance ECS task  (service advone-mastra-<instance>, port 4111)
   └── Mastra Studio UI
```

- The private VPC is `10.30.0.0/16`.
- Tailscale advertises that route requires approval in the tailnet admin console (the script calls the Tailscale API to approve it, but the **ACL + first-time route approval** may still need a manual step).
- The Mastra security group (`sg-0cb30e1f82be42227`) allows inbound `4111/tcp` **only from the Tailscale subnet-router security group** (`sg-0d5fa59ecebfdae61`), via rule `sgrule-3477425641`.

---

## Prerequisites (must all be true)

1. **Tailscale connected on your laptop** — `tailscale status` shows the `advone-mastra-subnet-router` device as online.
2. **Subnet route approved** — the `10.30.0.0/16` route is approved for `advone-mastra-subnet-router`:
   <https://login.tailscale.com/admin/machines> → find the router → Edit route settings → approve `10.30.0.0/16`.
   (The script auto-approves via API, but verify here if it fails.)
3. **ECS Mastra service healthy** — the instance task is `RUNNING` and `HEALTHY`:
   ```bash
   aws ecs describe-services --cluster advone-mastra-cluster \
     --services advone-mastra-main-mastra --region us-east-2 \
     --query 'services[0].{status:status,running:runningCount,desired:desiredCount}'
   ```
4. **Tailscale ACL** allows your device to reach `tag:mastra`:
   ```json
   { "action": "accept", "src": ["autogroup:member"], "dst": ["tag:mastra:*"] }
   ```
5. **Tailscale auth key valid** — if the subnet router isn't authenticating, run `./scripts/set-tailscale-key.sh` with a fresh reusable, pre-authorized, `tag:mastra` key, then re-run `open-studio.sh`.

---

## Per-instance access

The same `open-studio.sh` command works for any instance — just pass the instance name. The script derives everything from the instance registry:

| Instance | Command | ECS service | Hostname (internal DNS) |
|----------|---------|-------------|--------------------------|
| `main-mastra` | `./scripts/open-studio.sh main-mastra` | `advone-mastra-main-mastra` | `main.mastra.internal` |
| `finance` | `./scripts/open-studio.sh finance` | `advone-mastra-finance` | `finance.mastra.internal` |
| `<new>` | `./scripts/open-studio.sh <new>` | `advone-mastra-<new>` | `<new>.mastra.internal` |

> Note: the script opens Studio by the task's **private IP** (more reliable than the internal DNS hostname from outside the VPC), not the `*.mastra.internal` name. The internal hostname exists for VPC-internal traffic (ALB listener rules, Lambdas).

---

## After every deployment

The ECS task gets a **new private IP on every deployment**. Just re-run the script — it always looks up the current IP:

```bash
./scripts/open-studio.sh main-mastra
```

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `No active Tailscale subnet router found` | Tailscale ECS service down or not connected | Check `aws ecs describe-services --cluster advone-mastra-cluster --services advone-mastra-tailscale --region us-east-2`; logs `aws logs tail /ecs/advone-mastra-tailscale --follow --region us-east-2`. If auth key bad → `./scripts/set-tailscale-key.sh`. |
| Chrome opens but page doesn't load | Subnet route not approved (first time) | Approve `10.30.0.0/16` at <https://login.tailscale.com/admin/machines> |
| Proxy connects but times out | Task unhealthy or still starting | Tail `aws logs tail /ecs/advone-mastra-<instance> --follow --region us-east-2` |
| `No running tasks found for service '...'` | ECS service has 0 running tasks | Check service events in ECS console; redeploy with `./scripts/deploy.sh --config mastra/configs/<instance>-instance.yml --skip-build` |
| Task `UNHEALTHY` warning shown | Health check failing | Studio may still partially work; confirm `aws ecs describe-tasks ... healthStatus`; investigate the `/health` endpoint via the proxy |
| `Could not determine task IP` | ENI attachment details missing | `aws ecs describe-tasks --cluster advone-mastra-cluster --tasks <arn> --region us-east-2` and inspect `attachments[].details` for `privateIPv4Address` |

### Manual fallback (debugging only)

If `open-studio.sh` fails partway, you can reproduce its steps manually:

```bash
# 1. Router IP (the subnet router on your tailnet)
tailscale status | grep -i subnet

# 2. Current task IP for the instance
TASK_ARN=$(aws ecs list-tasks --cluster advone-mastra-cluster \
  --service-name advone-mastra-main-mastra --region us-east-2 \
  --query 'taskArns[0]' --output text)
aws ecs describe-tasks --cluster advone-mastra-cluster --tasks "$TASK_ARN" \
  --region us-east-2 \
  --query 'tasks[0].attachments[0].details[?name==`privateIPv4Address`].value' --output text

# 3. Test the proxy
curl -s --socks5 <router-ip>:1055 http://<task-ip>:4111/health

# 4. Launch Chrome
open -na "Google Chrome" --args \
  --proxy-server="socks5://<router-ip>:1055" \
  --user-data-dir="/tmp/chrome-mastra-main-mastra" \
  "http://<task-ip>:4111"
```

---

## Reference

- Script: [`scripts/open-studio.sh`](../scripts/open-studio.sh) · helpers: [`scripts/lib/tailscale.sh`](../scripts/lib/tailscale.sh)
- Deployment details: [deployment.md](deployment.md)
- Scripts overview: [scripts.md](scripts.md)
- Project rule for Studio access: [`.kilo/rules/studio-access.md`](../.kilo/rules/studio-access.md)
