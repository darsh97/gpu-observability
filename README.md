# GPU Observability Stack (Single Node)

Production-oriented GPU observability stack for an HPC/ML node running NIM/LLM workloads.

## What this provides

- GPU telemetry from NVIDIA GPUs via `dcgm-exporter`
- Time-series storage and alert evaluation in Prometheus
- Notification routing in Alertmanager
- Dashboards in Grafana
- Persistent metrics and dashboards across container restarts/reboots
- High-frequency sampling (`5s`) for short-lived VRAM spikes

## Architecture

`NVIDIA Driver -> DCGM -> dcgm-exporter -> Prometheus -> Grafana`

`Prometheus -> Alertmanager`

Services run on one node via Docker Compose.

## Repository layout

```text
gpu-observability/
|- docker-compose.yml
|- prometheus/
|  |- prometheus.yml
|  |- alerts.yml
|- alertmanager/
|  |- alertmanager.yml
|- grafana/
   |- provisioning/
   |  |- datasources/prometheus.yml
   |  |- dashboards/dashboard.yml
   |- dashboards/gpu-observability.json
```

## Prerequisites

- NVIDIA GPU drivers installed on host
- NVIDIA Container Toolkit installed and working
- Docker Engine + Docker Compose plugin

Validate GPU runtime before deploy:

```bash
docker run --rm --gpus all nvidia/cuda:12.2.0-base nvidia-smi
```

If this command fails, `dcgm-exporter` will not provide valid GPU metrics.

## HPC node context (your environment)

The stack was designed to run alongside your existing ML infra on `ml01` (single node).

host profile from infra checks:

- GPUs: `2 x NVIDIA H100 NVL` (`~95.8 GiB` each)
- Driver: `575.57.08`
- CUDA (reported by nvidia-smi): `12.9`
- Docker runtimes include `nvidia` (default runtime still `runc`, which is OK when services use `gpus: all`)

Port planning note:

- This stack maps Grafana to host port `3030` (`3030:3000`) to avoid conflict with Attu on `3000`.

## Deploy

From repository root:

```bash
docker compose up -d
```

Check service status:

```bash
docker compose ps
```

## Access endpoints

- Grafana: `http://localhost:3030`
- Prometheus: `http://localhost:9090`
- Alertmanager: `http://localhost:9093`
- dcgm-exporter metrics: `http://localhost:9400/metrics`

Default Grafana credentials (set in compose):

- User: `admin`
- Password: `admin`

Change credentials before production exposure.

## Persistence model

Named volumes are used:

- `prometheus_data` for TSDB and WAL
- `grafana_data` for Grafana state
- `alertmanager_data` for Alertmanager runtime state

Important:

- Deleting `prometheus_data` removes all historical metrics
- Deleting `grafana_data` removes local Grafana state

## Prometheus configuration

Configured in `prometheus/prometheus.yml`:

- Scrape interval: `5s`
- Evaluation interval: `5s`
- Scrape target: `dcgm-exporter:9400` (container DNS)
- Alertmanager target: `alertmanager:9093` (container DNS)
- Rule file: `/etc/prometheus/alerts.yml`

Retention is set in `docker-compose.yml`:

- `--storage.tsdb.retention.time=15d`

## DCGM exporter compatibility mode

Some nodes fail DCGM profiling initialization with errors like:

- `The third-party Profiling module returned an unrecoverable error`

To keep telemetry stable across mixed environments, this stack mounts a custom counter set:

- `dcgm-exporter/counters.csv`

and starts exporter with:

- `-f /etc/dcgm-exporter/counters.csv`

This keeps required observability signals (VRAM used/free, utilization, thermals, XID/ECC faults) while avoiding profiling-only fields that commonly cause collector init failures on some driver/GPU/runtime combinations.

## Alerts included

Defined in `prometheus/alerts.yml`:

- `GPUVRAMSaturationHigh`
- `GPUVRAMAllocationSpike`
- `GPUVRAMSustainedPressure`

These are based on:

- VRAM saturation ratio: `FB_USED / (FB_USED + FB_FREE)`
- VRAM growth derivative: `deriv(FB_USED[2m])`

Tune thresholds to your model size, batching policy, and acceptable headroom.

## Grafana dashboard

Provisioned dashboard includes:

- VRAM Used (GiB)
- VRAM Free (GiB)
- VRAM Saturation Ratio
- GPU Utilization
- VRAM Spike Detector (MiB/s)

