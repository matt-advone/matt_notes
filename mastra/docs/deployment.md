# Mastra Deployment Guide

> **Audience:** DevOps / on-call. This document maps every step of a deployment to the **exact AWS resources** it touches, so you can jump straight to the failing component in the AWS console or CLI when something breaks.
>
> All resources live in **region `us-east-2`**, AWS account **`628338935433`**.
> State is managed by **OpenTofu** with backends in S3 bucket **`advone-east-2-deploy`**.

---

## 1. Architecture overview

Mastra is deployed as **two OpenTofu stacks**:

| Stack | Path | State key in S3 | Purpose |
|-------|------|-----------------|---------|
| **root** | `infra/root/` | `mastra/infra/root/terraform.tfstate` | Shared, once-per-environment infra: VPC, ECS cluster, internal ALB, Route53 zone, ECR repos, API Gateway, event Lambdas, Tailscale subnet router. |
| **instance** | `infra/instance/` | `mastra/instance/<name>/terraform.tfstate` | Per-instance resources: ECS service + task definition, DynamoDB table, Secrets Manager secret, ALB listener rule + target group. Reads shared outputs from root via `terraform_remote_state`. |

Each **instance** (e.g. `main-mastra`, `finance`) is a separate Mastra server image, deployed independently with its own state file and its own ECS service.

Instances are registered in [`instances/instanceConfig.yml`](../instances/instanceConfig.yml). An instance is only deployable when its `deploy:` flag is `true`.

```
advone-mastra (root)                       advone-mastra-<instance>
├── VPC 10.30.0.0/16                       ├── ECS service  advone-mastra-<instance>
├── ECS cluster  advone-mastra-cluster     ├── Task definition  advone-mastra-<instance>
├── Internal ALB  advone-mastra-internal-alb├── Target group  advone-mastra-<instance>-tg
├── Route53 zone  mastra.internal           ├── ALB listener rule (host: <instance>.mastra.internal)
├── ECR repos  advone-mastra/<instance>     ├── DynamoDB table  advone-mastra-<instance>
├── API Gateway  advone-mastra-events       └── Secrets Manager secret  mastra/<instance>
├── Event Lambdas (zoho/generic/s3-ingest)
└── Tailscale subnet router service
```

---

## 2. Deployment tooling

All deploys go through **`./scripts/deploy.sh`** — never run `tofu apply` directly (see [scripts.md](scripts.md)).

Prerequisites the script checks for:

- `tofu`, `aws`, `yq`, `jq`, `docker`, `python3` on PATH
- AWS credentials with access to account `628338935433`
- State bucket `advone-east-2-deploy` exists (created one-time on first deploy)
- For instance deploys: the root stack must already be applied (state file present)

A deploy is driven by a **YAML config file uploaded to S3** under `s3://advone-east-2-deploy/mastra/`. The config selects the stack (`root` or `instance`) and provides tfvars + backend settings.

---

## 3. Deploying the ROOT stack

The root stack is deployed **once per environment** (production). It is the prerequisite for every instance deploy.

### 3.1 Steps

1. Create a root config file (e.g. `/tmp/mastra-root.yml`):

   ```yaml
   instance: root
   stack: root

   backend:
     bucket: advone-east-2-deploy
     key: mastra/infra/root/terraform.tfstate
     region: us-east-2

   tfvars:
     instance_names:
       - main-mastra
     aws_region: us-east-2
     environment: production
     tailscale_authkey_secret_arn: "arn:aws:secretsmanager:us-east-2:628338935433:secret:mastra/tailscale-authkey-XXXXXX"
   ```

2. Upload it:
   ```bash
   aws s3 cp /tmp/mastra-root.yml s3://advone-east-2-deploy/mastra/mastra-root.yml
   ```

3. Deploy:
   ```bash
   ./scripts/deploy.sh --config mastra/mastra-root.yml
   ```

4. The script runs `tofu init` → `tofu plan` → prompts `yes` → `tofu apply`, then approves Tailscale subnet routes and prints next steps.

