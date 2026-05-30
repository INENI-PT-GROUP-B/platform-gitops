# Chart Contract — `app-chart` ↔ Crossplane Composition

This document is the source of truth for the `values.yaml` surface of the
tenant app Helm chart (`app-chart`, sources at `app-backend/chart/`) that the
Crossplane Composition writes to at runtime via `provider-helm`. It pins which
keys are part of the contract, which side owns each, what happens when a key
is omitted, and the workflow for changing the contract.

Two repositories must stay aligned: the chart in `app-backend/` and the
Composition in `platform-gitops/crossplane/compositions/` (S3-06, not yet
written). The Composition author treats this document as the canonical
interface; the chart author treats every key listed here as a stable API.
Changes on either side go through a coordinated PR pair — see
[Changing the contract](#changing-the-contract).

## Contract surface

The keys below mirror the layout of `app-backend/chart/values.yaml` at the
tip of `app-backend@main`. Per-key annotations:

- **Owner** — which side sets it at runtime:
  - *Composition (per-tenant)* — written into the rendered `Release` by the
    S3-06 Composition, computed from the tenant claim and surrounding state.
  - *`values/app-version.yaml` (rolling)* — read by the Composition from the
    central tag file and forwarded into the release; a tenant claim's
    `spec.imageTagOverride.*` beats the central value for that tenant only.
  - *Chart default (static)* — declared in `app-backend/chart/values.yaml`,
    not overridden by the Composition under normal operation.
- **Required?** — *Required* means the chart template carries a `required`
  guard and the release fails when the value is missing or empty.
  *Optional, defaulted* means the chart `values.yaml` ships a default.
  *Optional, suppressed when empty* means the field is conditionally
  rendered (template `if`-block) and absence is a no-op.
- **Example** — a literal placeholder showing the expected shape.

### Tenant identity

#### `tenant`

- **Owner:** Composition (per-tenant)
- **Required?** Optional, defaulted — when empty, `app-chart.fullname` falls
  back to `.Release.Name` (see `_helpers.tpl` definition of
  `app-chart.fullname`).
- **Example:** `"demo"`
- **Notes:** drives the resource name prefix for every rendered object (the
  fullname is truncated to 63 chars) and is emitted as the
  `app.kubernetes.io/part-of` label when set. The Composition should set it
  to the tenant claim's name so resources remain identifiable even when the
  Helm release name diverges.

#### `host`

- **Owner:** Composition (per-tenant)
- **Required?** Required — `ingress.yaml` carries
  `required "host is required"` on both the TLS host and the rule host.
- **Example:** `"demo.fhuebung.lol"`
- **Notes:** must resolve under the wildcard `*.fhuebung.lol` covered by the
  shared TLS Secret referenced below.

### Image references

#### `image.backend.repository`, `image.frontend.repository`

- **Owner:** Chart default (static)
- **Required?** Optional, defaulted —
  `ghcr.io/ineni-pt-group-b/app-backend` and
  `ghcr.io/ineni-pt-group-b/app-frontend`.
- **Example:** `"ghcr.io/ineni-pt-group-b/app-backend"`
- **Notes:** the Composition does not override these — repository moves are
  a chart-side concern and ride through a chart version bump.

#### `image.backend.tag`, `image.frontend.tag`

- **Owner:** `values/app-version.yaml` (rolling), per-tenant
  `imageTagOverride` beats it.
- **Required?** Required — both deployments carry
  `required "image.{backend,frontend}.tag is required"` (see
  `backend-deployment.yaml`, `frontend-deployment.yaml`).
- **Example:** `"v0.1.0"`
- **Notes:** tags follow the `v`-prefixed semver convention (matches the
  release tags produced by the `app-backend` and `app-frontend` release
  workflows). See [`app-version.yaml` ↔ `image.*.tag`](#app-versionyaml--imagetag).

### Image pull

#### `imagePullSecrets`

- **Owner:** Chart default (static); Composition only touches it if the
  shared GHCR secret name ever changes.
- **Required?** Optional, defaulted — `["ghcr-pull-secret"]`. Applied to the
  frontend Deployment only; the backend image is public and needs no pull
  secret.
- **Example:** `["ghcr-pull-secret"]`
- **Notes:** the named `Secret` is materialised per tenant namespace by the
  Composition via an ExternalSecret pulling from
  `shared/ghcr-pull-secret` in Google Secret Manager (see S3-05 acceptance
  criteria).

### Ingress

#### `ingress.className`

- **Owner:** Chart default (static)
- **Required?** Optional, defaulted — `"traefik"` (the platform's ingress
  controller, S2-06).
- **Example:** `"traefik"`

#### `ingress.tls.secretName`

- **Owner:** Chart default (static)
- **Required?** Optional, defaulted — `"wildcard-fhuebung-lol-tls"`.
- **Example:** `"wildcard-fhuebung-lol-tls"`
- **Notes:** **shared** wildcard certificate produced by
  `platform-gitops/traefik/certificate.yaml`. The cert-manager `Certificate`
  writes its Secret into the `traefik` namespace; each tenant Ingress
  references it by name only — Traefik resolves the Secret via its default
  `TLSStore` regardless of which namespace the Ingress lives in. The
  Composition does not create or copy this Secret per tenant.

#### `ingress.basicAuthMiddlewareRef`

- **Owner:** Composition (per-tenant)
- **Required?** Optional, suppressed when empty —
  `ingress.yaml` only renders the
  `traefik.ingress.kubernetes.io/router.middlewares` annotation when this
  value is set (see the `if .Values.ingress.basicAuthMiddlewareRef` block).
- **Example:** `"tenant-demo-basicauth@kubernetescrd"`
- **Notes:** Traefik addresses Middleware CRs by
  `<namespace>-<name>@kubernetescrd` (the `@kubernetescrd` suffix is the
  Traefik provider identifier, not the namespace). The Middleware itself is
  produced per tenant by S3-05.

### Database

#### `db.secretName`

- **Owner:** Composition (per-tenant)
- **Required?** Required — `backend-deployment.yaml` carries
  `required "db.secretName is required"` on the `secretKeyRef.name`.
- **Example:** `"demo-cluster-app"`
- **Notes:** points at the CloudNativePG-generated credentials Secret in the
  tenant namespace (the operator creates it next to the `Cluster` resource).

#### `db.urlKey`

- **Owner:** Chart default (static)
- **Required?** Optional, defaulted — `"uri"` (matches the key
  CloudNativePG writes into its generated Secrets).
- **Example:** `"uri"`
- **Notes:** override only if CloudNativePG's Secret schema changes upstream.

### Sizing

#### `backend.replicas`, `frontend.replicas`

- **Owner:** Chart default (static) for now; future per-tier overrides
  belong to the Composition.
- **Required?** Optional, defaulted — `1` each.
- **Example:** `1`

#### `backend.resources`, `frontend.resources`

- **Owner:** Chart default (static) for now; future per-tier overrides
  belong to the Composition (the XRD's `tier` field, S3-01).
- **Required?** Optional, defaulted — see `app-backend/chart/values.yaml`
  for the current numbers (50m/64Mi → 250m/256Mi for backend; 25m/32Mi →
  100m/128Mi for frontend).
- **Example:**

  ```yaml
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 250m
    memory: 256Mi
  ```

## `app-version.yaml` ↔ `image.*.tag`

`platform-gitops/values/app-version.yaml` holds the rolling tag the
Composition uses by default for every tenant:

```yaml
chart:
  version: "0.1.0"
images:
  backend: "v0.1.0"
  frontend: "v0.1.0"
```

Schema and rollout semantics are documented in
`platform/docs/claude/architecture-decisions.md` § Application updates and
staging. The Composition reads this file once per reconcile and renders
`image.backend.tag` / `image.frontend.tag` from it.

Per-tenant override: a tenant claim's
`spec.imageTagOverride.backend` and `spec.imageTagOverride.frontend` (either
or both) beat the central values for that tenant only. The dedicated
`staging` tenant uses this to receive new versions before the central
rollout. `chart.version` is the OCI chart tag pulled from
`ghcr.io/ineni-pt-group-b/app-chart` (published by the app-backend release
workflow once S2-12 lands); it has no per-tenant override and applies
uniformly.

## Chart-internal — not part of the contract

These exist in the chart but are *not* contract keys. The Composition does
not address them, and the chart author can change them without touching this
doc, **as long as the contract surface above stays stable.**

- **`_helpers.tpl` template definitions** — `app-chart.fullname` (derives the
  resource-name prefix from `tenant` or the release name), common
  `app-chart.labels`, and the backend / frontend selector helpers.
- **`frontend-config-cm.yaml` ConfigMap** — ships the runtime config
  delivered to the SPA as `/config.js`; carries a static `apiBaseUrl: "/api"`
  matching the path-based routing the platform enforces (see
  `platform/docs/claude/architecture-decisions.md` § Ingress). The frontend
  image must serve `/config.js` statically and must not also write it at
  runtime.
- **Service ports** — backend on `3000`, frontend on `80`. Referenced by
  the Ingress backend definitions.
- **Path-based routing** — `/api` → backend Service, `/` → frontend Service
  (`ingress.yaml`). Locked to path-based by the wildcard certificate
  covering only one subdomain level.
- **Probes** — `startupProbe` / `livenessProbe` / `readinessProbe` against
  `GET /healthz` for the backend; HTTP probes against `/` for the frontend.

## Changing the contract

Any change to a documented key requires a coordinated PR pair:

1. **Frame the change explicitly.** Name the key, the new owner / required
   state / shape, the motivation.
2. **Land an `app-backend` PR** that adjusts `chart/values.yaml` and the
   templates, and bumps `chart/Chart.yaml#version`. While the chart is on
   the `0.x` line, treat breaking surface changes as a minor bump
   (`0.x` → `0.(x+1)`) and additive defaults as a patch bump.
3. **Land a paired `platform-gitops` PR** that updates this document and,
   once S3-06 exists, the Composition. Bump
   `platform-gitops/values/app-version.yaml#chart.version` to the new
   chart version in the same PR so the Composition picks up the new release
   on the next reconcile.
4. **Image-only rollouts** (no chart change) are a single
   `platform-gitops` PR bumping `images.backend` and/or `images.frontend`
   in `app-version.yaml`. No chart bump needed.

The cross-repo nature of this contract is the same drift-risk class as the
root App-of-Apps spec-identity contract surfaced in #28 and tracked as #29.
Today the contract is held together by review discipline; a CI-enforcement
check is a sensible follow-up but is intentionally out of scope here.

## References

- `platform/docs/claude/architecture-decisions.md` § The application —
  deployment contract this document implements.
- `platform/docs/claude/architecture-decisions.md` § Application updates and
  staging — rollout semantics for `app-version.yaml` and
  `imageTagOverride`.
- `platform/docs/claude/repository-layout.md` § platform-gitops — directory
  layout, including `values/app-version.yaml`.
- `app-backend/chart/values.yaml` — canonical default values.
- `app-backend/chart/Chart.yaml` — chart name (`app-chart`) and current
  version.
