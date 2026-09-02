# Module 10: Production Infra — Terraform + EKS

**Environment:** AWS (real account, real spend — see the Cost & Cleanup section before you start)
**Prerequisites:** Module 9 complete; an AWS account with credentials configured locally (`aws sts get-caller-identity` should return your account, not an error)

---

## Learning Objectives

By the end of this module, you should be able to:
- Explain why real teams provision Kubernetes clusters with Terraform instead of clicking through a console
- Recognize a Terraform module's basic shape: providers, resources, state, inputs, outputs
- Provision a real EKS cluster using the official `terraform-aws-modules/eks/aws` module — VPC, managed node group, and modern EKS access entries
- Install ArgoCD on EKS via Helm, and expose its UI two different ways
- Back up ArgoCD's configuration as part of a production go-live checklist
- Tear the whole thing down cleanly with `terraform destroy`, and confirm nothing is left running

---

## 1. Why Terraform for Cluster Provisioning

Everything through Module 9 ran on `kind` — free, disposable, `kind delete cluster` undoes any mistake in seconds. Production infrastructure doesn't work that way: a cluster costs money every hour it exists, mistakes are harder to undo, and "how was this cluster actually configured" needs to be answerable months later, by someone who wasn't in the room when it was created.

Terraform solves the same category of problem ArgoCD solves, one layer down the stack: **infrastructure as code, reviewed and versioned in Git, instead of state that only exists in someone's memory or a console session.**

| | Clicking in the AWS Console | Terraform |
|---|---|---|
| Reproducibility | Depends on someone remembering every setting | The `.tf` files *are* the settings |
| Review | No diff, no approval step | A PR shows exactly what infrastructure is about to change |
| Teardown | Manual, easy to miss a resource (and keep paying for it) | `terraform destroy` removes everything it created, and only what it created |
| "What's actually running?" | Whatever the console currently shows | `terraform plan` diffs reality against Git, same idea as ArgoCD's `Sync Status` |

That last row is worth sitting with: **ArgoCD keeps a Kubernetes cluster's contents honest against Git. Terraform keeps the cluster's own existence — and everything around it (VPC, IAM, node groups) — honest against Git, one layer below.** Different tool, same discipline.

---

## 2. Terraform Basics Recap

Three ideas carry the rest of this module:

- **Providers** — a plugin that knows how to talk to a specific API. `provider "aws" { region = "us-east-1" }` is what lets Terraform create real AWS resources; without it, Terraform is just a YAML-adjacent language with nothing to talk to.
- **State** — Terraform's record of what it actually created, stored in a `terraform.tfstate` file. This is how `terraform plan` knows the difference between "doesn't exist yet" and "already exists, matches config" and "exists but drifted." Local state (a file on your laptop) is fine for this module; real teams use **remote state** (an S3 bucket + DynamoDB lock table, commented out in `versions.tf` for when you're ready) so the state isn't lost if a laptop dies, and so a team can safely run `terraform apply` from more than one machine.
- **Modules** — a reusable, versioned bundle of resources someone else already wrote and tested. This module reaches for two community-maintained ones instead of hand-rolling a VPC and an EKS cluster from raw AWS resources: `terraform-aws-modules/vpc/aws` and `terraform-aws-modules/eks/aws` — both are the de facto standard for this in real teams, maintained by the Terraform AWS community, not something built from scratch here.

---

## 3. Provisioning EKS

The full Terraform lives in `online-boutique-gitops/terraform/` — five files, each with one job:

```
terraform/
├── versions.tf     # Terraform + provider version constraints, provider config
├── variables.tf    # every input this config accepts, with sane defaults
├── vpc.tf          # networking: VPC, public/private subnets, NAT gateway
├── eks.tf          # the cluster itself: control plane, node group, access entries
└── outputs.tf      # what to do with the cluster once it exists
```

### VPC

```hcl
# vpc.tf
data "aws_availability_zones" "available" {
  filter {
    name   = "opt-in-status"
    values = ["opt-in-not-required"]
  }
}

locals {
  azs = slice(data.aws_availability_zones.available.names, 0, 3)
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.0"

  name = "${var.cluster_name}-vpc"
  cidr = var.vpc_cidr

  azs             = local.azs
  private_subnets = [for k, v in local.azs : cidrsubnet(var.vpc_cidr, 4, k)]
  public_subnets  = [for k, v in local.azs : cidrsubnet(var.vpc_cidr, 8, k + 48)]

  enable_nat_gateway = true
  single_nat_gateway = true # one shared NAT, not one per AZ — cheaper, fine for a course cluster

  public_subnet_tags  = { "kubernetes.io/role/elb" = 1 }
  private_subnet_tags = { "kubernetes.io/role/internal-elb" = 1 }
}
```

