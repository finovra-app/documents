# Finovra — Course Documents

Course syllabus and module docs for the ArgoCD/GitOps Udemy course. This repo
holds reference material only — nothing here is watched by ArgoCD or built
into an image.

Application source code and the syllabus's app-vs-config split are covered by
two other repos:

- [`finovra-app/webapp`](https://github.com/finovra-app/webapp) — service
  source code, Dockerfiles
- [`finovra-app/gitops`](https://github.com/finovra-app/gitops) — Kubernetes
  manifests, the ArgoCD `Application` resource(s)

## Layout

```
documents/
├── ArgoCD_Crash_Course_Syllabus.md
└── modules/
    ├── module-00-course-introduction.md
    ├── module-01-why-gitops-why-argocd.md
    ├── ...
    └── reference-ci-pipeline-github-actions.md
```
