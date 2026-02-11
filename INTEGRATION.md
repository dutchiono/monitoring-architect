# Monitoring Architect - Integration Guide

## Integration Overview

This guide covers integrating Monitoring Architect with various platforms, services, and tools to create a comprehensive observability ecosystem. Each integration includes authentication setup, configuration examples, and best practices.

## Application Instrumentation

### Prometheus Metrics

**Go Application**:
```go
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promauto"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

var (
    requestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total number of HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )
    
    requestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "http_request_duration_seconds",
            Help: "HTTP request duration in seconds",
            Buckets: []float64{.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10},
        },
        []string{"method", "endpoint"},
    )
)

func instrumentHandler(handler http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        timer := prometheus.NewTimer(requestDuration.WithLabelValues(r.Method, r.URL.Path))
        defer timer.ObserveDuration()
        
        handler(w, r)
        
        requestsTotal.WithLabelValues(r.Method, r.URL.Path, "200").Inc()
    }
}

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.HandleFunc("/api/users", instrumentHandler(handleUsers))
    http.ListenAndServe(":8080", nil)
}
```

**Python Application (Flask)**:
```python
from flask import Flask
from prometheus_client import Counter, Histogram, generate_latest
from werkzeug.middleware.dispatcher import DispatcherMiddleware
from prometheus_client import make_wsgi_app
import time

app = Flask(__name__)

# Metrics
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

REQUEST_DURATION = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint']
)

@app.before_request
def start_timer():
    request.start_time = time.time()

@app.after_request
def record_metrics(response):
    duration = time.time() - request.start_time
    REQUEST_DURATION.labels(
        method=request.method,
        endpoint=request.endpoint
    ).observe(duration)
    
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.endpoint,
        status=response.status_code
    ).inc()
    
    return response

# Expose metrics
app.wsgi_app = DispatcherMiddleware(app.wsgi_app, {
    '/metrics': make_wsgi_app()
})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

**Node.js Application (Express)**:
```javascript
const express = require('express');
const client = require('prom-client');

const app = express();

// Create metrics
const httpRequestsTotal = new client.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status']
});

const httpRequestDuration = new client.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10]
});

// Middleware
app.use((req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    
    httpRequestDuration
      .labels(req.method, req.route?.path || req.path)
      .observe(duration);
    
    httpRequestsTotal
      .labels(req.method, req.route?.path || req.path, res.statusCode)
      .inc();
  });
  
  next();
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', client.register.contentType);
  res.end(await client.register.metrics());
});

app.listen(8080);
```

### OpenTelemetry Tracing

**Go Application**:
```go
package main

import (
    "context"
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    "go.opentelemetry.io/otel/sdk/trace"
    semconv "go.opentelemetry.io/otel/semconv/v1.17.0"
)

func initTracer(ctx context.Context) (*trace.TracerProvider, error) {
    exporter, err := otlptracegrpc.New(ctx,
        otlptracegrpc.WithEndpoint("tempo:4317"),
        otlptracegrpc.WithInsecure(),
    )
    if err != nil {
        return nil, err
    }
    
    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName("my-service"),
            semconv.ServiceVersion("1.0.0"),
        )),
    )
    
    otel.SetTracerProvider(tp)
    return tp, nil
}

func handleRequest(ctx context.Context) {
    tracer := otel.Tracer("my-service")
    ctx, span := tracer.Start(ctx, "handleRequest")
    defer span.End()
    
    // Add attributes
    span.SetAttributes(
        attribute.String("user.id", "12345"),
        attribute.Int("http.status_code", 200),
    )
    
    // Nested span
    ctx, childSpan := tracer.Start(ctx, "database.query")
    // ... do database work
    childSpan.End()
}
```

**Python Application**:
```python
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.resources import Resource

# Initialize tracer
resource = Resource(attributes={
    "service.name": "my-service",
    "service.version": "1.0.0"
})

provider = TracerProvider(resource=resource)
processor = BatchSpanProcessor(
    OTLPSpanExporter(endpoint="tempo:4317", insecure=True)
)
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

tracer = trace.get_tracer(__name__)

# Use tracer
def handle_request():
    with tracer.start_as_current_span("handle_request") as span:
        span.set_attribute("user.id", "12345")
        span.set_attribute("http.status_code", 200)
        
        # Nested span
        with tracer.start_as_current_span("database.query"):
            # ... database work
            pass
