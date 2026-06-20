# platform-gitops

GitOps repository for the platform. Read by ArgoCD from the cluster.

Contains ArgoCD applications, Crossplane manifests, tenant claims, and platform Helm values.

## Directory Layout

| Path | Purpose |
| ---- | ------- |
| `applications/` | ArgoCD `Application` manifests for platform components |
| `applicationsets/` | Reserved — tenant apps are deployed by Crossplane via `provider-helm`, not via ApplicationSets |
| `cert-manager/clusterissuers/` | `ClusterIssuer` manifests (Let's Encrypt staging / prod) |
| `crossplane/providers/` | Crossplane provider configurations |
| `crossplane/provider-installs/` | Crossplane `Provider` package installs (provider-helm, provider-kubernetes) |
| `crossplane/xrds/` | Composite Resource Definitions (XRDs) |
| `crossplane/compositions/` | Compositions implementing the XRDs |
| `docs/` | Repository documentation (e.g. `chart-contract.md`) |
| `external-secrets/clustersecretstores/` | `ClusterSecretStore` manifests (Google Secret Manager via Workload Identity) |
| `tenants/` | Tenant claims (one file or folder per tenant) |
| `traefik/` | Traefik-owned resources synced by Argo CD (e.g. the wildcard `Certificate`) |
| `values/` | Helm values for platform components |

## How It Works

1. ArgoCD runs in the cluster and watches this repo.
2. Changes merged to `main` are picked up automatically and reconciled.
3. New tenants are onboarded by adding a claim under `tenants/`. ArgoCD syncs the claim into the cluster, then Crossplane reconciles it against the matching Composition — provisioning the per-tenant resources (namespace, DB, network policies, secrets) and rendering the application's Helm chart via `provider-helm` into the tenant namespace.

## Tenant onboarding

Day-2 tenant provisioning is the primary daily action against this repo. The end-to-end runbook lives at [`docs/tenant-onboarding.md`](docs/tenant-onboarding.md): write a seven-line `XTenant` claim under `tenants/`, open a PR, squash-merge, and Argo CD + Crossplane do the rest (namespace, NetworkPolicies, CloudNativePG cluster, BasicAuth, Helm release via `provider-helm`).

Validation of the model — both isolation between tenants and the full lifecycle including deletion — is recorded in [`docs/multi-tenancy-validation.md`](docs/multi-tenancy-validation.md) and [`docs/lifecycle-tests.md`](docs/lifecycle-tests.md).

## Validation

Two CI workflows run on every PR:

- `lint` (`.github/workflows/lint.yml`) — yamllint and markdownlint via the org-wide reusable workflow.
- `validate` (`.github/workflows/validate.yml`) — schema validation for everything Argo CD reconciles from this repo:
  - `helm lint --strict` against every `values/*.yaml`, using the chart and version pinned in the matching `applications/<name>.yaml`.
  - `kubeconform -strict` against `applications/`, `cert-manager/`, `crossplane/providers/`, `external-secrets/`, `tenants/`, and `traefik/`, with CRD schemas pulled from a pinned commit of [`datreeio/CRDs-catalog`](https://github.com/datreeio/CRDs-catalog).

Reproduce locally (matches CI):

```bash
# helm lint a values file against its pinned chart
yq -r '.spec.sources[] | select(.chart) | [.repoURL, .chart, .targetRevision] | @tsv' \
  applications/externaldns.yaml \
  | while IFS=$'\t' read -r repo chart version; do
      helm repo add tmp "$repo" >/dev/null
      tmpdir=$(mktemp -d)
      helm pull "tmp/$chart" --version "$version" --untar --untardir "$tmpdir"
      helm lint --strict --values values/externaldns.yaml "$tmpdir/$chart"
    done

# kubeconform across the validated paths (CRDs-catalog SHA matches the CI pin)
SCHEMA="https://raw.githubusercontent.com/datreeio/CRDs-catalog/bc3592a812136ae4eac1effcee6a1b895a066bee/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json"
kubeconform -strict -summary -skip ProviderConfig \
  -kubernetes-version 1.31.0 \
  -schema-location default \
  -schema-location "$SCHEMA" \
  applications/ cert-manager/ crossplane/providers/ external-secrets/ tenants/ traefik/
```

`crossplane/xrds/` and `crossplane/compositions/` are intentionally excluded until Sprint 3 lands the manifests and their CRD schemas.

## Contributing

See the org-wide [CONTRIBUTING.md](https://github.com/INENI-PT-GROUP-B/platform/blob/main/CONTRIBUTING.md) for branch naming, commit message format, PR rules, and security requirements.

Key points:

- No direct commits to `main`, all changes via Pull Request
- Every PR references an issue

## Related docs

- Platform entry point and cross-cutting documentation: [`INENI-PT-GROUP-B/platform`](https://github.com/INENI-PT-GROUP-B/platform)
- Day-2 architecture diagram: [`platform/docs/architecture/day2-tenant-provisioning.drawio.png`](https://github.com/INENI-PT-GROUP-B/platform/blob/main/docs/architecture/day2-tenant-provisioning.drawio.png)
- Day-1 bootstrap (Terraform + `bootstrap.sh`): [`INENI-PT-GROUP-B/platform-iac`](https://github.com/INENI-PT-GROUP-B/platform-iac)
- Chart contract for tenant apps: [`docs/chart-contract.md`](docs/chart-contract.md)
- Per-tenant offboarding evidence: [`docs/tenant-offboarding-validation.md`](docs/tenant-offboarding-validation.md)
- Cluster redeploy validation: [`docs/cluster-redeploy-validation.md`](docs/cluster-redeploy-validation.md)
