# Add Grafana Helm repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create namespace
kubectl create namespace observability

# Install Loki
helm upgrade --install loki grafana/loki \
  --namespace observability \
  -f k8s/loki-values.yaml

# Install Grafana
helm upgrade --install grafana grafana/grafana \
  --namespace observability \
  -f k8s/grafana-values.yaml

# Install Alloy


# Logging format guidance
{
  "ts": "2026-03-03T10:15:20Z",
  "level": "error",
  "service": "checkout-api",
  "message": "payment provider timeout",
  "route": "/api/orders/{id}/pay",
  "status": 504,
  "tenant": "internal"
}