```

### Structured Logging

**Go Application (Zap)**:
```go
package main

import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

func initLogger() *zap.Logger {
    config := zap.NewProductionConfig()
    config.EncoderConfig.TimeKey = "timestamp"
    config.EncoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder
    
    logger, _ := config.Build()
    return logger
}

func main() {
    logger := initLogger()
    defer logger.Sync()
    
    logger.Info("Request received",
        zap.String("method", "GET"),
        zap.String("path", "/api/users"),
        zap.Int("status", 200),
        zap.Duration("duration", time.Millisecond*150),
    )
}
```

**Python Application (structlog)**:
```python
import structlog

# Configure structlog
structlog.configure(
    processors=[
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.add_log_level,
        structlog.processors.JSONRenderer()
    ],
    context_class=dict,
    logger_factory=structlog.PrintLoggerFactory(),
)

logger = structlog.get_logger()

# Log with context
logger.info("request_received",
    method="GET",
    path="/api/users",
    status=200,
    duration_ms=150
)
```

## Cloud Provider Integration

### AWS CloudWatch

**Prometheus CloudWatch Exporter**:
```yaml
# cloudwatch-exporter-config.yaml
region: us-east-1
metrics:
  - aws_namespace: AWS/EC2
    aws_metric_name: CPUUtilization
    aws_dimensions: [InstanceId]
    aws_statistics: [Average]
    period_seconds: 300
    
  - aws_namespace: AWS/RDS
    aws_metric_name: DatabaseConnections
    aws_dimensions: [DBInstanceIdentifier]
    aws_statistics: [Average, Maximum]
    period_seconds: 300
    
  - aws_namespace: AWS/ELB
    aws_metric_name: RequestCount
    aws_dimensions: [LoadBalancerName]
    aws_statistics: [Sum]
    period_seconds: 60
```

**CloudWatch Logs to Loki**:
```python
import boto3
import json
import requests

def lambda_handler(event, context):
    # Parse CloudWatch Logs event
    log_data = json.loads(event['awslogs']['data'].decode('base64'))
    
    # Transform to Loki format
    streams = []
    for log_event in log_data['logEvents']:
        streams.append({
            "stream": {
                "log_group": log_data['logGroup'],
                "log_stream": log_data['logStream']
            },
            "values": [
                [str(log_event['timestamp'] * 1000000), log_event['message']]
            ]
        })
    
    # Send to Loki
    response = requests.post(
        'http://loki:3100/loki/api/v1/push',
        json={"streams": streams},
        headers={'Content-Type': 'application/json'}
    )
    
    return {'statusCode': 200}
```

### Google Cloud Platform

**GCP Monitoring Integration**:
```python
from google.cloud import monitoring_v3

def export_to_prometheus():
    client = monitoring_v3.MetricServiceClient()
    project_name = f"projects/{project_id}"
    
    # Query GCP metrics
    interval = monitoring_v3.TimeInterval({
        "end_time": {"seconds": int(time.time())},
        "start_time": {"seconds": int(time.time()) - 300},
    })
    
    results = client.list_time_series(
        request={
            "name": project_name,
            "filter": 'metric.type = "compute.googleapis.com/instance/cpu/utilization"',
            "interval": interval,
            "view": monitoring_v3.ListTimeSeriesRequest.TimeSeriesView.FULL,
        }
    )
    
    # Convert to Prometheus format
    for result in results:
        print(f"# HELP gcp_cpu_utilization CPU utilization")
        print(f"# TYPE gcp_cpu_utilization gauge")
        for point in result.points:
            print(f'gcp_cpu_utilization{{instance="{result.resource.labels["instance_id"]}"}} {point.value.double_value}')
```

### Azure Monitor

**Azure to Prometheus Bridge**:
```yaml
# azure-exporter-config.yaml
subscriptions:
  - subscription_id: "your-subscription-id"
    resource_groups:
      - resource_group: "production"
        resources:
          - resource_name: "web-app-prod"
            resource_type: "Microsoft.Web/sites"
            metrics:
              - name: "CpuTime"
                aggregation: "Total"
              - name: "Requests"
                aggregation: "Total"
