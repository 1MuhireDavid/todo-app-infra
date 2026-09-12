# IAM roles for deploying the application stack

Two roles, created **before** the Git sync configuration, because the console
wizard asks for both and there is no way to add them afterwards without editing
the sync config.

| Role | Assumed by | Does what |
|---|---|---|
| `todo-app-cfn-exec-app` | `cloudformation.amazonaws.com` | creates every resource in the root stack and its seven nested stacks |
| `todo-app-git-sync` | `cloudformation.sync.codeconnections.amazonaws.com` | reads the repo through the CodeConnections connection and drives change sets |

They are separate on purpose. The sync role only needs to *start* a deployment;
the execution role is what actually holds VPC, RDS and IAM permissions. Merging
them would hand the repository-reading service the ability to create IAM roles.

---

## 1. Execution role — `todo-app-cfn-exec-app`

### Console

1. IAM → Roles → **Create role** → **Custom trust policy**
2. Paste [`trust-policy.json`](trust-policy.json)
3. Skip the permissions page, name it `todo-app-cfn-exec-app`, create
4. Open it → **Add permissions** → **Create inline policy** → JSON → paste
   [`permissions-policy-1-network-compute.json`](permissions-policy-1-network-compute.json),
   name it `todo-app-cfn-exec-app-1`
5. Repeat for
   [`permissions-policy-2-data-delivery.json`](permissions-policy-2-data-delivery.json)
   as `todo-app-cfn-exec-app-2`

**Two policies, not one, because of an IAM limit**, not for tidiness: a customer
managed policy caps at 6,144 characters and the combined policy is about 7,900.
Splitting by lifecycle — network and compute, then data and delivery — keeps each
side independently readable.

### CLI

```bash
aws iam create-role \
  --role-name todo-app-cfn-exec-app \
  --assume-role-policy-document file://iam/trust-policy.json \
  --max-session-duration 3600 \
  --tags Key=Project,Value=todo-app

aws iam put-role-policy --role-name todo-app-cfn-exec-app \
  --policy-name todo-app-cfn-exec-app-1 \
  --policy-document file://iam/permissions-policy-1-network-compute.json

aws iam put-role-policy --role-name todo-app-cfn-exec-app \
  --policy-name todo-app-cfn-exec-app-2 \
  --policy-document file://iam/permissions-policy-2-data-delivery.json
```

### The trust policy, and the nested stack problem

```
arn:aws:cloudformation:us-east-1:047719661196:stack/todo-app/*
arn:aws:cloudformation:us-east-1:047719661196:stack/todo-app-*Stack-*/*
```

The second line is the one people miss. A nested stack is a stack in its own
right, with a generated name like `todo-app-NetworkStack-1A2B3C4D5E6F`, and it
assumes the same execution role. An `aws:SourceArn` condition matching only
`stack/todo-app/*` passes for the root and then fails on the first nested stack
with an unhelpful `AccessDenied`.

Every nested logical ID in `root.yaml` ends in `Stack`, so `todo-app-*Stack-*`
matches all seven and deliberately does **not** match `todo-app-bootstrap` — the
bootstrap stack has its own execution role, in its own repository.

### Where `Resource: "*"` appears, and why

Resource-scoped: nested stacks, the templates bucket prefix, the Redis secret,
every `todo-app-*` role, the log group, the CodeDeploy application and group, the
pipeline, the EventBridge rule.

`Resource: "*"` in five statements, in each case because the API genuinely takes
no resource ARN:

| Statement | Why |
|---|---|
| `NetworkingNoResourceArnOnCreate` | `ec2:CreateVpc`, `CreateSubnet`, `CreateSecurityGroup` and friends create the resource being named — there is nothing to scope to at call time |
| `LoadBalancing` | same, `CreateLoadBalancer` and `CreateTargetGroup` |
| `EcsClusterServiceAndTasks` | `ecs:RegisterTaskDefinition` and `CreateCluster` take no ARN |
| `Autoscaling` | Application Auto Scaling supports no resource-level permissions at all |
| `GenerateSecretStringHasNoResource` | `secretsmanager:GetRandomPassword` is an account-level call |
| `ServiceLinkedRolesOnFirstUse` | constrained by an `iam:AWSServiceName` condition instead, listing the five services |

RDS and ElastiCache are also `"*"`: their `Create*` calls do accept ARNs, but the
ARN of a database that does not exist yet cannot be predicted before
CloudFormation names it. Tightening those means switching to a tag-based
condition, which is worth doing in production and is noise for a lab.

`PassRolesToTheServicesThatUseThem` is scoped to `role/todo-app-*` **and**
conditioned on `iam:PassedToService`, so these roles can only be handed to the
five services that legitimately receive them.

---

## 2. Git sync role — `todo-app-git-sync`

```bash
aws iam create-role \
  --role-name todo-app-git-sync \
  --assume-role-policy-document file://iam/git-sync-role-trust-policy.json \
  --tags Key=Project,Value=todo-app

aws iam put-role-policy --role-name todo-app-git-sync \
  --policy-name todo-app-git-sync-policy \
  --policy-document file://iam/git-sync-role-permissions-policy.json
```

**If the console rejects the trust principal, let the wizard create this role
instead.** The service principal for Git sync is version-specific and it is the
one place here where a generated role is the safer bet — it is a narrow role, and
getting the principal wrong produces a sync configuration that silently never
syncs. Everything else on this page is worth creating yourself.

After creating the CodeConnections connection, tighten the connection wildcard in
`git-sync-role-permissions-policy.json` to the real ARN:

```bash
aws codeconnections list-connections --region us-east-1 \
  --query "Connections[].{Name:ConnectionName,Arn:ConnectionArn,Status:ConnectionStatus}" \
  --output table
```

---

## Order

1. Create both roles here, and `todo-app-cfn-exec-bootstrap` in the
   `todo-app-bootstrap` repository
2. Deploy the bootstrap stack with its execution role
3. Create the CodeConnections connection, confirm it reads **Available**
4. Create the Git sync configuration: stack name `todo-app`, deployment file
   `cfn/deployment-file.yaml`, sync role `todo-app-git-sync`, execution role
   `todo-app-cfn-exec-app`, capability `CAPABILITY_NAMED_IAM`

Step 4 is where a missing role costs you the most time, because the wizard
validates late.

---

## The lab shortcut, and when it is defensible

Attaching `AdministratorAccess` to a CloudFormation execution role in a throwaway
sandbox is a real option and not automatically wrong — the stack can only create
what its template says, and the role is assumable only by CloudFormation for one
named stack.

It is wrong here for one reason: least-privilege IAM is part of what this lab is
demonstrating, and "the execution role is admin" undoes the argument made by
every scoped policy in the templates. Use these policies. When one denies
something, add the specific action rather than widening the resource.
