# Monitoring Configuration

This directory contains configuration files for the observability stack used by the Quarkus Super-Heroes application.

## Contents

- **`config/`** - Configuration files for monitoring tools
- **`docker-compose/`** - Docker Compose files for local monitoring stack
- **`k8s/`** - Kubernetes manifests for monitoring deployment

## Quick Start

### Local Development (Docker Compose)

Start the full monitoring stack with the application:

```bash
cd ../deploy/docker-compose
docker compose -f java21.yml -f monitoring.yml up
```

Access monitoring tools:
- **Prometheus**: http://localhost:9090
- **Jaeger UI**: http://localhost:16686
- **OpenTelemetry Collector**: http://localhost:4317 (gRPC), http://localhost:4318 (HTTP)

### Kubernetes Deployment

The monitoring stack is automatically deployed with the application:

```bash
kubectl apply -f ../deploy/k8s/java21-kubernetes.yml
```

## Components

### Prometheus

Metrics collection and storage.

- **Port**: 9090 (HTTP)
- **Config**: `config/prometheus.yml`
- **Scrape configs**: `../deploy/docker-compose/monitoring/prometheus_scrape_configs.yml`

### Jaeger

Distributed tracing visualization.

- **UI Port**: 16686
- **Collector Port**: 14250
- **Query Port**: 16686
- **Storage**: In-memory (development), Elasticsearch/Cassandra (production)

### OpenTelemetry Collector

Telemetry data collection and export.

- **OTLP gRPC**: 4317
- **OTLP HTTP**: 4318
- **Prometheus Metrics**: 8889
- **Health Check**: 13133
- **Config**: `../deploy/docker-compose/monitoring/otel-collector-config.yml`

## Configuration Files

### Prometheus Scrape Configuration

Located at `../deploy/docker-compose/monitoring/prometheus_scrape_configs.yml`

Defines which services to scrape for metrics:

```yaml
scrape_configs:
  - job_name: 'rest-heroes'
    static_configs:
      - targets: ['rest-heroes:8083']
  # ... additional services
```

### OpenTelemetry Collector Configuration

Located at `../deploy/docker-compose/monitoring/otel-collector-config.yml`

Defines receivers, processors, and exporters:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  jaeger:
    endpoint: jaeger:14250
  prometheus:
    endpoint: 0.0.0.0:8889
```

## Customization

### Adding Custom Metrics

Services automatically expose metrics at `/q/metrics`. To add custom business metrics:

```java
@Counted(name = "fights.total", description = "Total number of fights")
@Timed(name = "fights.duration", description = "Fight duration")
public Fight createFight(Fight fight) {
    // Business logic
}
```

### Modifying Scrape Intervals

Edit `prometheus_scrape_configs.yml`:

```yaml
scrape_configs:
  - job_name: 'rest-fights'
    scrape_interval: 30s  # Change from default 15s
    static_configs:
      - targets: ['rest-fights:8082']
```

### Adding Alert Rules

Create `prometheus-rules.yml`:

```yaml
groups:
  - name: superheroes
    rules:
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
```

Reference in `prometheus.yml`:

```yaml
rule_files:
  - /etc/prometheus/prometheus-rules.yml
```

## Production Deployment

For production environments, consider:

1. **Persistent storage** for Prometheus data
2. **External database** for Jaeger (Elasticsearch or Cassandra)
3. **High availability** configuration for all components
4. **Authentication** for accessing monitoring tools
5. **Network policies** to restrict access
6. **Resource limits** appropriate for your scale

### Example Kubernetes Resources

```yaml
# Prometheus with persistent storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: prometheus-storage
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi
```

## Documentation

For detailed monitoring setup and usage, see:

- **[Monitoring and Observability Guide](../docs/monitoring.md)** - Complete guide to observability
- **[Automation Guide](../docs/automation.md)** - CI/CD and monitoring deployment
- **[Deployment Guides](../deploy/)** - Platform-specific deployment instructions

## Troubleshooting

### Prometheus not scraping metrics

1. Check target health: http://localhost:9090/targets
2. Verify services are exposing metrics: `curl http://service:port/q/metrics`
3. Check network connectivity between Prometheus and services
4. Review Prometheus logs: `docker compose logs prometheus`

### Jaeger showing no traces

1. Verify OTLP endpoint configuration in services
2. Check OpenTelemetry Collector logs: `docker compose logs otel-collector`
3. Ensure tracing is enabled: `quarkus.otel.enabled=true`
4. Check sampling configuration: `quarkus.otel.traces.sampler`

### High resource usage

1. Reduce Prometheus retention period
2. Increase scrape intervals
3. Enable trace sampling in high-volume services
4. Use metric relabeling to drop unnecessary metrics

## Resources

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Quarkus Observability](https://quarkus.io/guides/observability)

---

**Note**: Configuration files in this directory are designed for development and testing. Adjust settings for production workloads.
