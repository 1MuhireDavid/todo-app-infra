# Architecture

Source for the diagrams. Both render on GitHub as-is; paste either block into
<https://mermaid.live> to export SVG or PNG for a slide.

## 1. Network, subnet tiers and security groups

Every arrow is an allowed security group rule. There is no NAT gateway and no
route to the internet from any private subnet — the dashed arrows to the
interface endpoints are the only way out of the VPC.

```mermaid
flowchart TB
    internet([Internet])
    client([Browser])

    subgraph vpc["VPC 10.40.0.0/16 — us-east-1"]
        igw[Internet Gateway]

        subgraph public["Public tier — 10.40.0.0/24, 10.40.1.0/24"]
            alb["Application Load Balancer<br/>sg-alb"]
        end

        subgraph app["App private tier — 10.40.10.0/24, 10.40.11.0/24"]
            ecs["ECS Fargate tasks<br/>sg-ecs<br/>AssignPublicIp DISABLED"]
            vpce["Interface VPC endpoints<br/>sg-vpce<br/>ecr.api · ecr.dkr · logs · sts · secretsmanager"]
        end

        subgraph data["Data private tier — 10.40.20.0/24, 10.40.21.0/24"]
            proxy["RDS Proxy<br/>sg-rdsproxy<br/>RequireTLS"]
            rds[("RDS PostgreSQL<br/>sg-rds<br/>db.t3.micro, encrypted")]
        end

        subgraph cache["Cache private tier — 10.40.30.0/24, 10.40.31.0/24"]
            redis[("ElastiCache Redis<br/>sg-cache<br/>TLS + AUTH, 1 primary + 1 replica")]
        end

        s3gw{{"S3 gateway endpoint<br/>on both private route tables"}}
    end

    sm[["Secrets Manager<br/>RDS-managed master secret<br/>generated Redis AUTH token"]]
    ecr[["ECR"]]
    logs[["CloudWatch Logs"]]

    client --> internet --> igw --> alb
    alb -- "tcp 8080 from sg-alb only" --> ecs
    ecs -- "tcp 5432 from sg-ecs only" --> proxy
    proxy -- "tcp 5432 from sg-rdsproxy only<br/>(sg-ecs is NOT allowed here)" --> rds
    ecs -- "tcp 6379 from sg-ecs only" --> redis
    ecs -. "tcp 443 from sg-ecs" .-> vpce
    proxy -. "tcp 443 from sg-rdsproxy<br/>reads the master secret" .-> vpce
    ecs -. "image layers" .-> s3gw

    vpce -.-> sm
    vpce -.-> ecr
    vpce -.-> logs
    s3gw -.-> ecr
```

### Security group rules in table form

| Group | Ingress | Egress |
|---|---|---|
| `sg-alb` | 80 from `0.0.0.0/0`; 9000 from `TestListenerCidr` | 8080 to `sg-ecs` |
| `sg-ecs` | 8080 from `sg-alb` | 5432 to `sg-rdsproxy`; 6379 to `sg-cache`; 443 to `sg-vpce` |
| `sg-rdsproxy` | 5432 from `sg-ecs` | 5432 to `sg-rds`; 443 to `sg-vpce` |
| `sg-rds` | 5432 from `sg-rdsproxy` **only** | none (`127.0.0.1/32` placeholder) |
| `sg-cache` | 6379 from `sg-ecs`; 6379 from `sg-cache` (replication) | 6379 to `sg-cache` |
| `sg-vpce` | 443 from `sg-ecs`; 443 from `sg-rdsproxy` | none (`127.0.0.1/32` placeholder) |

Every group declares explicit egress. A CloudFormation security group with no
`SecurityGroupEgress` silently gets allow-all to `0.0.0.0/0`, so the two groups
that never initiate a connection carry a `127.0.0.1/32` rule to suppress it.

## 2. CI/CD — two repos, two OIDC roles, one trigger

```mermaid
flowchart LR
    subgraph infra["GitHub: todo-app-infra"]
        icommit([push to main]) --> ilint[cfn-lint]
        ilint --> ihash["hash each nested template<br/>git hash-object | cut -c1-12"]
        ihash --> iupload["upload changed only<br/>(head-object check)"]
        iupload --> iwrite["write hashes into<br/>cfn/deployment-file.yaml"]
        iwrite --> ipush([commit with skip ci])
    end

    subgraph appgh["GitHub: todo-app"]
        acommit([push to main]) --> abuild[docker build]
        abuild --> asha["push sha-short<br/>no trigger"]
        asha --> arender["render taskdef.json<br/>from stack outputs"]
        arender --> azip["zip appspec + taskdef<br/>to artifact bucket"]
        azip --> alatest(["push latest<br/>THE trigger"])
    end

    role1{{"OIDC role: infra packaging<br/>aud + repository + ref + job_workflow_ref"}}
    role2{{"OIDC role: app build<br/>aud + repository + ref + job_workflow_ref"}}

    ilint -.assumes.-> role1
    abuild -.assumes.-> role2

    s3t[(templates bucket)]
    ecrrepo[(ECR)]
    s3a[(artifact bucket)]

    iupload --> s3t
    asha --> ecrrepo
    alatest --> ecrrepo
    azip --> s3a

    ipush --> sync["CloudFormation Git sync"]
    sync --> stack["root stack → 7 nested stacks"]
    s3t --> stack

    ecrrepo --> eb["EventBridge rule<br/>ECR Image Action / PUSH / SUCCESS<br/>filtered to repo + latest"]
    eb --> pipe["CodePipeline<br/>2 source actions, 1 deploy action<br/>S3 source PollForSourceChanges false"]
    s3a --> pipe
    pipe --> cd["CodeDeploy blue/green<br/>ECSAllAtOnce, 10 min blue wait"]
    cd --> svc["ECS service<br/>DeploymentController CODE_DEPLOY"]
```

## 3. Blue/green traffic shift

```mermaid
sequenceDiagram
    participant EB as EventBridge
    participant CP as CodePipeline
    participant CD as CodeDeploy
    participant ALB as ALB listeners
    participant ECS as ECS service

    EB->>CP: latest pushed to ECR
    CP->>CP: S3 source (appspec + taskdef) and ECR source (imageDetail.json)
    CP->>CD: CreateDeployment, IMAGE1_NAME substituted
    CD->>ECS: start green task set on the new revision
    ECS-->>ALB: register green targets on the test listener :9000
    ALB-->>CD: green healthy (2 checks, 15s apart)
    CD->>ALB: shift prod listener :80 to green
    Note over CD,ECS: blue kept for 10 minutes — rollback is a second shift, not a redeploy
    CD->>ECS: terminate blue task set
```
