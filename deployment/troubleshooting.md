# Troubleshooting Guide

## Common Issues and Solutions

### Data Collector Issues

#### 1. Collection Not Starting

**Symptoms:**
- No data appearing in database
- Collector logs show startup errors
- Process exits immediately

**Diagnosis:**
```bash
# Check collector logs
docker-compose logs data-collector

# Check database connectivity
docker-compose exec data-collector node -e "
const pool = require('./app/database');
pool.query('SELECT NOW()').then(r => console.log('Connected:', r.rows[0])).catch(e => console.log('Error:', e.message));
"

# Verify environment variables
docker-compose exec data-collector env | grep -E "(DATABASE_URL|COINGECKO_API_KEY)"
```

**Common Causes & Solutions:**

| Cause | Solution |
|-------|----------|
| Invalid DATABASE_URL | Check connection string format and credentials |
| Missing COINGECKO_API_KEY | Verify API key is set and valid |
| Database not ready | Add healthcheck dependency in docker-compose |
| Network connectivity | Check firewall and network configuration |

**Fix Examples:**
```bash
# Fix database URL format
DATABASE_URL=postgresql://user:pass@host:5432/dbname

# Test API key validity
curl -H "X-Cg-Demo-Api-Key: your-key" "https://api.coingecko.com/api/v3/ping"

# Wait for database to be ready
docker-compose up database
docker-compose exec database pg_isready -U pulseguard_user
```

#### 2. Rate Limit Exceeded

**Symptoms:**
- HTTP 429 errors in logs
- Collection taking much longer than expected
- Many failed API calls

**Diagnosis:**
```bash
# Check for rate limit errors
docker-compose logs data-collector | grep -i "429\|rate limit"

# Check current API usage
curl -H "X-Cg-Demo-Api-Key: your-key" "https://api.coingecko.com/api/v3/ping" -I
```

**Solutions:**
```bash
# Increase batch delay (in .env file)
BATCH_DELAY_SECONDS=10

# Reduce collection scope
COLLECTION_MODE=top1000
COINS_PER_BATCH=100

# Upgrade to paid CoinGecko plan
# Pro plan: 500 calls/minute vs Demo: 30 calls/minute
```

#### 3. Database Connection Pool Exhausted

**Symptoms:**
- "connection pool exhausted" errors
- Timeouts on database operations
- Slow performance

**Diagnosis:**
```bash
# Check active connections
docker-compose exec database psql -U pulseguard_user -d crypto_db -c "
SELECT count(*) as connections, state 
FROM pg_stat_activity 
GROUP BY state;
"

# Check for long-running queries
docker-compose exec database psql -U pulseguard_user -d crypto_db -c "
SELECT now() - query_start as duration, query 
FROM pg_stat_activity 
WHERE state = 'active' 
ORDER BY duration DESC;
"
```

**Solutions:**
```javascript
// Increase pool size in database config
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 30, // Increase from default 20
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 5000
});

// Add proper connection cleanup
process.on('SIGINT', () => {
  pool.end(() => {
    console.log('Pool has ended');
    process.exit(0);
  });
});
```

### API Service Issues

#### 1. API Not Responding

**Symptoms:**
- HTTP connection refused
- Timeouts on API calls
- 502/503 errors

**Diagnosis:**
```bash
# Check if service is running
docker-compose ps api-service

# Check service logs
docker-compose logs api-service

# Test basic connectivity
curl -I http://localhost:3000/health

# Check port binding
netstat -tlnp | grep :3000
```

**Solutions:**
```bash
# Restart the service
docker-compose restart api-service

# Check for port conflicts
sudo lsof -i :3000
# Kill conflicting process if needed

# Verify environment configuration
docker-compose exec api-service env | grep PORT
```

#### 2. High Response Times

**Symptoms:**
- API responses taking >2 seconds
- Timeouts in dashboard
- Poor user experience

**Diagnosis:**
```bash
# Test response times
time curl -H "X-API-Key: your-key" "http://localhost:3000/api/v1/system/status"

# Check database query performance
docker-compose exec database psql -U pulseguard_user -d crypto_db -c "
SELECT query, mean_time, calls 
FROM pg_stat_statements 
ORDER BY mean_time DESC 
LIMIT 10;
"

# Monitor system resources
docker stats api-service
```

**Solutions:**
```sql
-- Add missing indexes
CREATE INDEX CONCURRENTLY idx_minute_close_recent 
ON minute_close (minute_ts DESC, coin_id) 
WHERE minute_ts > NOW() - INTERVAL '1 hour';

-- Analyze table statistics
ANALYZE minute_close;
ANALYZE hourly_close;
ANALYZE coin_priorities;
```