> **Tailscale is optional but recommended** — without it Studio is not reachable from a laptop. Create the auth key first with `./scripts/set-tailscale-key.sh` (see [scripts.md](scripts.md)) and put the returned ARN in `tailscale_authkey_secret_arn`.

### 3.2 Root stack — exact AWS resources

> **State source:** `s3://advone-east-2-deploy/mastra/infra/root/terraform.tfstate`
> Pull with: `aws s3 cp s3://advone-east-2-deploy/mastra/infra/root/terraform.tfstate - | jq`

#### Network (`module.network`)

| Resource | Address | ID / value |
|----------|---------|------------|
| VPC | `module.network.aws_vpc.main` | `vpc-071616bf39177803d` (CIDR `10.30.0.0/16`) |
| Public subnet 2a | `module.network.aws_subnet.public[0]` | `subnet-0f0946ebcce380337` (`10.30.0.0/24`) |
| Public subnet 2b | `module.network.aws_subnet.public[1]` | `subnet-02d9d1ab8ef57e770` (`10.30.1.0/24`) |
| Private subnet 2a | `module.network.aws_subnet.private[0]` | `subnet-0d4ed8f39f0da2d2d` (`10.30.10.0/24`) |
| Private subnet 2b | `module.network.aws_subnet.private[1]` | `subnet-0b5af70949d3e034d` (`10.30.11.0/24`) |
| Internet Gateway | `module.network.aws_internet_gateway.main` | `igw-0e971a6f41bcde18a` |
| NAT Gateway | `module.network.aws_nat_gateway.main` | `nat-03f39d9dfe219bdaf` |
| NAT EIP | `module.network.aws_eip.nat` | `eipalloc-0390cf51e32207abb` |
| Public route table | `module.network.aws_route_table.public` | `rtb-0fdad87d289a8d9c6` |
| Private route table | `module.network.aws_route_table.private` | `rtb-0f69442a0cdfa280d` |
| ALB security group | `module.network.aws_security_group.alb` | `sg-0c19d7163858fd675` |
| Mastra task security group | `module.network.aws_security_group.mastra` | `sg-0cb30e1f82be42227` |
| Lambda security group | `module.network.aws_security_group.lambda` | `sg-065aed3647b361494` |
| VPC endpoint SG | `module.network.aws_security_group.vpce` | `sg-016f5ee733e018033` |
| S3 VPC endpoint (Gateway) | `module.network.aws_vpc_endpoint.s3` | `vpce-09691fad9e7179816` |
| DynamoDB VPC endpoint (Gateway) | `module.network.aws_vpc_endpoint.dynamodb` | `vpce-02665fa12243e8b6f` |
| Interface endpoints (ECR api, etc.) | `module.network.aws_vpc_endpoint.interface` | `vpce-00c884fcf0097fb76`, `vpce-00aa57c2ca96b03b0`, `vpce-056b96b417bb965f8`, `vpce-0a8759504068ef051`, `vpce-0e19a4ad8f522616e` |

#### ECS cluster (`module.ecs_cluster`)

| Resource | ID / value |
|----------|------------|
| Cluster | `arn:aws:ecs:us-east-2:628338935433:cluster/advone-mastra-cluster` |
| Cluster name | `advone-mastra-cluster` |

#### Internal ALB (`module.alb_internal`)

| Resource | Address | ID / value |
|----------|---------|------------|
| Load balancer | `module.alb_internal.aws_lb.internal` | `arn:aws:elasticloadbalancing:us-east-2:628338935433:loadbalancer/app/advone-mastra-internal-alb/7fc13c80a8b3a99b` |
| ALB DNS | output | `internal-advone-mastra-internal-alb-563951502.us-east-2.elb.amazonaws.com` |
| HTTP listener | `module.alb_internal.aws_lb_listener.http` | `arn:aws:elasticloadbalancing:us-east-2:628338935433:listener/app/advone-mastra-internal-alb/7fc13c80a8b3a99b/9a95d0c320d60622` |

#### ECR (`module.ecr`)

