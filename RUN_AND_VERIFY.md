# How To Run And Verify (Layered)

Use this runbook in order. Do not move to the next layer until the current one passes.

## 1. Start the stack

From repo root:

```bash
docker compose up -d
docker compose ps
```

Expected:

- `dcgm-exporter`, `prometheus`, `alertmanager`, `grafana` are `Up`.

## 2. Layer 1: dcgm-exporter (GPU telemetry source)

Check exporter logs:

```bash
docker compose logs --tail=200 dcgm-exporter
```

Expected:

- No restart loop.
- No `wrong number of fields` error for `counters.csv`.
- No fatal `GPU collector ... cannot be initialized`.

Check exporter endpoint:

```bash
curl -sf http://localhost:9400/metrics | head
```

Expected:

- Prometheus text output appears.
- Query includes key GPU metrics such as:
- `DCGM_FI_DEV_FB_USED`
- `DCGM_FI_DEV_FB_FREE`
- `DCGM_FI_DEV_GPU_UTIL`

If this layer fails, fix exporter/runtime first. Prometheus/Grafana validation is not meaningful until this is healthy.

## 3. Layer 2: Prometheus (scrape + storage + rules)

Check Prometheus readiness:

```bash
curl -sf http://localhost:9090/-/ready
```

Expected:

- Response is `Prometheus is Ready.`

Check scrape target status:

- Open `http://localhost:9090/targets`

Expected:

- Job `dcgm-exporter` is `UP`.
- No DNS error like `lookup dcgm-exporter ... no such host`.

Check core queries:

- Open `http://localhost:9090/graph`
- Run:
- `DCGM_FI_DEV_FB_USED`
- `DCGM_FI_DEV_FB_FREE`
- `DCGM_FI_DEV_GPU_UTIL`

Expected:

- Time series returned per GPU label.

Check rules loaded:

- Open `http://localhost:9090/rules`

Expected:

- `GPUVRAMSaturationHigh`
- `GPUVRAMAllocationSpike`
- `GPUVRAMSustainedPressure`

## 4. Layer 3: Alertmanager (notification pipeline)

Check readiness:

```bash
curl -sf http://localhost:9093/-/ready
```

Expected:

- HTTP 200 response.

Check Prometheus alertmanager wiring:

- In Prometheus UI, open `Status -> Runtime & Build Information` and `Status -> Config` if needed.
- Confirm target `alertmanager:9093` is present from `prometheus.yml`.

Expected:

- Prometheus can reach Alertmanager with no connection errors in Prometheus logs.

## 5. Layer 4: Grafana (visualization)

Check Grafana health:

```bash
curl -sf http://localhost:3030/api/health
```

Expected:

- JSON health response with `"database":"ok"`.

UI verification:

1. Open `http://localhost:3030`.
2. Login.
3. Open dashboard `GPU Observability`.

Expected:

- Panels render data (not `No data`) for:
- VRAM Used (GiB)
- VRAM Free (GiB)
- VRAM Saturation Ratio
- GPU Utilization
- VRAM Spike Detector (MiB/s)

## 6. End-to-end persistence check

Restart stack:

```bash
docker compose restart
```

Expected after recovery:

- Prometheus target `dcgm-exporter` returns to `UP`.
- Grafana dashboard still exists.
- Historical data remains available (within retention window).

## 7. Fast failure map

If Layer 1 fails:

- Check NVIDIA runtime:
- `docker run --rm --gpus all nvidia/cuda:12.2.0-base nvidia-smi`
- Check `dcgm-exporter/counters.csv` and exporter logs.

If Layer 1 passes but Layer 2 fails:

- Ensure both containers are on the same compose network.
- Confirm scrape target is `dcgm-exporter:9400` (not localhost).

If Layer 2 passes but Layer 4 fails:

- Check Grafana datasource points to `http://prometheus:9090`.
- Confirm datasource UID `prometheus` matches dashboard datasource UID.
- If folder is empty, check Grafana logs for `invalid character 'ï'` on dashboard JSON (BOM encoding issue), then rewrite dashboard/provisioning files as UTF-8 without BOM and restart Grafana.

## 8. Useful commands

```bash
docker compose logs -f
docker compose logs -f dcgm-exporter
docker compose logs -f prometheus
docker compose down
docker compose down -v
```

## 9. UI checks (manual)

Use this as a quick visual validation after CLI checks pass.

Prometheus UI:

1. Open `http://localhost:9090/targets`
2. Verify `dcgm-exporter` shows `State = UP`
3. Open `http://localhost:9090/graph`
4. Run `DCGM_FI_DEV_FB_USED`
5. Click `Execute`

Expected:

- Target row is green/healthy.
- Graph/table shows one or more series with GPU labels.

Prometheus rules UI:

1. Open `http://localhost:9090/rules`
2. Find group `gpu-observability`

Expected:

- Rules are listed:
- `GPUVRAMSaturationHigh`
- `GPUVRAMAllocationSpike`
- `GPUVRAMSustainedPressure`

Alertmanager UI:

1. Open `http://localhost:9093`
2. Go to `/#/status` (Status page)

Expected:

- Alertmanager is reachable and shows routing config/uptime.
- No fatal config errors.

Grafana UI:

1. Open `http://localhost:3030`
2. Login
3. Open `Dashboards -> GPU -> GPU Observability`
4. Set time range to `Last 15 minutes`
5. Set refresh to `5s`

Expected:

- Panels populate with non-empty data.
- VRAM Used/Free and GPU Utilization lines move over time.
- No datasource error banners.

Metric quick meaning (operator view):

- `VRAM Used (GiB)`: active framebuffer allocation on GPU. Rising baseline means shrinking headroom.
- `VRAM Free (GiB)`: remaining framebuffer memory. Near-zero values are pre-OOM risk.
- `VRAM Saturation Ratio`: normalized VRAM pressure (`used / total`). `>90%` sustained is dangerous.
- `GPU Utilization`: compute activity. High memory + low utilization often indicates memory pressure without useful compute.
- `VRAM Spike Detector (MiB/s)`: rate of VRAM change. Repeated positive spikes indicate allocation bursts/leak-like behavior.