```javascript
// Add response caching
const cache = new Map();
const CACHE_TTL = 30000; // 30 seconds

app.get('/api/v1/system/status', (req, res) => {
  const cacheKey = 'system_status';
  const cached = cache.get(cacheKey);
  
  if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
    return res.json(cached.data);
  }
  
  // ... fetch data
  cache.set(cacheKey, { data: result, timestamp: Date.now() });
  res.json(result);
});
```

#### 3. Authentication Failures

**Symptoms:**
- 401 Unauthorized errors
- "Invalid API key" messages
- Access denied

**Diagnosis:**
```bash
# Check API key configuration
docker-compose exec api-service env | grep API_KEY_SECRET

# Test with correct API key
curl -H "X-API-Key: correct-key" "http://localhost:3000/api/v1/system/status"

# Check logs for auth errors
docker-compose logs api-service | grep -i "auth\|401\|unauthorized"
```

**Solutions:**
```bash
# Generate new API key
openssl rand -hex 32

# Update environment
echo "API_KEY_SECRET=new-secret-key" >> .env

# Restart service
docker-compose restart api-service

# Update clients with new key
```

### Dashboard Issues

#### 1. Dashboard Not Loading

**Symptoms:**
- Blank page or loading spinner
- Network errors in browser console
- Unable to connect to API

**Diagnosis:**
```bash
# Check if dashboard is served
curl -I http://localhost:3001

# Check browser console for errors
# Open browser dev tools → Console tab

# Verify API connectivity from dashboard
curl -H "X-API-Key: your-key" "http://localhost:3000/api/v1/system/status"
```

**Solutions:**
```bash
# Check environment variables
cat dashboard/.env

# Ensure API URL is accessible
VITE_API_BASE_URL=http://localhost:3000/api/v1

# Rebuild dashboard
cd dashboard
npm run build
```

#### 2. CORS Errors

**Symptoms:**
- CORS policy errors in browser console
- API calls failing from dashboard
- "Access-Control-Allow-Origin" errors

**Diagnosis:**
```javascript
// Check browser console for CORS errors
// Look for messages like: "Access to fetch at ... has been blocked by CORS policy"
```

**Solutions:**
```javascript
// In API service (api-service/src/server.js)
const cors = require('cors');

app.use(cors({
  origin: [
    'http://localhost:3001',
    'https://your-dashboard-domain.com'
  ],
  credentials: true
}));
```

```bash
# Or set environment variable
CORS_ORIGIN=http://localhost:3001,https://your-dashboard-domain.com
```

#### 3. Real-time Updates Not Working

**Symptoms:**
- Data not refreshing automatically
- Manual refresh required
- WebSocket connection failures

**Diagnosis:**
```javascript
// Check WebSocket connection in browser console
const ws = new WebSocket('ws://localhost:3000/ws');
ws.onopen = () => console.log('WebSocket connected');
ws.onerror = (error) => console.log('WebSocket error:', error);
```

**Solutions:**
```javascript
// Implement fallback to polling
const useRealtimeData = () => {
  const [data, setData] = useState(null);
  
  useEffect(() => {
    // Try WebSocket first
    const ws = new WebSocket(config.wsUrl);
    
    ws.onmessage = (event) => {
      setData(JSON.parse(event.data));
    };
    
    ws.onerror = () => {
      // Fallback to polling
      const interval = setInterval(() => {
        fetch('/api/v1/system/status')
          .then(res => res.json())
          .then(setData);
      }, 30000);
      
      return () => clearInterval(interval);
    };
    
    return () => ws.close();
  }, []);
  
  return data;
};
```

### Database Issues

#### 1. Database Connection Failures

**Symptoms:**
- "connection refused" errors
- "password authentication failed"
- Services unable to start

**Diagnosis:**
```bash
# Check if PostgreSQL is running
docker-compose ps database

# Test direct connection
docker-compose exec database psql -U pulseguard_user -d crypto_db -c "SELECT NOW();"

# Check connection string format
echo $DATABASE_URL
```

**Solutions:**
```bash
# Start database if not running
docker-compose up -d database

# Check credentials
docker-compose exec database psql -U postgres -c "
SELECT usename, passwd FROM pg_shadow WHERE usename = 'pulseguard_user';
"

# Reset password if needed
docker-compose exec database psql -U postgres -c "
ALTER USER pulseguard_user PASSWORD 'new_password';
"
```

#### 2. Database Performance Issues

**Symptoms:**
- Slow query responses
- High CPU usage on database
- Timeouts on complex queries

**Diagnosis:**
```sql
-- Check slow queries
SELECT query, mean_time, calls, total_time
FROM pg_stat_statements
ORDER BY mean_time DESC
LIMIT 10;

-- Check index usage
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;

-- Check table sizes
SELECT tablename, pg_size_pretty(pg_total_relation_size(tablename::regclass))
FROM pg_tables
WHERE schemaname = 'public';
```

