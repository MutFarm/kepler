# Kepler

![Kepler](https://img.shields.io/badge/Kepler-Power%20Monitoring-blueviolet?style=flat)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat&logo=prometheus&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat&logo=helm&logoColor=white)

[Kepler](https://sustainable-computing.io) (Kubernetes-based Efficient Power Level Exporter) is a Prometheus exporter that estimates energy consumption at the container, pod, and node level by reading hardware power sensors and attributing usage based on resource utilization.

Deployed on MutFarm to track cluster power consumption and estimate electricity cost per node and per workload.

## How it works

Kepler runs as a DaemonSet on each node and reads:

| Source | What it measures |
|---|---|
| **RAPL sysfs** (`/sys/class/powercap`) | CPU package & DRAM — real hardware counters |
| **eBPF** | Per-container CPU time, cache misses, IRQs |
| **Regression model** | Platform-level estimate when ACPI sensors unavailable |

On MutFarm nodes (Intel i5-7500, no ACPI power meters), Kepler uses **RAPL sysfs** for CPU + DRAM and a software regression model for platform-level estimates. RAPL values are real hardware measurements; platform estimates are approximate.

## Metrics exposed

| Metric | Description |
|---|---|
| `kepler_node_package_joules_total` | CPU package energy (real, RAPL) |
| `kepler_node_dram_joules_total` | DRAM energy (real, RAPL) |
| `kepler_node_platform_joules_total` | Total platform estimate (model-based) |
| `kepler_container_package_joules_total` | Per-container CPU attribution |

> **Note:** RAPL measures CPU + RAM only. Add ~5–10W per node for motherboard, fans, storage, and NIC to estimate real wall power.

## Grafana dashboard

Custom dashboard **MutFarm - Power Consumption (RAPL)** in the `MutFarm` folder:
- Live watts per node (CPU + RAM)
- Cluster total with node selector (filter by one or multiple nodes)
- Estimated daily kWh and monthly cost (configurable €/kWh)
- Top 10 pods and namespaces by power consumption
- Cumulative kWh today per node

The official Kepler Exporter dashboard (v0.8.0) is also imported for detailed process/container breakdowns.

## Namespace setup

The `kepler` namespace requires privileged pod security to allow host access:

```bash
kubectl label namespace kepler \
  pod-security.kubernetes.io/enforce=privileged \
  pod-security.kubernetes.io/enforce-version=latest \
  --overwrite
```

## Helm

```bash
# Add repo
helm repo add kepler https://sustainable-computing-io.github.io/kepler-helm-chart
helm repo update

# Install / upgrade
helm upgrade --install kepler kepler/kepler \
  -n kepler --create-namespace \
  -f values.yaml
```

## Prometheus integration

ServiceMonitor deployed in the `kube-prometheus-stack` namespace with label `release: kube-prometheus-stack` so it is picked up by the Prometheus operator automatically.
