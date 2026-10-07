# Pelias New York City on Kubernetes (Argo CD sync waves)

Kustomize layout translated from `pelias/docker/projects/new-york-city`.
Point the Argo CD Application at this directory; it renders `kustomization.yaml`.

## Sync waves (replaces the `pelias` helper script)

| Wave | Resources | Replaces |
|---|---|---|
| 0 | PVC `pelias-data`, ConfigMaps, Elasticsearch StatefulSet + Service, libpostal | `elastic start`, `elastic wait` (readiness probe = yellow health) |
| 1 | Job `pelias-schema-create` | `elastic create` |
| 2 | Download Jobs: wof, oa, osm, tiger, transit (parallel) | `download all` |
| 3 | Prepare Jobs: polylines, placeholder (parallel) | `prepare all` (first half) |
| 4 | Prepare Job: interpolation | `prepare all` (second half) |
| 5-9 | Import Jobs, one per wave: wof, oa, osm, polylines, transit | `import all` |
| 10 | Deployments + Services: placeholder, pip, interpolation, api | `compose up` |
| PostSync | Job `pelias-fuzzy-tester` | `test run` |

Omitted on purpose: geonames (only runs when `ENABLE_GEONAMES=true`) and csv-importer
(no CSV config in `pelias.json`). Add them back if you need them.

## Changes from the upstream project

- `config/pelias.json`: only `api.defaultParameters.focus.point` changed (Portland -> NYC).
- Compose service hostnames are preserved as Service names, so the rest of `pelias.json` is untouched.
- `config/osm-blacklist.txt` is the (empty) upstream `blacklist/osm.txt`, mounted at `/data/blacklist/osm.txt`.

## Before the first sync

1. **Storage.** Uses the two vSphere CSI classes on this cluster. `pelias-data` (50Gi, `ReadWriteOnce`)
   uses `pitt-vcf-vks-storage-policy` (Immediate binding, which Argo CD needs so wave 0 can finish).
   Elasticsearch's volume (20Gi) uses `pitt-vcf-vks-storage-policy-latebinding`.
   Because the data volume is RWO, every pod that mounts it carries the label `pelias.io/uses-data: "true"`
   and a required podAffinity on that label (topology `kubernetes.io/hostname`), so they all land on one
   node. The first such pod schedules anywhere; later ones follow it. That node must have room for the
   concurrent pods in a wave. If your vSphere environment provides an RWX class (vSAN File Services),
   switch `pelias-data` to `ReadWriteMany` and delete the affinity block.
   Sizes (50Gi data, 20Gi Elasticsearch) are estimates; adjust after the first build.
2. **Security context.** Pods run as uid/gid 1000 (the `DOCKER_USER` equivalent) with `seccompProfile: RuntimeDefault`, non-root, no privilege escalation and all capabilities dropped, to satisfy the Pod Security `restricted` level. Change
   `runAsUser`/`fsGroup` if your cluster requires a specific range.
3. **Elasticsearch host settings.** Heap is `-Xms2g -Xmx2g` with a 4Gi limit (guess). If ES
   exits on `vm.max_map_count`, the node setting must be raised by a cluster admin.
   The compose file's `memlock`/`IPC_LOCK` is not reproduced.
4. **Resources.** Requests/limits (including `ephemeral-storage`) are starting points, not measured values.
   Ephemeral storage is node disk: the container's writable layer, `/tmp`, `emptyDir` and logs (not the
   PVC, and not the image). A pod that exceeds its ephemeral limit is evicted, so if an import is evicted
   for that reason, raise its limit. The OSM importer uses `/tmp` for its leveldb (`leveldbpath`), so it has
   the largest allowance (2Gi request / 8Gi limit).
5. **Upstream URLs.** The OSM extract (Nextzen S3) and MTA GTFS URLs in `pelias.json` may be
   stale. A failed download Job fails wave 2 and stops the sync; remove that source from
   `pelias.json` and its Job if it is dead.
6. **Image tags.** `:master` matches the compose file; pin tags for reproducibility.
7. **Argo CD project.** Needs `Job`, `StatefulSet`, `Deployment`, `Service`, `ConfigMap`,
   `PersistentVolumeClaim` whitelisted.

## Rerunning / refreshing the build

Jobs are normal resources, so a successful sync does not rerun them and their specs are
immutable. Do **not** add `ttlSecondsAfterFinished` (Argo CD would recreate the Jobs).

To rebuild:

```bash
kubectl delete jobs -l app.kubernetes.io/component=build
kubectl apply -f ops/drop-index.yaml      # elastic drop (then delete that Job)
# then re-sync the Application
```

If you change the downloaded data sources, also clear the relevant directories on the
`pelias-data` PVC. A failed import usually requires drop-index + rerun of the schema and
import Jobs.

`pelias-data` is annotated `Prune=false` so removing it from git does not delete the data.

## Not included

- Ingress for `api` (depends on your ingress class and host).
- Auto-sync is best left off, or the PostSync test job runs on every sync.
