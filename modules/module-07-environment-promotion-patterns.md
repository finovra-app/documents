# Module 7: Environment Promotion Patterns

**Environment:** `kind` (local)
**Prerequisites:** Module 6 complete — dashboard deployed as an Argo Rollouts canary, `Synced`/`Healthy` on `1.0.2`

---

## Learning Objectives

By the end of this module, you should be able to:
- Explain why most real teams organize multiple environments with directories/overlays on one branch, not one long-lived branch per environment
- Build `staging` and `prod` Kustomize overlays on top of Finovra's existing base, each with its own environment-specific overrides
- Set up a PR-based promotion flow, and explain the difference between the two separate gates a change passes through before it's live in prod
- Use the App-of-Apps pattern to manage multiple `Application` objects from one root, instead of `kubectl apply`-ing each by hand
- Use `sync-wave` annotations to control the order resources apply within one sync
- Explain what a `PreSync` hook is for, and why it isn't useful on every kind of sync

---

## 1. Branch-per-Environment vs. Overlay-per-Environment

There are two common ways teams structure "one app, three environments" in a GitOps repo, and the choice shapes everything else in this module.

**Branch-per-environment:** a long-lived `dev`, `staging`, and `prod` branch, each with its own copy of the manifests. Promotion = merging `dev` → `staging` → `prod`. This looks appealing at first — it mirrors how some teams already branch application code — but it comes with real problems in practice: three branches drift out of sync with each other over time, merge conflicts show up in manifests instead of in a clean diff, and it's easy to lose track of which branch is actually "ahead." ArgoCD also has to track three separate `targetRevision`s, one per `Application`.

**Overlay-per-environment (directory-per-environment):** one branch (`main`), with each environment as its own folder — `kustomize/overlays/dev`, `kustomize/overlays/staging`, `kustomize/overlays/prod` — all layered on the same shared base. Promotion = a PR that changes one file in one overlay folder. There's only ever one `main` to reason about, and a promotion PR's diff shows you *exactly* what's about to change in that environment — nothing more.

| | Branch-per-environment | Overlay-per-environment |
|---|---|---|
| Number of long-lived branches | 3+ | 1 (`main`) |
| Promotion mechanism | Merge one branch into another | PR that edits one overlay folder |
| Risk of drift | Branches silently diverge over time | Low — everything lives on `main` together |
| Diff clarity | Merge diff can include unrelated noise | PR diff is exactly the intended change |
| What most real teams use | Minority, mostly for other reasons (e.g. release branches) | **The default for GitOps config repos** |

This module builds the overlay-per-environment version — it's what Finovra's `kustomize/` layout has been quietly set up for since Module 4, and it's what you'll see in the overwhelming majority of real ArgoCD repos.

---

## 2. Promoting dev → staging → prod

Finovra's existing `finovra` Application (Helm-based, namespace `finovra`) has been "dev" all along, without ever needing the label — every module so far has deployed straight to it. This module adds two new environments alongside it: **staging** and **prod**, each its own `Application`, each pointed at its own Kustomize overlay.

**Dev deliberately stays exactly as it is — Helm-based, running `dashboard` as a canary `Rollout` — rather than getting rebuilt on Kustomize to match.** That does mean dev and staging/prod use different packaging tools, which is worth naming rather than glossing over: most real teams keep one tool per app across all its environments. It's a reasonable exception here specifically because dev is the environment where you've been canarying and deliberately breaking things; staging/prod are where the *promotion pattern* is the lesson, and rebuilding dev on Kustomize would mean either dropping the canary `Rollout` entirely or re-porting it into the Kustomize base — real extra work that teaches nothing new about promotion.

A promotion is nothing more than **a PR that bumps one value in one overlay** — normally the image tag, once you've proven it's good somewhere earlier in the chain:

```mermaid
flowchart LR
    Dev["dev\n(finovra ns)\nautomated sync"]
    PR1["PR: bump staging\noverlay's image tag"]
    Staging["staging\n(finovra-staging ns)\nautomated sync"]
    PR2["PR: bump prod\noverlay's image tag"]
    Prod["prod\n(finovra-prod ns)\nmanual sync only"]

    Dev -->|proven good| PR1
    PR1 -->|reviewed + merged| Staging
    Staging -->|proven good| PR2
    PR2 -->|reviewed + merged| Prod2["Git: prod overlay\nnow says 1.0.2"]
    Prod2 -.->|"still requires a\nhuman argocd app sync"| Prod
```

Notice there are **two separate gates**, not one, and they guard different things:

1. **The PR review gate** — a human approves the PR before it merges to `main`. This gates *what's allowed into Git*. Set this up the same way you would for any repo: on GitHub, enable branch protection on `main` with "Require a pull request before merging."
2. **The manual-sync gate on `prod`** — even after the PR merges and Git says prod should be on `1.0.2`, nothing actually deploys until someone runs `argocd app sync finovra-prod` (or clicks **Sync** in the UI). This is `syncPolicy` with no `automated:` block at all, deliberately left manual here.

That second gate is what the syllabus calls "a manual approval gate before prod," and it's a real, common pattern — merging code and deploying code are two different actions, and prod is exactly where teams want that gap to be explicit rather than automatic. `staging`, by contrast, stays `automated` — the whole point of a staging environment is that it deploys itself the moment something merges, so it stays a faithful preview of what prod is about to get.

---

## 3. App-of-Apps: One Root, Many Applications

You've been running one `Application` at a time, applied by hand with `kubectl apply -f apps/finovra.yaml`. That's fine for one environment. It gets tedious fast once you're managing three — and error-prone, since nothing stops someone from forgetting to apply one of them, or applying a stale copy.

**App-of-Apps** solves this by making the list of `Application` objects itself something ArgoCD manages: one root `Application` whose "app" is a folder of other `Application` manifests.

```yaml
# apps/root.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: finovra-root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/gitops.git
    targetRevision: main
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

This is the plain-YAML directory source type — nothing new there — except what it's managing is a folder of `Application` objects rather than a folder of Deployments/Services. Apply `root.yaml` once, and everything inside `apps/` (`finovra.yaml`, `finovra-staging.yaml`, `finovra-prod.yaml`) gets created and kept in sync automatically. Add a fourth environment later, and it's a fourth file in `apps/` plus a `git push` — no new `kubectl apply` command to remember.

`apps/` staying one level deep (only `Application` manifests, no subfolders) is what you want here — that's `source.directory.recurse`'s job to control, and its default is already `false`. **Don't write `directory: {recurse: false}` explicitly, even though it's tempting to be explicit about it:** `recurse` is a boolean that gets silently dropped whenever it's `false`, since `false` and "not set" serialize identically. Git would keep declaring it, the live `Application` object would never actually store it, and every reconciliation would see phantom drift and report `OutOfSync` forever, even though nothing is actually wrong — a real, easy-to-hit gotcha, not hypothetical. Leaving the field out entirely means there's nothing for that mismatch to happen to. If you ever need nested app-of-apps (a root managing other roots), that's when you'd set `recurse: true` for real — a non-default value serializes and persists just fine.

---

## 4. Add-on: Sync Waves & Lifecycle Hooks

Both of these solve a related problem — **ordering** — but at different scopes. A `sync-wave` controls the order resources within *the same sync* get applied. A hook runs a one-off task tied to a specific *phase* of a sync (before it starts, after it finishes, if it fails). Neither is something most plain-microservices teams reach for often — but Finovra's dashboard genuinely does depend on its four backend services being up first, which makes it a fair example to build once.

### Sync Waves

Every resource ArgoCD manages defaults to `sync-wave: "0"`. Resources in the same wave apply together, in no particular order; ArgoCD waits for one wave to be healthy before starting the next. Annotate the four backend Deployments and Services to stay at wave `"0"` (the default — no annotation needed), and bump `dashboard`'s Deployment and Service to wave `"1"`:

```yaml
# kustomize/base/dashboard-deployment.yaml (excerpt)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: dashboard
  annotations:
    argocd.argoproj.io/sync-wave: "1"
spec:
  # ...unchanged...
```

Do the same on `dashboard-service.yaml`. On the next full sync, ArgoCD applies all four backends first, waits until they're `Healthy`, *then* applies `dashboard` — instead of firing all ten resources at once and letting Kubernetes sort out the timing. This mostly matters on a **from-scratch deploy**: without it, `dashboard`'s Pods could start before the backend Services even exist, and its first few requests would just fail until the backends caught up (usually self-correcting within seconds, but visible in logs, and avoidable).

### Lifecycle Hooks

A hook is a Kubernetes `Job` (usually) that ArgoCD runs at a specific point in the sync lifecycle, identified purely by an annotation — `PreSync`, `Sync`, `PostSync`, or `SyncFail`. Here's a `PreSync` hook that checks every backend's `/healthz` before letting the rest of the sync proceed:

```yaml
# kustomize/base/backend-healthcheck-hook.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backend-healthcheck
  annotations:
    argocd.argoproj.io/hook: PreSync
    argocd.argoproj.io/hook-delete-policy: BeforeHookCreation
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: check
          image: curlimages/curl:8.9.1
          command:
            - sh
            - -c
            - |
              for svc in accounts-service insurance-service investments-service loans-service; do
                curl -sf "http://$svc:8000/healthz" || exit 1
              done