Nodes go in the **private** subnets (no direct internet route in) with a NAT gateway for outbound (pulling container images, talking to AWS APIs); load balancers for anything you expose go in the **public** subnets. The `kubernetes.io/role/...` tags aren't decoration — EKS's own controllers read them to decide where an internal vs. internet-facing `LoadBalancer` Service should place its load balancer.

### EKS Cluster

```hcl
# eks.tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 21.0"

  name               = var.cluster_name
  kubernetes_version = var.kubernetes_version

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  addons = {
    coredns                = {}
    kube-proxy              = {}
    eks-pod-identity-agent  = { before_compute = true }
    vpc-cni                 = { before_compute = true }
  }

  eks_managed_node_groups = {
    default = {
      instance_types = var.node_instance_types
      ami_type       = "AL2023_x86_64_STANDARD"

      min_size     = var.node_min_size
      max_size     = var.node_max_size
      desired_size = var.node_desired_size # ignored after initial creation
    }
  }

  enable_cluster_creator_admin_permissions = true

  access_entries = {
    for arn in var.additional_cluster_admin_arns : arn => {
      principal_arn = arn
      policy_associations = {
        admin = {
          policy_arn   = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
          access_scope = { type = "cluster" }
        }
      }
    }
  }
}
```

Three things worth calling out:

- **`addons`** — EKS's own managed add-ons (CoreDNS, kube-proxy, the VPC CNI, and the Pod Identity agent). `kind` gives you all of this for free out of the box; on EKS, the control plane and the node-level networking stack are provisioned separately, and this is how you ask AWS to manage them for you instead of installing each by hand.
- **`enable_cluster_creator_admin_permissions = true`** — grants whoever runs `terraform apply` cluster-admin, automatically. This is EKS's modern **access entry** system, not the older `aws-auth` ConfigMap you may see in older tutorials — access entries are IAM-native, and this is the field the syllabus's "cluster access entries" topic refers to.
- **`access_entries`** — for anyone *besides* the applying identity who needs access (a teammate, a CI role). Empty by default (`additional_cluster_admin_arns = []`); add ARNs there, not by hand-editing anything inside the cluster.

### Applying It

```bash
cd terraform
terraform init
terraform plan
```

Read the plan before you approve it — this is the review step that never existed when everything ran on `kind`. Confirm you're seeing one VPC, subnets, NAT gateway, one EKS cluster, one managed node group, nothing you don't recognize. Then:

```bash
terraform apply
```

Expect 15–20 minutes — EKS control planes are genuinely slow to provision, this isn't a timeout or a stuck terminal. Once it completes:

```bash
aws eks update-kubeconfig --region us-east-1 --name online-boutique
kubectl get nodes
```

You should see your managed node group's nodes, `Ready`.

---

## 4. Installing ArgoCD on EKS

Same Helm chart you'd use anywhere — nothing EKS-specific about ArgoCD itself:

```bash
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd -n argocd
```

### Exposing the UI: LoadBalancer vs. Port-Forward

Two real options, different trade-offs:

**Port-forward** — what you've used for the whole course so far:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
Works immediately, no AWS resources created, but only reachable from your own machine, and only while the command is running. Fine for this module's lab.

**LoadBalancer** — what a real team exposes ArgoCD's UI through:
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc argocd-server -n argocd
```
EKS provisions a real AWS load balancer (Network Load Balancer by default on recent EKS) with a public DNS name — reachable by anyone on your team, but also a real AWS resource billing by the hour, and (with no further configuration) reachable by anyone on the internet who finds that DNS name. A real rollout puts this behind a proper ingress/auth layer; this module just demonstrates the mechanism, not a production-hardened front door. **Remember to remove it (`kubectl delete svc argocd-server-lb` or patch the type back) before you tear down the cluster** — a `LoadBalancer` Service creates an AWS load balancer that `terraform destroy` doesn't know about and won't clean up, since Terraform never created it.

---

## 5. Backing Up ArgoCD's Configuration

Before this cluster becomes anyone's actual production system, back up what makes it *this* ArgoCD instance — its `Application`/`ApplicationSet`/`AppProject` objects, repo credentials, RBAC config:

```bash
argocd admin export > argocd-backup-$(date +%Y%m%d).yaml
```

This is a real step on a production go-live checklist, not a course-only exercise: if the cluster is ever lost or needs rebuilding, `argocd admin import` against that same file restores ArgoCD's own configuration — though notably not the workloads it manages, since those are already declared in Git and would resync on their own the moment a fresh ArgoCD points at the same repo. Store this file somewhere durable (not just your laptop) and treat it like any other credential-adjacent artifact — it can contain repo tokens.

---

## Lab: Provision EKS, Install ArgoCD

All of this happens in `online-boutique-gitops/terraform/`.

### Step 1 — Confirm AWS access

```bash
aws sts get-caller-identity
```

Should return your account ID, not an error. If it errors, resolve your AWS credentials before continuing — nothing in this module works without them.

### Step 2 — Review, then apply

```bash
terraform init
terraform plan
```

Read it. Then:

```bash
terraform apply
```

Confirm when prompted. Wait for it to finish — genuinely 15–20 minutes.

### Step 3 — Connect and verify

```bash
$(terraform output -raw configure_kubectl 2>/dev/null || echo "aws eks update-kubeconfig --region us-east-1 --name online-boutique")
kubectl get nodes
kubectl get pods -A
```

Confirm your managed node group's nodes are `Ready` and core system Pods (CoreDNS, kube-proxy, VPC CNI, Pod Identity agent) are `Running`.

### Step 4 — Install ArgoCD

```bash
kubectl create namespace argocd
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install argocd argo/argo-cd -n argocd
kubectl get pods -n argocd -w
```

Wait for every ArgoCD Pod to reach `Running`. Then log in (same pattern as Module 2):

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
argocd login localhost:8080 --username admin \
  --password $(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d) \
  --insecure
```

