# Kubernetes Anomaly Detection Service

FastAPI-based service that performs anomaly detection on Kubernetes pod metrics
fetched from Prometheus. It uses `StandardScaler` + `IsolationForest` to score
pods based on:

- CPU usage: `rate(container_cpu_usage_seconds_total{container!="",container!="POD"}[1m])`
- Memory usage: `container_memory_usage_bytes{container!="",container!="POD"}`
- Pod restart count: `kube_pod_container_status_restarts_total`

The service exposes:

- REST endpoint `/detect` – anomaly flag and score per pod.
- Prometheus metrics endpoint `/metrics` – Gauges `anomaly_flag` and
  `anomaly_score` per pod.

---

## Project Structure

- `app/`
  - `config.py` – configuration and environment variables.
  - `prometheus/`
    - `api_client.py` – thin Prometheus HTTP API client.
  - `ml/`
    - `anomaly_detector.py` – model training + inference logic.
  - `api/`
    - `main.py` – FastAPI app and HTTP endpoints.
- `requirements.txt` – Python dependencies.
- `Dockerfile` – container image definition.
- `k8s-deployment.yaml` – example Kubernetes Deployment + Service.

---

## Configuration

All configuration is done via environment variables (see `app/config.py`):

- `PROMETHEUS_URL` (default `http://prometheus:9090`)
  - Base URL for Prometheus HTTP API.
- `NAMESPACE_REGEX` (default `.*`)
  - Regex for Kubernetes namespaces to include in queries.
- `POD_LABEL` (default `pod`)
  - Label key in Prometheus metrics that holds the pod name.
- `TRAINING_LOOKBACK_MINUTES` (default `60`)
  - Lookback window for initial model training.
- `QUERY_STEP_SECONDS` (default `60`)
  - Step for `/api/v1/query_range` training queries.
- `REQUEST_TIMEOUT_SECONDS` (default `5`)
  - HTTP timeout for Prometheus calls.
- IsolationForest tuning:
  - `IFOREST_N_ESTIMATORS` (default `100`)
  - `IFOREST_MAX_SAMPLES` (default `256`)
  - `IFOREST_CONTAMINATION` (default `auto`)

---

## Running Locally (Python)

Prerequisites:

- Python 3.11+
- Access to a Prometheus instance that scrapes Kubernetes metrics

Install dependencies and run:

```bash
pip install -r requirements.txt

export PROMETHEUS_URL="http://localhost:9090"      # adjust to your Prometheus
export NAMESPACE_REGEX=".*"

uvicorn app.api.main:app --host 0.0.0.0 --port 8000
```

Open:

- Swagger UI: `http://localhost:8000/docs`
- Health check: `http://localhost:8000/healthz`
- Metrics: `http://localhost:8000/metrics`

Note: If Prometheus is unreachable during startup, training will fail but the
service will still come up. `/detect` will return HTTP 503 until training
completes successfully.

---

## Running with Docker

Build the image:

```bash
docker build -t k8s-anomaly-detector .
```

Run the container:

```bash
docker run --rm -p 8000:8000 \
  -e PROMETHEUS_URL=http://host.docker.internal:9090 \  # adjust for your setup
  -e NAMESPACE_REGEX='.*' \
  k8s-anomaly-detector
```

Then access:

- `http://localhost:8000/healthz`
- `http://localhost:8000/detect`
- `http://localhost:8000/metrics`

---

## Prometheus Integration

The service exposes metrics on `/metrics` using `prometheus_client`. It provides:

```text
anomaly_flag{pod="nginx-xx"} 0
anomaly_score{pod="nginx-xx"} -0.12
```

### Local Prometheus (static scrape)

In `prometheus.yml`:

```yaml
scrape_configs:
  - job_name: 'k8s-anomaly-detector'
    static_configs:
      - targets:
          - 'localhost:8000'  # or host.docker.internal:8000
    metrics_path: /metrics
```

Reload Prometheus and you should see job `k8s-anomaly-detector` and the metrics.

### Kubernetes Prometheus (kubernetes_sd_configs)

Assuming you deployed `k8s-deployment.yaml` into the `default` namespace and the
service name is `k8s-anomaly-detector`:

```yaml
scrape_configs:
  - job_name: 'k8s-anomaly-detector'
    kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
            - default
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        action: keep
        regex: k8s-anomaly-detector
      - source_labels: [__meta_kubernetes_endpoint_port_name]
        action: keep
        regex: http
    metrics_path: /metrics
```

If you are using Prometheus Operator, create a `ServiceMonitor` targeting the
`k8s-anomaly-detector` service instead, with `path: /metrics`.

---

## Kubernetes Deployment

Example manifest: `k8s-deployment.yaml`.

Apply it (after pushing your image to a registry and updating the `image` field):

```bash
kubectl apply -f k8s-deployment.yaml
```

Key points:

- Deployment runs container exposing port 8000 and probes `/healthz`.
- Service exposes the app inside the cluster as `k8s-anomaly-detector:8000`.
- Environment variables `PROMETHEUS_URL` and `NAMESPACE_REGEX` are configurable
  directly in the manifest.

---

## API Overview

- `GET /healthz`
  - Returns service status, whether the model is trained, and configured
    `PROMETHEUS_URL`.

- `GET /detect`
  - On-demand anomaly detection over current metrics.
  - Response:
    ```json
    {
      "pods": [
        {
          "pod": "nginx-xxx",
          "cpu": 0.0012,
          "memory": 123456789.0,
          "restarts": 0.0,
          "anomaly_flag": 0,
          "anomaly_score": -0.12
        }
      ]
    }
    ```

- `GET /metrics`
  - Prometheus metrics endpoint exposing `anomaly_flag` and `anomaly_score`
    per pod.

---

## Notes

- The quality of anomaly detection depends on:
  - Having enough historical data in Prometheus (default 60 minutes).
  - Correct Prometheus queries and label conventions (especially pod label).
  - Appropriate IsolationForest configuration (`IFOREST_CONTAMINATION`, etc.).

Feel free to adjust the model configuration and PromQL queries in
`app/ml/anomaly_detector.py` and `app/config.py` to better fit your cluster and
workload characteristics.

