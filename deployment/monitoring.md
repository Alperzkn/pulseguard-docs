# Monitoring Guide

## Overview

Comprehensive monitoring is essential for maintaining PulseGuard's reliability, performance, and data quality. This guide covers monitoring strategies for all system components.

## Monitoring Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Monitoring Stack                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐ │
│  │   Prometheus    │───▶│    Grafana      │───▶│   Alerts    │ │
│  │   (Metrics)     │    │   (Dashboard)   │    │ (Slack/Email)│ │
│  └─────────────────┘    └─────────────────┘    └─────────────┘ │
│           │                       │                       │   │
│           ▼                       ▼                       ▼   │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────┐ │
│  │   Node Exporter │    │  Custom Metrics │    │ Health Checks│ │
│  │  (System Stats) │    │ (App Specific)  │    │ (Uptime/SLA) │ │
│  └─────────────────┘    └─────────────────┘    └─────────────┘ │
│           │                       │                       │   │
│           ▼                       ▼                       ▼   │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │              PulseGuard Services                       │   │
│  │  Data Collector | API Service | Dashboard | Database  │   │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## System Monitoring

### 1. Prometheus Setup

Create `monitoring/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  # System metrics
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  # PulseGuard API metrics
  - job_name: 'pulseguard-api'
    static_configs:
      - targets: ['api-service:3000']
    metrics_path: '/metrics'
    scrape_interval: 30s

  # PostgreSQL metrics
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']

  # Docker metrics
  - job_name: 'docker'
    static_configs:
      - targets: ['cadvisor:8080']

  # Custom application metrics
  - job_name: 'pulseguard-collector'
    static_configs:
      - targets: ['data-collector:9090']
    metrics_path: '/metrics'
    scrape_interval: 60s
```

### 2. Docker Compose for Monitoring Stack

```yaml
# monitoring/docker-compose.monitoring.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./rules:/etc/prometheus/rules
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

  postgres-exporter:
    image: prometheuscommunity/postgres-exporter:latest
    container_name: postgres-exporter
    ports:
      - "9187:9187"
    environment:
      DATA_SOURCE_NAME: "postgresql://pulseguard_user:password@database:5432/crypto_db?sslmode=disable"
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - alertmanager_data:/alertmanager
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:
```

## Application Metrics

### 1. API Service Metrics

Add metrics endpoint to API service:

```javascript
// api-service/src/middleware/metrics.js
const promClient = require('prom-client');

// Create a Registry
const register = new promClient.Registry();

// Add default metrics
promClient.collectDefaultMetrics({ register });

// Custom metrics
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.5, 1, 2, 5]
});

const httpRequestTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code']
});

const apiKeyUsage = new promClient.Counter({
  name: 'api_key_requests_total',
  help: 'Total requests per API key',
  labelNames: ['api_key_hash']
});

const databaseQueryDuration = new promClient.Histogram({
  name: 'database_query_duration_seconds',
  help: 'Database query duration',
  labelNames: ['query_type', 'table'],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2]
});

const activeConnections = new promClient.Gauge({
  name: 'database_connections_active',
  help: 'Number of active database connections'
});

// Register metrics
register.registerMetric(httpRequestDuration);
register.registerMetric(httpRequestTotal);
register.registerMetric(apiKeyUsage);
register.registerMetric(databaseQueryDuration);
register.registerMetric(activeConnections);

// Middleware to collect HTTP metrics
const metricsMiddleware = (req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const route = req.route ? req.route.path : req.path;
    
    httpRequestDuration
      .labels(req.method, route, res.statusCode)
      .observe(duration);
    
    httpRequestTotal
      .labels(req.method, route, res.statusCode)
      .inc();
    
    // Track API key usage (hashed for privacy)
    if (req.headers['x-api-key']) {
      const crypto = require('crypto');
      const keyHash = crypto.createHash('sha256')
        .update(req.headers['x-api-key'])
        .digest('hex')
        .substring(0, 8);
      
      apiKeyUsage.labels(keyHash).inc();
    }
  });
  
  next();
};

// Metrics endpoint
const metricsEndpoint = async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
};

module.exports = {
  metricsMiddleware,
  metricsEndpoint,
  databaseQueryDuration,
  activeConnections,
  register
};
```

