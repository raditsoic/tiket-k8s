# tiket-k8s — GitOps source of truth for the tiket lab cluster

The Kubernetes workload manifests for the `tiket` lab live here. ArgoCD
(running in-cluster on the lab's control-plane VM, `cp`) watches this repo and
applies everything under [`tiket/`](tiket/) to the cluster automatically.

## What's here

| File | Purpose |
|---|---|
| `tiket/deployment.yaml` | `tiket-app` Deployment: 2 replicas spread one-per-worker, registry prefix `10.0.2.2:5000` |
| `tiket/service.yaml` | `tiket-app` NodePort Service (`:30080`) — HAProxy on `cp` load-balances across the workers |

## Deploy flow

Push to `main` → ArgoCD picks it up on its next poll (~3 min) and syncs the
cluster. No `kubectl` needed: the cluster converges to whatever this repo
says. Auto-sync runs with **prune** (objects deleted from the repo are
removed from the cluster) and **self-heal** (manual `kubectl` mutations are
reverted to the repo state).

## CI flow

Jenkins (building `Raditsoic/tiket-app`) pushes the image-tag bump commit to
this repo on every `main` build using the `tiket-ci` deploy key — that commit
**is** the deployment. See the `vagrant-lab/k8s` README, section CI/CD, for
the full pipeline.

## Rollback

`git revert` the tag-bump commit and push — ArgoCD rolls the Deployment back.
Rollbacks are plain git operations; the cluster follows the repo.

## Note on secrets

The app Secret (`tiket-app-env`, DB credentials) is **not** in this repo —
it carries vault values and is applied out-of-band by Ansible from the
`vagrant-lab` vault. This repo must stay secret-free so it can stay public
(ArgoCD reads it anonymously).
