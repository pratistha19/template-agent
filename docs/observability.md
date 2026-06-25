# Observability Guide

This guide covers OpenTelemetry (OTEL) metrics and tracing for the template agent.

## Overview

The agent supports two complementary observability systems:

1. **Langfuse** - LLM-specific tracing (prompts, completions, tokens, costs)
2. **OpenTelemetry** - Operational metrics and distributed tracing (requests, latency, errors)

Both systems coexist without conflict. Langfuse traces LLM calls; OTEL traces infrastructure.

## Local Development

### Quick Start

Start the full observability stack:

```bash
docker compose --profile observability up
```

This launches:
- **OTEL Collector** - Receives metrics/traces from agent
- **Jaeger** - Trace visualization UI at http://localhost:16686
- **Prometheus** - Metrics storage/query UI at http://localhost:9090

### Enable OTEL in Agent

Uncomment these lines in `compose.yaml` under `template-agent` service:

```yaml
environment:
  - ENABLE_OTEL=true
  - OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
```

Or set in `.env`:

```bash
ENABLE_OTEL=true
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
```

### View Metrics

**Prometheus UI**: http://localhost:9090

Example queries:
```promql
# Conversation rate
rate(template_agent_conversations_total[5m])

# Active conversations
template_agent_active_conversations

# Stream latency p95
histogram_quantile(0.95, rate(template_agent_stream_duration_seconds_bucket[5m]))

# Time to first token p99
histogram_quantile(0.99, rate(template_agent_time_to_first_token_seconds_bucket[5m]))
```

**OTEL Collector Metrics**: http://localhost:8889/metrics (raw Prometheus format)

### View Traces

**Jaeger UI**: http://localhost:16686

1. Select service: `template-agent`
2. Click "Find Traces"
3. Drill into specific requests to see:
   - HTTP request spans (FastAPI auto-instrumentation)
   - Custom spans (when wired)
   - W3C trace context propagation

## OpenShift Deployment

### Architecture

```
Agent Pod → OTEL Collector Pod → Prometheus/Jaeger (external)
```

The OTEL collector runs as a separate deployment in the same namespace.

### Configuration

OTEL is **disabled by default**. Enable via ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: agent-config
data:
  ENABLE_OTEL: "true"
  OTEL_EXPORTER_OTLP_ENDPOINT: "http://otel-collector:4317"
```

Apply collector manifests:

```bash
oc apply -f deployment/overlays/openshift/otel-collector-configmap.yaml
oc apply -f deployment/overlays/openshift/otel-collector-deployment.yaml
oc apply -f deployment/overlays/openshift/otel-collector-service.yaml
oc apply -f deployment/overlays/openshift/otel-collector-route.yaml
```

### Verify OTEL Setup

Check agent logs for successful initialization:

```bash
oc logs deployment/agent | grep -i otel
# Should show: "OTEL enabled — exporting to http://otel-collector:4317"
```

Check collector health:

```bash
oc get pods -l component=otel-collector
oc logs deployment/otel-collector
```

Access collector metrics endpoint:

```bash
oc port-forward deployment/otel-collector 8889:8889
curl http://localhost:8889/metrics
```

### Trace Backend

The OpenShift collector is configured with **debug exporter only** for traces. To send traces to a backend:

Edit `deployment/overlays/openshift/otel-collector-configmap.yaml`:

```yaml
exporters:
  otlp/jaeger:
    endpoint: "jaeger-collector.observability.svc.cluster.local:4317"
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/jaeger, debug]  # Add trace backend
```

### Prometheus Integration

Scrape OTEL collector metrics from Prometheus:

```yaml
- job_name: 'otel-collector'
  kubernetes_sd_configs:
    - role: pod
      namespaces:
        names:
          - <your-namespace>
  relabel_configs:
    - source_labels: [__meta_kubernetes_pod_label_component]
      regex: otel-collector
      action: keep
```

Metrics endpoint: `http://otel-collector:8889/metrics`

