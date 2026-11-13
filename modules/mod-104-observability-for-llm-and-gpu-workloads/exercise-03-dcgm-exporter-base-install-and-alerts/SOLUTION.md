# Exercise 03 — DCGM Exporter Base Install and Alerts (Solution)

## 1. Solution overview

Stand up the NVIDIA **DCGM Exporter** on a Kubernetes cluster (or a single GPU host) so Prometheus can scrape per-GPU health, utilization, and error metrics, then wire a minimum viable set of alerts that fire on the failure modes that matter for AI inference / training SRE: XID errors, thermal throttling, ECC errors, memory pressure, and idle-but-reserved GPUs.

Scope boundaries:

- Install only — advanced Grafana dashboards are Exercise 07.
- Alerts are the **base set** (XID, thermal, ECC, OOM proxy, utilization). Per-workload alerts (goodput, slot stall) layer on top later.
- We rely on the official exporter and its documented metric names. Any metric not listed in the exporter's shipped CSV counter definitions is called out inline as a note to verify against the running chart version.

## 2. Worked implementation

### 2.1 Prereqs

| Requirement | Why |
|---|---|
| NVIDIA driver on nodes (same version family as DCGM) | Exporter talks to `nvidia-smi`/`libdcgm`; version skew surfaces as missing counters |
| NVIDIA container toolkit + device plugin (on k8s) | Pods must be able to request GPUs |
| Prometheus operator OR a plain scrape config | Something to scrape `:9400/metrics` |
| `PodMonitor` or `ServiceMonitor` RBAC if using the operator | Autodiscovery of the exporter pods |

### 2.2 Install (Helm, k8s path — recommended)

```bash
helm repo add gpu-helm-charts https://nvidia.github.io/dcgm-exporter/helm-charts
helm repo update

helm install dcgm-exporter gpu-helm-charts/dcgm-exporter \
  --namespace gpu-operator \
  --create-namespace \
  --set serviceMonitor.enabled=true \
  --set serviceMonitor.interval=15s
```

Values worth setting explicitly:

- `arguments: ["-f", "/etc/dcgm-exporter/dcp-metrics-included.csv"]` — DCP (profiling) metrics give you SM activity, PCIe, NVLink. Only works on supported GPUs; falls back cleanly otherwise.
- `kubeletPath` — override if your cluster uses a non-default kubelet socket path.
- `nodeSelector: { "nvidia.com/gpu.present": "true" }` — avoids scheduling the DaemonSet on CPU-only nodes.

### 2.3 Install (plain Docker / bare node)

```bash
docker run -d --rm --gpus all --net host --cap-add SYS_ADMIN \
  --name dcgm-exporter \
  nvcr.io/nvidia/k8s/dcgm-exporter:latest
# Metrics on http://<host>:9400/metrics
```

### 2.4 Prometheus scrape config (plain Prometheus)

```yaml
scrape_configs:
  - job_name: dcgm-exporter
    scrape_interval: 15s
    static_configs:
      - targets: ['gpu-node-1:9400', 'gpu-node-2:9400']
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):\d+'
        target_label: host
```

With the Prometheus Operator, the chart's `ServiceMonitor` handles this.

### 2.5 Key metrics (as exported)

The exporter's CSV defines which DCGM fields become Prometheus series. The important ones for the base SRE alert set:

| Metric | Type | Meaning |
|---|---|---|
| `DCGM_FI_DEV_XID_ERRORS` | gauge | Last XID error code on the device |
| `DCGM_FI_DEV_GPU_TEMP` | gauge | GPU core temperature (°C) |
| `DCGM_FI_DEV_POWER_USAGE` | gauge | Current board power draw (W) |
| `DCGM_FI_DEV_GPU_UTIL` | gauge | Coarse % busy over sample interval |
| `DCGM_FI_DEV_MEM_COPY_UTIL` | gauge | % memory-bus busy |
| `DCGM_FI_DEV_FB_USED` | gauge | Framebuffer memory used (MiB) |
| `DCGM_FI_DEV_FB_FREE` | gauge | Framebuffer memory free (MiB) |
| `DCGM_FI_DEV_ECC_SBE_VOL_TOTAL` | counter | Single-bit ECC errors (volatile) |
| `DCGM_FI_DEV_ECC_DBE_VOL_TOTAL` | counter | Double-bit ECC errors (volatile) |
| `DCGM_FI_PROF_SM_ACTIVE` | gauge | Fraction of SMs active (profiling, finer than `GPU_UTIL`) |
| `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE` | gauge | Tensor-core pipe activity |

