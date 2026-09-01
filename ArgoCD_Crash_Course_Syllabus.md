# ArgoCD & GitOps Crash Course — Syllabus
**Audience:** DevOps Engineers & Developers
**Format:** Crash course — real-world-first, no rarely-used detours
**Demo App:** Finovra — a purpose-built 4-backend-service + dashboard app (2 languages), owned and versioned by us on Docker Hub — used for Modules 1–10. All four backend services deploy together from v1.0.0 onward; only the `dashboard` gets a new version each release, gaining one real, visible feature per module. **Online Boutique** (Google's 11-service e-commerce demo) is reserved for the Module 11 capstone, where its scale and realism become an asset rather than a distraction.

---

## Design Philosophy

This syllabus is deliberately **cut down**. Anything ArgoCD supports but that most teams don't touch day-to-day has been excluded or pushed to an optional appendix. Every module maps to something a student will actually do at work in their first month using ArgoCD.

**Why a custom demo app:** third-party demo repos (Podtato-Head, Sock Shop) risk going stale or disappearing over a course's multi-year shelf life, and existing "real" demos (Online Boutique) are too large to teach fundamentals against — students get lost in the app instead of focused on ArgoCD. Building our own small app means every failure scenario (a broken release, a slow release, a bad config) is staged intentionally to match exactly what a given module is teaching, and it never breaks from someone else's repo changing underneath us.

**Why the dashboard, not new services, carries each version:** most real releases bump an image tag rather than add a whole new microservice — students should practice the release pattern they'll actually use at work, not "add a service" five times in a row. Finovra's dashboard gains one visible feature per version (a changelog panel, a stats bar, a polish pass), so a version bump is always something you can *see*, without repeating the same "new folder + new manifests" motion every module.

**Why CI moved to the Capstone instead of its own module:** a dedicated GitHub Actions module this early tested well as a standalone technical walkthrough, but it duplicates ground the Capstone already covers once students reach it, and pulls focus away from ArgoCD itself. The mechanics (build → push → bump tag → ArgoCD syncs) are folded into the Capstone project instead, where they land as one step in a bigger, already-meaningful exercise rather than a detour. Reference material for the pipeline (a working `.github/workflows/dashboard-image.yml` against Finovra) already exists in `webapp` for whenever the Capstone gets built out.

**Cut from scope (rarely used, or used by a minority of teams in practice):**
- Jsonnet/Ksonnet as a config source (legacy, low adoption)
- Advanced ApplicationSet generators (Matrix, SCM Provider, Pull Request) — mentioned in passing only
- Internal mTLS between ArgoCD components (infra-team/platform-admin concern, not day-to-day)
- Argo CD Agent (emerging, not yet mainstream)
- Deep RBAC policy authoring, full SSO/OIDC setup — covered at "get it working" level only in the optional Running ArgoCD Safely module, not exhaustively
- Custom health checks / Lua scripting (mentioned as "exists," not built)
- **Sync Waves & Lifecycle Hooks as a standalone module** — genuinely useful for stateful/multi-tier apps, but most teams running plain microservices on ArgoCD never need explicit ordering. Lifecycle Hooks survive as a short add-on inside Module 7; Sync Waves are cut entirely rather than kept as a token example.
- **Access, Secrets & Notifications as a required module** — real and commonly needed at some point, but not universally day-one for every learner. Kept as a complete optional module (see below) rather than core curriculum, so the required path stays focused on ArgoCD's core loop.

---

## Environment Strategy

| Environment | Used for | Why |
|---|---|---|
| **kind** (Kubernetes in Docker) | Modules 1–9 (all core concepts, rollback, promotion, multi-cluster) | Free, resets cleanly between takes/labs, no AWS account required to follow along |
| **AWS EKS via Terraform** | Module 10 (Production Infra) + Module 11 (Capstone) | Realistic cloud environment, teaches IaC skills, ties the course together on "real" infrastructure |

Every module before Production Infra runs entirely free on a student's own laptop. AWS is only required for the final production module and capstone, and every EKS lecture ends with `terraform destroy` on camera to avoid surprise billing.

---

## Section 0: Course Introduction *(orientation)*

- Course introduction and what you'll build
- Who this course is for / prerequisites (Kubernetes core concepts, YAML, Git basics; Helm familiarity recommended but not required)
- What is GitOps & ArgoCD (short teaser — Module 1 covers this in depth)
- How ArgoCD fits into a CI/CD pipeline (short teaser)
- Meet your instructor
- Resources, course repo, and how to get help

---

## Part 1: GitOps Foundations & Core Workflow `[kind]`

### Module 1: Why GitOps, Why ArgoCD
- The problem with push-based CI/CD pipelines (credentials in CI, no drift detection)
- Pull-based reconciliation model — Git as single source of truth
- Where ArgoCD fits: CI does build/test, ArgoCD does deploy
- **Lab:** None (discussion + diagram)

### Module 2: Installation & Architecture
- Core components: API server, repo-server, application controller, Redis
- Installing Docker + kind, spinning up a local cluster
- Installing ArgoCD via Helm, CLI login, UI tour
- **Lab:** Install Docker + kind, create a cluster, install ArgoCD, log in via CLI and UI

### Module 3: Your First Application
- Anatomy of the `Application` CRD (source, destination, project)
- Manual sync vs automated sync
- Prune and self-heal — introduced and demoed one at a time, each with its own before/after
- Reading the resource tree, health/sync status, and pod logs
- GUI walkthrough: resource tree, Diff, Sync panel, History and Rollback
- **Lab:** Deploy Finovra v1.0.0 — all four backend services plus the dashboard, all live — trigger manual sync, watch it go healthy, then enable automated/prune/selfHeal one at a time with a live demo of each

### Module 4: Deployment Sources — Helm & Kustomize
*(These two cover the vast majority of real-world repos — Jsonnet skipped entirely)*
- Deploying with a Kustomize base + overlay (a real patch, not just a rendered label)
- Deploying via Finovra's Helm chart (values files, parameter overrides)
- Choosing between plain YAML / Helm / Kustomize for a given team
- **Lab:** Convert the Module 3 deployment to a Kustomize overlay, then to Helm (demo both override mechanisms) — same live Application, three source types, landing on Helm as the ongoing baseline

---

## Part 2: Rollbacks, Failures & Promotion `[kind]`

### Module 5: Rollbacks & Failure Recovery — **Core Module**
Covered as three distinct, real-world techniques:

1. **Native ArgoCD rollback** — history tab, `argocd app rollback`, when this is appropriate (fast, but bypasses Git as source of truth)
2. **Git revert** — the "correct" GitOps way to roll back, keeping Git history honest
3. **Pausing reconciliation during an incident** — stopping ArgoCD from fighting an emergency `kubectl` hotfix while you stabilize, then reconciling Git afterward

- Reading failed sync/health states and diagnosing *why* before rolling back
- **Lab:** Deploy a deliberately broken dashboard release (staged in advance via a bad image tag, separate from the main version roadmap), recover it three ways

### Module 6: Progressive Delivery with Argo Rollouts
- Why plain ArgoCD sync isn't enough for risky deploys
- Canary and blue-green strategies (concepts only — pick one to lab)
- Automated rollback on failed analysis (metrics-based promotion/abort)
- **Lab:** Deploy the dashboard as a canary rollout, inject a failure, watch automatic rollback

### Module 7: Environment Promotion Patterns
- A deliberate step back from Module 6's canary Rollout to a plain Deployment, so promotion is the only new concept this module introduces
- Promoting dev → staging → prod: PR-based promotion workflow, with a manual approval gate before prod
- App-of-Apps pattern for managing multiple environments/apps from one root — built by hand first, then adopted by the root, so the problem it solves is visible before the fix is
- **Add-on: Lifecycle Hooks** — what `PreSync`/`PostSync` hooks are for (migrations, smoke tests) — concept plus one small example, not a full standalone treatment
- **Lab:** Build a 2-environment (staging/prod) promotion flow using per-environment Helm values files and a PR-based promotion; apply staging/prod by hand, delete them, then watch an App-of-Apps root adopt the same running resources without redeploying anything

### Module 8: Scaling to Many Apps — ApplicationSets (Essentials only)
- The problem ApplicationSets solve (don't hand-write 50 Application YAMLs)
- **Only** the two generators teams actually reach for: **List** and **Git directory** generators
- **Lab:** Generate one Application per Finovra backend service from a single ApplicationSet using the Git generator

---

## Part 3: Multi-Cluster, Production Infrastructure & Capstone `[kind → EKS + Terraform]`

### Module 9: Multi-Cluster Basics
- Registering an external cluster with ArgoCD
- Deploying the same app to two clusters (common in real setups: staging cluster + prod cluster)
- **Lab:** Spin up a second kind cluster, register it, deploy Finovra to both

### Module 10: Production Infra — Terraform + EKS
- Why Terraform for cluster provisioning (reproducibility, version control, teardown discipline)
- Terraform basics recap: providers, state, modules
- Provisioning EKS using the official `terraform-aws-modules/eks/aws` module (VPC, managed node group, cluster access entries)
- Installing ArgoCD on EKS via Helm, exposing the UI (LoadBalancer vs port-forward)
- Backing up ArgoCD config (`argocd admin export`) as part of a production go-live checklist
- **Lab:** Provision an EKS cluster with Terraform, install ArgoCD
- **Mini-lecture — Cost & Cleanup:** Running `terraform destroy` on camera, confirming nothing is left running in the AWS console.

### Module 11: Capstone Project — Online Boutique on EKS
This is where we switch to **Online Boutique** — its 11 real, polyglot services make for a genuinely impressive final showcase now that every core concept is already second nature. This is also where CI finally enters the picture:
1. Set up an App-of-Apps repo with staging + prod for Online Boutique
2. Deploy via Helm with automated sync
3. Wire up a GitHub Actions pipeline for one Online Boutique service — build, push, bump the tag in the manifests repo — closing the CI/CD loop for real, end to end
4. Push a broken change, diagnose, and recover using the appropriate rollback method
5. Promote a fix from staging to prod via PR
6. Wrap with `terraform destroy` to tear down the environment

---

## Optional Module: Running ArgoCD Safely — Access, Secrets & Notifications
*(Not part of the required path — genuinely useful, but not something every learner needs on day one. Take this on whenever it's relevant to you, any time after Module 3.)*
- RBAC basics: AppProjects as an access boundary, binding a role to a group — "get it working," not full policy authoring
- SSO/OIDC: what it solves and where it plugs in (concept-level; standing up a full identity provider is out of scope)
- Managing multiple repositories and their credentials
- External secrets: why committing plaintext secrets to Git is a real, common mistake, and one working example of pulling a secret from outside Git
- ArgoCD Notifications: a Slack alert on sync failure or degraded health
- **Lab:** Create a restricted AppProject, wire up one external secret, and configure one Slack notification trigger

---

## Optional Appendix (only if time/interest allows, not core curriculum)
- OCI registry support for Helm charts
- Source Hydrator (separating "dry" source from "hydrated" deployed manifests)
- Impersonation for multi-tenant clusters
- Advanced ApplicationSet generators (Matrix, SCM Provider, PR generator)
- Argo CD Agent for large multi-cluster fleets
- Full SSO/OIDC identity provider setup (beyond the concept-level treatment in the optional Running ArgoCD Safely module)
- Deep RBAC policy authoring (beyond the AppProject basics in the optional Running ArgoCD Safely module)
- Sync Waves entirely, and deeper Hooks patterns beyond Module 7's add-on (e.g. `SyncFail`/`PostDelete` hooks, wave-based multi-tier orchestration)