### 2. Data Collector Metrics

```javascript
// data-collector/app/metrics.js
const promClient = require('prom-client');
const express = require('express');

// Create registry
const register = new promClient.Registry();
promClient.collectDefaultMetrics({ register });

// Collection metrics
const collectionDuration = new promClient.Histogram({
  name: 'collection_duration_seconds',
  help: 'Time spent collecting data',
  labelNames: ['collection_type', 'status'],
  buckets: [10, 30, 60, 120, 300, 600]
});

const coinsProcessed = new promClient.Counter({
  name: 'coins_processed_total',
  help: 'Total number of coins processed',
  labelNames: ['status'] // success, failed
});

const apiCalls = new promClient.Counter({
  name: 'coingecko_api_calls_total',
  help: 'Total CoinGecko API calls',
  labelNames: ['status', 'endpoint']
});

const apiResponseTime = new promClient.Histogram({
  name: 'coingecko_api_response_time_seconds',
  help: 'CoinGecko API response time',
  labelNames: ['endpoint'],
  buckets: [0.5, 1, 2, 5, 10, 30]
});

const queueSize = new promClient.Gauge({
  name: 'collection_queue_size',
  help: 'Number of items in collection queue'
});

const lastSuccessfulCollection = new promClient.Gauge({
  name: 'last_successful_collection_timestamp',
  help: 'Timestamp of last successful collection'
});

// Register metrics
register.registerMetric(collectionDuration);
register.registerMetric(coinsProcessed);
register.registerMetric(apiCalls);
register.registerMetric(apiResponseTime);
register.registerMetric(queueSize);
register.registerMetric(lastSuccessfulCollection);

// Start metrics server
function startMetricsServer(port = 9090) {
  const app = express();
  
  app.get('/metrics', async (req, res) => {
    res.set('Content-Type', register.contentType);
    res.end(await register.metrics());
  });
  
  app.get('/health', (req, res) => {
    res.json({ status: 'healthy', timestamp: new Date().toISOString() });
  });
  
  app.listen(port, () => {
    console.log(`Metrics server listening on port ${port}`);
  });
}

module.exports = {
  collectionDuration,
  coinsProcessed,
  apiCalls,
  apiResponseTime,
  queueSize,
  lastSuccessfulCollection,
  startMetricsServer
};
```

## Alert Rules

### 1. Prometheus Alert Rules

Create `monitoring/rules/pulseguard.yml`:

```yaml
groups:
  - name: pulseguard.rules
    rules:
      # System alerts
      - alert: HighCPUUsage
        expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage is above 80% for more than 5 minutes"

      - alert: HighMemoryUsage
        expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
          description: "Memory usage is above 85% for more than 5 minutes"

      - alert: DiskSpaceLow
        expr: (node_filesystem_size_bytes - node_filesystem_free_bytes) / node_filesystem_size_bytes * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space is running low"
          description: "Disk usage is above 90%"

      # Application alerts
      - alert: APIServiceDown
        expr: up{job="pulseguard-api"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PulseGuard API service is down"
          description: "The API service has been down for more than 1 minute"

      - alert: DataCollectorDown
        expr: up{job="pulseguard-collector"} == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "PulseGuard data collector is down"
          description: "The data collector has been down for more than 5 minutes"

      - alert: HighAPIErrorRate
        expr: rate(http_requests_total{status_code=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High API error rate"
          description: "API error rate is above 10% for the last 5 minutes"

      - alert: SlowAPIResponses
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow API responses"
          description: "95th percentile response time is above 2 seconds"

      # Database alerts
      - alert: DatabaseDown
        expr: up{job="postgres"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL database is down"
          description: "The database has been unreachable for more than 1 minute"

      - alert: HighDatabaseConnections
        expr: pg_stat_database_numbackends > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High number of database connections"
          description: "Number of database connections is above 80"

      - alert: SlowDatabaseQueries
        expr: histogram_quantile(0.95, rate(database_query_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow database queries"
          description: "95th percentile query time is above 1 second"

      # Collection specific alerts
      - alert: CollectionFailed
        expr: increase(coins_processed_total{status="failed"}[1h]) > 50
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "High collection failure rate"
          description: "More than 50 coins failed to collect in the last hour"

      - alert: NoRecentCollection
        expr: time() - last_successful_collection_timestamp > 3600
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "No recent data collection"
          description: "No successful data collection in the last hour"

      - alert: CoinGeckoAPIIssues
        expr: rate(coingecko_api_calls_total{status!="success"}[10m]) > 0.2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CoinGecko API issues"
          description: "CoinGecko API error rate is above 20%"
```

