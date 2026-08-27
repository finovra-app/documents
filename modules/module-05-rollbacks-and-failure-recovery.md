# Module 5: Rollbacks & Failure Recovery

**Environment:** `kind` (local)
**Prerequisites:** Module 4 complete — Finovra deployed via its Helm chart, with `automated`, `prune`, and `selfHeal` all enabled

---

## Learning Objectives

By the end of this module, you should be able to:
- Explain why ArgoCD's `Health Status` alone doesn't prove a release is actually working
- Perform a native ArgoCD rollback, and explain exactly what it does and doesn't fix
- Pause ArgoCD's reconciliation during an incident, apply an emergency mitigation, and resume cleanly once a real fix is ready
- Roll back the "correct" GitOps way, with `git revert`, and explain why it's preferred over rewriting history

---

## 1. "Healthy" Doesn't Mean "Working"

This module's practice material is `arsr319/finovra-dashboard:1.0.1` — a real, deliberately broken build.

The bug: `public/index.html`'s client-side JS calls `fetch("/api/tile")` instead of `fetch("/api/tiles")` — a typo'd, non-existent endpoint. The request 404s. The failure is silently caught. The page just never finishes loading its content.

Here's what makes this bug worth building a module around: **deploy it, and ArgoCD tells you everything is fine.**
- The container starts
- `/` still returns `200`, so liveness and readiness probes pass
- `Sync Status` reads `Synced`
- `Health Status` reads `Healthy`

Every signal `kubectl get pods` and `argocd app get` can give you says this release is good. The dashboard is visibly broken the moment you open it in a browser.

**ArgoCD's health checks tell you the container is running the way Kubernetes expects — not that the application is doing its job correctly.** A crash-looping Pod is easy to catch. A Pod that starts fine and quietly serves a broken page is not, and it's a far more common real-world failure mode than a crash. Before reaching for any rollback technique, the actual diagnostic step is: **look at the thing.**

---

## 2. Technique 1: Native ArgoCD Rollback

ArgoCD remembers every sync it's ever performed. That history powers the UI's **History and Rollback** tab and the `argocd app rollback` command. Rolling back is, in principle, as simple as pointing at an earlier entry:

```bash
argocd app history finovra
argocd app rollback finovra <ID>
```

There's a catch worth hitting deliberately rather than just reading about: if `automated` sync is enabled, ArgoCD refuses outright.

```
rollback cannot be initiated when auto-sync is enabled
```

This makes sense: automated sync's job is "make the cluster match the latest commit in Git." A native rollback tells the cluster to run something *other* than the latest commit — those two instructions directly conflict. You have to disable automated sync first:

```bash
argocd app set finovra --sync-policy none
argocd app rollback finovra <ID>
```

Here's the real lesson of this technique. Once the rollback completes, the *cluster* is back on the good version — but `Sync Status` now reads `OutOfSync`, because **Git still has the bad commit at `HEAD`.** You fixed what's running without fixing what Git says should be running. Anyone who looks at the repo still sees the broken release as "current." If automated sync gets re-enabled before a real fix lands, ArgoCD will redeploy the broken version right back — Git is still the source of truth, and native rollback never touched it.

Native rollback is the **fastest** way to get a good version running again. But it's a live-cluster action that leaves Git lying about reality until someone follows up with a real fix.

---

## 3. Technique 2: Pausing Reconciliation During an Incident

Native rollback fixes the cluster but not Git. If nobody follows up, the next time automated sync gets re-enabled, ArgoCD dutifully redeploys the broken release again — because Git still says that's correct. That's the situation this technique deals with: **the incident isn't actually over, and you need to buy time without ArgoCD undoing whatever you do next.**

With `selfHeal` enabled, ArgoCD fights any manual `kubectl` change that isn't reflected in Git. Watch this happen — try to take the broken dashboard offline so it stops serving broken pages to users:

```bash
kubectl scale deployment dashboard -n finovra --replicas=0
```

Within seconds, it's back at `1/1`. `selfHeal` did exactly what it's supposed to do: Git still says `replicas: 1`, so ArgoCD reverted the change. Correct behavior in general, actively unhelpful in this specific moment.

**The fix: pause reconciliation before making the emergency change.**

```bash
argocd app set finovra --sync-policy none
kubectl scale deployment dashboard -n finovra --replicas=0
```

This time it sticks. `Sync Status` flips to `OutOfSync` — ArgoCD notices the drift and says so — but nothing reverts it, because there's no automated sync left to do the reverting.

**Be clear about what this did and didn't do:** scaling to zero doesn't fix anything. Nobody gets a working dashboard — they get no dashboard at all. This is containment, not a fix. The actual fix is still Technique 3, next.

---

## 4. Technique 3: Git Revert — The Fix

`git revert` fixes the actual problem: Git's dishonesty. Instead of telling the *cluster* to ignore Git, you create a new commit that undoes the bad one.

```bash
git revert <bad-commit-sha>
git push origin main
```

With automated sync re-enabled, ArgoCD picks this up on its own. No `argocd app sync`, no `argocd app rollback` — nothing that touches the live `Application` object directly. `Sync Status` and `Health Status` both settle back to `Synced`/`Healthy`, and this time they mean it: Git and the cluster agree, and Git's history stays honest — it shows a bad release *and* the commit that reverted it, not a rewritten past.

This is the technique most teams reach for by default. It's slower than a native rollback, since you're waiting on the normal Git → sync path, but it never leaves a gap between what's running and what Git claims is running.

> **Why not just `kubectl edit` the broken code directly?** You can't — the bug is baked into the container image itself. A `kubectl` hotfix can change what's *running* (replica count, resource limits, env vars, taking a bad Pod out of rotation), but it can't rewrite application code sitting inside an already-built image. The defect always needs a real fix through Git.

