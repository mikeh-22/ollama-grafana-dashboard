# ollama-grafana-dashboard

Grafana provisioning files for monitoring [Ollama](https://ollama.com) inference servers. Designed to work with [ollama-exporter](https://github.com/mikeh-22/ollama-exporter), which supports both **AMD ROCm** and **NVIDIA CUDA** GPU hosts.

## What's included

```
provisioning/
├── dashboards/
│   ├── dashboard.yml            # Grafana dashboard provider config
│   └── ollama-inference.json    # "Ollama Inference Monitor" dashboard
├── datasources/
│   └── prometheus.yml           # Prometheus datasource (uid: prometheus)
└── alerting/
    └── rules.yml                # Unified alerting rules
```

## Dashboard: Ollama Inference Monitor

Six panel rows covering the full inference stack:

| Row | Panels |
|---|---|
| **GPU Overview** | Compute utilisation %, total VRAM used, junction temps, API up/down |
| **GPU Compute & Power** | Utilisation time series (both GPUs), power draw |
| **GPU Memory** | VRAM used per GPU (bytes + %) |
| **GPU Temperature** | All sensors over time, threshold at 85°C |
| **Ollama Model Info** | Loaded model, context length, VRAM footprint, API latency |
| **Inference Job Detail** | Active job elapsed time, last request duration/details, request rate, duration percentiles (p50/p95/p99) |

## Alert rules

| Alert | Condition | Severity |
|---|---|---|
| `OllamaAPIDown` | `/api/ps` unreachable for 2m | critical |
| `InferenceJobHung` | GPU >80% util sustained for 25+ min | warning |
| `GPUOverheat` | Junction temp >87°C for 3m | critical |
| `GPUMemoryNearCapacity` | VRAM >95% for 5m | warning |
| `OllamaContainerHighCPU` | Container CPU >300% for 10m | info |
| `OllamaContextWindowLarge` | Configured context length >32K tokens | warning |

## Usage

### Mount into a Grafana container

```yaml
services:
  grafana:
    image: grafana/grafana:11.5.2
    container_name: grafana
    restart: unless-stopped
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_UNIFIED_ALERTING_ENABLED=true
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./provisioning:/etc/grafana/provisioning:ro

volumes:
  grafana-data:
```

### Prometheus requirements

The datasource config points to `http://prometheus:9090`. Your Prometheus instance must scrape [ollama-exporter](https://github.com/mikeh-22/ollama-exporter) targets.

For multi-host setups (e.g. one AMD host + one NVIDIA host), add a `gpu_host` label in your Prometheus scrape config to distinguish hosts in the dashboard:

```yaml
scrape_configs:
  - job_name: ollama-exporter
    static_configs:
      - targets: ['amd-host:9101']
        labels:
          gpu_host: amd-host
          gpu_backend: amd
      - targets: ['nvidia-host:9101']
        labels:
          gpu_host: nvidia-host
          gpu_backend: nvidia
```

## Related

- **[ollama-exporter](https://github.com/mikeh-22/ollama-exporter)** — the Prometheus exporter this dashboard is built for
