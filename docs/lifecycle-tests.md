# Lifecycle Tests

Evidence for the platform's lifecycle and self-healing behaviour observed
against the running cluster. Tests in this file exercise single-component
recovery (S4-04a backend crash), full-cluster rebuild (S4-04b), and tenant
deletion (S4-03). Isolation, quota, and tier evidence is in
[`multi-tenancy-validation.md`](./multi-tenancy-validation.md).

All outputs in this document are verbatim from the live cluster on the date
marked above each test.

## Test 1 — Backend pod crash recovery

Verifies the Deployment + readiness-probe contract from S4-04a (#64):
killing a backend pod must produce a replacement automatically, and the new
pod must not receive traffic before its readiness probe passes. Run against
`demotenant1` on 2026-06-12.

### Setup

`demotenant1` is the `small`-tier demo tenant reconciled from
`tenants/demotenant1.yaml` through the `xtenant-default` Composition. The
backend Deployment is `demotenant1-backend` in the `tenant-demotenant1`
namespace, `RollingUpdate` strategy with `terminationGracePeriodSeconds: 30`.
Readiness probe: `GET /healthz` on the `http` port, `periodSeconds: 10`,
`failureThreshold: 3`, `timeoutSeconds: 1`.

Three terminals run in parallel:

- pod-state watcher
- `/healthz` polling via a backend-Service port-forward, every 0.5 s
- the `kubectl delete pod` itself

The polling bypasses Traefik and the BasicAuth middleware on purpose:
`kubectl port-forward svc/...` tunnels straight to the Service endpoint
set. The Deployment + readiness-probe behaviour is what's under test; the
Traefik/BasicAuth path adds no new evidence and would need a plaintext
password in the doc.

### Procedure

```bash
# Terminal A — watch pod state
kubectl get pod -n tenant-demotenant1 \
  -l app.kubernetes.io/component=backend -w

# Terminal B — start the port-forward, then poll
kubectl port-forward -n tenant-demotenant1 \
  svc/demotenant1-backend 13000:3000 &

while true; do
  printf '%s ' "$(date +%H:%M:%S)"
  curl -s -o /dev/null -w "%{http_code}\n" \
    --max-time 2 http://localhost:13000/healthz
  sleep 0.5
done

# Terminal C — kill the live backend pod once polling baseline is steady
kubectl delete pod -n tenant-demotenant1 \
  demotenant1-backend-5754bbfff-wq9rd
```

### Outputs

Local times below are CEST (UTC+2); pod-condition timestamps from the
Kubernetes API are UTC.

Pod-state watcher:

```text
21:45:19   demotenant1-backend-5754bbfff-wq9rd   1/1   Running             0     3h4m
21:45:21   demotenant1-backend-5754bbfff-h4l88   0/1   ContainerCreating   0     1s
           demotenant1-backend-5754bbfff-wq9rd   1/1   Terminating         0     3h4m
21:45:30   demotenant1-backend-5754bbfff-h4l88   0/1   Running             0     11s
           demotenant1-backend-5754bbfff-wq9rd   1/1   Terminating         0     3h4m
21:45:41   demotenant1-backend-5754bbfff-h4l88   1/1   Running             0     21s
```

Pod-condition timestamps on the new pod
(`kubectl get pod ... -o jsonpath='{.status.conditions[*]}'`):

```text
creation:                  2026-06-12T19:45:20Z
PodScheduled:              2026-06-12T19:45:20Z  True
Initialized:               2026-06-12T19:45:20Z  True
PodReadyToStartContainers: 2026-06-12T19:45:30Z  True
ContainersReady:           2026-06-12T19:45:41Z  True
Ready:                     2026-06-12T19:45:41Z  True
```

`/healthz` via the port-forward:

```text
21:45:19  200       baseline
21:45:20  200       delete issued
21:45:21  200       port-forward still tunnelling to the Terminating pod
...
21:45:50  200       last successful response on the old tunnel
21:45:51  000       port-forward connection broke
...
21:46:04  200       port-forward restarted, picks the new pod
```

The `000` window is a `kubectl port-forward svc/...` artefact: the tunnel
binds to one Pod IP at start time, and when that pod terminates the tunnel
breaks regardless of whether the replacement is ready. End-users hitting
the Service through Traefik see a smaller window — the Service endpoint
set is empty only between the old pod losing readiness and the new pod
gaining it, i.e. the same 21 s as the pod-condition window above.

### Post-state

```text
$ kubectl get deploy -n tenant-demotenant1 demotenant1-backend
NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
demotenant1-backend   1/1     1            1           3d20h
```

Deployment back to `1/1` ready; no operator intervention required.

### Proves

- The ReplicaSet controller (`demotenant1-backend-5754bbfff`, owned by
  the Deployment) spawned a replacement within 1 s (`ContainerCreating`
  at +1 s) — the pod-template hash stayed the same, so no new
  ReplicaSet was created, just a `desired=1, actual=0` self-heal.
- The container reached `Running` after 10 s (image cached on the node).
- The readiness probe held the new pod out of the Service endpoint set for
  a further 11 s — one full `periodSeconds: 10` cycle plus probe
  processing — matching the configured probe semantics.
- Service availability gap: **21 s** (delete → new pod Ready). No operator
  action; standard rolling-restart self-heal.