### Step 5 — Back up ArgoCD's config

```bash
argocd admin export > argocd-backup-$(date +%Y%m%d).yaml
```

**Checkpoint:** you have a real EKS cluster, provisioned and reviewable from Git, with ArgoCD installed and logged into — the same ArgoCD you've been using all course, just running on real infrastructure instead of `kind`. Module 11 deploys Online Boutique onto it.

---

## Mini-Lecture: Cost & Cleanup

**Don't skip this if you're not moving straight into Module 11.** An idle EKS cluster with a managed node group and a NAT gateway costs real money by the hour, whether or not anything is deployed to it.

Before tearing down:
```bash
kubectl get svc -A | grep LoadBalancer
```
Delete or patch back to `ClusterIP` any `LoadBalancer` Services first (per Section 4) — Terraform doesn't know about the AWS load balancers they create, so `terraform destroy` won't remove them, and they'll keep billing indefinitely.

Then:
```bash
cd terraform
terraform destroy
```

Confirm when prompted, wait for it to complete, then verify in the AWS Console (EC2, VPC, EKS, Load Balancers) that nothing from this module is still running. **This is the discipline every EKS module in this course ends with** — provisioning infrastructure without a verified teardown step is how course-related AWS bills quietly spiral.

---

## Key Terms Glossary

| Term | Meaning |
|---|---|
| **Terraform state** | Terraform's record of what it actually created, stored in `terraform.tfstate` — what `terraform plan` diffs reality against |
| **Remote state** | State stored somewhere shared (e.g. S3 + DynamoDB for locking) instead of a local file, so it survives a lost laptop and supports a team applying from multiple machines |
| **Terraform module** | A reusable, versioned bundle of resources — `terraform-aws-modules/vpc/aws` and `terraform-aws-modules/eks/aws` here, not hand-rolled from raw resources |
| **Managed node group** | AWS-managed EC2 worker nodes for EKS — AWS handles the launch template, scaling, and AMI updates, versus a self-managed node group where you own that yourself |
| **EKS access entry** | The modern, IAM-native way to grant a principal access to an EKS cluster's Kubernetes API — replaces the older `aws-auth` ConfigMap approach |
| **`enable_cluster_creator_admin_permissions`** | Module flag that grants the Terraform-applying identity cluster-admin automatically, via an access entry |
| **`argocd admin export`** | Backs up ArgoCD's own configuration (Applications, ApplicationSets, AppProjects, repo credentials, RBAC) to a file — restorable with `argocd admin import` |

---

## Recap Questions

1. What's the practical difference between what ArgoCD keeps honest against Git, and what Terraform keeps honest against Git?
2. Why do the EKS module's worker nodes go in the VPC's *private* subnets, not the public ones?
3. What's the difference between `enable_cluster_creator_admin_permissions` and the `access_entries` block — when would you need both?
4. Why would a `LoadBalancer` Service survive a `terraform destroy`, and what do you have to do about it before tearing the cluster down?
5. `argocd admin export` backs up ArgoCD's own config. Why doesn't it need to also back up the actual application workloads (Deployments, Services) it manages?

---

## What's Next

In **Module 11**, the Capstone, we deploy Online Boutique — Google's real 11-service e-commerce demo — onto this cluster with its own App-of-Apps repo, wire up a real GitHub Actions pipeline for one service, and close the loop: build, push, deploy, break something on purpose, recover it, and promote a real fix from staging to prod.
