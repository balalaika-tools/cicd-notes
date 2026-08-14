# Provisioning the Reference Platform

> **Who this is for**: Engineers standing up the AWS and GitHub resources the end-to-end production example assumes already exist — the ECR repository, the staging and production ECS clusters and services, the three per-environment OIDC deployment roles, and the first task-definition revision.

## The short version

The end-to-end walkthrough's first command is `docker buildx build --push -t $IMAGE:git-$GITHUB_SHA .` — that only succeeds if `$IMAGE`'s registry already exists, already refuses to let a pushed tag get silently overwritten, and already trusts the workflow's OIDC (OpenID Connect — a signed workload-identity token GitHub issues per job) token to push to it. Nowhere else in this collection shows where that registry — Amazon ECR (Elastic Container Registry, AWS's managed Docker registry) — the two Amazon ECS (Elastic Container Service, AWS's container orchestrator) clusters it deploys to, or the three IAM (Identity and Access Management — the cloud platform's system for what an identity may do) roles that push and deploy actually come from. This note provisions all four with Terraform: one ECR repository, two ECS clusters with one `orders` service each, three narrowly-scoped IAM roles, and the first task-definition revision — the one `register-task-definition.sh` later mutates into revision 183.

**What you need (3 things):**

1. Terraform with the AWS provider already authenticated for account `123456789012` in `eu-west-1` — the same account and region the rest of this platform lives in.
2. A repository name matching what the release workflow already pushes to — `orders`.
3. Tag immutability turned on, so a pushed tag can never be silently overwritten by a later push.

**The code:**

```hcl
terraform {
  required_version = ">= 1.5"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
provider "aws" {
  region = "eu-west-1"
}

resource "aws_ecr_repository" "orders" {
  name                 = "orders"
  image_tag_mutability = "IMMUTABLE"
}

output "repository_uri" {
  value = aws_ecr_repository.orders.repository_url
}
```

**Success signal:** `terraform apply` prints `Apply complete! Resources: 1 added, 0 changed, 0 destroyed`, and the `repository_uri` output reads `123456789012.dkr.ecr.eu-west-1.amazonaws.com/orders` — the exact registry string the release workflow in the end-to-end example already pushes to.