```

## Kubernetes Integration

### Service Discovery

**Prometheus Kubernetes SD Config**:
```yaml
scrape_configs:
  # Kubernetes pods
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      
      # Use custom port if specified
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
      
      # Add pod metadata as labels
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: kubernetes_pod_name
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
```

### Fluentd Kubernetes Integration

**Fluentd DaemonSet Configuration**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      read_from_head true
      <parse>
        @type json
        time_key time
        time_format %Y-%m-%dT%H:%M:%S.%NZ
      </parse>
    </source>
    
    # Enrich with Kubernetes metadata
    <filter kubernetes.**>
      @type kubernetes_metadata
      @id filter_kube_metadata
    </filter>
    
    # Send to Loki
    <match kubernetes.**>
      @type loki
      url http://loki:3100
      extra_labels {"cluster":"production"}
      <label>
        namespace
        pod_name
        container_name
      </label>
      <buffer>
        flush_interval 10s
        flush_at_shutdown true
      </buffer>
    </match>
```

## Database Monitoring

### PostgreSQL Integration

**Postgres Exporter**:
```yaml
# postgres-exporter deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-exporter
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: postgres-exporter
        image: prometheuscommunity/postgres-exporter:latest
        env:
        - name: DATA_SOURCE_NAME
          value: "postgresql://user:password@postgres:5432/database?sslmode=disable"
        ports:
        - containerPort: 9187
        args:
          - --collector.database
          - --collector.locks
          - --collector.replication
          - --collector.stat_statements
```

**Custom PostgreSQL Queries**:
```yaml
# Custom queries for postgres_exporter
queries:
  - name: "table_sizes"
    query: |
      SELECT 
        schemaname,
        tablename,
        pg_total_relation_size(schemaname||'.'||tablename) as size_bytes
      FROM pg_tables
      WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
    metrics:
      - schemaname:
          usage: LABEL
      - tablename:
          usage: LABEL
      - size_bytes:
          usage: GAUGE
```

### MySQL Integration

**MySQL Exporter**:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-exporter
data:
  .my.cnf: |
    [client]
    user=exporter
    password=secret
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql-exporter
spec:
  template:
    spec:
      containers:
      - name: mysql-exporter
        image: prom/mysqld-exporter:latest
        env:
        - name: DATA_SOURCE_NAME
          value: "exporter:secret@(mysql:3306)/"
        ports:
        - containerPort: 9104
        volumeMounts:
        - name: config
          mountPath: /root/.my.cnf
          subPath: .my.cnf
      volumes:
      - name: config
        secret:
          secretName: mysql-exporter
```

### Redis Integration

**Redis Exporter**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-exporter
spec:
  template:
    spec:
      containers:
      - name: redis-exporter
        image: oliver006/redis_exporter:latest
        env:
        - name: REDIS_ADDR
          value: "redis:6379"
        - name: REDIS_PASSWORD
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: password
        ports:
        - containerPort: 9121
```

## Message Queue Integration

### RabbitMQ

**RabbitMQ Prometheus Plugin**:
```bash
# Enable plugin
rabbitmq-plugins enable rabbitmq_prometheus

# Metrics available at http://rabbitmq:15692/metrics
```

**Prometheus Scrape Config**:
```yaml
scrape_configs:
  - job_name: 'rabbitmq'
    static_configs:
      - targets: ['rabbitmq:15692']
    metrics_path: /metrics
```

### Kafka

**Kafka Exporter**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kafka-exporter
spec:
  template:
    spec:
      containers:
      - name: kafka-exporter
        image: danielqsj/kafka-exporter:latest
        args:
          - --kafka.server=kafka:9092
          - --kafka.version=2.8.0
        ports:
        - containerPort: 9308
```

## APM Integration

### Datadog

**Datadog Agent Configuration**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: datadog-config
data:
  datadog.yaml: |
    api_key: YOUR_API_KEY
    site: datadoghq.com
    logs_enabled: true
    apm_config:
      enabled: true
      env: production
    process_config:
      enabled: true
```

### New Relic

**New Relic Integration**:
```go
package main

import (
    "github.com/newrelic/go-agent/v3/newrelic"
)

func main() {
    app, err := newrelic.NewApplication(
        newrelic.ConfigAppName("My Application"),
        newrelic.ConfigLicense("YOUR_LICENSE_KEY"),
        newrelic.ConfigDistributedTracerEnabled(true),
    )
    
    // Instrument HTTP handler
    http.HandleFunc(newrelic.WrapHandleFunc(app, "/api/users", handleUsers))
}
```

