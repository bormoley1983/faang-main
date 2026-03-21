# Deployment Guide

## Goal

Build a stable CI/CD path for the FAANG solution on k3s while keeping the process understandable, repeatable, and safe to document as part of the project.

This document is intentionally written as a runbook:

- what to verify
- what to change first
- what to postpone
- what is safe to document in Git

## Target Operating Model

### Ownership model

- Each service repository owns its own CI build.
- The deployment repository owns the Kubernetes desired state.
- Only the changed service image tag or digest gets updated, so only that deployment rolls.

### Platform model

- `k3s` is the target cluster.
- `ArgoCD` owns deployment synchronization.
- `Jenkins` owns builds and manifest update automation.
- `Harbor` is the preferred container registry platform.
- `MinIO` remains a separate object store and can later be used by Harbor as S3-compatible backend storage.

### Configuration model

- `faang-infra/k8s/base` stays generic.
- `faang-infra/k8s/overlays/homelab` becomes the real environment layer.
- Secrets are not stored as plain YAML in public Git.
- First-time bootstrap is separated from normal deployment sync.

## Why This Model

This gives the cleanest story for a portfolio or demo setup:

- code changes trigger builds
- builds produce immutable images
- deployment manifests are updated in one central place
- ArgoCD reconciles the cluster to the desired state
- environment-specific secrets and private values stay out of public manifests

It also avoids the main problem of the current repo: rebuilding and redeploying everything on every change.

## Safe Documentation To Keep In Repo

What is safe to document:

- how to point `kubectl` to the cluster
- namespace layout
- DNS naming conventions
- external infrastructure hostnames
- CI/CD flow
- bootstrap sequence
- what files and values must never be committed

What must not be committed:

- kubeconfig with real credentials
- `.env` files with secrets
- Kubernetes Secret manifests with real values
- Jenkins credentials
- Harbor admin credentials
- private keys for SOPS or other secret tooling

## Suggested Repo Snippet

This kind of snippet is good to keep in docs because it teaches the process without leaking credentials:

```powershell
# Example only. Do not commit real kubeconfig files.
mkdir "$HOME\.kube" -ErrorAction SilentlyContinue
scp <ssh-user>@192.168.100.14:/etc/rancher/k3s/k3s.yaml "$HOME\.kube\config-homelab"
(Get-Content "$HOME\.kube\config-homelab") -replace '127.0.0.1', '192.168.100.14' | Set-Content "$HOME\.kube\config-homelab"
$env:KUBECONFIG = "$HOME\.kube\config-homelab"

kubectl config current-context
kubectl get nodes -o wide
```

Add one warning next to it:

> Never commit the generated kubeconfig file into Git.

## Current Deployment Direction

### External infrastructure

The cluster will use already-running infrastructure discovered via DNS:

- `postgres-main`
- `redis-main`
- `kafka-main`
- `elasticsearch-main`
- `minio-main`

These names may also resolve with the homelab domain, for example:

- `postgres-main.office.aviv.com.ua`

Because these systems already exist, the application manifests should consume them as external dependencies instead of trying to redeploy them inside k3s by default.

### Missing dependency

`user_service` is a known missing dependency and will be added later.

This means the first rollout should be planned carefully:

- either exclude the services that strictly require `user_service`
- or accept partial functionality until that service is added

## Phased Plan

## Phase 0: Verify Cluster and Platform State

Run these commands from the shell that is actually connected to your cluster:

```powershell
kubectl config current-context
kubectl cluster-info
kubectl get nodes -o wide
kubectl get ns
kubectl get ingressclass
kubectl get pods -A -o wide
kubectl -n argocd get applications
kubectl -n argocd get pods,svc
kubectl -n jenkins get pods,svc
```

Verify external DNS from inside the cluster:

```powershell
kubectl run dns-test --image=busybox:1.36 --restart=Never --command -- sleep 3600
kubectl exec dns-test -- nslookup postgres-main
kubectl exec dns-test -- nslookup redis-main
kubectl exec dns-test -- nslookup kafka-main
kubectl exec dns-test -- nslookup elasticsearch-main
kubectl exec dns-test -- nslookup minio-main
kubectl delete pod dns-test
```

If you want to verify HTTP endpoints:

```powershell
kubectl run curl-test --image=curlimages/curl --restart=Never --command -- sleep 3600
kubectl exec curl-test -- curl -I http://elasticsearch-main:9200
kubectl exec curl-test -- curl -I http://minio-main:9000
kubectl delete pod curl-test
```

## Phase 1: Normalize the Kubernetes Layout

Create a dedicated runtime namespace:

- `faang`

Keep platform namespaces separate:

- `argocd`
- `jenkins`
- `cattle-system` or Rancher namespace
- `harbor` later

### Rule for manifests

- `base` must not contain real domains
- `base` must not contain real secrets
- `base` must not contain environment-specific infra values
- `base` must not use `latest`

