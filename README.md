# omap-stack

Open-source, provider-agnostic orienteering map generation at country scale.
Declare your LiDAR, DEM and vector sources in `omap.yaml`, and get a
self-hosted vector mapant with ISOM 2017-2 styling and exports.

One image runs in three modes:

- `omap queen`: PostGIS, the task queue, validation and ingest, serving through Martin, the dashboards. No heavy computation.
- `omap ant`: joins a queen over HTTPS on 443, takes Leases on Tasks, uploads Results. Runs anywhere: k8s, cloud VMs, personal machines.
- `omap colony`: a queen plus one local ant, for a single box.

The pipeline core is [karttapullautin](https://github.com/karttapullautin/karttapullautin).

Status: planning. Nothing to run yet.

## Licence

MIT. See [LICENSE](LICENSE).
