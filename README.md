# k8s_fluent-bit

Fluent Bit DaemonSet for microk8s — collects PowerDNS logs and forwards them to Loki.

## What it does

- Tails `/var/log/containers/pdns*.log` on each node
- Enriches logs with Kubernetes metadata (pod, namespace)
- Extracts zone name from pdns log messages via Lua script
- Forwards to Loki with labels: `job=pdns`, `namespace`, `pod`, `zone`

## Directory layout

```
k8s_fluent-bit/
├── helm/
│   └── Chart.yaml          # Fluent Bit wrapper chart
└── helm_values/
    └── values-micro.yaml   # DaemonSet config with pdns input + Loki output
```

## Prerequisites

- Loki is running in `monitoring` namespace (`k8s_loki`)

## Installation

```bash
helm repo add fluent https://fluent.github.io/helm-charts
helm repo update

helm dependency update helm/
helm upgrade --install fluent-bit helm/ \
  -f helm_values/values-micro.yaml \
  -n monitoring --create-namespace
```

## Verify logs are flowing

```bash
# Check Fluent Bit pods
kubectl get pods -n monitoring -l app.kubernetes.io/name=fluent-bit

# Check Fluent Bit logs for errors
kubectl logs -n monitoring -l app.kubernetes.io/name=fluent-bit

# Port-forward Loki and query pdns logs
kubectl port-forward svc/loki 3100:3100 -n monitoring
curl -G http://localhost:3100/loki/api/v1/query_range \
  --data-urlencode 'query={job="pdns"}' \
  --data-urlencode 'limit=20'
```

## Log labels

| Label       | Value                        |
|-------------|------------------------------|
| `job`       | `pdns`                       |
| `namespace` | `dns`                        |
| `pod`       | pdns pod name                |
| `zone`      | extracted zone (e.g. `example.com`) or `unknown` |
