---
title: Building a Greenfield EKS Platform on AWS
date: 2026-08-25 12:30:00 +0530
categories: [devops, aws]
tags: [aws, kubernetes, eks, opentofu, terraform, iam, networking, gitops]
---

I recently designed a production-grade Amazon EKS platform from scratch — multi-account AWS Organization, OpenTofu end to end, GitOps delivery, zero-trust access, the full observability stack. What made it interesting wasn't the components. It was the constraints.

The environment isn't hosting customer traffic. It's a **permanent demo and training environment**: something to show prospects, and something the team learns to build on. That sounds like it should be easier than production. In one important way it's harder, and that single fact reshaped almost every decision below.

## The brief, and why it changed everything

A demo environment has to *look* like production. Prospects poke at it. If the Kubernetes API is sitting on the public internet, or IAM roles are carrying `AdministratorAccess`, the demo undermines the pitch. So all the production-grade controls stay.

But the workloads are disposable, and the environment gets torn down and rebuilt constantly — because greenfield rebuild *is* the training exercise. Three consequences:

- **Rebuildability is a feature.** Every resource is OpenTofu-managed with a clean destroy order. Nothing gets clicked into existence, because anything clicked won't survive the next rebuild.
- **Cost tracks usage, not calendar.** An environment idling at production cost is pure waste when it's genuinely used a few hours a week.
- **A third-party product had to plug into it** — one I couldn't modify. More on that later, because it turned out to be the most interesting design problem in the whole build.

## Decision one: where does the blast radius stop?

The existing AWS Organization had exactly one account — the management account — running a live third-party product. Everything in one place, no OUs, no SCPs beyond the default `FullAWSAccess`.

The obvious move is separate accounts. IAM does not cross an account boundary without an explicit trust policy on *both* sides, which makes an account the strongest isolation primitive AWS offers. Far stronger than tags, naming conventions, or careful policy writing.

The layout I landed on:

| Account | Holds |
|---|---|
| **Management + Shared Services** | OpenTofu state, central ECR, org CloudTrail, Config, GuardDuty, IAM Identity Center, CI/CD OIDC roles — plus the pre-existing vendor stack |
| **Workloads — Dev** | EKS cluster, always on, scheduled scale-down |
| **Workloads — Prod** | EKS cluster, ephemeral, stood up per demo |

Note what *didn't* get its own account: the tooling. CI/CD, the registry and the state backend stay in the management account alongside the vendor product. That's cheaper — no duplicate registry, no cross-account state backend, no second Identity Center — and it's a real trade-off, not a free win. The tooling shares a blast radius with something I don't control.

## Decision two: a wall where you can't have a wall

When you can't separate by account, you separate by **permission boundary**.