### 2. Alertmanager Configuration

Create `monitoring/alertmanager.yml`:

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@your-domain.com'
  smtp_auth_username: 'alerts@your-domain.com'
  smtp_auth_password: 'your-app-password'

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'
  routes:
    - match:
        severity: critical
      receiver: 'critical-alerts'
    - match:
        severity: warning
      receiver: 'warning-alerts'

receivers:
  - name: 'web.hook'
    webhook_configs:
      - url: 'http://127.0.0.1:5001/'

  - name: 'critical-alerts'
    email_configs:
      - to: 'admin@your-domain.com'
        subject: 'CRITICAL: PulseGuard Alert - {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          Labels: {{ range .Labels.SortedPairs }}{{ .Name }}={{ .Value }} {{ end }}
          {{ end }}
    slack_configs:
      - api_url: 'your-slack-webhook-url'
        channel: '#alerts'
        title: 'CRITICAL: PulseGuard Alert'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

  - name: 'warning-alerts'
    email_configs:
      - to: 'team@your-domain.com'
        subject: 'WARNING: PulseGuard Alert - {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          {{ end }}
```

## Grafana Dashboards

### 1. PulseGuard Overview Dashboard

Create `monitoring/grafana/dashboards/pulseguard-overview.json`:

```json
{
  "dashboard": {
    "id": null,
    "title": "PulseGuard Overview",
    "tags": ["pulseguard"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "System Overview",
        "type": "stat",
        "targets": [
          {
            "expr": "up{job=~\"pulseguard.*\"}",
            "legendFormat": "{{job}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "color": {
              "mode": "thresholds"
            },
            "thresholds": {
              "steps": [
                {"color": "red", "value": 0},
                {"color": "green", "value": 1}
              ]
            }
          }
        }
      },
      {
        "id": 2,
        "title": "API Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{route}}"
          }
        ]
      },
      {
        "id": 3,
        "title": "Collection Success Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(coins_processed_total{status=\"success\"}[1h]) / rate(coins_processed_total[1h]) * 100",
            "legendFormat": "Success Rate %"
          }
        ]
      },
      {
        "id": 4,
        "title": "Database Connections",
        "type": "graph",
        "targets": [
          {
            "expr": "pg_stat_database_numbackends",
            "legendFormat": "Active Connections"
          }
        ]
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "30s"
  }
}
```

### 2. Custom Health Check Dashboard

```bash
# Create health check script
cat > monitoring/health-dashboard.sh << 'EOF'
#!/bin/bash

# Generate simple HTML health dashboard
cat > /tmp/health.html << HTML
<!DOCTYPE html>
<html>
<head>
    <title>PulseGuard Health Dashboard</title>
    <meta http-equiv="refresh" content="30">
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .healthy { color: green; }
        .unhealthy { color: red; }
        .warning { color: orange; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
    </style>
</head>
<body>
    <h1>PulseGuard System Health</h1>
    <p>Last updated: $(date)</p>
    
    <table>
        <tr><th>Service</th><th>Status</th><th>Details</th></tr>
HTML

# Check each service
services=("data-collector" "api-service" "dashboard" "database")
for service in "${services[@]}"; do
    if docker-compose ps $service | grep -q "Up"; then
        status="<span class='healthy'>✅ Running</span>"
        uptime=$(docker-compose ps $service | awk 'NR==2 {print $5" "$6}')
        details="Uptime: $uptime"
    else
        status="<span class='unhealthy'>❌ Down</span>"
        details="Service not running"
    fi
    
    echo "<tr><td>$service</td><td>$status</td><td>$details</td></tr>" >> /tmp/health.html
done

# Add system metrics
echo "<tr><td>CPU Usage</td><td>" >> /tmp/health.html
cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | sed 's/%us,//')
if (( $(echo "$cpu_usage > 80" | bc -l) )); then
    echo "<span class='warning'>⚠️ ${cpu_usage}%</span>" >> /tmp/health.html
