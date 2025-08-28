# OpenTelemetry LGTM Stack

A production-ready observability stack for collecting, storing, and visualizing telemetry data from any OpenTelemetry-compatible source.

**LGTM = Loki + Grafana + Tempo + Mimir/Prometheus**

## 🎯 Features

- **Universal Telemetry Collection**: Accept metrics, logs, and traces from any OpenTelemetry source
- **Offline Buffering**: WAL (Write-Ahead Log) ensures no data loss during network outages
- **Multi-tenant Support**: Isolate data by service, team, or environment
- **Auto-retry Logic**: Exponential backoff for failed exports
- **Production Ready**: Memory limits, health checks, and proper retention policies
- **Easy Integration**: Works with any language or framework that supports OpenTelemetry

## 🚀 Quick Start

### 1. Clone and Configure

```bash
# Copy and edit environment variables
cp .env.example .env
nano .env  # Edit passwords and settings

# Start the stack
docker-compose up -d

# Check status
docker-compose ps
```

### 2. Access Services

- **Grafana**: http://localhost:8439 (admin/changeme)
- **Prometheus**: http://localhost:9090
- **Collector Health**: http://localhost:13133/health

### 3. Send Telemetry

Configure your application to send data to the collector:

```bash
# Environment variables for any OTel application
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_RESOURCE_ATTRIBUTES="service.name=my-app,environment=production"
```

## 📊 Architecture

```
Your Applications
      ↓ (OTLP)
OpenTelemetry Collector (:4317/:4318)
      ↓
┌─────────────┬──────────────┬─────────────┐
│ Prometheus  │     Loki     │    Tempo    │
│  (Metrics)  │    (Logs)    │  (Traces)   │
└─────────────┴──────────────┴─────────────┘
                     ↓
                 Grafana
              (Visualization)
```

## 🔧 Configuration Examples

### Python Application

```python
from opentelemetry import metrics, trace
from opentelemetry.exporter.otlp.proto.grpc import (
    OTLPMetricExporter,
    OTLPSpanExporter
)

# Configure exporters
metric_exporter = OTLPMetricExporter(
    endpoint="http://localhost:4317",
    insecure=True
)
trace_exporter = OTLPSpanExporter(
    endpoint="http://localhost:4317",
    insecure=True
)
```

### Node.js Application

```javascript
const { OTLPMetricExporter } = require('@opentelemetry/exporter-otlp-grpc');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');

const metricExporter = new OTLPMetricExporter({
  url: 'http://localhost:4317',
});

const traceExporter = new OTLPTraceExporter({
  url: 'http://localhost:4317',
});
```

### Docker Container

```yaml
services:
  my-app:
    image: my-app:latest
    environment:
      - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
      - OTEL_RESOURCE_ATTRIBUTES=service.name=my-app,team=backend
      - OTEL_METRICS_EXPORTER=otlp
      - OTEL_LOGS_EXPORTER=otlp
      - OTEL_TRACES_EXPORTER=otlp
```

### Kubernetes

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-config
data:
  OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector.monitoring:4317"
  OTEL_RESOURCE_ATTRIBUTES: "service.name=my-service,k8s.namespace.name=production"
```

## 📈 Multi-Service Setup

The stack automatically handles multiple services sending data simultaneously:

```bash
# Service 1: API Backend
OTEL_RESOURCE_ATTRIBUTES="service.name=api,env=prod" ./api-server

# Service 2: Worker
OTEL_RESOURCE_ATTRIBUTES="service.name=worker,env=prod" ./worker

# Service 3: Frontend
OTEL_RESOURCE_ATTRIBUTES="service.name=frontend,env=prod" ./frontend
```

All services appear in Grafana with automatic correlation between metrics, logs, and traces.

## 🛠️ Advanced Configuration

### Offline Buffering

The collector stores up to 10,000 batches on disk when backends are unreachable:

```yaml
# config/otel-collector.yaml
sending_queue:
  enabled: true
  storage: file_storage
  queue_size: 10000  # Increase for longer outages
```

### Custom Dashboards

Add your dashboards to the `dashboards/` directory:

```bash
cp my-dashboard.json dashboards/
docker-compose restart grafana
```

### Resource Filtering

Filter data in Grafana using resource attributes:

```promql
# Metrics by service
rate(http_requests_total{service_name="api"}[5m])

# Logs by environment
{environment="production"} |= "error"

# Traces by team
{resource.team="backend"} | duration > 1s
```

## 📝 Retention Policies

Default retention settings:
- **Metrics**: 30 days (configurable via `PROMETHEUS_RETENTION`)
- **Logs**: 30 days (configured in Loki)
- **Traces**: 14 days (configured in Tempo)

Adjust in respective config files under `config/`.

## 🔍 Troubleshooting

### No Data in Grafana

1. Check collector health: `curl http://localhost:13133/health`
2. Verify your app connects: `telnet localhost 4317`
3. Check collector logs: `docker-compose logs otel-collector`
4. Enable debug mode: Set `LOG_LEVEL=debug` in `.env`

### High Memory Usage

Adjust limits in `config/otel-collector.yaml`:
```yaml
memory_limiter:
  limit_percentage: 80  # Lower this value
```

### Disk Space

Monitor storage usage:
```bash
du -sh storage/
docker system df
```

Clean old data:
```bash
docker-compose down
rm -rf storage/*
docker-compose up -d
```

## 🚢 Production Deployment

### Security Checklist

- [ ] Change default Grafana password
- [ ] Enable TLS on collector endpoints
- [ ] Configure firewall rules
- [ ] Set up authentication for Prometheus/Loki
- [ ] Use secrets management for credentials
- [ ] Enable audit logging

### Scaling

For high-volume environments:
1. Deploy multiple collectors behind a load balancer
2. Use external storage backends (S3 for Loki/Tempo)
3. Consider managed services (Grafana Cloud, AWS Managed Prometheus)

## 📚 Learn More

- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Grafana LGTM Stack](https://grafana.com/oss/)
- [Example Dashboards](./examples/)

## 🤝 Contributing

See example configurations in the `examples/` directory for specific use cases.

## 📄 License

MIT - Use freely for any project!