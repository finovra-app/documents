# Module 0: Course Introduction

**Environment:** None — just read
**Prerequisites:** None yet — this module tells you what you'll need before Module 2

---

## What You'll Build

Across this course you'll deploy, break, roll back, and promote **Finovra** — a
small demo fintech app — a personal finance dashboard. It's not a production
app with a real database; it exists so every GitOps concept in this course
has something concrete and visual to happen to.

Finovra is 5 tiny microservices:

| Service | Language | What it does |
|---|---|---|
| `dashboard` | Node.js | Renders a dark-themed page with one tile per backend service |
| `accounts-service` | Node.js | Sample balance + transactions |
| `insurance-service` | Node.js | Sample insurance policies |
| `investments-service` | Python (FastAPI) | Sample investment portfolio |
| `loans-service` | Python (FastAPI) | Sample loan balance |

The dashboard polls each backend's `/healthz` and `/version` endpoint every
few seconds. All four backends deploy together from the very first lab — every
tile shows real, live data as soon as Finovra is up. If a tile ever does grey
out with **"Unavailable"**, that's not a placeholder — it means that backend
is genuinely down. That's a deliberate design choice: it's what makes drift,
failure, and rollback exercises later in this course visible on screen
instead of something you have to take on faith from a `kubectl` command.

### Architecture

```mermaid
flowchart LR
    User((You: Browser)) -->|HTTP| Dashboard[dashboard]
    Dashboard -->|"GET /healthz, /version"| Accounts[accounts-service]
    Dashboard -->|"GET /healthz, /version"| Insurance[insurance-service]
    Dashboard -->|"GET /healthz, /version"| Investments[investments-service]
    Dashboard -->|"GET /healthz, /version"| Loans[loans-service]
    style Dashboard fill:#6cf,stroke:#333
```

Every arrow out of `dashboard` is independent — each backend can be deployed,
broken, rolled back, or removed entirely without touching the others or the
dashboard's own code. That independence is what makes Finovra useful for
practicing GitOps on: every operation you perform targets one Kubernetes
Deployment/Service pair, with an immediate, visible effect on exactly one
tile.

### The version story

Unlike most demo apps, Finovra's four backend services don't get their own
version numbers — they stay pinned at `1.0.0` for almost the entire course.
Instead, it's the **`dashboard`** that carries a new version each time you
reach a module that needs one, and each version adds one real, visible
feature — not a new service, just like a real release usually does:

| Version | Adds | Introduced in |
|---|---|---|
| `1.0.0` | Baseline dashboard, all four tiles live | Module 3 |
| `1.0.2` | Dashboard repackaged as an Argo Rollouts canary; the version staging/prod get promoted to | Module 6 |

Two modules also use their own separate, one-off broken releases as
practice material — both deliberately off this roadmap, and both
recovered from before their module ends. Module 5 uses
`arsr319/finovra-dashboard:1.0.1`. Module 6 uses its own separate broken
build, `1.0.3`, to demonstrate an automated canary rollback.

Every image is already built and published for you on Docker Hub as
`arsr319/finovra-<service>:<version>` — for example
`arsr319/finovra-accounts-service:1.0.0`. **You never need to build anything
yourself** in the required path — CI (building an image yourself with GitHub
Actions) is covered in the **Capstone** project instead, once you're working
with a real pipeline end to end. Until then, every lab just points ArgoCD at
a pre-built image tag.

---

## Who This Course Is For

DevOps engineers and developers who already know the basics of Kubernetes
(Pods, Deployments, Services) and Git, and want hands-on, practical GitOps
skills — not a certification-style tour of every ArgoCD feature. If you've
never used `kubectl` or committed to a Git repo before, do that first.

## Prerequisites

- Docker Desktop (or Docker Engine on Linux)
- Basic familiarity with Kubernetes (Pods, Deployments) and Git — Module 1
  assumes you know what these are, not how to master them
- A GitHub account (free) — you'll fork the [Finovra GitOps repo](https://github.com/finovra-app/gitops)
  in Module 3 to practice the GitOps commit loop yourself
- Nothing else yet — Module 2 walks you through installing `kind`, `kubectl`,
  `helm`, and ArgoCD itself from scratch

---

## A Two-Sentence Preview of GitOps & ArgoCD

**GitOps** means Git is the single source of truth for what should be
running, and an agent *inside* your cluster continuously pulls and applies
that state — nothing outside the cluster ever pushes into it. **ArgoCD** is
that agent: a Kubernetes controller that watches a Git repo (like the Finovra
GitOps repo you'll fork) and keeps your cluster in sync with it. Module 1 covers
both in full depth — this is just enough to orient you.

## Where ArgoCD Fits in CI/CD

Your CI pipeline (GitHub Actions, in this course) builds, tests, and pushes
container images — it never touches your cluster. ArgoCD does the other
half: watching the manifests repo and deploying what it finds. Module 1
covers this split with a full diagram.

---

## Meet Your Instructor

_[Add 2–3 sentences here: your background, why you teach this, and what
makes your take on ArgoCD practical rather than theoretical.]_

## Resources, Course Repo & Getting Help

- **Course repo (app code, Dockerfiles):** https://github.com/finovra-app/webapp
- **GitOps repo (Kubernetes manifests + Application definitions — this is the one you'll fork):** https://github.com/finovra-app/gitops
- **Documents repo (these module docs + the course syllabus):** https://github.com/finovra-app/documents
- **Pre-built images:** [hub.docker.com/u/arsr319](https://hub.docker.com/u/arsr319) — every `finovra-*` image referenced in this course
- **Questions during the course:** _[Udemy Q&A / Discord / email — fill in your preferred support channel]_

---

## What's Next

**Module 1: Why GitOps, Why ArgoCD** — no lab, just the problem GitOps
solves and where ArgoCD fits, in depth.