| Resource | Address | ID / value |
|----------|---------|------------|
| main-mastra repo | `module.ecr.aws_ecr_repository.instance["main-mastra"]` | `advone-mastra/main-mastra` (URL `628338935433.dkr.ecr.us-east-2.amazonaws.com/advone-mastra/main-mastra`) |
| Lifecycle policy | `module.ecr.aws_ecr_lifecycle_policy.instance` | on `advone-mastra/main-mastra` |

#### Route53 (private zone)

| Resource | Address | ID / value |
|----------|---------|------------|
| Private zone `mastra.internal` | `aws_route53_zone.mastra_internal` | `Z0244475V2WYBFXX1U79` |
| Wildcard `*.mastra.internal` → ALB | `aws_route53_record.wildcard` | record on `Z0244475V2WYBFXX1U79` |

#### Tailscale subnet router (`module.tailscale`)

| Resource | Address | ID / value |
|----------|---------|------------|
| ECS service | `module.tailscale.aws_ecs_service.tailscale` | `arn:aws:ecs:us-east-2:628338935433:service/advone-mastra-cluster/advone-mastra-tailscale` |
| Task definition | `module.tailscale.aws_ecs_task_definition.tailscale` | `advone-mastra-tailscale` |
| Task role | `module.tailscale.aws_iam_role.tailscale` | `advone-mastra-tailscale-task-role` |
| Security group | `module.tailscale.aws_security_group.tailscale` | `sg-0d5fa59ecebfdae61` |
| CloudWatch log group | `module.tailscale.aws_cloudwatch_log_group.tailscale` | `/ecs/advone-mastra-tailscale` |
| Studio ingress rule (4111) | `aws_security_group_rule.tailscale_to_mastra_studio` | `sgrule-3477425641` (on Mastra SG, source = Tailscale SG) |

#### API Gateway (`module.api_gateway`)

| Resource | Address | ID / value |
|----------|---------|------------|
| HTTP API | `module.api_gateway.aws_apigatewayv2_api.main` | `6y1fzc2d0l` (name `advone-mastra-events`) |
| Invoke URL | output | `https://6y1fzc2d0l.execute-api.us-east-2.amazonaws.com/` |
| Stage | `module.api_gateway.aws_apigatewayv2_stage.default` | `$default` |
| Access log group | `module.api_gateway.aws_cloudwatch_log_group.api_gw` | `/aws/apigateway/advone-mastra-events` |

#### Event Lambdas (`module.events_lambda`)

| Resource | Function name | Log group |
|----------|---------------|-----------|
| Zoho webhook | `advone-mastra-zoho-webhook` | `/aws/lambda/advone-mastra-zoho-webhook` |
| Generic webhook | `advone-mastra-generic-webhook` | `/aws/lambda/advone-mastra-generic-webhook` |
| S3 ingest | `advone-mastra-s3-ingest` | `/aws/lambda/advone-mastra-s3-ingest` |

| Resource | Address | ID / value |
|----------|---------|------------|
| Lambda role | `module.events_lambda.aws_iam_role.lambda` | `advone-mastra-events-lambda-role` |
| Lambda policy | `module.events_lambda.aws_iam_role_policy.lambda_permissions` | `advone-mastra-events-lambda-role:advone-mastra-events-lambda-policy` |
| VPC access attachment | `module.events_lambda.aws_iam_role_policy_attachment.lambda_vpc` | attached to `advone-mastra-events-lambda-role` |
| S3 bucket notification | `module.events_lambda.aws_s3_bucket_notification.ingest` | on bucket `advone-mastra-artifacts` |
| API GW → Lambda permission | `module.events_lambda.aws_lambda_permission.apigw` | `AllowAPIGateway` |
| S3 → Lambda permission | `module.events_lambda.aws_lambda_permission.s3` | `AllowS3Invoke` |
| API GW integrations | `module.events_lambda.aws_apigatewayv2_integration.webhook` | `ke1ayo0`, `vy3lg0m` |
| API GW routes | `module.events_lambda.aws_apigatewayv2_route.webhook` | `4gpvkbk`, `vr4dcbq` |

