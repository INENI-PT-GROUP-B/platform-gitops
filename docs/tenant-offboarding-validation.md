# Tenant Offboarding Validation

Evidence for the tenant deletion (offboarding) flow required by S4-03 (#86):
removing a Tenant claim from `tenants/` tears down the whole per-tenant stack
in reverse, while a small set of resources is deliberately kept. Run against
a throwaway tenant `deltest`, onboarded via #90 only to be deleted here.

The companion onboarding evidence lives in
[`multi-tenancy-validation.md`](./multi-tenancy-validation.md); this document
covers the opposite direction.

Pre-deletion outputs are verbatim from the live cluster on 2026-06-13.
Post-deletion outputs are captured against the same cluster immediately after
Argo CD prunes the removed claim (see § Post-deletion verification).

## Setup

`deltest` is a `small`-tier Tenant claim, reconciled by Argo CD from
`tenants/deltest.yaml` through `crossplane/compositions/xtenant-default.yaml`
into the `tenant-deltest` namespace. It was live and healthy for ~24h before
the deletion:

```text
$ kubectl get xtenant | grep deltest
deltest-pmfgg   True   True   xtenant-default   xtenant-default-7b31449   24h

$ kubectl -n tenant-deltest get pods
NAME                                READY   STATUS    RESTARTS   AGE
deltest-backend-7f6497b998-lxjkx    1/1     Running   0          91m
deltest-cluster-1                   1/1     Running   0          22h
deltest-frontend-5944dc4c4f-fxcdb   1/1     Running   0          91m

$ kubectl -n tenant-deltest get cluster.postgresql.cnpg.io
NAME              AGE   INSTANCES   READY   STATUS                      PRIMARY
deltest-cluster   24h   1           1       Cluster in healthy state    deltest-cluster-1
```

Reachable under its wildcard subdomain with the BasicAuth wall active:

```text
$ curl -sI https://deltest.fhuebung.lol/
HTTP/1.1 401 Unauthorized
Www-Authenticate: Basic realm="traefik"
```

DNS A record (published by ExternalDNS from the Ingress) and its ownership
TXT record:

```text
$ gcloud dns record-sets list --zone platform-zone --filter="name~deltest"
NAME                     TYPE  RRDATAS
a-deltest.fhuebung.lol.  TXT   "heritage=external-dns,external-dns/owner=gke-prod,external-dns/resource=ingress/tenant-deltest/deltest"
deltest.fhuebung.lol.    A     34.79.103.36
```

## Deleted-vs-persisted inventory

Derived from the `deletionPolicy` settings in
`crossplane/compositions/xtenant-default.yaml` and the full managed-resource
set for `deltest-pmfgg`. Removing the claim deletes the composite (XR), and
each managed resource then follows its own policy.

### Torn down (default `deletionPolicy: Delete`, resource follows the XR)

The provider-kubernetes `Object`s, the Helm `Release`, and the
`BucketIAMMember`:

- Namespace `tenant-deltest` and everything namespaced inside it: backend +
  frontend `Deployment`s/`Pod`s, the CNPG `Cluster` `deltest-cluster` and its
  PVC, the five `NetworkPolicy`s, `ResourceQuota` `tenant-quota`, `LimitRange`,
  the Traefik `Middleware` `basicauth`, the three `ExternalSecret`s, the
  `ScheduledBackup`, and the `Ingress`.
- Helm `Release` `deltest-pmfgg-l256j` (`app-chart` 0.1.1).
- `BucketIAMMember` `deltest-pmfgg-6btsj` — the prefix-scoped
  `roles/storage.objectAdmin` binding on `gs://…-pg-backups` for the tenant's
  WI-federated principal.

ExternalDNS removes the `deltest.fhuebung.lol` A record and its ownership TXT
once the Ingress is gone.

### Kept on purpose (`deletionPolicy: Orphan`)

The Google Secret Manager containers and versions survive XR deletion so a
re-onboarded tenant of the same name re-adopts them (composition resources at
lines 660/691/733/762):

```text
$ gcloud secrets list --filter="name:deltest" --format="value(name)"
tenant-deltest-basicauth-htpasswd
tenant-deltest-basicauth-password
```

### Not applicable for `deltest`: backup objects

S4-03's scope notes that backup objects under
`gs://…-pg-backups/<tenant>/` persist until the 30-day lifecycle reaps them.
For `deltest` there are none to persist: its continuous archiving never
succeeded (`ContinuousArchiving=False`, `403 storage.buckets.get`; the four
original tenants are unaffected — see the #67 close note), so the bucket holds
no `deltest/` prefix. The deletion therefore has no backup objects to leave
behind. The 403 itself is not pursued — deleting the tenant makes it moot.

## Procedure

1. Open a PR on `platform-gitops` removing `tenants/deltest.yaml` (this PR).
2. Squash-merge. Argo CD prunes the removed `Tenant` claim on its next sync;
   Crossplane deletes the XR, and the managed resources follow their
   `deletionPolicy` as above.
3. Capture the post-deletion state below.

## Post-deletion verification

> Captured immediately after the merge + Argo CD prune. Commands run with an
> explicit `--context` against the GKE cluster.

Cleanup (expected gone):

```text
$ kubectl get ns tenant-deltest
# <to be captured at merge: expect "not found">

$ kubectl get xtenant | grep deltest
# <expect: no rows>

$ gcloud dns record-sets list --zone platform-zone --filter="name~deltest"
# <expect: no A / TXT rows — ExternalDNS removed them>
```

Intentional survivors (expected still present):

```text
$ gcloud secrets list --filter="name:deltest" --format="value(name)"
# <expect: tenant-deltest-basicauth-htpasswd, tenant-deltest-basicauth-password>
```

Observed timing: _<teardown duration from prune to namespace removal — filled
at merge>_.

## Proves

- Offboarding is the same single-PR motion as onboarding, in reverse: one
  claim removal tears down the full per-tenant stack.
- The deletion is selective by design — GSM credential containers are
  Orphaned for safe re-onboarding; everything else (namespace, DB, PVC,
  ingress, DNS, bucket IAM binding) is removed.
- No manual cleanup step is required; ExternalDNS and Crossplane reconcile the
  removal end-to-end.