**Not handled yet:** [expiring images nobody promoted](#hardening-expire-what-nobody-promoted), [the two ECS clusters and the `orders` service in each](#2-two-ecs-clusters-and-the-orders-service-each-one-deploys), [the three IAM roles that push and deploy](#3-three-iam-roles-creating-the-resource-not-the-trust-boundary), and [the first task-definition revision](#4-task-definition-revision-1-before-183-can-exist).

---

For how this Terraform actually gets reviewed and applied — speculative plans, saved-plan custody, state locking — see [Infrastructure and Database Changes](../04_delivery_operations/03_infrastructure_and_database_changes.md). This note owns the shape of what gets created; that one owns the pipeline that creates it.

## 1. The ECR Repository the Release Workflow Already Assumes

`docker buildx build --push` needs somewhere to push to, and the somewhere needs to already agree with the rest of the pipeline on two things: image-tag immutability (so promoting by digest — the content hash the registry returns for a push — actually means something) and a name the trusted build job's role is allowed to reach. The baseline above creates exactly that: one repository, `orders`, with `image_tag_mutability = "IMMUTABLE"` so a second `docker push` of `git-a1b2c3d` fails loudly instead of quietly replacing what staging already verified.

> **Core:** every resource in this note exists to satisfy an exact-name lookup the end-to-end pipeline already performs — the repository named `orders`, an `orders` service in a `staging` and a `production` cluster, and three role names matching the `vars.AWS_ROLE_ARN` each environment reads. Get the names right and that pipeline needs nothing else from this note.

### Hardening: Expire What Nobody Promoted

The baseline repository keeps every image forever. A build that never gets promoted — a failed PR-triggered rebuild, an abandoned branch's release attempt — still leaves a layer set sitting in ECR, and storage cost grows with every commit to `main`, not just every release. An **ECR lifecycle policy** — a set of rules ECR evaluates on a schedule to expire images automatically — bounds that without deleting anything a task definition still points to:

```hcl
resource "aws_ecr_lifecycle_policy" "orders" {
  repository = aws_ecr_repository.orders.name

  policy = jsonencode({
    rules = [
      {
        rulePriority = 1
        description  = "Expire untagged images after 14 days"
        selection = {
          tagStatus   = "untagged"
          countType   = "sinceImagePushed"
          countUnit   = "days"
          countNumber = 14
        }
        action = { type = "expire" }
      }
    ]
  })
}
```

> **Production:** this rule only ever matches images with zero tags pointing at them — every image this pipeline pushes keeps its `git-<sha>` tag for good, so nothing the pipeline produces is ever "untagged." [§5](#5-what-breaks-first-and-when-not-to-provision-this-way) covers what happens if a later rule is written more broadly.

**How you know it's working:** `aws ecr get-lifecycle-policy --repository-name orders` returns the JSON above. The silent failure is applying a lifecycle policy to the wrong repository name after a rename — Terraform reports success either way, since `aws_ecr_lifecycle_policy` only checks that some repository by that name exists, not that it's the one you meant.

---

## 2. Two ECS Clusters, and the `orders` Service Each One Deploys

The end-to-end example's deploy step is `aws ecs update-service --cluster staging --service orders --task-definition orders:183` — an *update*, not a create. Something has to create the `staging` cluster, the `production` cluster, and an `orders` service in each before that command has anything to target; run it against a cluster or service that doesn't exist yet and ECS returns `ClusterNotFoundException` or `ServiceNotFoundException`, not a helpful "creating it now."

```hcl
resource "aws_ecs_cluster" "this" {
  for_each = toset(["staging", "production"])
  name     = each.value

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}
```

Each cluster gets one `orders` service, sized as a starting baseline rather than a production target — one task, on **Fargate** (AWS's serverless compute for containers; no EC2 instance to patch or size):

```hcl
resource "aws_ecs_service" "orders" {
  for_each        = aws_ecs_cluster.this
  name            = "orders"
  cluster         = each.value.arn
  task_definition = aws_ecs_task_definition.orders.arn
  desired_count   = 1
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = var.subnet_ids
    security_groups  = var.security_group_ids
    assign_public_ip = true
  }
}
```

`var.subnet_ids` and `var.security_group_ids` are left as required input variables on purpose — which VPC, subnets, and security groups the tasks land in is a networking decision this note doesn't own, and hardcoding a placeholder VPC ID here would be more likely to get copied into a real environment than a variable with no default ever would.

**Success signal:** `aws ecs describe-services --cluster staging --services orders --query 'services[0].{status:status,running:runningCount,desired:desiredCount}'` reports `"status": "ACTIVE"` and `running` climbing to match `desired`. If `runningCount` stays at `0` while `status` is `ACTIVE`, the task is failing to start — check `aws ecs describe-tasks` for a `stoppedReason` before assuming the service itself is broken.

> **Production:** circuit-breaker rollback, deployment alarms, and rolling-vs-blue-green strategy are deployment-time behavior, not provisioning — see [Verification, Observability, and Rollback](../04_delivery_operations/04_verification_observability_and_rollback.md) and [Deployment Strategies](../04_delivery_operations/02_deployment_strategies.md) for those. Provisioning only needs the service to exist and reach steady state once.

---

## 3. Three IAM Roles: Creating the Resource, Not the Trust Boundary

OIDC federation only works if the role it's meant to hand credentials to already exists — assume a role that was never created and `configure-aws-credentials` fails before it ever evaluates a trust condition, with `NoSuchEntity` where a trust-policy mismatch would instead say `AccessDenied`. This section creates `orders-artifact-publisher`, `orders-staging-deploy`, and `orders-production-deploy` as resources. It does **not** re-derive why each role's trust condition is scoped the way it is — [End-to-End Production Example §3](02_end_to_end_production_example.md#3-every-environment-gets-its-own-trust-boundary-not-just-its-own-name) owns the full trust-policy JSON and the `aud`/`sub` table across all three roles, and the attack that a shared deploy role would open up. This note creates the role; that section explains what makes its trust boundary safe.

The three roles differ only in which GitHub environment's token they trust:

```hcl
locals {
  deploy_roles = {
    "orders-artifact-publisher" = "artifact-publish"
    "orders-staging-deploy"     = "staging"
    "orders-production-deploy"  = "production"
  }

  # The subset of the roles above that also deploy to a specific ECS cluster.
  ecs_deploy_roles = {
    "orders-staging-deploy"    = "staging"
    "orders-production-deploy" = "production"
  }
}

data "aws_iam_policy_document" "oidc_trust" {
  for_each = local.deploy_roles

  statement {
    effect  = "Allow"
    actions = ["sts:AssumeRoleWithWebIdentity"]

    principals {
      type        = "Federated"
      identifiers = ["arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:aud"
      values   = ["sts.amazonaws.com"]
    }

    condition {
      test     = "StringEquals"
      variable = "token.actions.githubusercontent.com:sub"
      values   = ["repo:acme/orders:environment:${each.value}"]
    }
  }
}

resource "aws_iam_role" "deploy" {
  for_each           = local.deploy_roles
  name               = each.key
  assume_role_policy = data.aws_iam_policy_document.oidc_trust[each.key].json
}
```

This assumes the account already has GitHub's OIDC identity provider registered at `arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com` — a one-time, account-level resource shared by every repository, not something this per-repository platform recreates. See [Permissions, Secrets, and OIDC](../03_security_and_supply_chain/01_permissions_secrets_and_oidc.md) if that provider doesn't exist yet.

Trust decides *who* can assume a role; a separate permission policy decides *what* it can do once assumed, and each role here gets only what its own job performs. The publisher pushes images and nothing else:

```hcl
data "aws_iam_policy_document" "publisher_permissions" {
  statement {
    sid       = "AuthenticateToECR"
    effect    = "Allow"
    actions   = ["ecr:GetAuthorizationToken"]
    resources = ["*"] # this action has no resource-level permissions in IAM
  }

  statement {
    sid    = "PushToOrdersRepositoryOnly"
    effect = "Allow"
    actions = [
      "ecr:BatchCheckLayerAvailability",
      "ecr:PutImage",
      "ecr:InitiateLayerUpload",
      "ecr:UploadLayerPart",
      "ecr:CompleteLayerUpload",
      "ecr:BatchGetImage",
    ]
    resources = [aws_ecr_repository.orders.arn]
  }
}

resource "aws_iam_role_policy" "publisher" {
  name   = "ecr-push-only"
  role   = aws_iam_role.deploy["orders-artifact-publisher"].id
  policy = data.aws_iam_policy_document.publisher_permissions.json
}
```

`orders-staging-deploy` and `orders-production-deploy` each get the same shape of policy, scoped to their own cluster's `orders` service only — the staging role's policy document can never resolve to a resource ARN that contains `production`:

```hcl
data "aws_iam_policy_document" "deploy_permissions" {
  for_each = local.ecs_deploy_roles

  statement {
    sid       = "UpdateOwnClusterServiceOnly"
    effect    = "Allow"
    actions   = ["ecs:DescribeServices", "ecs:UpdateService"]
    resources = ["arn:aws:ecs:eu-west-1:123456789012:service/${each.value}/orders"]
  }

  statement {
    sid       = "RegisterTaskDefinitionRevisions"
    effect    = "Allow"
    actions   = ["ecs:RegisterTaskDefinition", "ecs:DescribeTaskDefinition"]
    resources = ["*"] # task definitions have no ARN to scope to before they're registered
  }

  statement {
    sid       = "PassExecutionRoleToECSOnly"
    effect    = "Allow"
    actions   = ["iam:PassRole"]
    resources = [aws_iam_role.execution.arn]

    condition {
      test     = "StringEquals"
      variable = "iam:PassedToService"
      values   = ["ecs-tasks.amazonaws.com"]
    }
  }
}

resource "aws_iam_role_policy" "deploy" {
  for_each = local.ecs_deploy_roles
  name     = "ecs-deploy-own-cluster-only"
  role     = aws_iam_role.deploy[each.key].id
  policy   = data.aws_iam_policy_document.deploy_permissions[each.key].json
}
```

`ecs:RegisterTaskDefinition` has to stay `Resource: "*"` — IAM has no ARN for a task definition that doesn't exist yet, so the narrowing has to come entirely from the trust policy and from which pipeline is allowed to call it, not from this statement. `aws_iam_role.execution` is the ECS task's own execution role, created in [§4](#4-task-definition-revision-1-before-183-can-exist) below — a different role from the three above, assumed by `ecs-tasks.amazonaws.com` rather than by GitHub.

**Success signal:** `aws iam get-role --role-name orders-staging-deploy` returns the role with the trust policy above attached; a workflow that assumes it and runs `aws sts get-caller-identity` lands as `arn:aws:sts::123456789012:assumed-role/orders-staging-deploy/<session>`, exactly as [Permissions, Secrets, and OIDC](../03_security_and_supply_chain/01_permissions_secrets_and_oidc.md) verifies for any OIDC role.

---

## 4. Task-Definition Revision 1, Before 183 Can Exist

`register-task-definition.sh` in the end-to-end example only ever *edits* an existing task-definition family — it fetches the current revision, swaps one container's image, and registers the result as the next revision. It has nothing to fetch on the very first deploy, because the family doesn't exist yet.

> **Key insight**: the platform has to exist before the pipeline that fills it in can run, but that pipeline is the only thing that will ever produce a real `orders` image — so revision 1 cannot be shaped like revision 183. It has to point at a placeholder that satisfies ECS's requirements (a real image, a real port, a real execution role) without pretending to be a tested release. Bootstrapping a new service into an existing pipeline almost always has one step like this: a manually-registered, deliberately-fake first version that exists only so the automated version has something to replace.

A Fargate task also needs its own **execution role** — distinct from the three deploy roles above, assumed by the ECS agent itself (`ecs-tasks.amazonaws.com`) to pull the image and write logs, not by GitHub:

```hcl
resource "aws_iam_role" "execution" {
  name = "orders-ecs-execution"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ecs-tasks.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "execution" {
  role       = aws_iam_role.execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
```

Revision 1 itself, registered with family `orders` and container `orders` — the exact two names `register-task-definition.sh` later takes as its `$1` and `$2` arguments — pointing at a `:bootstrap` tag pushed by hand once, before any pipeline exists to push `git-<sha>` tags:

```hcl
resource "aws_cloudwatch_log_group" "orders" {
  name              = "/ecs/orders"
  retention_in_days = 14
}

resource "aws_ecs_task_definition" "orders" {
  family                   = "orders"
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.execution.arn

  container_definitions = jsonencode([{
    name      = "orders"
    image     = "${aws_ecr_repository.orders.repository_url}:bootstrap"
    essential = true
    portMappings = [
      { containerPort = 8080, protocol = "tcp" }
    ]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = aws_cloudwatch_log_group.orders.name
        "awslogs-region"        = "eu-west-1"
        "awslogs-stream-prefix" = "orders"
      }
    }
  }])
}
```

**Success signal:** `aws ecs describe-task-definition --task-definition orders --query 'taskDefinition.{family:family,revision:revision,container:containerDefinitions[0].name}'` reports `"family": "orders"`, `"revision": 1`, `"container": "orders"` — the exact family and container name the release workflow's first real deploy will pass straight into `register-task-definition.sh`.

---

## 5. What Breaks First, and When Not to Provision This Way

⚠️ **The ECS service and its task definition live in different Terraform states.** Inside one `terraform apply`, the reference from `aws_ecs_service.orders` to `aws_ecs_task_definition.orders.arn` makes Terraform create the task definition first automatically — there's no ordering problem as written above. Split them into separate states (a common instinct: clusters feel like "infrastructure," task definitions feel like "application") and that automatic ordering disappears. The service's apply then either fails outright with `ClientException: Unable to run task... no active task definition family`, or worse, succeeds against a stale ARN read from remote state, silently redeploying an old bootstrap image. Keep a task definition and the service that runs it in the same state, or pass the ARN through an explicit `terraform_remote_state` data source — never a hardcoded string.

⚠️ **The lifecycle policy is widened to expire tagged images too.** The rule in [§1](#hardening-expire-what-nobody-promoted) only matches images with zero tags, which is safe here specifically because this pipeline never removes the `git-<sha>` tag from an image it pushed. Add a rule that also expires *tagged* images past some age or count, and the same policy can delete the exact digest a production task definition is still running — silently breaking the promotion-by-digest guarantee the whole platform exists to provide, with no error until the next deploy tries to pull an image that's gone.

⚠️ **The repository opted into GitHub's immutable-subject format after these roles were written.** The `sub` value baked into `local.deploy_roles` above — `repo:acme/orders:environment:staging` — assumes the historical name-based subject. If `acme/orders` is later created fresh, or opts in, GitHub starts issuing `repo:acme@<id>/orders@<id>:environment:staging` instead, the condition stops matching, and `AssumeRoleWithWebIdentity` fails with a bare `AccessDenied` — nothing in the failing workflow or in this Terraform says which format changed. [End-to-End Production Example §3](02_end_to_end_production_example.md#3-every-environment-gets-its-own-trust-boundary-not-just-its-own-name) has the check for which format a given repository actually issues; run it before writing the `sub` value in, not after the role stops working.

**When not to provision this way:**

- **A platform team already vends ECS clusters centrally.** If `staging` and `production` are shared, multi-tenant clusters created by a landing-zone module, don't create per-repository clusters like `aws_ecs_cluster.this` above — create only the `orders` service, attached to the cluster that already exists, and drop the cluster resource entirely.
- **The target is Kubernetes, not ECS.** None of these resource shapes transfer — a task definition has no Kubernetes equivalent as a Terraform resource, and pod identity (IRSA or Pod Identity) replaces the execution-role pattern in §4 entirely.
- **The account has no GitHub OIDC provider registered yet.** The `aws_iam_role` resources in [§3](#3-three-iam-roles-creating-the-resource-not-the-trust-boundary) apply successfully either way — Terraform doesn't validate that the `Federated` principal ARN resolves to anything — but every `AssumeRoleWithWebIdentity` call fails until the provider exists. Register it first; see [Permissions, Secrets, and OIDC](../03_security_and_supply_chain/01_permissions_secrets_and_oidc.md).

---

## 6. Verify the Whole Configuration Together

Every resource block above, concatenated into one `main.tf` plus the two input variables from [§2](#2-two-ecs-clusters-and-the-orders-service-each-one-deploys), is a real Terraform configuration — not fragments that only look plausible next to each other. Running it through `terraform init` and `terraform validate` catches the failure mode this note would most embarrassingly ship with: a resource argument or reference that doesn't actually exist in the AWS provider.

```text
$ terraform init -backend=false -input=false -no-color
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.100.0...
- Installed hashicorp/aws v5.100.0 (signed by HashiCorp)

Terraform has been successfully initialized!

$ terraform validate -no-color
Success! The configuration is valid.
```

**Success signal:** both commands exit `0`, and `validate` prints exactly `Success! The configuration is valid.` This step never contacts AWS — `-backend=false` skips remote state, and `validate` only checks syntax and provider schema — so it catches typos and wrong argument names, but not a `NoSuchEntity` from a role that doesn't exist or a quota this account has already hit. `terraform plan` against real credentials is what proves the rest, using the flow in [Infrastructure and Database Changes](../04_delivery_operations/03_infrastructure_and_database_changes.md).

---

## 7. References

- [Amazon ECR lifecycle policies](https://docs.aws.amazon.com/AmazonECR/latest/userguide/LifecyclePolicies.html)
- [Amazon ECS task execution IAM role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-execution-IAM-role.html)
- [`aws_iam_policy_document` data source](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/data-sources/iam_policy_document)

(checked 2026-08-14)

---

**Next**: [End-to-End Production Example](02_end_to_end_production_example.md)