Datasource is provisioned to Prometheus at `http://prometheus:9090`.

## Dashboard metrics explained

`VRAM Used (GiB)`:

- Query: `DCGM_FI_DEV_FB_USED / 1024`
- Source metric: `DCGM_FI_DEV_FB_USED` (MiB)
- Meaning: Current framebuffer memory allocated on each GPU.
- What to watch:
- Steady high baseline near card capacity means little headroom for new requests/batches.
- Step jumps often map to model load, engine rebuild, or larger batch admission.

`VRAM Free (GiB)`:

- Query: `DCGM_FI_DEV_FB_FREE / 1024`
- Source metric: `DCGM_FI_DEV_FB_FREE` (MiB)
- Meaning: Remaining framebuffer memory available on each GPU.
- What to watch:
- Free memory collapsing toward zero is pre-failure pressure for allocators.
- Oscillation (sawtooth) can indicate aggressive cache/allocator churn.

`VRAM Saturation Ratio`:

- Query: `DCGM_FI_DEV_FB_USED / clamp_min(DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE, 1)`
- Unit: Percent of total VRAM in use (`0.0` to `1.0` rendered as `0%` to `100%`).
- Meaning: Normalized pressure signal independent of GPU size.
- What to watch:
- Sustained `>90%` is high-risk for CUDA OOM in real inference traffic.
- Sustained `>85%` with rising trend usually indicates degradation before hard failure.

`GPU Utilization`:

- Query: `DCGM_FI_DEV_GPU_UTIL / 100`
- Source metric: `DCGM_FI_DEV_GPU_UTIL` (percent from DCGM)
- Meaning: GPU compute activity level.
- What to watch:
- High VRAM saturation with low GPU utilization can indicate memory hoarding, not compute saturation.
- High GPU utilization with low VRAM usage suggests compute-bound kernels.

`VRAM Spike Detector (MiB/s)`:

- Query: `deriv(DCGM_FI_DEV_FB_USED[2m])`
- Meaning: Rate of VRAM growth/shrink over time window.
- What to watch:
- Large positive spikes often align with sudden batch growth, warmup/rebuild, or leak behavior.
- Repeated spikes without workload changes are a strong anomaly indicator.

Interpretation patterns:

- High `VRAM Saturation` + high `VRAM Spike Detector`:
- Approaching memory fault zone rapidly (likely allocator failures if sustained).
- High `VRAM Saturation` + low `GPU Utilization`:
- Reserved/fragmented memory pressure without proportional compute.
- Low `VRAM Saturation` + high `GPU Utilization`:
- Compute-bound behavior; memory is not current bottleneck.

Scale sanity check:

- `VRAM Saturation Ratio` and `GPU Utilization` should visually stay in `0%` to `100%`.
- If you see very large percent axes (for example `10000%`), check query scaling and panel unit assumptions.

## Operations

Restart stack:

```bash
docker compose restart
```

Stop stack:

```bash
docker compose down
```

Stop stack and remove volumes (destructive):

```bash
docker compose down -v
```

View logs:

```bash
docker compose logs -f
```

## Validation checklist

After startup, verify:

- `dcgm-exporter` exposes metrics on `:9400/metrics`
- Prometheus target `dcgm-exporter:9400` is `UP`
- Alerts appear in Prometheus rules page
- Grafana dashboard loads and shows per-GPU signals
- Restarting containers preserves metrics and dashboard state

## Hardening recommendations

- Replace default Grafana credentials and set secrets via env/secret manager
- Restrict exposed ports with host firewall rules
- Pin image tags (already done) and update via controlled change windows
- Adjust Prometheus retention to fit available disk budget
- Configure real Alertmanager receivers (Slack/email/webhook) in `alertmanager/alertmanager.yml`

## Troubleshooting

`No GPU metrics`:

- Re-run NVIDIA runtime check command above
- Confirm Docker sees GPUs with `--gpus all`
- Check exporter logs: `docker compose logs dcgm-exporter`

`Prometheus target down`:

- Verify exporter container is running
- Confirm target is `dcgm-exporter:9400` (not localhost)

`Grafana has no data`:

- Verify Prometheus datasource URL is `http://prometheus:9090`
- Check Prometheus query UI for `DCGM_FI_DEV_FB_USED`

## Files to edit for customization

- `docker-compose.yml`
- `prometheus/prometheus.yml`
- `prometheus/alerts.yml`
- `alertmanager/alertmanager.yml`
- `grafana/dashboards/gpu-observability.json`
