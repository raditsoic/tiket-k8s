# tiket-k8s — GitOps source of truth for the tiket lab cluster

The Kubernetes workload manifests for the `tiket` lab live here as a Helm
chart. ArgoCD (running in-cluster on the lab's control-plane VM, `cp`)
renders the chart under [`tiket/`](tiket/) with its bundled Helm (nothing
Helm-related is installed in the cluster) and syncs the cluster to it
automatically.

## What's here

| Path | Purpose |
|---|---|
| `tiket/Chart.yaml` | chart metadata (name `tiket`) |
| `tiket/values.yaml` | the knobs: `image.tag` (CI bumps it per deploy), `image.registry` (`10.0.2.2:5000`), `replicas`, `service.nodePort` |
| `tiket/templates/` | Deployment (2 replicas spread one-per-worker) + NodePort Service (`:30080`) — HAProxy on `cp` load-balances across the workers |

## Deploy flow

Push to `main` → ArgoCD picks it up on its next poll (~3 min) and syncs the
cluster. No `kubectl` needed: the cluster converges to whatever this repo
says. Auto-sync runs with **prune** (objects deleted from the repo are
removed from the cluster) and **self-heal** (manual `kubectl` mutations are
reverted to the repo state).

Preview locally (no cluster needed): `helm lint tiket/` and
`helm template tiket tiket/` from the repo root.

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