Standard labels every series carries: `gpu`, `UUID`, `device`, `modelName`, `Hostname`, plus k8s labels (`namespace`, `pod`, `container`) when running under the operator.

> Note: confirm exact label names shipped by the current dcgm-exporter chart; label casing (e.g. `Hostname` vs `hostname`) has changed across releases, so verify before writing alert rules against a specific casing.

### 2.6 Base alert rules

```yaml
# prometheus/rules/dcgm.yml
groups:
  - name: gpu.dcgm
    interval: 30s
    rules:
      # 1. XID errors — any non-zero reading is worth a page.
      - alert: GPUXidError
        expr: DCGM_FI_DEV_XID_ERRORS > 0
        for: 1m
        labels:
          severity: page
        annotations:
          summary: "GPU {{ $labels.gpu }} on {{ $labels.Hostname }} reported XID {{ $value }}"
          runbook: "https://docs.nvidia.com/deploy/xid-errors/"

      # 2. Thermal throttle risk (threshold is per GPU family; 85C is a conservative shared value).
      - alert: GPUHighTemperature
        expr: DCGM_FI_DEV_GPU_TEMP > 85
        for: 5m
        labels:
          severity: ticket
        annotations:
          summary: "GPU {{ $labels.gpu }} sustained >85C for 5m — check cooling / neighbor load"

      # 3. ECC double-bit error (uncorrectable) — immediate drain.
      - alert: GPUECCDoubleBitError
        expr: increase(DCGM_FI_DEV_ECC_DBE_VOL_TOTAL[10m]) > 0
        for: 0m
        labels:
          severity: page
        annotations:
          summary: "Uncorrectable ECC error on GPU {{ $labels.gpu }} — drain the node"

      # 4. Memory saturation — <1 GiB free sustained is almost always an incident precursor.
      - alert: GPUMemoryPressure
        expr: DCGM_FI_DEV_FB_FREE < 1024
        for: 5m
        labels:
          severity: ticket
        annotations:
          summary: "GPU {{ $labels.gpu }} has <1 GiB framebuffer free for 5m"

      # 5. Reserved but idle — GPU allocated to a pod but doing no work (typical: engine crashloop, dead client).
      - alert: GPUReservedIdle
        expr: |
          (DCGM_FI_DEV_GPU_UTIL == 0)
          and on(gpu, Hostname) (DCGM_FI_DEV_FB_USED > 500)
        for: 15m
        labels:
          severity: ticket
        annotations:
          summary: "GPU {{ $labels.gpu }} holds memory but has 0% util for 15m — likely a hung workload"

      # 6. Exporter health — missing GPU metrics are themselves a signal.
      - alert: GPUExporterDown
        expr: absent(DCGM_FI_DEV_GPU_UTIL)
        for: 5m
        labels:
          severity: page
        annotations:
          summary: "DCGM exporter has not reported GPU metrics in 5m — scrape or driver issue"
```

### 2.7 Why these alerts (and not others, yet)

- **XID errors** cover hardware fault codes spanning power, PCIe, memory, and driver assertions. Non-zero is always worth investigating; NVIDIA publishes the XID code table.
- **ECC DBE** (double-bit) errors are uncorrectable and historically correlate with device failure; SBE (single-bit) errors are corrected in hardware and worth a ticket, not a page.
- **Temperature** varies by SKU (datacenter GPUs tolerate higher junction temps). 85°C board temperature is a conservative trigger; tune to your fleet. Cross-reference `DCGM_FI_DEV_THERMAL_VIOLATION` when it is exposed in your chart version.
- **Framebuffer free** is the clearest proxy for "next request will OOM". Absolute MiB is easier to reason about across SKUs than a percent.
- **Reserved idle** catches crashed inference workers that still hold the device — one of the common "we're paying for GPUs doing nothing" failure modes.
- **Exporter down** keeps absence from being silent; `absent()` is the dual of your main alert.

Deferred (added in later exercises or by workload type):

- Per-model goodput alerts
- NVLink / PCIe error alerts (noisy without context — usually pulled in during incident investigation, not paged on)
- MIG partition-specific alerts

