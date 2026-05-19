# platform-gitops

GitOps repository for the platform. Read by ArgoCD from the cluster.

Contains ArgoCD applications, Crossplane manifests, tenant claims, and platform Helm values.

## Directory Layout

| Path | Purpose |
|------|---------|
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

## Contributing

See the org-wide [CONTRIBUTING.md](https://github.com/INENI-PT-GROUP-B/platform/blob/main/CONTRIBUTING.md) for branch naming, commit message format, PR rules, and security requirements.

Key points:
- No direct commits to `main`, all changes via Pull Request
- Every PR references an issue