```

`hook-delete-policy: BeforeHookCreation` means ArgoCD deletes any previous run of this Job right before creating a new one — without it, the second sync would fail immediately with "Job already exists."

**The catch, worth knowing before you reach for this pattern elsewhere:** a `PreSync` hook runs *before any of the sync's own resources are applied*. On a genuinely first-ever deploy to an empty namespace, this hook would fail every time — the backend Services it's curling don't exist yet. It only makes sense on an environment that's already running, where you're validating the existing backends stay healthy before rolling out a *new* dashboard release on top of them — which is exactly staging and prod's situation, never a fresh `dev` bootstrap. (The other common use — `PostSync` hooks for a smoke test *after* everything's up, or a one-off DB migration — doesn't have this problem, since by definition everything the hook needs already exists. This module builds only the `PreSync` example; `PostSync` follows the identical pattern with `argocd.argoproj.io/hook: PostSync` instead.)

---

## Lab: Build a Staging → Prod Promotion Flow

All of this happens in your fork of `gitops`.

### Step 1 — Add the staging overlay

Create `kustomize/overlays/staging/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

replicas:
  - name: dashboard
    count: 2

images:
  - name: arsr319/finovra-dashboard
    newTag: "1.0.2"
```

Same two dedicated transformers (`replicas:`, `images:`) — no separate patch file needed here either. The `images:` line is the piece that makes this a *promotion* overlay rather than just another environment copy — it's the one line a promotion PR will actually touch going forward. Render it locally before moving on:

```bash
kubectl kustomize kustomize/overlays/staging
```

Confirm `dashboard`'s image reads `arsr319/finovra-dashboard:1.0.2` and its `replicas: 2`, while all four backends stay at whatever the base declares.

### Step 2 — Add the prod overlay

Create `kustomize/overlays/prod/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base

replicas:
  - name: dashboard
    count: 3

images:
  - name: arsr319/finovra-dashboard
    newTag: "1.0.0"
```

Note prod deliberately starts pinned to `1.0.0`, one step behind staging's `1.0.2` — that gap is what you'll close with a real promotion PR in Step 5. Render and sanity-check the same way as Step 1.

> **If Finovra used Helm for this instead:** you wouldn't need overlay directories at all — just a `values-staging.yaml` and `values-prod.yaml` sitting next to the existing `values-dev.yaml`, each overriding `dashboard.image.tag`:
>
> ```yaml
> # values-staging.yaml
> dashboard:
>   replicas: 2
>   image:
>     tag: "1.0.2"
> ```
>
> The `Application` would point at the same `helm-chart` path dev already uses, swapping in a values file instead of a Kustomize path:
>
> ```yaml
> spec:
>   source:
>     path: helm-chart
>     helm:
>       valueFiles:
>         - values-staging.yaml
> ```
>
> A promotion PR would then bump one line inside `values-prod.yaml` instead of `kustomize/overlays/prod/kustomization.yaml`'s `images.newTag` — same promotion mechanic, same two gates from Section 2, just expressed through Helm's override system instead of Kustomize's. This module builds the Kustomize version for real, since that's the tool most teams reach for specifically for environment overlays — but the pattern itself isn't tool-specific, and you'd land in the same place either way.

### Step 3 — Add the two new Applications

Create `apps/finovra-staging.yaml`:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: finovra-staging
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/gitops.git
    targetRevision: main
    path: kustomize/overlays/staging
  destination:
    server: https://kubernetes.default.svc
    namespace: finovra-staging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Create `apps/finovra-prod.yaml` — same shape, **no `automated:` block**:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: finovra-prod
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/gitops.git
    targetRevision: main
    path: kustomize/overlays/prod
  destination:
    server: https://kubernetes.default.svc
    namespace: finovra-prod
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
```

### Step 4 — Wrap them in an App-of-Apps root

Create `apps/root.yaml` (the exact file shown in Section 3). Commit and push everything from Steps 1–4 in one commit:

```bash
git add kustomize/overlays/staging kustomize/overlays/prod apps/finovra-staging.yaml apps/finovra-prod.yaml apps/root.yaml
git commit -m "Add staging/prod overlays and an App-of-Apps root"
git push origin main
```