Environment-specific values belong in:

- `faang-infra/k8s/overlays/homelab`

## Phase 2: Split Bootstrap From Regular Deployments

Bootstrap tasks should not run on every sync:

- create PostgreSQL database and schemas
- create Kafka topics
- create Elasticsearch indices
- create MinIO buckets and policies if needed

Regular deployment should only do:

- apply deployment manifests
- roll updated image references
- reconcile services, ingress, config, and secrets references

### Result

Have two clearly separate workflows:

1. Bootstrap workflow
2. Normal application deployment workflow

## Phase 3: Remove `latest`

This is a mandatory improvement.

### Desired image pattern

Use one of these:

- `harbor.example.local/faang/post-service:<git-sha>`
- `harbor.example.local/faang/post-service@sha256:<digest>`

Do not use:

- `:latest`

### Why

- Kubernetes does not reliably roll workloads when only `latest` is overwritten.
- ArgoCD works best when manifests change explicitly.
- Immutable image references make rollbacks and debugging much easier.

## Phase 4: Introduce Harbor

### Why Harbor over plain Docker Registry

Harbor gives more than raw image storage:

- projects and access control
- vulnerability scanning support
- better UI
- image retention and governance
- easier team-friendly registry management

### Role of MinIO

MinIO stays separate.

Harbor may later use MinIO as backend object storage, but MinIO itself is not the registry service.

### Short-term note

If Harbor is not installed yet, the deployment plan should still assume Harbor is the destination registry so the manifests and CI logic are designed correctly from the start.

## Phase 5: Decide Secret Strategy

There are two practical paths.

### Recommended path

Use:

- private deployment repo or private overlay path
- encrypted secrets with `SOPS`

Why this is best:

- stable
- auditable
- GitOps-friendly
- safe to share the process publicly without leaking values

### Transitional path

Create runtime secrets manually outside Git:

- `kubectl create secret ...`
- keep local secret material in ignored files

Why this helps:

- fast to bootstrap
- no need to adopt SOPS immediately

Tradeoff:

- not full GitOps
- harder to reproduce automatically

## Phase 6: Fix the Manifest Contract

Before first real deployment, align the manifests with the applications.

Fix at least these areas:

- bad environment variable wiring
- wrong ports
- `S3_*` vs `MINIO_*`
- service-to-service host and port values
- `POSTGRES_PORT` reference bug in `achievement-service`
- placeholder secret usage in `base`

Also make sure the overlay, not local shell substitution, supplies:

- base domain
- runtime config
- private environment values

## Phase 7: CI/CD Flow

### CI

Each service repo pipeline should:

1. checkout code
2. run tests and build
3. build image
4. push immutable image to Harbor
5. update the deployment repo image reference for that service

### CD

ArgoCD should:

1. watch the deployment repo overlay
2. detect manifest changes
3. sync only the changed workload

### Expected deployment flow

1. code pushed to service repo
2. Jenkins builds image
3. Jenkins pushes image to Harbor
4. Jenkins updates deployment manifest
5. ArgoCD syncs cluster
6. only changed deployment rolls

## Phase 8: Documentation Deliverables

To show knowledge and architecture maturity, document these clearly:

### 1. Platform topology

- k3s control plane
- Jenkins
- ArgoCD
- Rancher
- Harbor
- external infra dependencies

### 2. Runtime topology

- app namespace
- service-to-service communication
- ingress strategy
- external infra via DNS

### 3. Deployment flow

- CI build
- image push
- manifest update
- Argo sync

### 4. Bootstrap flow

- database init
- Kafka topics
- Elasticsearch index
- MinIO bucket setup

### 5. Secrets policy

- what is stored where
- what is never committed
- how runtime secrets are created

## Immediate Next Steps

1. Verify the cluster and namespace state from your connected shell.
2. Confirm whether `argocd` and `jenkins` are healthy.
3. Decide whether Harbor will be installed now or in the next phase.
4. Refactor the manifests so `homelab` owns the real environment config.
5. Remove plain secret values from `base`.
6. Replace `latest`-based image references with an immutable tagging strategy.
7. Separate bootstrap jobs from normal deployment sync.

## Practical Note About This Workspace

At the moment, the shell used by this agent is not inheriting your local `kubectl` context automatically.

That means:

- your terminal may already be connected to k3s
- my current tool shell still sees no active Kubernetes context

So for cluster inspection, either:

- run the commands from your connected shell and paste back the output
- or re-export the same `KUBECONFIG` in the shell session used for tooling here

## What We Will Do Next

Once the cluster state is confirmed, the implementation order should be:

1. fix repository structure for overlays and environment config
2. fix manifest contract issues
3. prepare secret strategy
4. prepare Harbor adoption path
5. rewrite Jenkins and ArgoCD flow around immutable images

That order keeps the architecture stable before we automate anything.
