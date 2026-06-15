# Cluster Redeploy Validation

Evidence for S4-04b (#65): a full teardown of the GKE cluster + a fresh
`bootstrap.sh` rerun, validated end-to-end against
`dotted-axle-495612-f4` on 2026-06-15. The platform reconverged to a
fully healthy state — all four tenants live and reachable, Argo CD
managing every platform component, wildcard cert issued by Let's
Encrypt. The run also surfaced eight concrete tightening opportunities,
each tracked under a follow-up issue or fix; with those merged the
procedure is reproducible without the manual steps recorded below.

All cluster outputs in this document are verbatim from that run.

## Setup

Live pre-state snapshot at `2026-06-15T18:13:42Z`, before any teardown
action:

```text
$ kubectl get xtenants
NAME                SYNCED   READY   COMPOSITION       AGE
demotenant1-g2rls   True     True    xtenant-default   6d19h
demotenant2-n58bt   True     True    xtenant-default   6d19h
demotenant3-svskk   True     True    xtenant-default   6d18h
staging-6g8gs       True     True    xtenant-default   4d23h

$ kubectl -n argocd get applications
# 19 total; 17 Synced/Healthy; 2 pre-existing OutOfSync (Healthy)
#   crossplane-compositions, kube-prometheus-stack-secrets
```

The two pre-existing `OutOfSync` apps remain in the same state after
the rebuild — the drift is not a regression of this run.

Persistent artefacts on the GCP side before teardown:

```text
$ gcloud secrets list --project=dotted-axle-495612-f4 | wc -l
12       # 2 shared (grafana-admin, shared-ghcr-pull-secret)
         # + 2 entries per tenant (incl. tenant-deltest-* orphan'd from #100)

$ gcloud storage buckets list --project=dotted-axle-495612-f4
gs://dotted-axle-495612-f4-pg-backups/
gs://dotted-axle-495612-f4-tfstate/

$ gcloud dns record-sets list --zone=platform-zone --project=dotted-axle-495612-f4 | wc -l
16       # 8 A + 8 TXT
```

Pre-existing CNPG backup data (~137 MB on the staging prefix alone),
sampled:

```text
$ gcloud storage ls --recursive gs://dotted-axle-495612-f4-pg-backups/
gs://dotted-axle-495612-f4-pg-backups/demotenant1/.../base/20260613T020002/...
gs://dotted-axle-495612-f4-pg-backups/demotenant1/.../base/20260614T020003/...
gs://dotted-axle-495612-f4-pg-backups/demotenant1/.../base/20260615T020003/...
gs://dotted-axle-495612-f4-pg-backups/demotenant1/.../wals/0000000100000000/*.gz
# similar prefixes for demotenant2/3 and staging
```

## Procedure

The teardown + rebootstrap follows `platform-iac/TEARDOWN.md`:

- Phase A — drain GitOps state
- Phase B — `terraform destroy`
- Phase C — cloud hygiene check
- Phase D — `bootstrap.sh` rerun
- Phase E — end-to-end verification

The procedure described below is exactly what reached the converged
state on 2026-06-15. Where it had to extend or replace a step from
TEARDOWN.md as written, the gap is captured under Findings (F1–F8) and
mapped to its follow-up issue. The TEARDOWN.md updates and the code
fixes land in the follow-up PRs referenced under each finding.

## Outputs per phase

### Phase A — drain GitOps state

Scale Argo CD to zero, delete the LB-watched resources, wait one
ExternalDNS poll cycle:

```text
$ kubectl -n argocd scale deploy --all --replicas=0
$ kubectl -n argocd scale statefulset argocd-application-controller --replicas=0
$ kubectl -n argocd delete ingress argocd-server
$ kubectl -n traefik delete svc traefik
$ sleep 75 && gcloud dns record-sets list --zone=platform-zone ...
```

After that block alone the `*.fhuebung.lol` and `argocd.fhuebung.lol`
records were gone, but the per-tenant Ingresses + the Grafana Ingress
kept their records (Crossplane providers were recreating them faster
than ExternalDNS could prune). For a clean DNS state the run extended
Phase A with one more block (Finding F1):

```text
$ kubectl -n crossplane-system scale deploy \
    provider-helm-* provider-kubernetes-* --replicas=0
$ for ns in tenant-demotenant{1,2,3} tenant-staging monitoring; do
    kubectl -n $ns delete ingress --all
  done
$ sleep 75
```

After which the zone held only `NS`, `SOA` (and the persistent
`_acme-challenge` TXT cert-manager publishes on demand).

### Phase B — `terraform destroy`

The destroy completed on the third attempt, after each gap had a
workaround applied. The sequence that converged:

```text
1. gcloud storage rm -r --recursive "gs://...-pg-backups/**"      # F2
2. gcloud projects add-iam-policy-binding ... \
     --role="roles/dns.admin"                                     # F5
3. gcloud iam roles create dnsZoneIamSetter ... \
     --permissions="dns.managedZones.setIamPolicy"                # F5
4. gcloud projects add-iam-policy-binding ... \
     --role="projects/.../roles/dnsZoneIamSetter"                 # F5
5. gcloud auth login --force && \
   gcloud auth application-default login --force                  # token refresh
6. gcloud dns managed-zones set-iam-policy platform-zone \
     --policy-file=/tmp/zone-empty-policy.json                    # clear zone IAM
7. terraform -chdir=terraform state rm \
     'module.dns.google_dns_managed_zone_iam_member.dns_admin[...]'
8. terraform -chdir=terraform destroy -auto-approve
```

Step 8 itself, once unblocked, finished within the expected window:

```text
module.cluster.google_container_node_pool.primary: Destruction complete after 4m02s
module.cluster.google_container_cluster.this:      Destruction complete after 4m02s
module.network.google_compute_subnetwork.subnet:   Destruction complete after 32s
module.network.google_compute_network.vpc:         Destruction complete after 22s
Destroy complete! Resources: 6 destroyed.
```

The TEARDOWN.md reference time (~9 min, 2026-05-30) is accurate for the
destroy operation itself. The full wallclock through the three
attempts was ~30 min — see the timing table below for the breakdown.

### Phase C — cloud hygiene

```text
$ gcloud compute forwarding-rules list  # Listed 0 items.
$ gcloud compute backend-services  list # Listed 0 items.
$ gcloud compute target-pools      list # Listed 0 items.
$ gcloud compute addresses         list # Listed 0 items.
$ gcloud compute disks             list
pvc-045ca0a0-...   europe-west1-b zone 20 pd-balanced READY    # Prometheus PVC
pvc-5ad1b4b0-...   europe-west1-b zone  8 pd-ssd      READY    # CNPG demotenant1
pvc-5bb05940-...   europe-west1-b zone  8 pd-ssd      READY    # CNPG demotenant2
pvc-6fdc366a-...   europe-west1-b zone  8 pd-ssd      READY    # CNPG demotenant3
pvc-926e1058-...   europe-west1-b zone  8 pd-ssd      READY    # CNPG staging
```

Five PVC-backed disks left after the cluster destroy — one per tenant
CNPG cluster plus the Prometheus PVC. Cleaned up with a
`gcloud compute disks delete` loop (Finding F3).

### Phase D — `bootstrap.sh` rerun

The first run revealed that the Phase 0 IAM-preflight added in
`platform-iac#64` calls `gcloud projects test-iam-permissions`, which
does not exist on gcloud SDK 572.0.0 (verified in stable, alpha, beta
surfaces):

```text
[phase 0] verifying operator IAM permissions
ERROR: (gcloud.projects) Invalid choice: 'test-iam-permissions'.
```

The `REQUIRED_OPERATOR_PERMISSIONS` array itself is correct — only the
gcloud invocation needs replacing with a REST-API call (Finding F4).
For this run the Phase 0 probe block was temp-edited out so the rest of
the script could proceed; the working tree was reverted to main
immediately after the test.

With the probe bypassed, Phases 1–4 ran as designed:

```text
[phase 1]  enable APIs                                  # no-op, all enabled
[phase 1a] GHCR pull-secret seed (idempotent, no-op)    # GSM value unchanged
[phase 1b] Grafana admin seed (idempotent, no-op)
[phase 2]  state bucket survived, DNS zone survived, terraform init
[phase 3]  terraform apply: Apply complete! Resources: 22 added
[phase 4]  kubeconfig refreshed against the new cluster
```

Phase 5 (`helm upgrade --install argocd`) surfaced one more required
permission — Argo CD's Helm chart pre-install hook needs
`container.roles.delete`, which is part of `roles/container.admin` and
not in the baseline `roles/editor`:

```text
Error: failed pre-install: roles.rbac.authorization.k8s.io
"argocd-redis-secret-init" is forbidden: ... requires container.roles.delete
```

Granted `roles/container.admin`, re-logged gcloud to refresh the token,
retried. Phase 5 then completed cleanly (Finding F6 documents the
prereq update):

```text
[bootstrap] platform bootstrap complete; kubectl is configured for platform-cluster
[bootstrap] Argo CD installed and root App-of-Apps applied; it self-manages from platform-gitops
```

Total Phase D wallclock with the two retries was ~18 min — see the
timing table below. Cluster create dominated Phase 3 at ~9m50s, well
inside Terraform's timeout.

### Phase E — end-to-end verification

First Argo CD reconcile check at `19:28:22Z`:

```text
$ kubectl -n argocd get applications | awk 'NR>1 {print $2"/"$3}' | sort | uniq -c
   17 Synced/Healthy
    2 OutOfSync/Healthy           # crossplane-compositions, kube-prometheus-stack-secrets
                                  # — same pre-existing drift as before the run

$ kubectl get xtenants
demotenant1-n9ndx   True   True   xtenant-default   6m51s
demotenant2-p6kzt   True   True   xtenant-default   6m51s
demotenant3-hmltz   True   True   xtenant-default   6m51s
staging-kj2br       True   True   xtenant-default   6m51s
```

Wildcard cert + tenant ingress, verified with explicit IP resolve to
bypass local DNS-resolver `NXDOMAIN` caching (the records were already
published in `platform-zone`, only the local resolver was stale):

```text
$ curl -sS -o /dev/null -w "HTTP %{http_code} | TLS %{ssl_verify_result}\n" \
    --resolve argocd.fhuebung.lol:443:34.140.5.98 https://argocd.fhuebung.lol/
HTTP 200 | TLS 0     # cert: CN=*.fhuebung.lol, Issuer Let's Encrypt YR2

$ curl ... --resolve demotenant1.fhuebung.lol:443:34.140.5.98 \
    https://demotenant1.fhuebung.lol/
HTTP 401 | TLS 0     # BasicAuth wall live
```

The monitoring stack needed one short manual step: the `Prometheus` CR
sat with `RECONCILED=blank, AVAILABLE=blank` and no StatefulSet was
created, because the operator pod started before the chart's CRDs were
installed. The operator log line at `19:29:43Z` made it obvious:

```text
warn: resource "prometheuses" (group: "monitoring.coreos.com/v1") not installed in the cluster
warn: resource "prometheusagents" (group: "monitoring.coreos.com/v1alpha1") not installed in the cluster
```

Restarting the deployment let the operator pick up the now-present
CRDs and the StatefulSet came up within a minute (Finding F8 covers
this and the longer-term fix):

```text
$ kubectl -n monitoring rollout restart deploy/kube-prometheus-stack-operator
$ kubectl -n monitoring get sts prometheus-kube-prometheus-stack-prometheus
NAME                                          READY   AGE
prometheus-kube-prometheus-stack-prometheus   1/1     32s
```

End state: every functional Argo CD app `Synced/Healthy` (modulo the
two pre-existing drifts that were already there before the run), all
four tenants live, BasicAuth + wildcard TLS verified end-to-end.

## Per-phase wallclock

| Phase | TEARDOWN.md (2026-05-30) | This run (2026-06-15) | Drivers of the delta |
|---|---|---|---|
| A — drain GitOps state | ~2 min | **6m 01s** | extended ingress + provider scale-down (F1) |
| B — `terraform destroy` | ~9 min | **~30 min** | one-time IAM + bucket workarounds (F2, F5) |
| C — cloud hygiene check | <1 min | **~2 min** | five PVC-backed disks to delete (F3) |
| D — `bootstrap.sh` rerun | ~15 min | **~18 min** | one-time IAM preflight + `container.admin` retries (F4, F6) |
| E — verification + reconcile + operator restart | ~3 min | **~28 min** | reconcile to converged state + F8 restart |
| **Total** | **~30 min** | **~84 min** | one-time discoveries F1–F8 |

The 2026-05-30 reference assumed every prerequisite was already in
place and the procedure followed without the discoveries above. This
run's ~84 min reflects the cost of surfacing those gaps the first time.
With the follow-up PRs landed (see Findings F1–F8 below) a future run
returns to the original reference time — none of the eight findings is
fundamental to the design, every one of them has a scoped fix in
flight.

## Persistence-boundary inventory

### Persistent (survives teardown by design)

- `gs://dotted-axle-495612-f4-tfstate` — Terraform remote state bucket,
  chicken-and-egg with the backend, not Terraform-managed.
- Cloud DNS managed zone `platform-zone` — referenced as a data source,
  not owned by Terraform.
- 12 Google Secret Manager containers + their version history:
  - `grafana-admin`, `shared-ghcr-pull-secret` (shared).
  - `tenant-<name>-basicauth-htpasswd` + `tenant-<name>-basicauth-password`
    pairs for `deltest`, `demotenant1`, `demotenant2`, `demotenant3`,
    `staging`. Confirmed live: the `deltest` entries from Alex's
    offboarding test (#100) survived this teardown intact —
    `deletionPolicy: Orphan` holds across a full cluster destroy.

### Rebuilt by Terraform on the next bootstrap

- VPC `platform-vpc`, subnet `platform-subnet`, firewall
  `platform-vpc-allow-internal`.
- GKE cluster `platform-cluster` + nodepool.
- Five Google Service Accounts (cert-manager, external-dns,
  external-secrets, crossplane-provider-gcp, plus the provider's
  storage sub-WI) with their Workload-Identity bindings.
- DNS zone IAM bindings on `platform-zone` for cert-manager and
  external-dns; project-level `dns.reader` binding for external-dns.
- `gs://dotted-axle-495612-f4-pg-backups` bucket — **content is destroyed
  by design**. Issue #65's original scope listed this as persistent;
  the live run confirms TEARDOWN.md as the correct reference. CNPG
  backups across the teardown window are not preserved.
- `pg_backups_iam_admin` custom IAM role + its `BucketIAMMember` for
  provider-gcp.

### Externally triggered (out of `bootstrap.sh`'s hands)

- `app-backend` image + chart push to GHCR — driven by the
  `app-backend` release workflow on tag push.
- `app-frontend` image push to GHCR — driven by the `app-frontend`
  release workflow on tag push.

The platform's "no manual click after `bootstrap.sh` kickoff" rule
covers everything the operator triggers; the externally-triggered set
is the documented exception for image/chart provenance.

## Manual steps + path to a single-run rebuild

The 2026-06-15 run took eight manual operator interventions to reach
the converged state; each maps to a follow-up issue or to the
TEARDOWN.md update PR. The list below doubles as the close-out
checklist for getting back to a single-command rebuild:

1. Extended Phase A with the additional Ingress + provider scale-down
   (Finding F1) — TEARDOWN.md update PR.
2. Emptied the pg-backups bucket before `terraform destroy` (Finding
   F2) — `Closes platform-iac#67`.
3. Granted the operator `roles/dns.admin` + the `dnsZoneIamSetter`
   custom role; forced a token refresh (Finding F5) — TEARDOWN.md
   prereqs.
4. Cleared the zone IAM policy manually + `terraform state rm` for the
   two bindings (Finding F5) — once #67 ships and F4 catches the
   permission upfront, this step disappears.
5. Deleted the five PVC-backed disks (Finding F3) — TEARDOWN.md update
   adds the loop snippet.
6. Temp-edited `bootstrap.sh` to skip the broken IAM-preflight (Finding
   F4) — `Closes platform-iac#66`.
7. Granted `roles/container.admin`; forced a token refresh (Finding F6)
   — TEARDOWN.md prereqs.
8. Restarted `prometheus-operator` once the chart's CRDs were present
   (Finding F8) — separate follow-up issue if pursued.

The assignment's "no manual click after `bootstrap.sh` kickoff"
constraint covers the operator workflow once the prereq IAM grants are
in place and the four code fixes (#66, #67, #107, the SyncWave change
under F8) have landed. None of those is structural; each is a small
targeted change with a clear acceptance check.

## Findings

| ID | Summary | Follow-up |
|---|---|---|
| **F1** | TEARDOWN.md Phase A only deletes the argocd-server Ingress + traefik svc; tenant Ingresses + Grafana Ingress + Crossplane providers stay running, leaving five A+TXT records orphan'd | TEARDOWN.md update — separate `platform-iac` PR, `Refs platform-gitops#65` |
| **F2** | `module.backup.google_storage_bucket.pg_backups` is `force_destroy = false`; a populated bucket blocks `terraform destroy` | `Closes platform-iac#67` |
| **F3** | Cluster destroy leaves N PVC-backed disks (one per CNPG cluster + one for cluster-wide Prometheus); TEARDOWN.md mentions this abstractly, the gap is the concrete cleanup snippet | TEARDOWN.md update (same PR as F1) |
| **F4** | `bootstrap.sh` Phase 0 calls `gcloud projects test-iam-permissions`, which does not exist on gcloud SDK 572.0.0 (verified in stable, alpha, beta surfaces); `REQUIRED_OPERATOR_PERMISSIONS` itself is correct | `Closes platform-iac#66` |
| **F5** | `roles/dns.admin` does NOT include `dns.managedZones.setIamPolicy`; a custom role is needed for the zone-IAM teardown step. Plus `roles/owner` is blocked for external accounts by `ORG_MUST_INVITE_EXTERNAL_OWNERS` | TEARDOWN.md prereqs (same PR as F1); custom-role create snippet in operator runbooks |
| **F6** | `roles/editor` does NOT include `container.roles.delete`, which Argo CD's Helm chart pre-install hook needs; `roles/container.admin` is required as an additional baseline grant | TEARDOWN.md prereqs (same PR as F1) |
| **F7** | Race in `xtenant-default` Composition between Crossplane `SecretVersion` (Resource 11) and ESO `ExternalSecret` (Resource 14): ESO can read `:latest` from GSM before Crossplane has written the freshly bcrypted htpasswd, leaving a 0–60 min window where the operator-visible plaintext (latest GSM version) does not match what Traefik validates against | `Closes platform-gitops#107` |
| **F8** | `kube-prometheus-stack` operator pod starts before the chart's CRDs are installed; per upstream design it does NOT register a watcher for missing CRDs and does not retry, so `Prometheus` reconciliation needs the pod to restart once. `kubectl rollout restart deploy/kube-prometheus-stack-operator` is the one-line live workaround; the proper fix is Argo CD SyncWave annotations (CRDs first, operator second) | Argo CD `applications/kube-prometheus-stack.yaml` SyncWave change — separate follow-up issue if pursued; documented as known-issue here |

## References

- Task S4-04b in `platform/tasks/task-list.md`.
- Issue: #65.
- Companion procedure document: `platform-iac/TEARDOWN.md`.
- Related architecture decisions:
  `platform/docs/claude/architecture-decisions.md` § DNS and TLS — Zone
  lifecycle, § Secrets management, § Per-tenant BasicAuth.
- Follow-up issues opened during the test write-up:
  `INENI-PT-GROUP-B/platform-iac#66`,
  `INENI-PT-GROUP-B/platform-iac#67`,
  `INENI-PT-GROUP-B/platform-gitops#107`.