## 3. Validation steps

1. **Exporter reachable**: `curl -s http://<node-ip>:9400/metrics | head -30` — should return Prometheus text format, not HTML/404.
2. **Expected metrics present**: `curl -s :9400/metrics | grep -c ^DCGM_FI_DEV_GPU_UTIL` ≥ number of GPUs on the host.
3. **Scrape succeeds**: in Prometheus UI, `up{job="dcgm-exporter"} == 1` for every target.
4. **Alert evaluation**: `promtool check rules dcgm.yml` reports OK. Then run `promtool test rules` with a small fixture for each alert.
5. **Forced XID (controlled fault)**: on a test GPU, use `dcgmi diag -r` which can surface simulated faults, or wait for a benign fault during a stress run and confirm `GPUXidError` fires.
6. **Thermal drill**: cap one GPU's power with `nvidia-smi -pl <low-watts>`, run a warm-up kernel, watch `DCGM_FI_DEV_GPU_TEMP` on the dashboard, then revert.
7. **Idle-reserved drill**: schedule a sleep-forever pod with `nvidia.com/gpu: 1`; confirm the alert fires at ~15 minutes.
8. **Exporter-down drill**: `kubectl delete pod -l app=dcgm-exporter --wait=false` on one node; confirm `GPUExporterDown` fires in <5m, then recovers.

## 4. Rubric

| Criterion | Pass bar |
|---|---|
| Install path | Helm or DaemonSet manifest reproducible from the repo; no `kubectl apply` of an unpinned URL |
| Driver / toolchain gate | Documented driver and container-toolkit versions; CPU-only nodes excluded via nodeSelector |
| Scrape config | Prometheus actually scrapes `:9400/metrics` with explicit interval |
| Alert set | Covers at minimum: XID, ECC DBE, thermal, memory pressure, reserved-idle, exporter-down |
| Severity split | Page vs ticket is intentional (not all 6 are pages) |
| Runbook pointers | Every page-severity alert has a runbook annotation |
| Drills | At least two alerts proven to fire with a controlled trigger |
| Hygiene | `promtool check rules` passes |

## 5. Common mistakes

- **Scraping too fast.** 1-second intervals on DCGM add measurable load to the driver without useful resolution. 15s is the sweet spot; 30s is fine for alerts.
- **Treating every XID as a page.** Some XIDs are benign (e.g. driver resets during normal preemption). Read the NVIDIA XID page before tuning severity per code — but a conservative "any XID is a ticket" default is fine until the fleet has history.
- **Hardcoding one temperature threshold across heterogeneous SKUs.** A100 at 85°C is normal under load; a consumer-grade GPU at 85°C is near its throttle point. Alert per GPU family (`DCGM_FI_DEV_GPU_TEMP{modelName=~"NVIDIA A100.*"} > 90`) once you have the fleet composition.
- **Leaving `DCGM_FI_PROF_*` off because "it adds overhead".** It adds some, but the signal for inference workloads (SM active, tensor-core active) is high. The real cost is exclusivity with other profilers, not exporter overhead.
- **Letting the exporter pod request a GPU.** It does not need one; it reads host DCGM. Requesting a GPU wastes a device.
- **Running only one replica of the exporter per cluster.** It's a DaemonSet by design; one pod per node. A single cluster-scoped replica sees only one host.
- **No `absent()` alert.** Dashboards go blank when the exporter crashes, but PagerDuty won't fire unless `absent()` or a scrape-health alert is wired up.
- **Cardinality explosion from k8s labels.** Chart defaults join every pod's labels onto DCGM series. Trim with `metric_relabel_configs` if your TSDB struggles.

## 6. References

- NVIDIA DCGM Exporter (official repo): https://github.com/NVIDIA/dcgm-exporter
- NVIDIA DCGM (Data Center GPU Manager) overview: https://developer.nvidia.com/dcgm
- NVIDIA DCGM user guide: https://docs.nvidia.com/datacenter/dcgm/latest/user-guide/index.html
- NVIDIA XID error table: https://docs.nvidia.com/deploy/xid-errors/
- NVIDIA GPU Operator (full stack alt install): https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/overview.html
- Prometheus Alerting Rules documentation: https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- `promtool` (rules test): https://prometheus.io/docs/prometheus/latest/command-line/promtool/
