# Longhorn wrapper chart

## Purpose

This chart installs the unchanged Longhorn 1.8.2 dependency from
`https://charts.longhorn.io`. It also manages four StorageClasses, the daily
snapshot cleanup job, a VolumeSnapshotClass, a BackupTarget, a Traefik
IngressRoute, and the MinIO credential sync.

The release name is `longhorn`. The namespace is `longhorn-system`. Chart
version `1.0.0` is the source version. CI publishes this wrapper to GHCR.

## Before first install

1. Check that Traefik, the snapshot CRDs, and the Infisical operator with
   `InfisicalStaticSecret` support are available.
2. Check that `infisical-prod-universal-auth` in `default` can read
   `/minio/users/longhorn` recursively in project `k3s-ntpo`, environment `prod`.
   The two keys are `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.
3. Check that the TLS Secret `prod-raulstechtips-tls` and the `default-headers`
   and `oauth-chain` middlewares exist for the route.
4. Build and publish the chart through CI. Set `PENDING_CI_VERSION` in
   `cluster-config/argo/dev/longhorn.yaml` to the first published version.
5. Complete the secret handover in Adoption notes before the first manual sync.
   The Application is not set to sync automatically.

## Values reference

| Value | Default | Use |
| --- | --- | --- |
| `infisical.authRef` | Empty name and namespace | Shared InfisicalAuth reference |
| `infisical.projectSlug`, `environmentSlug` | Empty | Infisical project and environment |
| `minioSecret.enabled` | `false` | Enable credential sync |
| `minioSecret.secretName` | `minio-secret` | Existing target Secret and new static-secret name |
| `minioSecret.secretPath` | `/minio/users/longhorn` | Infisical source path |
| `minioSecret.recursive` | `true` | Include child paths |
| `minioSecret.refreshInterval` | `60s` | Sync interval |
| `minioSecret.endpoint` | Empty | Literal `AWS_ENDPOINTS` value; required when enabled |
| `ingressRoute.enabled` | `false` | Enable `longhorn-ingress` |
| `ingressRoute.entryPoints` | `[websecure]` | Traefik entry points |
| `ingressRoute.ingressClass` | `traefik-external` | Ingress class annotation |
| `ingressRoute.host`, `tlsSecretName` | Empty | Required when route is enabled |
| `ingressRoute.middlewares` | `[]` | Middleware name and namespace list |
| `storageClasses` | Four live policies | Name, reclaim policy and exact parameter strings |
| `recurringJobs` | Daily cleanup at 03:00 | Name, task, cron, retain and concurrency |
| `volumeSnapshotClasses` | One default Delete class | Name, default annotation and deletion policy |
| `backupTargets` | `[]` | Name, URL, credential Secret name and poll interval |
| `longhorn` | Live non-environment overrides | Values passed unchanged to the dependency |

Set a resource list to `[]` to disable it. Environment values are in
`cluster-config/apps/dev/longhorn.yaml`. No credential values are stored in
either file. All StorageClasses use `driver.longhorn.io`, volume expansion,
and `Immediate` binding. Do not change existing StorageClass parameters:
Kubernetes does not permit these changes.

The upstream overrides keep orphan auto-deletion on, log level `info`, one
UI replica and JSON manager logs. The inactive old `manifests/overlays/`
patch is not included.

## Upgrades

Keep the dependency pinned. Change `charts/Chart.yaml`, then run
`/opt/homebrew/bin/helm dependency update charts` and retain `Chart.lock`.
Review upstream release instructions before any Longhorn upgrade.

Run the local checks with a private temporary directory:

```sh
export TMPDIR=/tmp/ulw-wrappers/longhorn/tmp
mkdir -p "$TMPDIR"
/Users/raul/Desktop/Projects/HomeLab/.omo/wrapper-charts/tools/checks.sh \
  /Users/raul/Desktop/Projects/HomeLab/longhorn /tmp/ulw-wrappers/longhorn
```

Render the environment with:

```sh
/opt/homebrew/bin/helm template longhorn charts -n longhorn-system \
  -f ../cluster-config/apps/dev/longhorn.yaml
```

## Adoption notes

The live Application `longhorn-system` tracks 47 resources, including 22
Longhorn CRDs from upstream `templates/crds.yaml`. Keep those CRDs rendered;
do not set `skipCrds`. Keep the CRD ignore rule for
`/spec/preserveUnknownFields`, with `RespectIgnoreDifferences=true`.

The existing `longhorn-storage-class-replicas-1` is OutOfSync. Its storage
fields match the source, but its Argo tracking annotation is absent. Its
managed fields show `kubectl-client-side-apply` on August 22, 2026, and its
last-applied configuration has an empty annotations map. This supports a
tracking-metadata cause, not a need to replace immutable parameters. The
parity tool ignores tracking annotations, so it cannot expose that drift.
The new Application keeps the live name and will set tracking metadata on sync.

`longhorn-storage-class-replicas-1-delete` and BackupTarget `default` already
exist live without Argo tracking. Both have manifests in the operator's
working copy. This chart adopts them and keeps their policy fields.
The BackupTarget comments are retained. Its old manifest path in the comment
is historical; `templates/minio-secret.yaml` now manages that sync.
The controller-owned `syncRequestedAt` timestamp is not managed by this chart.

The RecurringJob keeps `longhorn.io/v1beta1`, as in the old source. The live
CRD serves both `v1beta1` and `v1beta2`, and stores `v1beta2`.

Before the first sync, the operator must orphan-delete the old credential
sync object so that the existing Secret stays:

```sh
kubectl delete infisicalsecret infisical-secret-longhorn-minio-secret \
  -n longhorn-system --cascade=orphan
