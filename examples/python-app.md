# Python Application Integration

Complete example of instrumenting a Python application with OpenTelemetry.

## Installation

```bash
pip install opentelemetry-distro \
            opentelemetry-exporter-otlp \
            opentelemetry-instrumentation-flask \
            opentelemetry-instrumentation-requests
```

## Basic Setup

```python
from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.resources import Resource
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader

# Define service resource
resource = Resource.create({
    "service.name": "python-api",
    "service.version": "1.0.0",
    "environment": "production",
    "team": "backend"
})

# Configure tracing
trace.set_tracer_provider(TracerProvider(resource=resource))
tracer = trace.get_tracer(__name__)

span_exporter = OTLPSpanExporter(
    endpoint="http://localhost:4317",
    insecure=True
)
span_processor = BatchSpanProcessor(span_exporter)
trace.get_tracer_provider().add_span_processor(span_processor)

# Configure metrics
metric_exporter = OTLPMetricExporter(
    endpoint="http://localhost:4317",
    insecure=True
)
metric_reader = PeriodicExportingMetricReader(
    exporter=metric_exporter,
    export_interval_millis=60000
)
metrics.set_meter_provider(
    MeterProvider(resource=resource, metric_readers=[metric_reader])
)
meter = metrics.get_meter(__name__)
```

## Flask Application Example

```python
from flask import Flask
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

app = Flask(__name__)

# Auto-instrument Flask
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

# Create custom metrics
request_counter = meter.create_counter(
    name="api_requests",
    description="Number of API requests",
    unit="1",
)

request_duration = meter.create_histogram(
    name="api_request_duration",
    description="API request duration",
    unit="ms",
)

@app.route('/api/users')
def get_users():
    with tracer.start_as_current_span("fetch_users") as span:
        span.set_attribute("user.count", 100)
        
        # Record metrics
        request_counter.add(1, {"endpoint": "/api/users"})
        
        # Your business logic here
        users = fetch_users_from_db()
        
        return jsonify(users)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

## Logging Integration

```python
import logging
from opentelemetry._logs import set_logger_provider
from opentelemetry.sdk._logs import LoggerProvider, LoggingHandler
from opentelemetry.exporter.otlp.proto.grpc._log_exporter import OTLPLogExporter

# Configure logging
logger_provider = LoggerProvider(resource=resource)
set_logger_provider(logger_provider)

log_exporter = OTLPLogExporter(
    endpoint="http://localhost:4317",
    insecure=True
)
logger_provider.add_log_record_processor(
    BatchLogRecordProcessor(log_exporter)
)

# Attach to Python logging
handler = LoggingHandler(level=logging.INFO, logger_provider=logger_provider)
logging.getLogger().addHandler(handler)

# Use standard Python logging
logger = logging.getLogger(__name__)
logger.info("Application started", extra={"user_id": 123})
```

## Environment Variables Alternative

```bash
# Auto-configure with environment variables
export OTEL_SERVICE_NAME=python-api
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_RESOURCE_ATTRIBUTES="environment=production,team=backend"
export OTEL_METRICS_EXPORTER=otlp
export OTEL_LOGS_EXPORTER=otlp
export OTEL_TRACES_EXPORTER=otlp

# Run with auto-instrumentation
opentelemetry-instrument python app.py
```

## Docker Deployment

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

ENV OTEL_SERVICE_NAME=python-api
ENV OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
ENV OTEL_RESOURCE_ATTRIBUTES="environment=production,version=1.0.0"

CMD ["opentelemetry-instrument", "python", "app.py"]
```

## Grafana Queries

```promql
# Request rate by endpoint
sum by (endpoint) (rate(api_requests_total[5m]))

# P95 latency
histogram_quantile(0.95, sum(rate(api_request_duration_bucket[5m])) by (le))

# Error rate
sum(rate(api_requests_total{status_code=~"5.."}[5m]))
```