#### Shared IAM (root)

| Resource | ID / value |
|----------|------------|
| ECS execution role | `advone-mastra-ecs-execution-role` |
| Execution role policy (secrets access) | `advone-mastra-ecs-execution-role:advone-mastra-ecs-execution-secrets` (grants `secretsmanager:GetSecretValue` on `arn:aws:secretsmanager:us-east-2:628338935433:secret:mastra/*`) |
| ECS task execution managed policy | `AmazonECSTaskExecutionRolePolicy` attached to the execution role |

> **External (manually created) resources** referenced but not managed by root state:
> - Secrets Manager secret `mastra/tailscale-authkey` (ARN `arn:aws:secretsmanager:us-east-2:628338935433:secret:mastra/tailscale-authkey-sCTQ1G`) — written by `set-tailscale-key.sh`.
> - S3 bucket `advone-mastra-artifacts` (Lambda artifacts / S3 ingest source).

---

## 4. Deploying an INSTANCE stack

Each instance is deployed independently. Repeat for every instance listed in `instanceConfig.yml` with `deploy: true`.

### 4.1 Prerequisites

- Root stack already applied (state file exists at the S3 key).
- Instance entry exists in `instances/instanceConfig.yml` with `deploy: true`.
- The instance's name is in the root `instance_names` tfvars (so an ECR repo + Lambda allowlist entry exist). Add it and re-apply root if missing.
- A deployment config YAML exists for the instance, e.g. `s3://advone-east-2-deploy/mastra/configs/main-mastra-instance.yml`. The `new-instance.sh` script scaffolds one under `scripts/examples/`.

Example instance config:

```yaml
instance: main-mastra
stack: instance

backend:
  bucket: advone-east-2-deploy
  key: mastra/instance/main-mastra/terraform.tfstate
  region: us-east-2

tfvars:
  instance_name: main-mastra
  routing_host: main.mastra.internal
  cpu: 256
  memory: 512
  desired_count: 1
  alb_listener_rule_priority: 100
  state_bucket: advone-east-2-deploy
  infra_state_key: mastra/infra/root/terraform.tfstate
  aws_region: us-east-2
  log_level: info
  extra_env: {}
```

### 4.2 Steps

1. (Optional, only first time) Populate secrets so the container can start:
   ```bash
   ./scripts/set-instance-secrets.sh main-mastra      # API keys, platform token
   ./scripts/set-zoho-token.sh desk main-mastra        # Zoho OAuth (if used)
   ```

2. Deploy the instance:
   ```bash
   ./scripts/deploy.sh --config mastra/configs/main-mastra-instance.yml
   ```

   Flags:
   - `--skip-build` — redeploys the existing ECR image (no Docker build/push) for infra-only changes.
   - `--image-tag <tag>` — override the image tag (default: current git SHA).
   - `--auto-approve` — skip the `yes` confirmation.

3. The script:
   - Builds + pushes the Docker image to ECR (unless `--skip-build`),
   - Runs `tofu init` → `plan` → `apply`,
   - Polls the ECS service until it is `RUNNING`/`HEALTHY` (or fails with a circuit-breaker message),
   - Approves Tailscale subnet routes and prints next-step instructions.

4. If the secret still has a placeholder (`REPLACE_ME`) after apply, populate it and force a new deployment:
   ```bash
   aws ecs update-service --cluster advone-mastra-cluster \
     --service advone-mastra-main-mastra --force-new-deployment --region us-east-2
   ```

### 4.3 Instance stack — exact AWS resources (`main-mastra`)

> **State source:** `s3://advone-east-2-deploy/mastra/instance/main-mastra/terraform.tfstate`
> Pull with: `aws s3 cp s3://advone-east-2-deploy/mastra/instance/main-mastra/terraform.tfstate - | jq`

All per-instance resources live in `module.mastra_service` and use the prefix `advone-mastra-main-mastra`.

