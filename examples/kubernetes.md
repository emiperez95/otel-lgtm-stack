# Kubernetes Integration

Deploy the OpenTelemetry Operator and configure applications to send telemetry to your LGTM stack.

## OpenTelemetry Operator Setup

```bash
# Install cert-manager (required)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml

# Install OpenTelemetry Operator
kubectl apply -f https://github.com/open-telemetry/opentelemetry-operator/releases/latest/download/opentelemetry-operator.yaml
```

## OpenTelemetry Collector in Kubernetes

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: OpenTelemetryCollector
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  mode: deployment
  replicas: 2
  config: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      
      k8s_cluster:
        auth_type: serviceAccount
        node_conditions_to_report: []
      
      kubeletstats:
        collection_interval: 10s
        auth_type: serviceAccount
        endpoint: "https://${env:K8S_NODE_NAME}:10250"
        insecure_skip_verify: true

    processors:
      batch:
        timeout: 10s
      
      k8sattributes:
        extract:
          metadata:
            - k8s.namespace.name
            - k8s.deployment.name
            - k8s.pod.name
            - k8s.node.name
          labels:
            - key: app
              from: pod
              tag_name: service.name
      
      resource:
        attributes:
          - key: cluster.name
            value: production-cluster
            action: insert

    exporters:
      otlp:
        endpoint: "YOUR_LGTM_STACK_IP:4317"
        tls:
          insecure: true

    service:
      pipelines:
        metrics:
          receivers: [otlp, k8s_cluster, kubeletstats]
          processors: [k8sattributes, resource, batch]
          exporters: [otlp]
        traces:
          receivers: [otlp]
          processors: [k8sattributes, resource, batch]
          exporters: [otlp]
        logs:
          receivers: [otlp]
          processors: [k8sattributes, resource, batch]
          exporters: [otlp]
```

## Auto-Instrumentation for Applications

```yaml
apiVersion: opentelemetry.io/v1alpha1
kind: Instrumentation
metadata:
  name: auto-instrumentation
  namespace: default
spec:
  exporter:
    endpoint: http://otel-collector.monitoring:4317
  propagators:
    - tracecontext
    - baggage
  
  java:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-java:latest
  
  nodejs:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-nodejs:latest
  
  python:
    image: ghcr.io/open-telemetry/opentelemetry-operator/autoinstrumentation-python:latest
```

## Application Deployment with Auto-Instrumentation

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    metadata:
      annotations:
        instrumentation.opentelemetry.io/inject-python: "true"
    spec:
      containers:
      - name: app
        image: my-app:latest
        env:
        - name: OTEL_SERVICE_NAME
          value: "my-app"
        - name: OTEL_RESOURCE_ATTRIBUTES
          value: "environment=production,version=1.2.3"
```

## Collecting Kubernetes Metrics

```yaml
apiVersion: v1
kind: ServiceMonitor
metadata:
  name: kubernetes-metrics
spec:
  endpoints:
  - port: metrics
    interval: 30s
  selector:
    matchLabels:
      app: kube-state-metrics
```

## Fluent Bit for Log Collection

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: monitoring
data:
  fluent-bit.conf: |
    [SERVICE]
        Daemon Off
        Flush 5
        Log_Level info
    
    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        Parser docker
        Tag kube.*
        Skip_Long_Lines On
    
    [FILTER]
        Name kubernetes
        Match kube.*
        Merge_Log On
        Keep_Log Off
    
    [OUTPUT]
        Name opentelemetry
        Match *
        Host otel-collector.monitoring
        Port 4318
        URI /v1/logs
        Format json
```

## Grafana Dashboard Queries

```promql
# Pod memory usage by namespace
sum by (namespace) (
  container_memory_usage_bytes{container!="POD"}
)

# Request rate by service
sum by (service_name) (
  rate(http_server_duration_count[5m])
)

# Pod restart count
increase(kube_pod_container_status_restarts_total[1h])

# Namespace resource quotas
kube_resourcequota{type="hard"} - kube_resourcequota{type="used"}
```

## Helm Values for External LGTM Stack

```yaml
# values.yaml for opentelemetry-collector helm chart
config:
  exporters:
    otlp:
      endpoint: "http://YOUR_LGTM_STACK_IP:4317"
      tls:
        insecure: true
  
  service:
    pipelines:
      metrics:
        exporters: [otlp]
      traces:
        exporters: [otlp]
      logs:
        exporters: [otlp]

resources:
  limits:
    memory: 2Gi
  requests:
    memory: 512Mi
    cpu: 100m

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 80
```

## Install with Helm

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm install otel-collector open-telemetry/opentelemetry-collector \
  -f values.yaml \
  --namespace monitoring \
  --create-namespace
```