## Available Metrics

All metrics use prefix `template_agent_`.

### Conversations

- `template_agent_conversations_total{status}` - Counter of conversations by status
- `template_agent_active_conversations` - Gauge of currently active conversations
- `template_agent_conversation_duration_seconds` - Histogram of conversation duration

### Messages

- `template_agent_messages_total{direction,message_type}` - Counter of messages sent/received

### Streaming

- `template_agent_stream_tokens_total` - Counter of tokens streamed
- `template_agent_stream_duration_seconds` - Histogram of stream duration
- `template_agent_stream_errors_total{error_type}` - Counter of stream failures
- `template_agent_time_to_first_token_seconds` - Histogram of TTFT latency

### Threads

- `template_agent_threads_created_total` - Counter of threads created
- `template_agent_threads_active` - Gauge of currently active threads
- `template_agent_threads_deleted_total` - Counter of threads deleted
- `template_agent_thread_messages_count` - Histogram of messages per thread

## Instrumentation Status

The OTEL module defines `record_*` helpers for all metrics, but they are **not yet wired** to runtime code. To enable metric recording:

1. **Conversations**: Call `record_conversation_started()` at conversation lifecycle start, `record_conversation_completed()` at end
2. **Messages**: Call `record_message_sent()` in message ingress/egress handlers
3. **Streams**: Call `record_stream_started()`, `record_first_token()`, `record_stream_completed()` in streaming endpoints
4. **Threads**: Call `record_thread_created()`, `record_thread_deleted()` in thread management APIs

See `deep_agent/aegra/otel.py` for full API.

## Configuration Reference

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `ENABLE_OTEL` | `false` | Enable OTEL metrics/tracing export |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | `http://localhost:4317` | OTLP gRPC endpoint |
| `OTEL_EXPORTER_OTLP_INSECURE` | `true` | Disable TLS for collector connection |
| `OTEL_METRIC_EXPORT_INTERVAL` | `5000` | Metric export interval (ms) |

### YAML Configuration

Config file: `config/agent/runtime/observability.yaml`

```yaml
otel:
  enabled: false
  exporter:
    endpoint: "http://localhost:4317"
    insecure: true
  metrics:
    export_interval_ms: 5000
  tracing:
    fastapi_auto_instrument: true
```

Environment variables override YAML values.

## Troubleshooting

### Metrics Not Appearing

1. Check agent logs for OTEL initialization:
   ```bash
   grep -i otel <agent-logs>
   ```
   Should show: `"OTEL enabled — exporting to ..."`

2. Verify collector is reachable:
   ```bash
   curl -v http://otel-collector:4317
   ```

3. Check collector logs for received data:
   ```bash
   oc logs deployment/otel-collector | grep -i "datapoints"
   ```

4. Verify Prometheus is scraping collector:
   ```
   http://localhost:9090/targets
   ```

### Traces Not Showing in Jaeger

1. Confirm FastAPI auto-instrumentation is enabled:
   ```yaml
   otel:
     tracing:
       fastapi_auto_instrument: true
   ```

2. Check collector pipeline includes trace exporter:
   ```yaml
   service:
     pipelines:
       traces:
         exporters: [otlp/jaeger, debug]
   ```

3. Verify Jaeger is receiving from collector:
   ```bash
   oc logs deployment/jaeger | grep -i "spans"
   ```

### High Memory Usage in Collector

Default limits are conservative (256Mi). If handling high trace volume, increase:

```yaml
resources:
  limits:
    memory: "512Mi"  # or higher
```

Monitor collector metrics for batch queue size and dropped spans.

## Cost Considerations

- **Metrics**: Low volume, negligible cost
- **Traces**: High volume if sampling 100%. Configure sampling in collector:

```yaml
processors:
  probabilistic_sampler:
    sampling_percentage: 10  # Sample 10% of traces

service:
  pipelines:
    traces:
      processors: [probabilistic_sampler, batch]
```

Or use tail-based sampling to keep only slow/error traces.
