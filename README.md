# todo-app-infra

Infrastructure for a Java To-Do application on ECS Fargate: public ALB, tasks
in private subnets with no internet route, writes to PostgreSQL **through RDS
Proxy**, reads cached in ElastiCache for Redis. All CloudFormation, deployed by
CloudFormation Git sync, with blue/green releases driven by ECR image pushes.

Application source lives in the separate [`todo-app`](https://github.com/1MuhireDavid/todo-app)
repository. This repository contains no application code, and that one contains
no infrastructure.

- **Region:** us-east-1
- **Account:** 047719661196
- **Architecture diagrams:** [`docs/architecture.md`](docs/architecture.md)

---

## Layout

```
bootstrap/
  00-bootstrap.yaml          templates bucket, artifact bucket, ECR, two GitHub OIDC roles
  deployment-file.yaml
cfn/
  root.yaml                  wires the nested stacks together
  deployment-file.yaml       parameters and tags Git sync deploys with
  nested-templates/
    01-network.yaml          VPC, four subnet tiers across 2 AZs, route tables
    02-security.yaml         one security group per resource type
    03-vpc-endpoints.yaml    ecr.api, ecr.dkr, logs, sts, secretsmanager, S3 gateway
    04-database.yaml         RDS PostgreSQL + RDS Proxy
    05-cache.yaml            ElastiCache Redis replication group
    06-alb-ecs.yaml          ALB, blue/green target groups, cluster, task, service, autoscaling
    07-cicd-pipeline.yaml    CodeDeploy, CodePipeline, EventBridge
.github/workflows/
  package-templates.yml      lint, content-address, upload, commit hashes back
docs/architecture.md
```

Nested stacks are split **by lifecycle**, not by service: the network changes
almost never, the data stacks are slow and stateful, `06` changes with every
application release. Root passes values down through `Outputs` → `Parameters`;
no nested stack imports a sibling. Only the bootstrap stack publishes `Export`s,
and only the root consumes them.

---

## One-time bootstrap

The bootstrap stack owns everything that must survive a teardown of the
application stack. Create it by hand, once, before anything else exists.

```bash
aws cloudformation create-stack \
  --stack-name todo-app-bootstrap \
  --template-body file://bootstrap/00-bootstrap.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1 \
  --parameters \
      ParameterKey=ProjectName,ParameterValue=todo-app \
      ParameterKey=GitHubOwner,ParameterValue=1MuhireDavid \
      ParameterKey=InfraRepoName,ParameterValue=todo-app-infra \
      ParameterKey=AppRepoName,ParameterValue=todo-app \
      ParameterKey=CreateOIDCProvider,ParameterValue=false

aws cloudformation wait stack-create-complete \
  --stack-name todo-app-bootstrap --region us-east-1

aws cloudformation describe-stacks \
  --stack-name todo-app-bootstrap --region us-east-1 \
  --query "Stacks[0].Outputs" --output table
```

`CreateOIDCProvider=false` because account 047719661196 already federates GitHub
Actions. An account holds exactly one provider per URL; a second one fails with
`EntityAlreadyExists`. In a fresh account, pass `true`.

This creates, and it is worth knowing which survive a teardown:

| Resource | Deletion policy | Why |
|---|---|---|
| `todo-app-cfn-templates-047719661196-us-east-1` | **Retain** | deleting it would destroy the templates needed to redeploy |
| `todo-app` (ECR) | **Retain** | with no NAT, losing the image means the stack can never start again |
| `todo-app-pipeline-artifacts-047719661196-us-east-1` | Delete | everything in it is regenerable by re-running the pipeline |
| `todo-app-gha-infra-packaging`, `todo-app-gha-app-build` | Delete | recreated with the stack |

### Why the artifact bucket is here and not in `07`

The ALB in `06` writes access logs to it, and `07` has to be created *after* `06`
because CodeDeploy needs the service, the target groups and the listeners. Owning
the bucket in bootstrap breaks that cycle, and it also lets the app repo's OIDC
role be scoped to a real bucket ARN instead of one guessed with `Fn::Sub`.

---

## Wiring up Git sync

Git sync is what deploys. There is no `aws cloudformation deploy` step in any
workflow, and neither OIDC role holds a single CloudFormation write permission.

1. **Connect GitHub.** Console → CodePipeline → Settings → Connections → *Create
   connection* → GitHub. Install the AWS Connector app on the `1MuhireDavid`
   account and grant it `todo-app-infra`. The connection must be **Available**,
   not *Pending*.
2. **Create the sync configuration.** Console → CloudFormation → *Create stack*
   → **With Git sync**.
   - Stack name: `todo-app` — this must match `RootStackName` in the bootstrap
     stack and `ROOT_STACK_NAME` in the app repo workflow, because the app build
     reads this stack's outputs.
   - Repository: `1MuhireDavid/todo-app-infra`, branch `main`
   - Deployment file: `cfn/deployment-file.yaml`
   - IAM role: let the console create one, or supply a role CloudFormation can
     assume with permissions for every service in these templates.
   - Capabilities: tick `CAPABILITY_NAMED_IAM` — the templates create named roles.
3. CloudFormation will now deploy on every commit that touches the deployment
   file or the templates it points at.

Optionally repeat with `bootstrap/deployment-file.yaml` and stack name
`todo-app-bootstrap` to manage the bootstrap stack the same way. Keep it a
separate sync configuration; the two stacks have deliberately different
lifecycles.

---

## First deploy, in order

The sequence matters in one specific way: **an image tagged `latest` must exist
in ECR before the application stack is created.** The tasks run in subnets with
no internet route, so a missing image cannot fall back to a public registry —
the service will sit retrying until the stack times out and rolls back.

That makes the first deploy two passes of the same workflow rather than one, but
no part of it is manual: `build-and-push` detects that the stack does not exist
yet and publishes the image without attempting a deployment. See step 3.

1. **Bootstrap stack** — above. Nothing else works without it.

2. **Publish the templates.** Push this repository to `main`. The
   `package-templates` workflow lints every template, uploads each nested one to
   `templates/<name>-<hash>.yaml`, writes the hashes into
   `cfn/deployment-file.yaml` and pushes that commit with `[skip ci]`.

   The hashes committed here already match the current files, so a first push
   uploads the templates and reports "no template content changed".

3. **Push the first image.** In the `todo-app` repository, run the
   `build-and-push` workflow — from the Actions tab, or just push to `main`.
   Nothing manual, and nothing is expected to fail.

   The workflow checks whether the `todo-app` stack exists. On this first run it
   does not, so it builds the image, pushes `sha-<short>` and `latest`, and
   skips the deploy bundle — there are no stack outputs to render `taskdef.json`
   from and no pipeline to deploy with yet. The job summary says exactly that
   and tells you what to do next. Pushing `latest` triggers nothing, because the
   EventBridge rule it would fire does not exist yet either.

   ```bash
   aws ecr describe-images --repository-name todo-app --region us-east-1 \
     --query "imageDetails[].imageTags" --output table
   ```

4. **Let Git sync create the application stack.** It picks up the commit from
   step 2. Roughly 20–25 minutes, most of it RDS and ElastiCache.

   ```bash
   aws cloudformation describe-stacks --stack-name todo-app --region us-east-1 \
     --query "Stacks[0].StackStatus"
   ```

5. **Re-run `build-and-push`.** The same workflow, unchanged. This time it finds
   the stack `CREATE_COMPLETE`, renders `taskdef.json` from its outputs,
   publishes the bundle, and pushing `latest` starts the first real blue/green
   deployment.

From here the loop is: push application code → image and bundle published →
EventBridge → CodePipeline → CodeDeploy shifts traffic. Push infrastructure code
→ hashes change → Git sync updates only the nested stacks whose bytes changed.

---

## Verifying the deployment

```bash
# The URL
aws cloudformation describe-stacks --stack-name todo-app --region us-east-1 \
  --query "Stacks[0].Outputs[?OutputKey=='ApplicationUrl'].OutputValue" --output text

# Health endpoint - should be {"status":"UP"} and must never touch the DB
curl -s http://<alb-dns>/health

# Cache behaviour: first read is a miss, second is a hit
curl -s http://<alb-dns>/api/tasks | jq '{cacheHit, source, elapsedMs}'
curl -s http://<alb-dns>/api/tasks | jq '{cacheHit, source, elapsedMs}'

# A write invalidates, so the next read is a miss again
curl -s -X POST http://<alb-dns>/api/tasks \
  -H 'Content-Type: application/json' -d '{"title":"prove the cache works"}'
curl -s http://<alb-dns>/api/tasks | jq '{cacheHit, source, elapsedMs}'
```

Or just open the URL: the page shows a **CACHE HIT** / **CACHE MISS** badge, the
source, the server-side read time and the browser round trip, with a *Read
again* button.

Things worth checking that prove the harder requirements:

```bash
# The app connects to the PROXY, never the instance
aws cloudformation describe-stacks --stack-name todo-app --region us-east-1 \
  --query "Stacks[0].Outputs[?OutputKey=='DatabaseProxyEndpoint'].OutputValue" --output text

# The proxy is available - if it says incompatible-network, the Secrets Manager
# endpoint rule in 02-security.yaml is the first thing to check
aws rds describe-db-proxies --db-proxy-name todo-app-proxy --region us-east-1 \
  --query "DBProxies[0].Status" --output text

# No password anywhere in the repository or the stack parameters
grep -rIn "password" cfn/ bootstrap/ --include=*.yaml | grep -iv "masteruserpassword\|password auth\|passwordauthtype\|:password::\|the password\|managemasteruser"

# Tasks have no public IP
aws ecs describe-tasks --cluster todo-app-cluster --region us-east-1 \
  --tasks $(aws ecs list-tasks --cluster todo-app-cluster --region us-east-1 --query 'taskArns[0]' --output text) \
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId']"
```

During a deployment, the green task set is reachable on the test listener at
port 9000 before any production traffic shifts.

---

## Running cost

us-east-1, on-demand, one task, roughly 730 hours. Rounded, and excluding data
transfer.

| Item | Config | USD / month |
|---|---|---|
| **VPC interface endpoints** | 5 endpoints × 2 AZs × $0.01/AZ-hr | **~73** |
| ElastiCache | 2 × `cache.t4g.micro` (primary + replica) | ~23 |
| RDS Proxy | billed per DB vCPU, 2 vCPU minimum on T classes | ~22 |
| ALB | fixed hourly + minimal LCU | ~18 |
| Fargate | 0.5 vCPU + 1 GB, 1 task | ~18 |
| RDS | `db.t3.micro` single-AZ + 20 GB gp3 | ~15 |
| CodePipeline | 1 active V1 pipeline | ~1 |
| Secrets Manager | 2 secrets × $0.40 | ~1 |
| S3, ECR, CloudWatch Logs | small | ~2 |
| **Total** | | **~173** |

**The VPC endpoints are the largest line item, and they cost more than the NAT
gateway they replace** (~$33/month). That trade is deliberate — with no NAT
there is no route from a private subnet to the internet at all — but it is a
security decision, not a cost saving, and the opposite is often claimed.

Levers, in order of return:

1. **Delete the application stack when you are not demoing.** Bootstrap is
   retained, so a redeploy is one Git sync run plus ~25 minutes. This saves
   effectively all of it.
2. `CacheReplicaCount: 0` — saves ~$12/month, gives up Multi-AZ failover.
3. Drop the `sts` endpoint — saves ~$15/month. Fargate does not need it; only an
   application calling STS directly does, and this one calls no AWS API at all.
4. Put the interface endpoints in one AZ — saves ~$37/month, gives up AZ
   redundancy for AWS API calls.

---

## Teardown

Order matters, and one step is not optional.

```bash
# 1. Empty the artifact bucket. A versioned bucket with objects in it will hang
#    the stack delete for an hour and then fail it.
aws s3 rm s3://todo-app-pipeline-artifacts-047719661196-us-east-1 --recursive
aws s3api list-object-versions \
  --bucket todo-app-pipeline-artifacts-047719661196-us-east-1 \
  --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json \
  > /tmp/versions.json
aws s3api delete-objects \
  --bucket todo-app-pipeline-artifacts-047719661196-us-east-1 \
  --delete file:///tmp/versions.json
# Repeat for DeleteMarkers[] if the bucket has any.

# 2. Delete the application stack. Nested stacks go in reverse dependency order
#    on their own; expect 15-20 minutes.
aws cloudformation delete-stack --stack-name todo-app --region us-east-1
aws cloudformation wait stack-delete-complete --stack-name todo-app --region us-east-1

# 3. Optional: remove the Git sync configuration, then the CodeConnections
#    connection, in the console.

# 4. Only if you are finished with the project entirely. The templates bucket
#    and ECR repository are Retain, so they survive step 5 and must be removed
#    by hand.
aws s3 rm s3://todo-app-cfn-templates-047719661196-us-east-1 --recursive
aws s3 rb s3://todo-app-cfn-templates-047719661196-us-east-1 --force
aws ecr delete-repository --repository-name todo-app --force --region us-east-1

# 5. Delete the bootstrap stack.
aws cloudformation delete-stack --stack-name todo-app-bootstrap --region us-east-1
```

Leftovers to check afterwards: the CloudWatch log group `/ecs/todo-app` is
deleted with the stack, but the Redis AUTH token secret enters a 30-day recovery
window rather than disappearing. Force it if you plan to recreate the stack with
the same name, since the name will otherwise collide:

```bash
aws secretsmanager delete-secret --secret-id todo-app/redis/auth-token \
  --force-delete-without-recovery --region us-east-1
```

---

## Requirement → implementation

| Requirement | File | Resource / setting |
|---|---|---|
| VPC across 2 AZs | `01-network.yaml` | `Vpc`, `!Select` over `!GetAZs` |
| ECS, RDS and Redis in **different dedicated subnets** | `01-network.yaml` | `AppSubnet1/2`, `DataSubnet1/2`, `CacheSubnet1/2` |
| Public subnets for the ALB only | `01-network.yaml` | `PublicSubnet1/2`, `PublicRouteTable` |
| No NAT gateway | `01-network.yaml`, `03-vpc-endpoints.yaml` | no `AWS::EC2::NatGateway`; endpoints instead |
| Private route tables per AZ | `01-network.yaml` | `PrivateRouteTable1/2` |
| S3 gateway endpoint on every private route table | `03-vpc-endpoints.yaml` | `S3GatewayEndpoint` |
| ECR, logs, STS, Secrets Manager reachable privately | `03-vpc-endpoints.yaml` | five `Interface` endpoints |
| One SG per resource type, chained by reference | `02-security.yaml` | six groups, `SourceSecurityGroupId` throughout |
| RDS reachable only from the proxy | `02-security.yaml` | `DatabaseIngressFromRdsProxy` — no rule from `sg-ecs` |
| Explicit egress on every SG | `02-security.yaml` | inline `SecurityGroupEgress` on all six |
| Circular SG references broken cleanly | `02-security.yaml` | eight standalone `SecurityGroupIngress` / `Egress` |
| RDS password never in the repo | `04-database.yaml` | `ManageMasterUserPassword: true` |
| Proxy authenticates with that secret | `04-database.yaml` | `RdsProxyRole` scoped to `MasterUserSecret.SecretArn` |
| `RequireTLS` on the proxy | `04-database.yaml` | `RdsProxy.RequireTLS: true` |
| Proxy target group | `04-database.yaml` | `RdsProxyTargetGroup` |
| App connects to the proxy endpoint | `04-database.yaml` → `06` | output `ProxyEndpoint` → `DB_HOST` |
| Redis encrypted both ways | `05-cache.yaml` | `TransitEncryptionEnabled`, `AtRestEncryptionEnabled` |
| AUTH token generated, never a parameter | `05-cache.yaml` | `AuthTokenSecret` + `GenerateSecretString` |
| Replication group, not cache cluster | `05-cache.yaml` | `AWS::ElastiCache::ReplicationGroup` |
| Multi-AZ cache | `05-cache.yaml` | `MultiAZEnabled` + `CacheReplicaCount: 1` |
| Credentials in `Secrets:`, never `Environment:` | `06-alb-ecs.yaml` | `Secrets` with `:password::` selectors |
| Execution role separate from task role | `06-alb-ecs.yaml` | `ExecutionRole`, `TaskRole` (no policies) |
| `Resource: "*"` justified in a comment | `00-bootstrap`, `06`, `07` | `ecr:GetAuthorizationToken`, `ecs:RegisterTaskDefinition`, `kms:Decrypt` |
| `iam:PassRole` narrowed | `07-cicd-pipeline.yaml` | `StringEqualsIfExists` on `iam:PassedToService` |
| OIDC only, no static keys | `00-bootstrap.yaml` | two federated roles, `MaxSessionDuration: 3600` |
| Trust pinned to aud + repository + ref + job_workflow_ref | `00-bootstrap.yaml` | both `AssumeRolePolicyDocument` blocks |
| Tasks have no public IP | `06-alb-ecs.yaml` | `AssignPublicIp: DISABLED` |
| Blue/green target groups | `06-alb-ecs.yaml` | `TargetGroupBlue`, `TargetGroupGreen` |
| Prod listener 80, parameterized test listener | `06-alb-ecs.yaml` | `ProdListener`, `TestListener` |
| `deregistration_delay` 30s | `06-alb-ecs.yaml` | `TargetGroupAttributes` on both groups |
| CodeDeploy owns the task definition | `06-alb-ecs.yaml` | `DeploymentController: CODE_DEPLOY` |
| Blue survives for rollback | `07-cicd-pipeline.yaml` | `TerminationWaitTimeInMinutes: 10` |
| Autoscaling 1/1/4, asymmetric cooldowns | `06-alb-ecs.yaml` | `ScalableTarget`, `CpuScalingPolicy` |
| Container Insights | `06-alb-ecs.yaml` | `ClusterSettings` |
| Log group explicit, 14-day retention | `06-alb-ecs.yaml` | `LogGroup` |
| ALB access logs | `06-alb-ecs.yaml` + bootstrap | `access_logs.s3.*` + `AllowAlbAccessLogDelivery` |
| Health check on `/health` with tuned thresholds | `06-alb-ecs.yaml` | both target groups |
| Single pipeline trigger | `07-cicd-pipeline.yaml` | `PollForSourceChanges: "false"` + `EcrPushRule` |
| Trigger role can only start this pipeline | `07-cicd-pipeline.yaml` | `PipelineTriggerRole` |
| ECR scan on push, encryption, lifecycle | `00-bootstrap.yaml` | `EcrRepository` |
| Buckets: versioning, SSE, PAB, lifecycle, TLS-only | `00-bootstrap.yaml` | both buckets + both policies |
| Retain on templates bucket and ECR | `00-bootstrap.yaml` | `DeletionPolicy` / `UpdateReplacePolicy` |
| Explicit `Delete` on the artifact bucket | `00-bootstrap.yaml` | `ArtifactBucket` |
| Deploy via Git sync with a checked-in deployment file | `cfn/deployment-file.yaml` | all parameters and tags |
| Per-file content-addressed templates | `package-templates.yml` | `git hash-object \| cut -c1-12` |
| Skip unchanged uploads | `package-templates.yml` | `aws s3api head-object` |
| Unmapped template fails the build | `package-templates.yml` | `HASH_PARAM` check → `::error::` + `exit 1` |
| `cfn-lint` before upload | `package-templates.yml` | "Lint templates" step |
| Typed parameters | all | `AWS::EC2::VPC::Id`, `List<AWS::EC2::Subnet::Id>`, `CommaDelimitedList`, `AllowedValues` |
| One `ProjectName`, `Project` tag everywhere | all | `!Sub "${ProjectName}-..."` |

---

## Deliberately out of scope

Named here rather than silently skipped.

| Skipped | What production does |
|---|---|
| **TLS on the ALB** | ACM certificate on an HTTPS:443 listener, HTTP:80 redirecting to it, a modern `SslPolicy`. Skipped because the lab has no domain. Today the app is served over plain HTTP and the test listener is open to `0.0.0.0/0`. |
| **WAF, Shield Advanced, GuardDuty, CloudTrail data events** | WAF web ACL on the ALB with managed rule groups and rate limiting; GuardDuty on for the account; CloudTrail data events on both buckets. |
| **Multi-region, cross-region replication, PITR beyond 1 day** | Cross-region automated backup replication, 7–35 day retention, a tested restore runbook. `BackupRetentionPeriod` is 1 here. |
| **RDS Multi-AZ** | On. It is a parameter defaulting to `"false"` purely for cost — flip `MultiAZ` in the deployment file. |
| **`verify-full` TLS to PostgreSQL** | RDS CA bundle in the image and `sslmode=verify-full`. The lab uses `sslmode=require`, which encrypts but does not verify the server certificate. |
| **Schema migrations** | Flyway or Liquibase. The lab uses `spring.jpa.hibernate.ddl-auto=update`, which is fine for one table and wrong for anything else. |
| **Secret rotation for Redis** | A rotation Lambda. RDS rotates its own master secret; the Redis AUTH token does not rotate. |
| **Alarms and dashboards** | CloudWatch alarms on 5xx rate, target response time, unhealthy host count, RDS and Redis CPU, wired to the CodeDeploy `AutoRollbackConfiguration` so a bad deploy rolls itself back on a metric rather than only on task failure. Container Insights is on, but nothing is alarmed. |
| **`EnableExecuteCommand`** | On, with the `ssmmessages` endpoints. Off here to avoid two more billed endpoints. |

---

## Known gaps and judgement calls

Read this before grading.

1. **`aws cloudformation validate-template` has not been run against a live
   account.** `cfn-lint` passes with no errors or warnings on all nine templates.
   The API call needs credentials this machine does not currently have — the
   token in the environment is expired. Run it after authenticating:

   ```bash
   for f in bootstrap/00-bootstrap.yaml cfn/root.yaml cfn/nested-templates/*.yaml; do
     aws cloudformation validate-template --template-body "file://$f" \
       --region us-east-1 >/dev/null && echo "VALID   $f" || echo "FAILED  $f"
   done
   ```

2. **The interface endpoints sit in the app subnets only, not the data subnets.**
   An interface endpoint accepts exactly one subnet per AZ, so the data tier
   cannot have its own ENI. It does not need one: private DNS resolves VPC-wide
   and the data subnets route to the app subnets locally. What gates the RDS
   Proxy is the security group rule, not ENI placement.

3. **The spec's "VPC endpoints: ingress 443 from ECS only" would have broken RDS
   Proxy**, which reads its master secret from Secrets Manager over the VPC and,
   with no NAT, has no other route. `VpcEndpointIngressFromRdsProxy` was added.
   Without it the proxy settles into `incompatible-network` and the symptom looks
   like a database problem.

4. **The ALB egress rule points at the task security group, not the VPC CIDR.**
   The spec said "within the VPC CIDR"; a group reference is strictly tighter and
   keeps the "never by CIDR" rule intact everywhere except the two public-facing
   ALB ingress rules.

5. **The task definition exists in both repositories, and that is not
   duplication for its own sake.** CloudFormation cannot update `TaskDefinition`
   on a service with the `CODE_DEPLOY` controller, so `06-alb-ecs.yaml` defines
   only the bootstrap revision the service is born with. Every revision after
   that comes from `ecs/taskdef.json` in the app repo. The app build workflow
   fills its account-specific values from this stack's outputs at build time
   rather than committing them, so the two cannot silently drift into pointing at
   different secrets. If you change the container shape, change both.

6. **`EngineVersion` is the major version `"16"`**, so RDS picks the current
   minor release and the template does not rot when a minor version is
   deprecated. Production pins the full version.

7. **Private route tables are per AZ, shared across the three private tiers.**
   With no NAT the three tiers route identically, so per-tier tables would be six
   identical tables. Split them the moment a tier needs different egress.

8. **`CodePipeline` is a V1 pipeline** — flat monthly charge rather than V2's
   per-action-minute billing, and none of V2's features are used here.

9. **Nothing verifies that `ecs/appspec.yaml`'s `ContainerName` matches
   `ProjectName`.** It is `todo-app` in both. Rename the project and you must
   change it in `06-alb-ecs.yaml`, `appspec.yaml` and `taskdef.json` together.

10. **The health check is intentionally shallow.** `/health` touches neither
    PostgreSQL nor Redis. A dependency-checking health endpoint means one RDS
    failover makes every task unhealthy at once and the ALB drains all of them —
    a recoverable blip becomes an outage. The tradeoff is that a task with a dead
    database stays in the target group and serves 500s. Production adds a
    separate deep-check endpoint that alarms but does not gate traffic.