**Solutions:**
```sql
-- Add missing indexes
CREATE INDEX CONCURRENTLY idx_minute_close_coin_time 
ON minute_close (coin_id, minute_ts DESC);

-- Update table statistics
ANALYZE;

-- Vacuum tables
VACUUM ANALYZE minute_close;
VACUUM ANALYZE hourly_close;
```

```bash
# Increase PostgreSQL memory settings
echo "shared_buffers = 256MB" >> postgresql.conf
echo "effective_cache_size = 1GB" >> postgresql.conf
echo "work_mem = 4MB" >> postgresql.conf
```

#### 3. Disk Space Issues

**Symptoms:**
- "No space left on device" errors
- Database write failures
- Slow performance

**Diagnosis:**
```bash
# Check disk usage
df -h

# Check database size
docker-compose exec database psql -U pulseguard_user -d crypto_db -c "
SELECT pg_size_pretty(pg_database_size('crypto_db'));
"

# Check largest tables
docker-compose exec database psql -U pulseguard_user -d crypto_db -c "
SELECT tablename, pg_size_pretty(pg_total_relation_size(tablename::regclass))
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(tablename::regclass) DESC;
"
```

**Solutions:**
```sql
-- Clean up old data
DELETE FROM minute_close WHERE minute_ts < NOW() - INTERVAL '1 hour';
DELETE FROM hourly_close WHERE hour_ts < NOW() - INTERVAL '24 hours';
DELETE FROM daily_close WHERE day_ts < NOW() - INTERVAL '7 days';

-- Vacuum to reclaim space
VACUUM FULL minute_close;
VACUUM FULL hourly_close;
```

```bash
# Set up automated cleanup
cat > cleanup-script.sh << 'EOF'
#!/bin/bash
docker-compose exec -T database psql -U pulseguard_user -d crypto_db -c "
DELETE FROM minute_close WHERE minute_ts < NOW() - INTERVAL '1 hour';
DELETE FROM hourly_close WHERE hour_ts < NOW() - INTERVAL '24 hours';
VACUUM ANALYZE;
"
EOF

# Schedule cleanup
echo "0 */6 * * * /path/to/cleanup-script.sh" | crontab -
```

### Network and Connectivity Issues

#### 1. Service Discovery Problems

**Symptoms:**
- Services can't communicate with each other
- "host not found" errors
- DNS resolution failures

**Diagnosis:**
```bash
# Check Docker network
docker network ls
docker network inspect pulseguard_default

# Test internal connectivity
docker-compose exec api-service ping database
docker-compose exec data-collector nslookup database
```

**Solutions:**
```yaml
# Explicitly define network in docker-compose.yml
networks:
  pulseguard-network:
    driver: bridge

services:
  database:
    networks:
      - pulseguard-network
  api-service:
    networks:
      - pulseguard-network
```

#### 2. External API Connectivity

**Symptoms:**
- CoinGecko API calls failing
- Network timeouts
- DNS resolution issues

**Diagnosis:**
```bash
# Test external connectivity
docker-compose exec data-collector curl -I https://api.coingecko.com/api/v3/ping

# Check DNS resolution
docker-compose exec data-collector nslookup api.coingecko.com

# Test with specific API key
docker-compose exec data-collector curl -H "X-Cg-Demo-Api-Key: your-key" https://api.coingecko.com/api/v3/ping
```

**Solutions:**
```bash
# Add DNS servers to docker-compose.yml
services:
  data-collector:
    dns:
      - 8.8.8.8
      - 1.1.1.1

# Check firewall rules
sudo ufw status
# Allow outbound HTTPS
sudo ufw allow out 443
```

## Diagnostic Tools and Scripts

### 1. Comprehensive Health Check

```bash
#!/bin/bash
# health-check.sh

echo "=== PulseGuard System Health Check ==="
echo "Timestamp: $(date)"
echo

# Check Docker services
echo "1. Docker Services:"
docker-compose ps
echo

# Check resource usage
echo "2. Resource Usage:"
docker stats --no-stream
echo

# Check database connectivity
echo "3. Database Connectivity:"
if docker-compose exec -T database pg_isready -U pulseguard_user; then
    echo "✅ Database is ready"
else
    echo "❌ Database connection failed"
fi
echo

# Check API health
echo "4. API Health:"
if curl -f -s http://localhost:3000/health > /dev/null; then
    echo "✅ API is responding"
else
    echo "❌ API health check failed"
fi
echo

# Check recent collections
echo "5. Recent Data Collection:"
last_collection=$(docker-compose exec -T database psql -U pulseguard_user -d crypto_db -t -c "
SELECT EXTRACT(EPOCH FROM (NOW() - MAX(created_at)))/60 as minutes_ago
FROM collection_status 
WHERE status = 'completed';" 2>/dev/null | xargs)

if [ -n "$last_collection" ] && [ $(echo "$last_collection < 60" | bc) -eq 1 ]; then
    echo "✅ Last collection: ${last_collection} minutes ago"
else
    echo "⚠️ No recent collection found"
fi
echo

# Check disk space
echo "6. Disk Space:"
df -h / | grep -v Filesystem
echo

# Check for errors in logs
echo "7. Recent Errors:"
error_count=$(docker-compose logs --since 1h | grep -i error | wc -l)
if [ $error_count -eq 0 ]; then
    echo "✅ No errors in the last hour"
else
    echo "⚠️ Found $error_count errors in the last hour"
    docker-compose logs --since 1h | grep -i error | tail -3
fi

echo
echo "=== Health Check Complete ==="
```