## Incident Management Integration

### PagerDuty

**Alertmanager PagerDuty Configuration**:
```yaml
receivers:
  - name: 'pagerduty'
    pagerduty_configs:
      - service_key: 'YOUR_SERVICE_KEY'
        description: '{{ .GroupLabels.alertname }}'
        details:
          firing: '{{ .Alerts.Firing | len }}'
          resolved: '{{ .Alerts.Resolved | len }}'
          severity: '{{ .GroupLabels.severity }}'

route:
  group_by: ['alertname', 'cluster']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 12h
  receiver: 'pagerduty'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true
```

### Slack

**Alertmanager Slack Configuration**:
```yaml
receivers:
  - name: 'slack'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        channel: '#alerts'
        title: '{{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Labels.alertname }}
          *Severity:* {{ .Labels.severity }}
          *Description:* {{ .Annotations.description }}
          *Details:*
          {{ range .Labels.SortedPairs }} • {{ .Name }}: {{ .Value }}
          {{ end }}
          {{ end }}
```

## Custom Exporters

### Writing a Custom Exporter (Go)**:
```go
package main

import (
    "net/http"
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
)

type CustomCollector struct {
    metric prometheus.Gauge
}

func NewCustomCollector() *CustomCollector {
    return &CustomCollector{
        metric: prometheus.NewGauge(prometheus.GaugeOpts{
            Name: "custom_metric",
            Help: "A custom metric",
        }),
    }
}

func (c *CustomCollector) Describe(ch chan<- *prometheus.Desc) {
    c.metric.Describe(ch)
}

func (c *CustomCollector) Collect(ch chan<- prometheus.Metric) {
    // Fetch data from your source
    value := fetchCustomData()
    c.metric.Set(value)
    c.metric.Collect(ch)
}

func main() {
    collector := NewCustomCollector()
    prometheus.MustRegister(collector)
    
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":9090", nil)
}
```

## API Integration

### Grafana API

**Create Dashboard Programmatically**:
```python
import requests
import json

GRAFANA_URL = "http://grafana:3000"
API_KEY = "your-api-key"

headers = {
    "Authorization": f"Bearer {API_KEY}",
    "Content-Type": "application/json"
}

dashboard = {
    "dashboard": {
        "title": "My Dashboard",
        "panels": [
            {
                "type": "graph",
                "title": "CPU Usage",
                "targets": [
                    {
                        "expr": "100 - (avg by (instance) (rate(node_cpu_seconds_total{mode='idle'}[5m])) * 100)"
                    }
                ]
            }
        ]
    },
    "overwrite": True
}

response = requests.post(
    f"{GRAFANA_URL}/api/dashboards/db",
    headers=headers,
    data=json.dumps(dashboard)
)

print(response.json())
```

### Prometheus Query API

**Query Prometheus**:
```python
import requests
from datetime import datetime, timedelta

PROMETHEUS_URL = "http://prometheus:9090"

# Instant query
response = requests.get(
    f"{PROMETHEUS_URL}/api/v1/query",
    params={
        "query": "up",
        "time": datetime.now().isoformat()
    }
)

# Range query
response = requests.get(
    f"{PROMETHEUS_URL}/api/v1/query_range",
    params={
        "query": "rate(http_requests_total[5m])",
        "start": (datetime.now() - timedelta(hours=1)).isoformat(),
        "end": datetime.now().isoformat(),
        "step": "15s"
    }
)

data = response.json()
for result in data['data']['result']:
    print(f"Labels: {result['metric']}")
    for timestamp, value in result['values']:
        print(f"  {timestamp}: {value}")
```

## Integration Checklist

- [ ] Application instrumented with metrics
- [ ] Logs formatted as JSON
- [ ] Distributed tracing configured
- [ ] Prometheus scraping endpoints
- [ ] Fluentd/Fluent Bit collecting logs
- [ ] OpenTelemetry collector configured
- [ ] Grafana data sources connected
- [ ] Alertmanager configured
- [ ] Notification channels tested
- [ ] Dashboards created
- [ ] Alert rules defined
- [ ] Runbooks documented