| Resource | Address | ID / value |
|----------|---------|------------|
| ECS service | `module.mastra_service.aws_ecs_service.mastra` | `arn:aws:ecs:us-east-2:628338935433:service/advone-mastra-cluster/advone-mastra-main-mastra` (name `advone-mastra-main-mastra`) |
| Task definition | `module.mastra_service.aws_ecs_task_definition.mastra` | `advone-mastra-main-mastra` (current rev: `:3` → `arn:aws:ecs:us-east-2:628338935433:task-definition/advone-mastra-main-mastra:3`) |
| Task role | `module.mastra_service.aws_iam_role.task` | `advone-mastra-main-mastra-task-role` |
| Task policy | `module.mastra_service.aws_iam_role_policy.task` | `advone-mastra-main-mastra-task-role:advone-mastra-main-mastra-task-policy` |
| CloudWatch log group | `module.mastra_service.aws_cloudwatch_log_group.task` | `/ecs/advone-mastra-main-mastra` |
| Secrets Manager secret | `module.mastra_service.aws_secretsmanager_secret.instance` | `arn:aws:secretsmanager:us-east-2:628338935433:secret:mastra/main-mastra-G0Dgpu` (name `mastra/main-mastra`) |
| Secret version | `module.mastra_service.aws_secretsmanager_secret_version.instance` | (placeholder until populated) |
| ALB target group | `module.mastra_service.aws_lb_target_group.mastra` | `arn:aws:elasticloadbalancing:us-east-2:628338935433:targetgroup/advone-mastra-main-mastra-tg/b3a9fa0bd771500f` (name `advone-mastra-main-mastra-tg`) |
| ALB listener rule | `module.mastra_service.aws_lb_listener_rule.mastra` | `arn:aws:elasticloadbalancing:us-east-2:628338935433:listener-rule/app/advone-mastra-internal-alb/7fc13c80a8b3a99b/9a95d0c320d60622/1b62b3ac0173aa76` (host `main.mastra.internal`) |
| DynamoDB table | `module.mastra_service.aws_dynamodb_table.mastra` | `advone-mastra-main-mastra` |

> **Read-only dependency:** `terraform_remote_state.infra` reads `s3://advone-east-2-deploy/mastra/infra/root/terraform.tfstate` to import the cluster ARN, VPC, subnets, SGs, ALB listener, execution role, and ECR repo URL. If a remote-state value is missing or stale, fix the **root** stack, not the instance stack.

### 4.4 Adding a new instance

```bash
./scripts/new-instance.sh finance
```

Then:
1. Add `finance` to the root `instance_names` tfvars and re-apply the **root** stack (creates the ECR repo + Lambda allowlist entry).
2. Set `deploy: true` for `finance` in `instances/instanceConfig.yml`.
3. Upload the generated config from `scripts/examples/finance-instance.yml` to S3, ensuring `alb_listener_rule_priority` is unique.
4. `./scripts/set-instance-secrets.sh finance`
5. `./scripts/deploy.sh --config mastra/finance-instance.yml`

---

## 5. Troubleshooting — where to look for each failure

Use this map to jump straight to the right AWS console / CLI when a deploy or runtime issue occurs.

### Deploy fails at `tofu init`
- **Cause:** state lock held, or backend config wrong.
- **Check:** `aws s3 ls s3://advone-east-2-deploy/mastra/ --recursive --region us-east-2` for the state key; check the lockfile object (`*.tflock`) in the same prefix.

### Deploy fails at `tofu plan` / `apply`
- **Cause:** drift, or missing remote-state value.
- **For instance stacks:** the error usually references a root output (e.g. `ecr_repo_urls`, `alb_listener_arn`). Re-apply **root**, then re-run the instance deploy.
- **Drift:** `tofu -chdir=infra/root plan` (or `infra/instance`) and diff against state pulled from S3.

