# Tenant Onboarding Guide

How to onboard a new tenant onto the platform, what happens after the PR
merges, how to verify the tenant is live, and how to hand over credentials.
Offboarding is covered at the end.

A *tenant* is one landlord instance of the property-management app: an
isolated namespace with its own PostgreSQL database, network policies,
resource governance, BasicAuth-protected URL, and automated backups —
all provisioned from a single small YAML file.

## TL;DR

1. Add `tenants/<name>.yaml` with `spec.name` and `spec.tier`.
2. Open a PR, get one approval, squash-merge.
3. Wait ~5–10 minutes. The tenant is live at `https://<name>.fhuebung.lol`.
4. Retrieve the BasicAuth password from Google Secret Manager and hand it
   to the tenant out-of-band.

No infrastructure change, no Terraform run, no manual cluster step.

## Step 1 — choose a name and a tier

### Name rules

`spec.name` becomes the namespace (`tenant-<name>`), the URL
(`<name>.fhuebung.lol`), and the prefix of most per-tenant resources.

- Lowercase alphanumerics and `-`, starting and ending alphanumeric
  (DNS-1123 label; enforced by the XRD schema).
- Keep it ≤ 56 characters: the `tenant-` namespace prefix consumes 7 of
  Kubernetes' 63-character limit.
- The name is permanent. Renaming means offboarding and re-onboarding.
- Reserved names: `staging` (the version-testing tenant, see
  [Image tag overrides](#image-tag-overrides-staging-workflow)).

### Tier-selection policy

The platform offers two resource-governance envelopes. The tier drives the
namespace `ResourceQuota` and the per-container `LimitRange.max`; request
defaults and minimums are identical across tiers (see
`platform/docs/claude/architecture-decisions.md` § Tiering).

**Default to `small`.** It fits the standard app stack (backend, frontend,
single-instance PostgreSQL — three pods) with roughly 3× headroom for pod
restarts, rolling updates, and the daily backup job.

**Pick `medium` only on demonstrated need**, not speculatively:

- the tenant's workload is sustained at or near the small quota
  (CPU throttling, evictions, or quota-denied pods in the namespace), or
- a per-container need exceeds the small `LimitRange.max`
  (500m CPU / 512Mi memory), or
- the tenant needs more than 10 pods, 2 PVCs, or 10Gi of storage.

| Envelope | `small` | `medium` |
| --- | --- | --- |
| CPU requests / limits (namespace) | 1 / 2 | 4 / 8 |
| Memory requests / limits (namespace) | 1Gi / 2Gi | 4Gi / 8Gi |
| Pods | 10 | 30 |
| PVCs / storage | 2 / 10Gi | 5 / 30Gi |
| Per-container max (CPU / memory) | 500m / 512Mi | 1 / 1Gi |

**Changing tier later** is the same PR motion: edit `spec.tier` in the
claim, merge. The new quota applies immediately; running pods are not
restarted (the `LimitRange.max` applies to newly created pods only), so a
tier change is downtime-free.

## Step 2 — write the claim

Create `tenants/<name>.yaml`:

```yaml
---
apiVersion: platform.fhuebung.lol/v1alpha1
kind: Tenant
metadata:
  name: <name>
spec:
  name: <name>
  tier: small
```

That is the whole file for a normal tenant. `metadata.name` and
`spec.name` must match.

### Image tag overrides (staging workflow)

Every tenant runs the central app version from `values/app-version.yaml`.
A claim may pin itself with `spec.imageTagOverride`:

```yaml
spec:
  name: staging
  tier: small
  imageTagOverride:
    backend: "v0.2.0"
    frontend: "v0.2.0"
```

This is the *staging* workflow: the dedicated `staging` tenant receives a
new release via override first; once validated, the central bump in
`values/app-version.yaml` rolls every other tenant forward while overridden
tenants stay pinned. Normal tenants should **not** set `imageTagOverride` —
leaving it out is what makes the single-change global rollout work.

## Step 3 — open the PR

Standard workflow per `CONTRIBUTING.md`: branch
`<initials>/feat/tenant-<name>`, commit
`feat(tenants): add <name> tenant claim`, open a PR referencing the
onboarding issue, one approval, squash-merge. CI validates the claim with
yamllint; the API server validates it against the XRD-generated schema at
sync time.

## What happens after the merge

No human touches anything past this point:

1. **Argo CD** (≈ ≤3 min polling) syncs the claim into the cluster.
2. **Crossplane** matches it to the `xtenant-default` Composition and
   provisions the full tenant set: namespace, ResourceQuota, LimitRange,
   four NetworkPolicies, BasicAuth chain (random password → bcrypt
   htpasswd → Google Secret Manager → ExternalSecret → Traefik
   Middleware), GHCR pull secret, CloudNativePG `Cluster`, backup bucket
   IAM binding, daily `ScheduledBackup`, and the app Helm `Release`.
3. **CloudNativePG** initializes PostgreSQL (~1–2 min) and generates the
   app credentials Secret consumed by the backend.
4. **provider-helm** renders the app chart (backend, frontend, ingress)
   into the namespace.
5. **ExternalDNS** publishes `<name>.fhuebung.lol`; the wildcard
   certificate already covers the host, so there is no per-tenant ACME
   wait.
6. The first base backup is taken immediately; WAL archiving runs
   continuously from then on.

End-to-end a tenant is typically reachable in **5–10 minutes**.

## Verify

```bash
# Claim accepted and reconciled (SYNCED + READY = True)
kubectl get tenants.platform.fhuebung.lol <name>

# Workloads up: backend, frontend, and <name>-cluster-1 Running
kubectl -n tenant-<name> get pods

# Database healthy
kubectl -n tenant-<name> get cluster <name>-cluster

# URL serves and demands auth (401 without credentials is success)
curl -sI https://<name>.fhuebung.lol | head -1
```

Then open `https://<name>.fhuebung.lol` in a browser: a BasicAuth prompt
with a valid (wildcard) certificate, and the app loads after login.

If something hangs, the usual suspects in order: the Argo CD `tenants`
Application (sync error), the XR's `kubectl describe xtenant` events
(patch/provider errors), and the CNPG operator logs (storage).

## Credential handover

The Composition generates a random password per tenant (username is always
`admin`) and stores it in Google Secret Manager. The operator retrieves it:

```bash
gcloud secrets versions access latest \
  --secret="tenant-<name>-basicauth-password"
```

Hand it to the tenant **out-of-band** (password manager share, encrypted
mail) — never via issue, PR, or chat. GSM IAM gates who can read the
secret; access is audited in Cloud Logging.

## Offboarding

Remove `tenants/<name>.yaml` via PR. On merge, Crossplane tears down the
namespace and everything in it — workloads, database **including its data
volume**, network policies, the DNS record (via ExternalDNS, once the
Ingress is gone), and the backup-bucket IAM binding.

Two things survive deliberately:

- the GSM secrets (`tenant-<name>-basicauth-htpasswd`, `-password`) are
  `Orphan`'d so a re-onboarded tenant of the same name keeps its audit
  trail;
- backup objects under `gs://<project>-pg-backups/<name>/` persist until
  the bucket's 30-day lifecycle deletes them — the recovery window after
  an accidental offboarding.

The validated deletion procedure and observed timings are documented in
the S4-03 validation doc (issue #86).
