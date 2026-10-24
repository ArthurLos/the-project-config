# the-project-config

Kubernetes configuration for [`the_project`](https://github.com/ArthurLos/devops-with-kubernetes-2026/tree/main/the_project), the main project from the University of Helsinki "DevOps with Kubernetes" course (exercise 4.10, "the grande finale": split application code and configuration into separate repositories).

**Application source code, Dockerfiles, and per-service READMEs live in the other repo**: [ArthurLos/devops-with-kubernetes-2026](https://github.com/ArthurLos/devops-with-kubernetes-2026), under `the_project/`. This repo holds *only* the Kubernetes manifests, Kustomize overlays, and the ArgoCD `Application` definitions that deploy them — nothing here is built into a container image, nothing in the other repo is applied to the cluster directly.

## Why split like this

The code repo's CI (if it were building/pushing images) shouldn't need write access to deployment config, and the config repo shouldn't need to know how to build an image — they're different concerns with different change cadences (a typo fix in `todo-app`'s HTML doesn't need a new Kustomize overlay; bumping a CPU limit doesn't need a new Docker image). Keeping them in one repo makes that boundary easy to blur; a separate repo makes ArgoCD's job explicit: it watches *this* repo, never the application source.

## Structure

```
todo-backend/manifests/   todo-app/manifests/   wiki-reminder/manifests/   broadcaster/manifests/   db-backup/manifests/
  — one folder per microservice, each with its own small kustomization.yaml (just a resources: list)
    so the overlays below can pull all of them in as nested Kustomize bases.

overlays/
  staging/      — namespace: staging, tracks this repo's main branch
  production/   — namespace: production, tracks this repo's production branch, plus the broadcaster
                  webhook patch and the externally-reachable HTTPRoute

argocd-application-staging.yaml      — ArgoCD Application, targetRevision: main
argocd-application-production.yaml   — ArgoCD Application, targetRevision: production

.github/workflows/promote-production.yml
  — pushing a v* tag to THIS repo fast-forwards the production branch to that
    commit, which ArgoCD's production Application then syncs automatically.
    This repo's own tags (not the code repo's course-exercise tags) are what
    "each tagged commit results in deployment to production" means here.

manifests/, kustomization.yaml (at the repo root)
  — historical only, from before exercise 4.9 split this into staging/production
    and before 4.10 moved it here. Kept for older exercise tags in the code repo
    that still reference this layout.
```

## Deploying

```bash
# one-time: secrets are applied by hand, never through ArgoCD (per exercise 4.9)
kubectl create namespace staging
kubectl create namespace production
kubectl create secret generic postgres-credentials -n staging --from-literal=... # see exercise 4.9 in the code repo's the_project/README.md
kubectl create secret generic postgres-credentials -n production --from-literal=...
kubectl create secret generic broadcaster-webhook -n production --from-literal=WEBHOOK_URL=...

# register both environments with ArgoCD (already installed on the cluster)
kubectl apply -f argocd-application-staging.yaml
kubectl apply -f argocd-application-production.yaml
```

From then on: commit + push to `main` → staging redeploys automatically; push a `v*` tag → production redeploys automatically. See the code repo's `the_project/README.md` ("Exercise 4.9"/"Exercise 4.10" sections) for the full history, design rationale, and live test screenshots.