else
    echo "<span class='healthy'>✅ ${cpu_usage}%</span>" >> /tmp/health.html
fi
echo "</td><td>Current CPU utilization</td></tr>" >> /tmp/health.html

# Close HTML
cat >> /tmp/health.html << HTML
    </table>
    
    <h2>Quick Actions</h2>
    <ul>
        <li><a href="/metrics">Prometheus Metrics</a></li>
        <li><a href="/grafana">Grafana Dashboard</a></li>
        <li><a href="/api/v1/system/status">API Status</a></li>
    </ul>
</body>
</html>
HTML

# Serve the dashboard
python3 -m http.server 8000 --directory /tmp &
echo "Health dashboard available at http://localhost:8000/health.html"
EOF

chmod +x monitoring/health-dashboard.sh
```

## Log Monitoring

### 1. Centralized Logging with ELK Stack

```yaml
# monitoring/docker-compose.logging.yml
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.15.0
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data

  kibana:
    image: docker.elastic.co/kibana/kibana:7.15.0
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

  logstash:
    image: docker.elastic.co/logstash/logstash:7.15.0
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
      - ./logs:/logs
    depends_on:
      - elasticsearch

volumes:
  elasticsearch_data:
```

### 2. Log Analysis Scripts

```bash
# Create log analysis script
cat > monitoring/analyze-logs.sh << 'EOF'
#!/bin/bash

echo "=== PulseGuard Log Analysis ==="
echo "Date: $(date)"
echo

# API error analysis
echo "1. API Errors (Last 24 hours):"
docker-compose logs --since 24h api-service | grep -i error | tail -10
echo

# Collection failures
echo "2. Collection Failures (Last 24 hours):"
docker-compose logs --since 24h data-collector | grep -i "failed\|error" | tail -10
echo

# Database connection issues
echo "3. Database Issues (Last 24 hours):"
docker-compose logs --since 24h | grep -i "database\|connection" | grep -i error | tail -5
echo

# High response times
echo "4. Slow Requests (>2s response time):"
docker-compose logs --since 24h api-service | grep -E "duration.*[2-9][0-9]{3}ms|duration.*[0-9]{5,}ms" | tail -5
echo

# Memory warnings
echo "5. Memory Warnings:"
docker-compose logs --since 24h | grep -i "memory\|oom" | tail -5
echo

echo "=== Analysis Complete ==="
EOF

chmod +x monitoring/analyze-logs.sh
```

## Automated Monitoring Scripts

### 1. Comprehensive Monitoring Script

```bash
# Create comprehensive monitoring script
cat > monitoring/monitor-all.sh << 'EOF'
#!/bin/bash

SLACK_WEBHOOK_URL="your-slack-webhook-url"
LOG_FILE="/var/log/pulseguard-monitor.log"

log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a $LOG_FILE
}

send_alert() {
    local message="$1"
    local severity="$2"
    
    log_message "ALERT [$severity]: $message"
    
    # Send to Slack
    if [ -n "$SLACK_WEBHOOK_URL" ]; then
        curl -X POST -H 'Content-type: application/json' \
            --data '{"text":"PulseGuard Alert ['"$severity"']: '"$message"'"}' \
            $SLACK_WEBHOOK_URL
    fi
    
    # Send email (if configured)
    # echo "$message" | mail -s "PulseGuard Alert [$severity]" admin@your-domain.com
}

