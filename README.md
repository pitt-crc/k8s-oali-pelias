# Pelias New York City

Pelias is an open-source geocoder built on Elasticsearch. It turns addresses and place names into coordinates, and
answers search queries against data built from OpenStreetMap, OpenAddresses, Who's On First, TIGER, and MTA subway
stops. The manifests here are a translation of the original
[Docker Compose project](https://github.com/pelias/docker/tree/master/projects/new-york-city)
for New York City, which is linked above.

The original project downloads data from multiple sources, including a free community-built map called OpenStreetMap
(OSM). The upstream OPSM URL used in the original documentation (commit
[fdb4a01](https://github.com/pelias/docker/commit/fdb4a01c7d5addb04bba8e5129497e50eba784ce))
no longer works, so this deployment switched to using data from Geofabrik instead.
Geofabrik’s file covers all of New York State rather than just the city, so the import is larger and takes longer.
The modified URL data can be found in `config/pelias.json` and are listed below:

- **Original URL:** https://s3.amazonaws.com/metro-extracts.nextzen.org/new-york_new-york.osm.pbf
- **Replacement URL:** https://download.geofabrik.de/north-america/us/new-york-latest.osm.pbf

## What is deployed

The original Pelias project is deployed using Docker Compose and requires the `pelias` helper command to run build
steps in a specific order. This repository replaces that manual process with Kubernetes Jobs and Argo CD sync waves.

Argo CD applies resources one wave at a time, in ascending order, and waits for everything in a wave to be healthy
before starting the next. For Jobs, healthy means completed. A failed Job stops the sync at that wave. The waves below
reproduce the order the helper command enforced:

| Wave     | Resources                                          | Replaces                                         |
|----------|----------------------------------------------------|--------------------------------------------------|
| 0        | ConfigMaps, data PVC, Elasticsearch, libpostal     | `pelias elastic start` and `pelias elastic wait` |
| 1        | Schema Job                                         | `pelias elastic create`                          |
| 2        | Download Jobs (parallel)                           | `pelias download all`                            |
| 3        | Prepare Jobs: polylines and placeholder (parallel) | `pelias prepare all`, first half                 |
| 4        | Prepare Job: interpolation                         | `pelias prepare all`, second half                |
| 5 to 9   | Import Jobs, one per wave                          | `pelias import all`                              |
| 10       | Pelias services                                    | `pelias compose up`                              |
| PostSync | Fuzzy-tester Job                                   | `pelias test run`                                |

The manifests are organized into directories by purpose.
The sections that follow describe each one in the order listed here.

| Directory     | Contents                                                                 |
|---------------|--------------------------------------------------------------------------|
| `base/`       | Storage, Elasticsearch, and libpostal. Everything else depends on these. |
| `build/`      | The one-shot Jobs that create the index and build the data.              |
| `serve/`      | The Pelias services that answer queries.                                 |
| `test/`       | The acceptance-test hook.                                                |
| `config/`     | `pelias.json` and the OpenStreetMap blacklist, turned into ConfigMaps.   |
| `test_cases/` | The fuzzy-tester test cases, turned into a ConfigMap.                    |

### Base (Wave 0)

The _base_ manifests deploy the foundational application components.
Data volumes are annotated `Prune=false`, so removing it from git does not delete the data.

| File                 | Resources             | Details                                                                |
|----------------------|-----------------------|------------------------------------------------------------------------|
| `pvc.yaml`           | PersistentVolumeClaim | (50Gi, `ReadWriteOnce`) use to store all downloaded and prepared data. |
| `elasticsearch.yaml` | StatefulSet, Service  | Single-node Elasticsearch with its own 20Gi volume.                    |
| `libpostal.yaml`     | Deployment, Service   | Address-parsing service on port 4400.                                  |

### Build (Waves 1 to 9)

The _build_ manifests are used to download and prepare data from external sources.
Downloads are spread across multiple waves to lessen disk pressure on host nodes.

| File               | Wave    | Jobs                                                                                              |
|--------------------|---------|---------------------------------------------------------------------------------------------------|
| `01-schema.yaml`   | 1       | Create the `pelias` index.                                                                        |
| `02-download.yaml` | 2       | Download Who's On First, OpenAddresses, OpenStreetMap, TIGER, and transit data.                   |
| `03-prepare.yaml`  | 3 and 4 | Build polylines and the placeholder database (wave 3), then the interpolation databases (wave 4). |
| `04-import.yaml`   | 5 to 9  | Import Who's On First, OpenAddresses, OpenStreetMap, polylines, and transit, in that order.       |

### Serve (Waves 10)

The _serve_ manifests deploy the application APIs and launch all end user services.

| File                 | Service                                  | Port |
|----------------------|------------------------------------------|------|
| `api.yaml`           | Pelias API                               | 4000 |
| `placeholder.yaml`   | Placeholder (administrative-area lookup) | 4100 |
| `pip.yaml`           | Point-in-polygon service                 | 4200 |
| `interpolation.yaml` | Address interpolation                    | 4300 |
| `quota.yaml`         | Namespace rsource limits                 | N/A  |

### Test

The _test_ manifests use an Argo CD PostSync hook to validate deployed resources and make sure they are healthy.
The **`fuzzy-tester.yaml`** manifest runs the Pelias fuzzy-tester against `http://api:4000/v1/` using the cases
from `test_cases/`. If any test fails, the sync is reported as failed.

### Config and Test Cases

The _config_ and _test_cases_ directories hold the inputs that Kustomize turns into ConfigMaps.
When changing the config files the application setup jobs must be deleted and recreated, as described in
[Apply a configuration change](#apply-a-configuration-change).

| File                       | ConfigMap              | Details                                                                                 |
|----------------------------|------------------------|-----------------------------------------------------------------------------------------|
| `config/pelias.json`       | `pelias-config`        | Read by every Pelias container: Elasticsearch settings, service URLs, and data sources. |
| `config/osm-blacklist.txt` | `pelias-osm-blacklist` | Mounted at `/data/blacklist/osm.txt`. Empty by default.                                 |

## Common tasks

### Smoke test the API

Send a search and a reverse-geocode query through a port-forward.

```bash
kubectl port-forward svc/api 4000:4000 -n <namespace>

# In a second terminal
curl 'http://localhost:4000/v1/search?text=111+8th+ave+nyc'
curl 'http://localhost:4000/v1/reverse?point.lat=40.741&point.lon=-74.004'
```

### Check index health and counts

Confirm Elasticsearch is healthy and each import loaded data.

```bash
# Cluster health and index size
kubectl exec elasticsearch-0 -n <namespace> -- curl -s 'localhost:9200/_cluster/health?pretty'
kubectl exec elasticsearch-0 -n <namespace> -- curl -s 'localhost:9200/_cat/indices?v'

# Document counts per source and layer
kubectl exec elasticsearch-0 -n <namespace> -- curl -s 'localhost:9200/pelias/_search?size=0&pretty' \
  -H 'Content-Type: application/json' \
  -d '{"aggs":{"sources":{"terms":{"field":"source","size":100},"aggs":{"layers":{"terms":{"field":"layer","size":100}}}}}}'
```

### Full rebuild

Drop the index and rerun every build step.

Use this when sources change or the index may be in a bad state.
If you changed which sources are used, also remove stale directories from the data volume.

```bash
# Delete the index, then the build Jobs
kubectl exec elasticsearch-0 -n <namespace> -- curl -s -X DELETE 'localhost:9200/pelias'
kubectl delete jobs -l app.kubernetes.io/component=build -n <namespace>

# Sync and watch progress
argocd app sync <argocd-app>
kubectl get jobs -n <namespace> -w
```

### Reimport one source

Rerun a single import.

To a single data source, delete the matching `pelias-download-*` job and rerun the matching `pelias-download-*` job.
A reimport updates documents in place but does not remove records dropped from the source, so
use a full rebuild when you need a clean index.

```bash
kubectl delete job pelias-import-oa -n <namespace>
argocd app sync <argocd-app>
```

### Apply a configuration change

To deploy an edited config file, delete the old sync job and do a full rebuild.

```bash
# Edit config/pelias.json, commit, and push, then:
kubectl delete jobs -l app.kubernetes.io/component=build -n <namespace>
argocd app sync <argocd-app>
```