### ECS task won't start / keeps restarting
| Symptom | Where to look | Command |
|---------|---------------|---------|
| Container `ResourceInitializationError: secrets` | Secrets Manager `mastra/<instance>` | `aws secretsmanager get-secret-value --secret-id mastra/main-mastra --region us-east-2 --query SecretString --output text \| jq` |
| Image pull failure | ECR repo `advone-mastra/<instance>` | `aws ecr describe-images --repository-name advone-mastra/main-mastra --region us-east-2` |
| App crash | CloudWatch log group `/ecs/advone-mastra-<instance>` | `aws logs tail /ecs/advone-mastra-main-mastra --follow --region us-east-2` |
| Task unhealthy | ECS task health | `aws ecs describe-tasks --cluster advone-mastra-cluster --tasks <arn> --region us-east-2 --query 'tasks[0].healthStatus'` |
| Deployment stuck / circuit breaker | ECS service deployments | `aws ecs describe-services --cluster advone-mastra-cluster --services advone-mastra-main-mastra --region us-east-2` |

### Studio won't load (see [studio-access.md](studio-access.md) for full steps)
| Symptom | Where to look |
|---------|---------------|
| No Tailscale subnet router online | ECS service `advone-mastra-tailscale`; log group `/ecs/advone-mastra-tailscale` |
| Subnet route not approved | Tailscale admin console → machines → `advone-mastra-subnet-router` → approve `10.30.0.0/16` |
| Proxy connects but times out | ECS task `HEALTHY`? Tail `/ecs/advone-mastra-<instance>` |

### Webhooks not reaching Mastra
| Component | Where to look |
|-----------|---------------|
| Public entry point | API Gateway `6y1fzc2d0l` (URL `https://6y1fzc2d0l.execute-api.us-east-2.amazonaws.com/`); logs `/aws/apigateway/advone-mastra-events` |
| Event Lambda | Functions `advone-mastra-zoho-webhook` / `advone-mastra-generic-webhook` / `advone-mastra-s3-ingest`; logs `/aws/lambda/<fn>` |
| Internal routing | ALB listener rule + target group `advone-mastra-<instance>-tg` on `advone-mastra-internal-alb` |

### Tailscale key rejected
- Re-run `./scripts/set-tailscale-key.sh` with a fresh reusable, pre-authorized, `tag:mastra` key, then approve the `10.30.0.0/16` route at <https://login.tailscale.com/admin/machines>.

### DynamoDB (instance storage)
- Table `advone-mastra-<instance>` — console: <https://us-east-2.console.aws.amazon.com/dynamodb/home?region=us-east-2#tables:selected=advone-mastra-main-mastra>

---

## 6. Useful one-liners

```bash
# Pull and inspect root state
aws s3 cp s3://advone-east-2-deploy/mastra/infra/root/terraform.tfstate - | jq '.resources[].type' | sort | uniq -c

# Pull and inspect an instance state
aws s3 cp s3://advone-east-2-deploy/mastra/instance/main-mastra/terraform.tfstate - | jq

# List all deployable instances
yq '.instances | to_entries[] | select(.value.deploy == true) | .key' instances/instanceConfig.yml

# Show current task definition + image
aws ecs describe-task-definition --task-definition advone-mastra-main-mastra --region us-east-2 \
  --query 'taskDefinition.containerDefinitions[0].image'

# Force a redeploy without rebuilding
aws ecs update-service --cluster advone-mastra-cluster --service advone-mastra-main-mastra \
  --force-new-deployment --region us-east-2
```

---

## 7. Reference

- Instance registry: [`instances/instanceConfig.yml`](../instances/instanceConfig.yml)
- Root infra: [`infra/root/`](../infra/root/) · variables: [`infra/root/variables.tf`](../infra/root/variables.tf)
- Instance infra: [`infra/instance/`](../infra/instance/) · variables: [`infra/instance/variables.tf`](../infra/instance/variables.tf)
- Modules: [`infra/modules/`](../infra/modules/) (`network`, `ecs-cluster`, `alb-internal`, `ecr`, `tailscale`, `api-gateway`, `events-lambda`, `mastra-service`)
- Scripts: see [scripts.md](scripts.md)
- Studio access: see [studio-access.md](studio-access.md)