---

## Lab: Break It, Then Recover It

All of this happens in your fork of `gitops`. You'll only need **two commits** for the whole lab — one to introduce the bad release, one to fix it for real.

### Step 1 — Deploy the bad release

Edit `helm-chart/values.yaml`: find the **`dashboard:`** block specifically and change its `image.tag` from `""` to `"1.0.1"`.

```yaml
dashboard:
  replicas: 1
  image:
    tag: "1.0.1"   # was ""
```

**Change only this one line.** `accounts-service`, `insurance-service`, `investments-service`, and `loans-service` each have their own `image.tag: ""` line in the same file. There's also a *global* `image.tag` near the top of the file — don't touch that one either, it's the fallback every service uses when its own tag is empty. Editing the global tag, or find-and-replacing `tag: ""` across the whole file, bumps all five services at once. Only `dashboard` has a `1.0.1` build on Docker Hub — the other four will fail to pull.

Because the template derives both the image tag *and* the `VERSION` env var from this one field, this single line is enough.

```bash
git add helm-chart/values.yaml
git commit -m "Bump dashboard to 1.0.1"
git push origin main
```

Wait for automated sync to pick it up, then confirm the trap:

```bash
argocd app get finovra
```

`Sync Status: Synced`, `Health Status: Healthy` — looks completely fine. Now open the dashboard in your browser. The header shows `v...` forever and the tile grid never populates. That's the diagnostic step from Section 1: the status line told you something true (the container's running), which you might mistake for something false (the app works).

### Step 2 — Recover with native rollback

```bash
argocd app history finovra
argocd app set finovra --sync-policy none
argocd app rollback finovra <ID-of-the-1.0.0-revision>
```

Confirm the dashboard is visibly fixed, then confirm the catch:

```bash
argocd app get finovra
```

`Sync Status: OutOfSync` — the cluster's fixed, Git isn't. Leave it like this for a moment before moving on.

### Step 3 — See the incident come back, then contain it

Sync is still disabled from Step 2, and Git still has `1.0.1` at `HEAD`. Re-enable automated sync and watch what happens:

```bash
argocd app set finovra --sync-policy automated --auto-prune --self-heal
argocd app sync finovra
argocd app get finovra
```

The dashboard goes broken again — `Health Status: Healthy`, visibly not working. No new commit was needed: Git already had `1.0.1` declared, and automated sync just did its job — redeploying whatever Git says, regardless of whether that's actually good.

Now run the before/after from Technique 2. First, the unprotected attempt:

```bash
kubectl scale deployment dashboard -n finovra --replicas=0
kubectl get deployment dashboard -n finovra -w
```

Press `Ctrl+C` once you see it bounce back to `1/1` — `selfHeal` reverted it. Now the protected version:

```bash
argocd app set finovra --sync-policy none
kubectl scale deployment dashboard -n finovra --replicas=0
kubectl get deployment dashboard -n finovra
```

Confirm it stays at `0/0`. The dashboard is offline — contained, not fixed.

### Step 4 — Ship the real fix

Find the bad commit's SHA (`git log --oneline`), then, while still paused:

```bash
git revert <bad-commit-sha>
git push origin main
kubectl get deployment dashboard -n finovra
```

Confirm nothing has changed yet — still paused, still `0/0`. Then hand control back to ArgoCD:

```bash
argocd app set finovra --sync-policy automated --auto-prune --self-heal
argocd app get finovra
```

Everything converges in one reconciliation: replicas back to `1`, image back to `1.0.0`, `Sync Status: Synced`, `Health Status: Healthy` — genuinely this time, because Git and the cluster now agree.

**Checkpoint:** you've recovered from the same broken release three different ways — a fast cluster-only fix, a contained incident, and a real Git-based fix — using two commits total, and watched each technique's specific tradeoff play out for real.

---

## Key Terms Glossary

| Term | Meaning |
|---|---|
| **Native rollback** | `argocd app rollback` — fastest recovery, but only changes the live cluster; Git still shows the bad release as current until you fix it separately |
| **`--sync-policy none`** | Disables automated sync on the live `Application` object without touching Git or your local manifest file |
| **Reconciliation pause** | Temporarily disabling automated sync so an emergency `kubectl` mitigation isn't immediately reverted by `selfHeal` |
| **Containment vs. a fix** | Containment (like scaling to zero) stops the damage; it doesn't restore working service. Only a real fix — via rollback or Git revert — does that |
| **Git revert** | A new commit that undoes a bad one — slower than native rollback, but keeps Git and the cluster in agreement the whole time |
| **Health Status vs. actual correctness** | A Pod can be `Healthy` (passing its probes) while the application it's running is completely broken — probes check what you told them to check, nothing more |

---

## Recap Questions

1. Why did `Health Status` stay `Healthy` throughout this entire module, even while the dashboard was visibly broken?
2. Why does ArgoCD refuse to run `argocd app rollback` while automated sync is enabled?
3. In Step 3, why did the dashboard go broken again the moment you re-enabled automated sync, without you pushing any new commit?
4. In Step 3, the exact same `kubectl scale --replicas=0` command produced two different outcomes. What changed in between, and why?
5. Scaling the dashboard to zero stops it serving broken pages. Why isn't that the same thing as fixing it?

---

## What's Next

In **Module 6**, we look at progressive delivery with Argo Rollouts — deploying the dashboard as a canary release, injecting a failure, and watching an automated analysis catch and roll it back before it ever reaches full rollout, without anyone running a manual rollback command at all.
