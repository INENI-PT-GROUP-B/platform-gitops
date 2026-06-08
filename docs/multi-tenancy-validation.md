# Multi-Tenancy Validation

Evidence for the four invariants required by S3-09 (#62): cross-tenant network
isolation, in-namespace traffic allowed, ResourceQuota enforcement, and
per-tier differentiation. Run against `demotenant1` (`small` tier) and
`demotenant2` (`medium` tier), both reconciled by Argo CD from `tenants/*.yaml`
through the `xtenant-default` Composition.

All outputs in this document are verbatim from the live cluster on 2026-06-09.

## Setup

Two persistent Tenant claims under `tenants/` synced into `crossplane-system`
by `applications/tenants.yaml`, each rendered into a `tenant-<name>` namespace
through `crossplane/compositions/xtenant-default.yaml`:

```text
$ kubectl get xtenants
NAME                SYNCED   READY   COMPOSITION       AGE
demotenant1-g2rls   True     True    xtenant-default   12m
demotenant2-n58bt   True     True    xtenant-default   12m
```

Both tenants reachable under their wildcard subdomain, BasicAuth wall active:

```text
$ curl -sI https://demotenant1.fhuebung.lol/
HTTP/2 401
content-type: text/plain
www-authenticate: Basic realm="traefik"
```

TLS terminated by Traefik's default `TLSStore` using the shared wildcard cert
from cert-manager (`traefik/certificate.yaml`):

```text
* SSL connection using TLSv1.3 / TLS_AES_128_GCM_SHA256
*  subject: CN=*.fhuebung.lol
*  issuer: C=US; O=Let's Encrypt; CN=YR1
```

`demotenant2.fhuebung.lol` returns identical headers and serves the same
wildcard cert.

## Test 1 — Cross-tenant DB connect is blocked

From a debug pod in `tenant-demotenant1`, attempt TCP 5432 to `demotenant2`'s
CNPG read-write Service. Expected outcome: `default-deny-egress-allow-dns`
(resource 5 in the Composition) drops the packet → timeout.

```text
$ kubectl run egress-test --image=nicolaka/netshoot -n tenant-demotenant1 \
    --restart=Never --command -- nc -zv -w 5 10.30.8.47 5432
$ kubectl logs -n tenant-demotenant1 egress-test
nc: connect to 10.30.8.47 port 5432 (tcp) timed out: Operation in progress
```

Sanity from the same source pod, this time targeting the **own** CNPG Service.
Expected to succeed via `allow-app-to-db` (resource 6):

```text
$ nc -zv -w 3 demotenant1-cluster-rw 5432
Connection to demotenant1-cluster-rw (10.30.14.181) 5432 port [tcp/postgresql] succeeded!
```

Proves: cross-tenant DB egress is blocked, in-namespace DB egress allowed.

## Test 2 — Cross-namespace pod-to-pod traffic is blocked

From the same source pod, HTTP GET against `demotenant2`'s backend Pod IP on
port 3000. Expected outcome: `default-deny-egress-allow-dns` drops the packet.

```text
$ curl --max-time 5 -sS http://10.20.0.19:3000/healthz
curl: (28) Connection timed out after 5002 milliseconds
```

Proves: cross-tenant pod traffic on application ports is blocked.

## Test 3 — ResourceQuota rejects an over-quota Deployment

In `tenant-demotenant1` (`small` tier, `requests.cpu` Quota = `1`), apply a
Deployment whose individual pods stay within the LimitRange (`max.cpu = 500m`)
but whose aggregate request exceeds the namespace Quota (4 replicas × 300m =
1200m additional, summed with the 175m already in use by the tenant app and
CNPG).

```text
$ kubectl apply -f deployment-overflow.yaml -n tenant-demotenant1
deployment.apps/quota-overflow created

$ kubectl get events -n tenant-demotenant1 --sort-by='.lastTimestamp' \
    | grep "exceeded quota"
Warning   FailedCreate   replicaset/quota-overflow-757445d74d   Error creating: pods
"quota-overflow-757445d74d-6jdbz" is forbidden: exceeded quota: tenant-quota,
requested: limits.cpu=400m,requests.cpu=300m,
used: limits.cpu=1650m,requests.cpu=775m,
limited: limits.cpu=2,requests.cpu=1
```

The Deployment object lands, but the ReplicaSet controller fails to create
additional pods once aggregate requests would exceed `requests.cpu=1`. Only 2
of the 4 replicas come up; the remaining 2 are rejected with `forbidden:
exceeded quota`.

## Test 4 — Tier differentiation

Side-by-side `kubectl describe` on the per-tenant `ResourceQuota` and
`LimitRange`. Each field maps back to the tier-mapped patches in
`xtenant-default.yaml`.

### ResourceQuota

```text
$ kubectl describe resourcequota tenant-quota -n tenant-demotenant1
Name:                   tenant-quota
Namespace:              tenant-demotenant1
Resource                Used   Hard
--------                ----   ----
limits.cpu              850m   2
limits.memory           896Mi  2Gi
persistentvolumeclaims  1      2
pods                    3      10
requests.cpu            175m   1
requests.memory         352Mi  1Gi
requests.storage        8Gi    10Gi

$ kubectl describe resourcequota tenant-quota -n tenant-demotenant2
Name:                   tenant-quota
Namespace:              tenant-demotenant2
Resource                Used   Hard
--------                ----   ----
limits.cpu              850m   8
limits.memory           896Mi  8Gi
persistentvolumeclaims  1      5
pods                    3      30
requests.cpu            175m   4
requests.memory         352Mi  4Gi
requests.storage        8Gi    30Gi
```

Hard caps differ on all seven fields, matching the `tier`-map transforms on
Composition resource 2.

### LimitRange

```text
$ kubectl describe limitrange tenant-limits -n tenant-demotenant1
Type        Resource  Min   Max    Default Request  Default Limit
----        --------  ---   ---    ---------------  -------------
Container   memory    32Mi  512Mi  64Mi             256Mi
Container   cpu       10m   500m   50m              250m

$ kubectl describe limitrange tenant-limits -n tenant-demotenant2
Type        Resource  Min   Max  Default Request  Default Limit
----        --------  ---   ---  ---------------  -------------
Container   cpu       10m   1    50m              250m
Container   memory    32Mi  1Gi  64Mi             256Mi
```

Only `Max` is tier-mapped (`small: 500m / 512Mi`, `medium: 1 / 1Gi`); `Min`,
`Default` and `Default Request` stay constant across tiers — by design, per
`platform/docs/claude/architecture-decisions.md` § Multi-tenancy § Tiering.

## Reproducing

The tests are stable across cluster rebuilds because GSM containers carry
`deletionPolicy: Orphan` and the rest is fully GitOps-managed. Re-running on a
fresh cluster:

1. `bootstrap.sh` brings up GKE + Argo CD + the platform components.
2. Argo CD's `tenants` Application picks up `tenants/demotenant{1,2}.yaml`.
3. Crossplane reconciles both XTenants via `xtenant-default`.
4. Wait until `kubectl get xtenants` shows both `READY=True` (~3-5 min).
5. Run tests 1-4 as above. Service ClusterIPs and Pod IPs are different but
   the assertions are identical.

## References

- Composition source: `crossplane/compositions/xtenant-default.yaml`
  (Namespace=1, ResourceQuota=2, LimitRange=3, NetworkPolicies=4-7, CNPG=16).
- Tenant XRD: `crossplane/xrds/xtenant.yaml` (`spec.name`, `spec.tier` enum
  `small`/`medium`).
- Tier semantics:
  `platform/docs/claude/architecture-decisions.md` § Multi-tenancy § Tiering.
- Issue: INENI-PT-GROUP-B/platform-gitops#62 (S3-09).