A permission boundary caps what a role can do regardless of how permissive its own policy becomes. Attach `AdministratorAccess` to a bounded role and it still can't exceed the boundary. That property is what makes it the right tool here: it survives future carelessness, which an identity policy does not.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowRegionalWork",
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringEquals": { "aws:RequestedRegion": ["ap-south-1", "us-east-1"] }
      }
    },
    {
      "Sid": "AllowGlobalEndpointServices",
      "Effect": "Allow",
      "Action": ["iam:*", "sts:*", "route53:*", "support:*", "ecr-public:*"],
      "Resource": "*"
    },
    {
      "Sid": "DenyTouchingVendorCompute",
      "Effect": "Deny",
      "Action": ["lambda:*", "states:*", "apigateway:*"],
      "Resource": [
        "arn:aws:lambda:*:*:function:vendor-*",
        "arn:aws:states:*:*:stateMachine:vendor-*"
      ]
    },
    {
      "Sid": "DenyLongLivedCredentials",
      "Effect": "Deny",
      "Action": ["iam:CreateUser", "iam:CreateAccessKey", "iam:CreateLoginProfile"],
      "Resource": "*"
    },
    {
      "Sid": "DenyBoundaryEscape",
      "Effect": "Deny",
      "Action": [
        "iam:DeleteRolePermissionsBoundary",
        "iam:PutRolePermissionsBoundary"
      ],
      "Resource": "arn:aws:iam::*:role/platform-*"
    }
  ]
}
```

Two details worth calling out.

The `us-east-1` allowance in the region lock isn't sloppiness — ACM certificates for CloudFront, the Pricing API and parts of the Route 53 control plane only exist there. A strict single-region boundary breaks in confusing ways about a week later.

`DenyBoundaryEscape` means the boundary can't remove itself. That's the point, and it's also a foot-gun: detaching it later needs a principal that *isn't* bounded by it. Confirm your break-glass path works **before** you attach this, not after.

> Permission boundaries are the most underused IAM feature. Most teams reach for more granular identity policies when the actual problem is that nothing stops a future policy from being too broad.
{: .prompt-tip }

Underneath, SCPs at the OU level handle the rest: deny root user actions, deny leaving the org, deny disabling CloudTrail/Config/GuardDuty, and — for the prod OU — require IMDSv2 on `RunInstances` and deny unencrypted EBS.

## Network: three tiers, and one subnet decision that bites

Each workload account gets a `/16` with a three-tier layout across three AZs:

| Tier | AZ-a | AZ-b | AZ-c | Usable | Holds |
|---|---|---|---|---|---|
| Public | `10.20.0.0/24` | `10.20.1.0/24` | `10.20.2.0/24` | 251 | ALB, NAT |
| Private — app | `10.20.16.0/20` | `10.20.32.0/20` | `10.20.48.0/20` | 4091 | Nodes and pod ENIs |
| Private — data | `10.20.64.0/24` | `10.20.65.0/24` | `10.20.66.0/24` | 251 | RDS, EFS, endpoints |
| Reserved | `10.20.128.0/17` | | | 32k | Secondary CNI CIDR |

The application tier gets `/20`s rather than `/24`s, and this is the one to get right the first time. The AWS VPC CNI assigns a **real VPC address to every pod**, not an overlay address. A `/24` gives you 251 addresses total — a full observability stack plus a few application deployments will exhaust that, and the failure mode is pods stuck in `ContainerCreating` with an error message that doesn't obviously say "you ran out of IPs." Resizing a subnet afterwards means rebuilding it.

The guardrails that turn "an EKS cluster" into something you can show a security-conscious audience:

| Guardrail | Implementation |
|---|---|
| Delete the default VPC | Removed in every new account on day one |
| Neuter the default security group | Zero ingress, zero egress rules |
| No public IPs on compute | `map_public_ip_on_launch = false`, no EIPs on nodes |
| Private-only EKS endpoint | Public access disabled once the mesh VPN is healthy |
| SGs reference SGs, not CIDRs | Node SG accepts from ALB SG; data SG from node SG |
| Gateway endpoints | S3 and DynamoDB — free, and keeps registry pulls off NAT |
| Flow logs | ALL traffic, 30-day retention |
| No SSH anywhere | No key pairs, no port 22; SSM Session Manager is break-glass |
| IMDSv2, hop limit 1 | Containers cannot reach instance metadata |

That last one deserves a paragraph. Setting `http_put_response_hop_limit = 1` on the node launch template means a container can't reach the instance metadata service, which means it can't steal the node's IAM role. It's a two-line change that closes the single most common EKS privilege-escalation path. It also *forces* you to use IRSA properly, because the lazy fallback stops working — which is the real benefit.

### The private endpoint sequencing problem

Turning off public API access sounds simple until you realise your CI runner can no longer reach the cluster.

The clean answer is that it shouldn't need to. With Argo CD, delivery is **pull-based**: CI builds an image, pushes it, and commits a manifest change. Argo CD, running *inside* the cluster, notices and syncs. The pipeline never needs API access at all.

What does still need access is OpenTofu — specifically the Kubernetes and Helm providers. So the code splits in two:

- an **infra** layer using only the AWS provider, which runs from anywhere
- a **platform** layer using Kubernetes and Helm providers, which runs over the VPN

Build the cluster with the endpoint public but CIDR-restricted, stand up the mesh VPN, *then* close it. Trying to do it in one pass produces a cluster you can't reach with the tool that created it.

## Compute: check the price list

The original design specified `m6i.large`. A quick query against the Pricing API in `ap-south-1`:

```
m6i.large  = $0.1010/hr
m6a.large  = $0.0555/hr
```

Same 2 vCPU / 8 GiB, same three AZs, AMD instead of Intel. **45% cheaper** for workloads that are overwhelmingly Go and JVM services where the silicon difference is noise. Across four nodes that's roughly $130/month for a one-line change.

> Instance family pricing shifts and regional availability varies. Query the Pricing API for your region rather than carrying a default from the last project — I've seen this exact substitution missed on three different builds.


Each cluster runs **two** node groups, not one:

| Node group | Capacity | Types | Min/Des/Max | Taint |
|---|---|---|---|---|
| `platform` | On-demand | `m6a.large` | 2 / 2 / 3 | `dedicated=platform:NoSchedule` |
| `apps` | Spot | `m6a.large`, `m5.large`, `m6i.large`, `t3.large` | 1 / 2 / 6 | none |

The split matters. The observability stack is memory-hungry and must not be evicted by a demo application, and a demo application is exactly what you want on interruptible capacity.

Four instance types on the spot group is deliberate. Spot capacity is allocated per type per AZ — a single-type spot group in one AZ *will* eventually fail to scale, usually during a demo. Four types across three AZs gives twelve pools to draw from.

On add-ons, the highest-leverage setting is **prefix delegation** on the VPC CNI. It raises pods-per-node on an `m6a.large` from 29 to 110. Without it you buy nodes to hold IP addresses rather than to run workloads.

## Autoscaling is four things, not one

| Layer | Mechanism | Responds in | Scales |
|---|---|---|---|
| Pods | HPA on metrics-server | 15–60s | Replica count |
| Nodes | Cluster Autoscaler | 1–3 min | Node group desired capacity |
| Hard bounds | Node group min/max | instant | The ceiling CA can't exceed |
| Calendar | EventBridge Scheduler | scheduled | Overnight/weekend shutdown |

The third layer is the one people skip, and it's the cost guarantee. HPA and Cluster Autoscaler will happily scale into a very large bill if a workload misbehaves. The node group `max` is the only thing that says no.

Cluster Autoscaler discovers node groups by ASG tag, and **managed node groups don't add those tags themselves** — a genuinely annoying gotcha:

```hcl
k8s.io/cluster-autoscaler/enabled      = "true"
k8s.io/cluster-autoscaler/platform-dev = "owned"
```

The same tag then scopes the IAM policy, so the autoscaler can only mutate groups it owns:

```json
{
  "Sid": "MutateOwnedGroupsOnly",
  "Effect": "Allow",
  "Action": [
    "autoscaling:SetDesiredCapacity",
    "autoscaling:TerminateInstanceInAutoScalingGroup",
    "autoscaling:UpdateAutoScalingGroup"
  ],
  "Resource": "*",
  "Condition": {
    "StringEquals": {
      "aws:ResourceTag/k8s.io/cluster-autoscaler/platform-dev": "owned"
    }
  }
}
```

Two supporting pieces make scale-down behave: a **PriorityClass** putting platform components above application pods, so pressure evicts the demo app and not Grafana, and **PodDisruptionBudgets** on CoreDNS, Argo CD and Prometheus so the autoscaler can't drain the last replica of something that matters.

Worth saying plainly: **Karpenter would be the better choice** on a greenfield cluster. It provisions right-sized nodes directly instead of nudging ASGs, consolidates underused nodes automatically, and typically cuts node spend 20–40%. Cluster Autoscaler here stays faithful to an already-approved design, and the swap is a clean follow-up.

## The IAM mistake almost every EKS tutorial makes

Search for how to build an EKS node role and you'll be told to attach four managed policies, including `AmazonEKS_CNI_Policy`.

Don't.

On the node role, **every pod on that host inherits ENI-manipulation rights**. The CNI policy belongs on an IRSA role scoped to the `aws-node` service account, where only the CNI can assume it. Combine that with the hop-limit-1 metadata setting above and you have a least-privilege story that holds up to scrutiny.

The node role should carry exactly three: `AmazonEKSWorkerNodePolicy`, `AmazonEC2ContainerRegistryPullOnly` (narrower than the usual `ReadOnly`), and `AmazonSSMManagedInstanceCore` for keyless access.

The other place to be precise is the GitHub OIDC trust policy:

```json
{
  "Effect": "Allow",
  "Principal": {
    "Federated": "arn:aws:iam::222233334444:oidc-provider/token.actions.githubusercontent.com"
  },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
      "token.actions.githubusercontent.com:sub":
        "repo:my-org/platform-infra:environment:dev"
    }
  }
}
```

Pinning `sub` to a specific repo **and** GitHub environment is what makes OIDC safe. A wildcard there — and wildcards are common, because they make the first attempt work — lets any workflow in any repo in the organization assume the role. Pair it with required reviewers on the `prod` environment and your approval gate is enforced by GitHub before AWS is ever called.

Also: use EKS **access entries** rather than the legacy `aws-auth` ConfigMap. Authentication mode `API`, roles mapped explicitly, nobody authenticating as an IAM user.

## Integrating with something you can't change

The most interesting constraint: a third-party product already lived in the management account — a serverless stack of container-image Lambdas, Step Functions and HTTP APIs. Vendor-owned. Not mine to modify, redeploy or refactor.

This is a common situation and it's usually handled badly, by either forking the thing or building a fragile wrapper around it. The better framing is to treat the vendor's **published interfaces as the API contract** and adapt on your side only:

1. **Shared registry.** Platform images push to the same ECR registry the vendor's scanners already read. Workload accounts get cross-account pull via repository policy.
2. **CI gate.** The pipeline calls the vendor's existing scan endpoint and blocks the merge on the verdict.
3. **Repository registration.** The new infra repo registers through the existing VCS connector, so pull requests get scanned automatically.
4. **Findings surface.** Reports already land in an S3 bucket; an event rule forwards them to Security Hub as custom findings and a Grafana panel renders them beside cluster metrics.
5. **Live targets.** A deliberately vulnerable demo app gives the scanner something real to find.

Vendor changes required: zero. Every seam is a published interface.

### A useful thing I found while auditing

The eventual goal is a case study on migrating that stack off Lambda onto EKS or ECS. Auditing it turned up something that makes it much more tractable than a typical migration: **every function was already a container image** — `PackageType: Image`, pulling from ECR. No zip bundle to repackage, no runtime to port.

What actually remains is narrower than it first appears:

- **The runtime contract.** Lambda container images speak the Lambda Runtime API — the entrypoint polls for an invocation rather than serving traffic or running to completion. Outside Lambda that needs the Runtime Interface Emulator or a thin adapter. This is the real work, and it's roughly per *function shape*, not per function.
- **Orchestration.** Step Functions state machines become Argo Workflows DAGs — a translation with close structural correspondence.
- **Invocation surface.** HTTP APIs become an ALB Ingress with the same routes.

The comparison worth making is economic, and it turns on utilisation. Lambda at high memory costs nothing idle and a great deal under sustained load; a spot node pool costs a fixed amount and absorbs load for free until it saturates. There's a real crossover point, computable from actual invocation data — and computing it is a far better artefact than a migration undertaken for its own sake.

## What it costs, and the trick that halves it

Reference figures at `ap-south-1` list prices:

| Component | Dev 24/7 | Dev scheduled | Prod 24/7 |
|---|---:|---:|---:|
| EKS control plane | 73.00 | 73.00 | 73.00 |
| Platform nodes | 81.03 | 28.36 | 81.03 |
| App nodes | 33.58 | 11.75 | 81.03 |
| NAT gateways | 38.00 | 38.00 | 110.00 |
| Load balancer | 20.00 | 20.00 | 20.00 |
| Interface endpoints | — | — | 29.20 |
| EBS gp3 | 11.86 | 11.86 | 16.42 |
| CloudWatch | 20.00 | 14.00 | 25.00 |
| S3, KMS, ECR, DNS, secrets | 8.50 | 8.50 | 9.50 |
| Data transfer | 6.00 | 4.00 | 10.00 |
| **Total (USD/month)** | **291.97** | **209.47** | **455.18** |

Two things stand out.

**NAT gateways are the quiet line item.** Three of them cost more than the EKS control plane. Dev runs one and routes all three private subnets to it — a deliberate single point of failure worth about $66/month, and the right trade when the failure mode is "the demo pauses." Prod runs three.

**The scheduled shutdown is where the money is.** An EventBridge rule scales dev node groups to zero at 21:00 and back at 08:00 on weekdays, staying at zero across weekends — roughly 65% of wall-clock hours removed from node spend, with no effect on a working day. The control plane charge continues regardless, and the cluster is back in about four minutes.

Then the actual trick: **prod is ephemeral**. It costs about $0.55/hour while it exists, so a full-day demo runs about $5. Running both environments permanently would be roughly $750/month for something used a few hours a week.

That isn't only cheaper. Destroying and rebuilding prod for each engagement *is* the greenfield exercise the team needs to practise — so the cost-optimal choice and the pedagogically useful one turn out to be the same choice. It also keeps the IaC honest: infrastructure rebuilt weekly cannot quietly accumulate drift.

## Teardown is a feature

If rebuild is the exercise, `destroy` has to work reliably rather than mostly. Three things break a naive `tofu destroy` on an EKS stack:

1. **Orphaned load balancers.** ALBs created by the AWS Load Balancer Controller aren't in OpenTofu state. Ingress and Service objects must be deleted *before* the cluster, or the ALBs and their security groups survive and block VPC deletion. Destroying the platform layer first — which depends on the controller — makes the ordering automatic.
2. **Dangling ENIs.** The VPC CNI leaves ENIs behind on abrupt node termination. Node groups go before subnets, with a wait.
3. **Retained resources.** State bucket, KMS keys and audit buckets carry `prevent_destroy`. Destroying an environment must never destroy the ability to rebuild it.

Fixed order: `platform` → `cluster` → `network`. A scheduled job then checks for orphaned load balancers, unattached ENIs and unassociated EIPs, so cost leaks surface within a day rather than on the next invoice.

## What I'd carry to the next build

- **Account boundaries first.** Every other isolation mechanism is a consolation prize. Reach for permission boundaries only where an account wall genuinely isn't available.
- **Query the price list.** A 45% compute saving sat behind one API call and a one-word change.
- **Size app subnets for pod IPs, not node counts.** The CNI hands out real VPC addresses, and the resulting failure is obscure.
- **Move the CNI policy off the node role.** It's the default in most tutorials and it's wrong.
- **Pin the OIDC `sub` claim.** Repo *and* environment. Wildcards there are a standing invitation.
- **Set node group `max` deliberately.** It's the only hard stop between an autoscaler and a very large bill.
- **Design the destroy path on day one.** Infrastructure you can't confidently tear down is infrastructure you'll be afraid to change.

The general lesson is that "production-grade" is mostly about which failures you've decided are unacceptable. For this build the unacceptable failures were an exposed Kubernetes API, a credential that could reach something it shouldn't, and a bill nobody predicted. Almost every decision above traces back to one of those three.
