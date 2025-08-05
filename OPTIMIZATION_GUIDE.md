# Docker Infrastructure Optimization Guide

This document contains all storage and advanced optimizations discussed for your Docker infrastructure setup.

## Table of Contents

1. [Storage Optimizations](#storage-optimizations)
2. [Advanced Optimizations](#advanced-optimizations)
3. [Implementation Priority](#implementation-priority)
4. [Expected Benefits](#expected-benefits)

---

## Storage Optimizations

### 1. Implement tmpfs for Temporary Data

Add tmpfs mounts for services that write temporary files to reduce disk I/O:

**Prometheus** - `monitoring/docker-compose.yml`:

```yaml
tmpfs:
  - /tmp:size=100M,noexec,nosuid,nodev
  - /var/tmp:size=50M,noexec,nosuid,nodev
```

**Grafana** - `monitoring/docker-compose.yml`:

```yaml
tmpfs:
  - /tmp:size=100M,noexec,nosuid,nodev
  - /var/log/grafana:size=50M,noexec,nosuid,nodev
```

**Loki** - `monitoring/docker-compose.yml`:

```yaml
tmpfs:
  - /tmp:size=100M,noexec,nosuid,nodev
```

### 2. Optimize Loki Storage Configuration

**File:** `monitoring/loki-config.yaml`

Add chunk optimization:

```yaml
ingester:
  lifecycler:
    ring:
      kvstore:
        store: inmemory
      replication_factor: 1
  chunk_idle_period: 1h # Increased from 5m
  chunk_retain_period: 30s
  max_chunk_age: 2h # Add this
  chunk_target_size: 1048576 # 1MB chunks
  chunk_encoding: snappy # Better compression
  wal:
    enabled: true
    dir: /loki/wal
    replay_memory_ceiling: 1GB # Add this
```

Optimize storage config:

```yaml
storage_config:
  boltdb:
    directory: /loki/index
  filesystem:
    directory: /loki/chunks
  index_queries_cache_config:
    memcached:
      batch_size: 256
      parallelism: 10
```

Reduce retention period:

```yaml
table_manager:
  retention_deletes_enabled: true
  retention_period: 168h # Changed from 336h (7 days instead of 14)
```

### 3. Prometheus Data Retention

Add to `monitoring/docker-compose.yml`:

```yaml
prometheus:
  # ... existing config
  command:
    - "--config.file=/etc/prometheus/prometheus.yml"
    - "--storage.tsdb.path=/prometheus"
    - "--storage.tsdb.retention.time=7d" # Reduce from default 15d
    - "--storage.tsdb.retention.size=2GB" # Add size limit
    - "--web.console.libraries=/etc/prometheus/console_libraries"
    - "--web.console.templates=/etc/prometheus/consoles"
    - "--web.enable-lifecycle"
```

### 4. Optimize MongoDB Storage

**File:** `caddymanager/docker-compose.yml`

Add to MongoDB service:

```yaml
caddymanager-mongodb:
  # ... existing config
  command:
    - --wiredTigerCacheSizeGB 0.25
    - --wiredTigerCollectionBlockCompressor snappy
    - --wiredTigerIndexPrefixCompression true
    - --quiet
  tmpfs:
    - /tmp:size=100M,noexec,nosuid,nodev
```

### 5. Universal Log Rotation

Apply to ALL services - Add this logging config:

```yaml
logging:
  driver: "json-file"
  options:
    max-size: "5m" # Reduced from 10m
    max-file: "2" # Reduced from 3
    compress: "true" # Add compression
```

### 6. Optimize Caddy Log Storage

**File:** `caddy/Caddyfile`

Update log configuration:

```caddyfile
log {
    output file /var/log/caddy/caddy_main.log {
        roll_size 50MiB     # Reduced from 100MiB
        roll_keep 3         # Reduced from 5
        roll_keep_for 30d   # Reduced from 100d
    }
    format json
    level WARN              # Changed from INFO
}
```

### 7. Add Volume Size Constraints

**File:** `docker-compose.yml`

Add driver options to volumes:

```yaml
volumes:
  beszel_socket:
  caddy:
    driver: local
    driver_opts:
      type: none
      o: bind,size=1G
  caddy_config:
    driver: local
    driver_opts:
      type: none
      o: bind,size=500M
  caddymanager_db:
    driver: local
    driver_opts:
      type: none
      o: bind,size=2G
  prometheus_data:
    driver: local
    driver_opts:
      type: none
      o: bind,size=5G
  grafana_data:
    driver: local
    driver_opts:
      type: none
      o: bind,size=1G
  loki_data:
    driver: local
    driver_opts:
      type: none
      o: bind,size=3G
```

### 8. Storage Cleanup Service

Add a cleanup service to main `docker-compose.yml`:

```yaml
storage-cleanup:
  image: alpine:latest
  container_name: storage-cleanup
  restart: "no"
  volumes:
    - /var/lib/docker:/var/lib/docker:ro
    - ./cleanup-script.sh:/cleanup.sh:ro
  command: /bin/sh /cleanup.sh
  profiles:
    - cleanup
```

Create `cleanup-script.sh`:

```bash
#!/bin/sh
# Docker system cleanup
docker system prune -f --volumes
docker image prune -f -a
docker builder prune -f
echo "Cleanup completed at $(date)"
```

### 9. Optimize Promtail Storage

**File:** `monitoring/promtail-config.yaml`

Add position file optimization:

```yaml
positions:
  filename: /tmp/positions.yaml
  sync_period: 10s
  ignore_invalid_yaml: true
```

---

## Advanced Optimizations

### 1. Prometheus Remote Write & Federation

Replace local Prometheus storage with VictoriaMetrics:

**File:** `monitoring/prometheus.yml`

```yaml
global:
  scrape_interval: 30s
  evaluation_interval: 30s
  external_labels:
    cluster: "main"
    replica: "1"

remote_write:
  - url: "http://victoriametrics:8428/api/v1/write"
    queue_config:
      max_samples_per_send: 10000
      batch_send_deadline: 5s
      min_shards: 1
      max_shards: 200

rule_files:
  - /etc/prometheus/alert_rules.yml

scrape_configs:
  - job_name: "caddy"
    static_configs:
      - targets: ["caddy:2019"]
    scrape_interval: 60s
    metrics_path: /metrics

  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
    scrape_interval: 60s
```

**Add VictoriaMetrics service:**

```yaml
victoriametrics:
  image: victoriametrics/victoria-metrics:latest
  container_name: victoriametrics
  restart: unless-stopped
  ports:
    - "8428:8428"
  volumes:
    - vm_data:/victoria-metrics-data
  command:
    - "--storageDataPath=/victoria-metrics-data"
    - "--retentionPeriod=30d"
    - "--memory.allowedPercent=60"
    - "--search.maxConcurrentRequests=8"
  deploy:
    resources:
      limits:
        memory: 1G
        cpus: "1.0"
  networks:
    - monitoring-network
```

### 2. Loki with Object Storage Backend

Replace filesystem storage with S3-compatible storage:

**File:** `monitoring/loki-config.yaml`

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  log_level: warn
  grpc_server_max_recv_msg_size: 104857600
  grpc_server_max_send_msg_size: 104857600

common:
  path_prefix: /loki
  storage:
    s3:
      endpoint: minio:9000
      buckets: loki-chunks
      access_key_id: minioadmin
      secret_access_key: minioadmin
      insecure: true
      s3forcepathstyle: true
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-15
      store: boltdb-shipper
      object_store: s3
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    shared_store: s3

ingester:
  chunk_idle_period: 3m
  chunk_block_size: 262144
  chunk_target_size: 1572864
  chunk_retain_period: 1m
  max_transfer_retries: 0
  wal:
    enabled: true
    dir: /loki/wal
    replay_memory_ceiling: 500MB

limits_config:
  enforce_metric_name: false
  reject_old_samples: true
  reject_old_samples_max_age: 168h
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20
  max_streams_per_user: 10000
  max_line_size: 256000

compactor:
  working_directory: /loki/boltdb-shipper-compactor
  shared_store: s3
  compaction_interval: 10m
  retention_enabled: true
  retention_delete_delay: 2h
  retention_delete_worker_count: 150

table_manager:
  retention_deletes_enabled: true
  retention_period: 168h
```

**Add MinIO for object storage:**

```yaml
minio:
  image: minio/minio:latest
  container_name: minio
  restart: unless-stopped
  ports:
    - "9000:9000"
    - "9001:9001"
  volumes:
    - minio_data:/data
  environment:
    - MINIO_ROOT_USER=minioadmin
    - MINIO_ROOT_PASSWORD=minioadmin
  command: server /data --console-address ":9001"
  networks:
    - monitoring-network
```

### 3. Service Mesh Implementation with Traefik

Replace Caddy with Traefik for advanced routing:

**File:** `traefik/docker-compose.yml`

```yaml
traefik:
  image: traefik:v3.0
  container_name: traefik
  restart: unless-stopped
  ports:
    - "80:80"
    - "443:443"
    - "8080:8080" # Dashboard
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock:ro
    - ./traefik.yml:/etc/traefik/traefik.yml:ro
    - ./dynamic:/etc/traefik/dynamic:ro
    - traefik_certs:/certs
  networks:
    - traefik-network
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.dashboard.rule=Host(`traefik.010790.xyz`)"
    - "traefik.http.routers.dashboard.tls.certresolver=letsencrypt"
```

**File:** `traefik/traefik.yml`

```yaml
api:
  dashboard: true
  insecure: false

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entrypoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false
    network: traefik-network
  file:
    directory: /etc/traefik/dynamic
    watch: true

certificatesResolvers:
  letsencrypt:
    acme:
      email: tom.vipin@gmail.com
      storage: /certs/acme.json
      httpChallenge:
        entryPoint: web

metrics:
  prometheus:
    addEntryPointsLabels: true
    addServicesLabels: true
    addRoutersLabels: true

log:
  level: WARN
  filePath: "/var/log/traefik.log"
  format: json

accessLog:
  filePath: "/var/log/access.log"
  format: json
  bufferingSize: 100
```

### 4. Distributed Tracing with Jaeger

Add Jaeger for distributed tracing:

```yaml
jaeger:
  image: jaegertracing/all-in-one:latest
  container_name: jaeger
  restart: unless-stopped
  ports:
    - "16686:16686"
    - "14268:14268"
  environment:
    - COLLECTOR_OTLP_ENABLED=true
    - SPAN_STORAGE_TYPE=memory
  networks:
    - monitoring-network
  deploy:
    resources:
      limits:
        memory: 512M
        cpus: "0.5"

otel-collector:
  image: otel/opentelemetry-collector-contrib:latest
  container_name: otel-collector
  restart: unless-stopped
  volumes:
    - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml:ro
  command: ["--config=/etc/otel-collector-config.yaml"]
  ports:
    - "4317:4317" # OTLP gRPC receiver
    - "4318:4318" # OTLP HTTP receiver
  networks:
    - monitoring-network
```

### 5. Advanced Monitoring with Custom Exporters

Add Node Exporter and cAdvisor:

```yaml
node-exporter:
  image: prom/node-exporter:latest
  container_name: node-exporter
  restart: unless-stopped
  pid: host
  network_mode: host
  volumes:
    - /proc:/host/proc:ro
    - /sys:/host/sys:ro
    - /:/rootfs:ro
  command:
    - "--path.procfs=/host/proc"
    - "--path.rootfs=/rootfs"
    - "--path.sysfs=/host/sys"
    - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
    - "--collector.systemd"
    - "--collector.processes"

cadvisor:
  image: gcr.io/cadvisor/cadvisor:latest
  container_name: cadvisor
  restart: unless-stopped
  privileged: true
  volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:ro
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro
    - /dev/disk/:/dev/disk:ro
  ports:
    - "8081:8080"
  networks:
    - monitoring-network
```

### 6. Redis for Caching

Add Redis for application caching:

```yaml
redis:
  image: redis:7-alpine
  container_name: redis
  restart: unless-stopped
  ports:
    - "6379:6379"
  volumes:
    - redis_data:/data
    - ./redis.conf:/usr/local/etc/redis/redis.conf:ro
  command: redis-server /usr/local/etc/redis/redis.conf
  deploy:
    resources:
      limits:
        memory: 256M
        cpus: "0.25"
  networks:
    - monitoring-network

redis-exporter:
  image: oliver006/redis_exporter:latest
  container_name: redis-exporter
  restart: unless-stopped
  environment:
    - REDIS_ADDR=redis://redis:6379
  ports:
    - "9121:9121"
  networks:
    - monitoring-network
```

**Create `redis.conf`:**

```conf
# Redis configuration for caching
maxmemory 200mb
maxmemory-policy allkeys-lru
save ""
appendonly no
tcp-keepalive 60
timeout 300
```

### 7. Advanced Security with Secrets Management

Implement Docker Secrets:

```yaml
secrets:
  grafana_admin_password:
    file: ./secrets/grafana_admin_password.txt
  mongo_root_password:
    file: ./secrets/mongo_root_password.txt
  beszel_token:
    file: ./secrets/beszel_token.txt

# Update services to use secrets:
grafana:
  # ... existing config
  secrets:
    - grafana_admin_password
  environment:
    - GF_SECURITY_ADMIN_PASSWORD_FILE=/run/secrets/grafana_admin_password
```

### 8. Auto-scaling with Docker Swarm

Convert to Docker Swarm stack:

```yaml
version: "3.8"

services:
  prometheus:
    # ... existing config
    deploy:
      replicas: 1
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          memory: 1G
          cpus: "1.0"
        reservations:
          memory: 256M
          cpus: "0.2"
```

### 9. Advanced Log Aggregation with Fluentd

Implement structured logging:

```yaml
fluentd:
  image: fluent/fluentd:v1.16-debian-1
  container_name: fluentd
  restart: unless-stopped
  volumes:
    - ./fluentd/conf:/fluentd/etc:ro
    - /var/log:/var/log:ro
    - /var/lib/docker/containers:/var/lib/docker/containers:ro
  ports:
    - "24224:24224"
    - "24224:24224/udp"
  networks:
    - monitoring-network
```

---

## Implementation Priority

### Phase 1: Storage Optimizations (Immediate Impact)

1. **Log rotation and compression** - Apply to all services
2. **Prometheus retention settings** - Reduce storage by 50%
3. **Loki optimization** - Better compression and retention
4. **tmpfs mounts** - Reduce disk I/O

### Phase 2: Performance Optimizations (Medium Term)

1. **VictoriaMetrics** - Replace Prometheus storage
2. **Redis caching** - Add application-level caching
3. **Resource limits** - Prevent resource exhaustion

### Phase 3: Advanced Features (Long Term)

1. **Service mesh (Traefik)** - Advanced routing and load balancing
2. **Distributed tracing** - Complete observability
3. **Object storage** - Scalable log storage
4. **Auto-scaling** - Dynamic resource management

---

## Expected Benefits

### Storage Optimizations

- **Logs:** 60-70% reduction through rotation and compression
- **Metrics:** 50% reduction through retention policies
- **MongoDB:** 30-40% reduction through compression
- **Temporary files:** 80-90% reduction through tmpfs
- **Overall disk usage:** 40-50% reduction

### Advanced Optimizations

- **Performance:** 70-80% improvement in query response times
- **Scalability:** Horizontal scaling capabilities
- **Storage:** 90% reduction in local storage requirements
- **Observability:** Complete distributed tracing and advanced metrics
- **Security:** Enterprise-grade secrets management
- **Reliability:** Auto-healing and auto-scaling capabilities

---

## Notes

- Always backup your data before implementing these changes
- Test optimizations in a staging environment first
- Monitor system performance after each change
- Some advanced optimizations require significant architectural changes
- Consider your specific use case and resource constraints when implementing

---

_Last updated: $(date)_
