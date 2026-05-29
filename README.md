# platform-gitops

GitOps repository for the platform. Read by ArgoCD from the cluster.

Contains ArgoCD applications, Crossplane manifests, tenant claims, and platform Helm values.

## Directory Layout

| Path | Purpose |
| ---- | ------- |
| `applications/` | ArgoCD `Application` manifests for platform components |
| `applicationsets/` | Reserved — tenant apps are deployed by Crossplane via `provider-helm`, not via ApplicationSets |
| `crossplane/providers/` | Crossplane provider configurations |
| `crossplane/xrds/` | Composite Resource Definitions (XRDs) |
| `crossplane/compositions/` | Compositions implementing the XRDs |
| `tenants/` | Tenant claims (one file or folder per tenant) |
| `values/` | Helm values for platform components |

## How It Works

1. ArgoCD runs in the cluster and watches this repo.
2. Changes merged to `main` are picked up automatically and reconciled.
3. New tenants are onboarded by adding a claim under `tenants/`. ArgoCD syncs the claim into the cluster, then Crossplane reconciles it against the matching Composition — provisioning the per-tenant resources (namespace, DB, network policies, secrets) and rendering the application's Helm chart via `provider-helm` into the tenant namespace.

## Validation

Two CI workflows run on every PR:

- `lint` (`.github/workflows/lint.yml`) — yamllint and markdownlint via the org-wide reusable workflow.
- `validate` (`.github/workflows/validate.yml`) — schema validation for everything Argo CD reconciles from this repo:
  - `helm lint --strict` against every `values/*.yaml`, using the chart and version pinned in the matching `applications/<name>.yaml`.
  - `kubeconform -strict` against `applications/`, `crossplane/providers/`, and `tenants/`, with CRD schemas pulled from a pinned commit of [`datreeio/CRDs-catalog`](https://github.com/datreeio/CRDs-catalog).

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

# kubeconform across the validated paths (pinned CRDs-catalog SHA matches CI)
SCHEMA="https://raw.githubusercontent.com/datreeio/CRDs-catalog/<pinned-sha>/{{.Group}}/{{.ResourceKind}}_{{.ResourceAPIVersion}}.json"
kubeconform -strict -summary -skip ProviderConfig \
  -kubernetes-version 1.31.0 \
  -schema-location default \
  -schema-location "$SCHEMA" \
  applications/ crossplane/providers/ tenants/
```

`crossplane/xrds/` and `crossplane/compositions/` are intentionally excluded until Sprint 3 lands the manifests and their CRD schemas.

## Contributing

See the org-wide [CONTRIBUTING.md](https://github.com/INENI-PT-GROUP-B/platform/blob/main/CONTRIBUTING.md) for branch naming, commit message format, PR rules, and security requirements.

Key points:

- No direct commits to `main`, all changes via Pull Request
- Every PR references an issue