check_services() {
    log_message "Checking service health..."
    
    services=("data-collector" "api-service" "database")
    for service in "${services[@]}"; do
        if ! docker-compose ps $service | grep -q "Up"; then
            send_alert "Service $service is down" "CRITICAL"
        fi
    done
}

check_api_health() {
    log_message "Checking API health..."
    
    if ! curl -f -s http://localhost:3000/health > /dev/null; then
        send_alert "API health check failed" "CRITICAL"
    fi
}

check_disk_space() {
    log_message "Checking disk space..."
    
    usage=$(df / | awk 'NR==2 {print $5}' | sed 's/%//')
    if [ $usage -gt 85 ]; then
        send_alert "Disk space usage is ${usage}%" "WARNING"
    fi
}

check_memory_usage() {
    log_message "Checking memory usage..."
    
    usage=$(free | awk 'NR==2{printf "%.0f", $3*100/$2 }')
    if [ $usage -gt 85 ]; then
        send_alert "Memory usage is ${usage}%" "WARNING"
    fi
}

check_collection_status() {
    log_message "Checking data collection status..."
    
    # Check if collection happened in last hour
    last_collection=$(docker-compose exec -T database psql -U pulseguard_user -d crypto_db -t -c "
        SELECT EXTRACT(EPOCH FROM (NOW() - MAX(created_at))) 
        FROM collection_status 
        WHERE status = 'completed' AND collection_type = 'complete';" 2>/dev/null | xargs)
    
    if [ -n "$last_collection" ] && [ $(echo "$last_collection > 3600" | bc) -eq 1 ]; then
        send_alert "No successful collection in the last hour" "WARNING"
    fi
}

# Run all checks
log_message "Starting monitoring cycle"
check_services
check_api_health
check_disk_space
check_memory_usage
check_collection_status
log_message "Monitoring cycle complete"
EOF

chmod +x monitoring/monitor-all.sh

# Schedule monitoring script
(crontab -l 2>/dev/null; echo "*/5 * * * * /path/to/monitoring/monitor-all.sh") | crontab -
```

### 2. Performance Monitoring

```bash
# Create performance monitoring script
cat > monitoring/performance-monitor.sh << 'EOF'
#!/bin/bash

METRICS_FILE="/var/log/pulseguard-metrics.log"

collect_metrics() {
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    
    # System metrics
    cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | sed 's/%us,//')
    memory_usage=$(free | awk 'NR==2{printf "%.1f", $3*100/$2 }')
    disk_usage=$(df / | awk 'NR==2 {print $5}' | sed 's/%//')
    
    # API metrics
    api_response_time=$(curl -o /dev/null -s -w '%{time_total}' http://localhost:3000/health 2>/dev/null || echo "0")
    
    # Database metrics
    db_connections=$(docker-compose exec -T database psql -U pulseguard_user -d crypto_db -t -c "SELECT count(*) FROM pg_stat_activity;" 2>/dev/null | xargs || echo "0")
    
    # Collection metrics
    recent_collections=$(docker-compose exec -T database psql -U pulseguard_user -d crypto_db -t -c "
        SELECT COUNT(*) FROM collection_status 
        WHERE created_at > NOW() - INTERVAL '1 hour';" 2>/dev/null | xargs || echo "0")
    
    # Log metrics
    echo "$timestamp,CPU:$cpu_usage,Memory:$memory_usage,Disk:$disk_usage,API_Response:$api_response_time,DB_Connections:$db_connections,Recent_Collections:$recent_collections" >> $METRICS_FILE
    
    # Keep only last 1000 lines
    tail -1000 $METRICS_FILE > /tmp/metrics.tmp && mv /tmp/metrics.tmp $METRICS_FILE
}

# Collect metrics
collect_metrics
EOF

chmod +x monitoring/performance-monitor.sh

# Schedule performance monitoring
(crontab -l 2>/dev/null; echo "* * * * * /path/to/monitoring/performance-monitor.sh") | crontab -
```

This comprehensive monitoring setup provides real-time visibility into all aspects of the PulseGuard system, enabling proactive issue detection and resolution.