Apply just the root — this is the last time you `kubectl apply` an `Application` by hand in this module:

```bash
kubectl apply -f apps/root.yaml
argocd app get finovra-root
```

Within moments, `finovra-staging` and `finovra-prod` should both exist as `Application` objects on their own, without you applying either directly:

```bash
argocd app list
```

Confirm `finovra-staging` shows `Synced`/`Healthy` on its own (it's `automated`). `finovra-prod` should show `OutOfSync` — that's expected: nothing has synced it yet, on purpose.

```bash
argocd app sync finovra-prod
argocd app get finovra-prod
```

That manual command **is** the approval gate from Section 2 — confirm `finovra-prod` settles to `Synced`/`Healthy` running `1.0.0`, one version behind staging's `1.0.2`.

### Step 5 — Run a real promotion

Open a PR (not a direct push to `main`) that changes exactly one line — `kustomize/overlays/prod/kustomization.yaml`'s `images.newTag`, from `"1.0.0"` to `"1.0.2"`:

```bash
git checkout -b promote-prod-1.0.2
# edit kustomize/overlays/prod/kustomization.yaml: newTag: "1.0.2"
git add kustomize/overlays/prod/kustomization.yaml
git commit -m "Promote prod to 1.0.2"
git push origin promote-prod-1.0.2
gh pr create --title "Promote prod to 1.0.2" --body "staging has been on 1.0.2 since Step 1 with no issues."
```

Review and merge it (through GitHub, same as any real PR). Then check ArgoCD:

```bash
argocd app get finovra-prod
```

**`Sync Status: OutOfSync`** — Git now says `1.0.2`, but prod is still running `1.0.0`. The PR merging didn't deploy anything; it only changed what Git declares. Only now does the second gate apply:

```bash
argocd app sync finovra-prod
argocd app get finovra-prod
kubectl get deployment dashboard -n finovra-prod -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Checkpoint:** you built a real two-environment promotion flow — a PR review gate on Git, and a separate manual-sync gate on ArgoCD before anything reaches prod — managed three `Application` objects from one root, and (if you added the sync-wave annotations and hook from Section 4) watched backends deploy before the dashboard on every one of these environments' next full sync.

---

## Key Terms Glossary

| Term | Meaning |
|---|---|
| **Overlay-per-environment** | One branch, one folder per environment, all layered on a shared Kustomize base — the pattern most real GitOps repos use over branch-per-environment |
| **Promotion** | A PR that changes one value (usually an image tag) in one environment's overlay, moving a proven release to the next environment |
| **PR review gate** | Branch protection requiring a reviewed, merged PR before a change lands on `main` — gates what's *allowed into Git* |
| **Manual-sync gate** | An `Application` with no `automated:` sync policy — gates what's *allowed to actually deploy*, independent of the PR gate |
| **App-of-Apps** | A root `Application` whose source is a folder of other `Application` manifests, so ArgoCD manages the app list itself instead of you applying each one by hand |
| **`sync-wave`** | An annotation (`argocd.argoproj.io/sync-wave`) controlling the order resources apply within one sync — lower numbers first, each wave waits for the previous to be healthy |
| **Hook** | A Job tied to a sync phase (`PreSync`, `Sync`, `PostSync`, `SyncFail`) via the `argocd.argoproj.io/hook` annotation — for one-off tasks like migrations or smoke tests, not ongoing workloads |
| **`hook-delete-policy`** | Controls whether/when ArgoCD deletes a completed hook Job — `BeforeHookCreation` clears the previous run so the next sync doesn't collide with it |

---

## Recap Questions

1. Why does a promotion PR's diff stay small and easy to review under the overlay-per-environment pattern, but not necessarily under branch-per-environment?
2. `finovra-staging` and `finovra-prod` are both `Synced` to the same Git commit at some point in Step 5. Why does only one of them actually have `1.0.2` running?
3. What specifically does the App-of-Apps root manage that a single `Application` doesn't?
4. Why would the `PreSync` backend-healthcheck hook fail on a brand-new environment's very first sync, and why isn't that a problem for a `PostSync` hook doing the same kind of check?
5. If `dashboard`'s Deployment didn't have a `sync-wave` annotation at all, what wave would it sync in, and what would that mean for its ordering relative to the four backends?

---

## What's Next

In **Module 8**, we stop hand-maintaining `apps/finovra-staging.yaml` and `apps/finovra-prod.yaml` as separate files and generate `Application` objects like them automatically with an `ApplicationSet` — the pattern that scales once you're managing many services or many environments instead of a handful of files in one folder.