```

This command is a handover step, not part of the local checks. The new
`InfisicalStaticSecret/minio-secret` uses `creationPolicy: Orphan`, the same
recursive source, the same two credential templates, and the same literal
endpoint `https://s3-api.prod.raulstechtips.dev`. Do not let both sync
controllers write the same Secret at the same time.

The old, untracked
`InfisicalSecret/infisical-secret-s3-backup-longhorn-backup-target` is not
adopted or removed. It targets a different Secret and reports a missing old
Infisical project. Its cleanup is outside this chart's scope.

The project changes from `default` to `platform`, the chart source changes
to GHCR, and the environment values use the existing dev directory shape.
The release name, namespace and Application name stay the same.

### Render-versus-live result

Local evidence is in `../.omo/evidence/wrappers/longhorn/`.

```text
SUMMARY SAME=32 CHANGED=16 NEW=1 PRUNE=1 HOOK=3 UNKNOWN-KIND=0
```

Every difference is listed below. No upstream workload or configuration
value differs. Eight upstream objects have explicit `false` fields that the
API omits. These fields are hard-coded by upstream, not controlled by values.
The dependency templates are not changed to hide these differences.

| Result | Object | Explanation |
| --- | --- | --- |
| CHANGED | PriorityClass `longhorn-critical` | Upstream emits `globalDefault: false`; live omits this optional false field. The class does not become the global default. |
| CHANGED | CRD `engineimages.longhorn.io` | Upstream emits `spec.preserveUnknownFields: false`; live omits it. The retained Argo ignore rule covers this field. |
| CHANGED | CRD `engines.longhorn.io` | Same omitted false field; covered by the retained CRD ignore rule. |
| CHANGED | CRD `instancemanagers.longhorn.io` | Same omitted false field; covered by the retained CRD ignore rule. |
| CHANGED | CRD `nodes.longhorn.io` | Same omitted false field; covered by the retained CRD ignore rule. |
| CHANGED | CRD `replicas.longhorn.io` | Same omitted false field; covered by the retained CRD ignore rule. |
| CHANGED | CRD `settings.longhorn.io` | Same omitted false field; covered by the retained CRD ignore rule. |
| CHANGED | CRD `volumes.longhorn.io` | Same omitted false field; covered by the retained CRD ignore rule. |
| CHANGED | StorageClass `longhorn-default-storage-class` | Added wrapper labels only; all policy fields and the complete parameter map match live exactly. |
| CHANGED | StorageClass `longhorn-storage-class-local` | Added wrapper labels only; all policy fields and the complete parameter map match live exactly. |
| CHANGED | StorageClass `longhorn-storage-class-replicas-1` | Added wrapper labels only; all policy fields and the complete parameter map match live exactly. |
| CHANGED | StorageClass `longhorn-storage-class-replicas-1-delete` | Added wrapper labels only; adopts the hand-applied class with its Delete policy. Complete parameters match live exactly. |
| CHANGED | BackupTarget `default` | Added wrapper labels only; adopts the hand-applied target without managing its runtime timestamp. |
| CHANGED | IngressRoute `longhorn-ingress` | Added wrapper labels only; route, middleware order, service and TLS reference match live. |
| CHANGED | RecurringJob `cleanup-snapshots` | Added wrapper labels only; schedule, retention, task and concurrency match live. |
| CHANGED | VolumeSnapshotClass `longhorn-volume-snapshot-class` | Added wrapper labels only; default annotation, driver and Delete policy match live. |
| NEW | InfisicalStaticSecret `minio-secret` | Intended migration to the new operator API. The target Secret already exists. |
| PRUNE | InfisicalSecret `infisical-secret-longhorn-minio-secret` | Replaced by the new static-secret object after the orphan-delete handover step. |

The three upstream hooks are unchanged: `longhorn-pre-upgrade` and
`longhorn-post-upgrade` run during Argo sync; the upstream `longhorn-uninstall`
pre-delete hook is not supported by the installed Argo CD v3.0.0 and is ignored.
The parity tool groups all three under `HOOK`; it does not compare their
live bodies. No unknown resource kind was found.

Local checks passed: 8 suites, 31 tests. Kubeconform checked the default
render (49 resources: 27 valid, 22 CRDs skipped) and all-on render
(52 resources: 29 valid, 23 CRDs/Infisical objects skipped). Both had
0 invalid resources and 0 errors. The catalog supplied schemas for
RecurringJob, BackupTarget, IngressRoute and VolumeSnapshotClass.
All authored YAML, the schema JSON, the lock file and the complete dev render
parsed with `yq`. Raw Helm templates are checked through their rendered YAML.
The YAML language server is not installed; Helm and kubeconform checks were
used instead. No package was installed to change the shared workspace.
