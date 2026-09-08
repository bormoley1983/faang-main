# FAANG deployment and operator guide

This is the canonical public entry point for delivery operations. It describes
the verified manual GitOps process without exposing private topology,
credentials, kubeconfig data, encrypted-secret material, or backup locations.

## Operating model

- Reuse the existing k3s, Rancher, Argo CD, Longhorn, MetalLB, Traefik,
  cert-manager, Jenkins, and Distribution registry installations.
- Jenkins builds, tests, signs, and proposes immutable image-digest updates; it
  never applies Kubernetes manifests or syncs Argo CD.
- Argo CD reconciles reviewed Git revisions manually. Automated sync, prune,
  self-heal, force, and replace are disabled.
- Public Git contains portable configuration. Real topology and encrypted
  runtime Secrets belong in the protected private environment source.

## Current Argo ownership

| Application | Responsibility |
| --- | --- |
| `faang-system` | Retained local-backup `s3-main` only |
| `faang-runtime-foundation` | `Namespace/faang` and `ConfigMap/faang-config` |
| `faang-selected-dependencies` | Dependency-selection metadata and external aliases |
| `faang-bootstrap` | Idempotent bootstrap ServiceAccount, scripts, and Jobs |
| `faang-workloads` | Nine application Deployments, Services, and Ingress |
| Existing named stateful, storage, object-storage, and secrets apps | Their already-scoped resources only |

`s3-main` is retained local backup storage, not the active application S3
endpoint. Do not move or prune it during ordinary delivery.

## Approved release path

1. A service’s trusted Jenkins delivery publishes a signed immutable image
   digest and proposes the matching GitOps update.
2. Review and merge the proposal to Argo CD’s watched branch.
3. Run the repository validation gates; see
   [`ops/validation/README.md`](faang-infra/ops/validation/README.md).
4. In Argo CD, refresh only the affected Application and inspect its exact
   diff, revision, health, resource set, and any proposed prune.
5. With explicit approval, manually sync only that Application with Prune,
   Force, and Replace disabled. Require `Synced/Healthy`.
6. Collect sanitized read-only post-deployment evidence.

Never bulk-sync all Applications. Do not use `kubectl apply -k` as the normal
delivery path, and stop if ownership or a destructive diff is unclear.

## Dependency selection and secrets

PostgreSQL, Redis, Kafka, and Elasticsearch currently use external aliases.
Application S3 uses the separate internal `faang-object-storage` endpoint.
Profile changes are separate stateful operations, not image releases.

Runtime Secrets are delivered from the private SOPS/age environment source by
the restricted `faang-secrets` Application. Never create public plaintext
Secret files or copy local credential mappings into Git.

## Verification and rollback

Use the read-only release collector before calling a release delivered.
In-cluster readiness alone does not prove trusted external HTTPS, dependency
connectivity, backup/restore, or rollback readiness.

For an explicitly approved workload-only rollback, use
[`workload-rollback.md`](faang-infra/ops/gitops/workload-rollback.md). A Git
revert does not undo schema migrations, dependency changes, Secrets, or data.

## Focused runbooks

- Infrastructure overview: [`faang-infra/README.md`](faang-infra/README.md)
- Argo/Jenkins/GitOps: [`faang-infra/ops/README.md`](faang-infra/ops/README.md)
- Validation evidence: [`faang-infra/ops/validation/README.md`](faang-infra/ops/validation/README.md)
- Dependency profiles: [`faang-infra/k8s/components/dependencies/README.md`](faang-infra/k8s/components/dependencies/README.md)
- Bootstrap: [`faang-infra/k8s/bootstrap/README.md`](faang-infra/k8s/bootstrap/README.md)
- Registry trust: [`faang-infra/k8s/registry/README.md`](faang-infra/k8s/registry/README.md)
- Private ingress/TLS: [`faang-infra/ops/gitops/private-ingress-tls.md`](faang-infra/ops/gitops/private-ingress-tls.md)