### 2. Performance Diagnostics

```bash
#!/bin/bash
# performance-check.sh

echo "=== Performance Diagnostics ==="

# API response times
echo "1. API Response Times:"
for endpoint in "/health" "/api/v1/system/status" "/api/v1/market/top/10"; do
    time=$(curl -o /dev/null -s -w '%{time_total}' "http://localhost:3000$endpoint" 2>/dev/null)
    echo "  $endpoint: ${time}s"
done
echo

# Database query performance
echo "2. Database Query Performance:"
docker-compose exec -T database psql -U pulseguard_user -d crypto_db -c "
SELECT query, calls, mean_time, max_time
FROM pg_stat_statements
WHERE calls > 10
ORDER BY mean_time DESC
LIMIT 5;
" 2>/dev/null
echo

# Collection metrics
echo "3. Collection Metrics (Last 24h):"
docker-compose exec -T database psql -U pulseguard_user -d crypto_db -c "
SELECT 
    COUNT(*) as total_collections,
    COUNT(*) FILTER (WHERE status = 'completed') as successful,
    COUNT(*) FILTER (WHERE status = 'failed') as failed,
    AVG(EXTRACT(EPOCH FROM (end_time - start_time))) as avg_duration_seconds
FROM collection_status
WHERE created_at > NOW() - INTERVAL '24 hours';
" 2>/dev/null
```

### 3. Log Analysis

```bash
#!/bin/bash
# log-analysis.sh

echo "=== Log Analysis ==="

# Error frequency
echo "1. Error Frequency (Last 24h):"
docker-compose logs --since 24h | grep -i error | \
    awk '{print $1, $2}' | sort | uniq -c | sort -nr | head -10
echo

# API rate limiting
echo "2. Rate Limiting Events:"
docker-compose logs --since 24h | grep -i "429\|rate limit" | wc -l
echo

# Database connection issues
echo "3. Database Connection Issues:"
docker-compose logs --since 24h | grep -i "connection.*error\|connection.*failed" | wc -l
echo

# Memory warnings
echo "4. Memory Warnings:"
docker-compose logs --since 24h | grep -i "memory\|oom" | wc -l
```

## Emergency Procedures

### 1. Service Recovery

```bash
#!/bin/bash
# emergency-recovery.sh

echo "Starting emergency recovery procedure..."

# Stop all services
echo "1. Stopping all services..."
docker-compose down

# Clean up containers and networks
echo "2. Cleaning up..."
docker system prune -f

# Restart database first
echo "3. Starting database..."
docker-compose up -d database

# Wait for database to be ready
echo "4. Waiting for database..."
sleep 30
until docker-compose exec database pg_isready -U pulseguard_user; do
    echo "Waiting for database..."
    sleep 5
done

# Start API service
echo "5. Starting API service..."
docker-compose up -d api-service

# Wait for API to be ready
sleep 10
until curl -f http://localhost:3000/health; do
    echo "Waiting for API..."
    sleep 5
done

# Start data collector
echo "6. Starting data collector..."
docker-compose up -d data-collector

# Start dashboard
echo "7. Starting dashboard..."
docker-compose up -d dashboard

echo "Recovery complete!"
```

### 2. Data Recovery

```bash
#!/bin/bash
# data-recovery.sh

# Restore from backup
restore_from_backup() {
    local backup_file=$1
    
    echo "Restoring database from $backup_file..."
    
    # Stop services
    docker-compose stop data-collector api-service
    
    # Drop and recreate database
    docker-compose exec database psql -U postgres -c "DROP DATABASE IF EXISTS crypto_db;"
    docker-compose exec database psql -U postgres -c "CREATE DATABASE crypto_db;"
    
    # Restore data
    gunzip -c $backup_file | docker-compose exec -T database psql -U pulseguard_user crypto_db
    
    # Restart services
    docker-compose start data-collector api-service
    
    echo "Database restoration complete!"
}

# Usage: ./data-recovery.sh backup_file.sql.gz
restore_from_backup $1
```

This troubleshooting guide provides systematic approaches to diagnosing and resolving the most common issues in PulseGuard deployments. Regular use of the diagnostic scripts can help prevent issues before they become critical.