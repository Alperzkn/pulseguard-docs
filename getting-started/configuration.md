# Configuration Guide

This guide covers the configuration options for all PulseGuard components.

## Environment Variables

### Data Collector Configuration

**File**: `pulseguard/.env`

```bash
# Database Configuration
DATABASE_URL=postgresql://username:password@host:5432/dbname

# CoinGecko API Configuration
COINGECKO_API_KEY=your_demo_api_key
QUOTE=usd

# Collection Settings
COLLECTION_MODE=top1000              # top1000 | top5000 | complete
COINS_PER_BATCH=250                  # Coins per API request
BATCH_DELAY_SECONDS=6                # Delay between batches

# Priority Management
PRIORITY_COINS=bitcoin,ethereum,cardano
PRIORITY_REFRESH_HOUR=0              # Daily refresh at midnight UTC
PRIORITY_CACHE_FILE=data/coin_priorities.json

# Rate Limiting
MAX_RETRIES=3
BACKOFF_MULTIPLIER=2
```

### API Service Configuration

**File**: `pulseguard-api/.env.api`

```bash
# Database Configuration
DATABASE_URL=postgresql://username:password@host:5432/dbname

# Server Configuration
PORT=3000
NODE_ENV=production                  # development | production

# Authentication
API_KEY_SECRET=your-super-secret-api-key-here

# Rate Limiting
RATE_LIMIT_MAX_REQUESTS=100         # Requests per window
RATE_LIMIT_WINDOW_MS=900000         # 15 minutes

# CORS Configuration
CORS_ORIGIN=*                       # Allowed origins
CORS_CREDENTIALS=false

# Logging
LOG_LEVEL=info                      # error | warn | info | debug
LOG_FORMAT=json                     # json | simple
```

### Dashboard Configuration

**File**: `pulseguard-dashboard/.env`

```bash
# API Configuration
VITE_API_BASE_URL=http://localhost:3000/api/v1
VITE_API_KEY=your-super-secret-api-key-here
VITE_WS_URL=ws://localhost:3000/ws

# Environment
VITE_ENVIRONMENT=development        # development | production
VITE_DEBUG=true

# Monitoring Configuration
VITE_REFRESH_INTERVAL=30000         # 30 seconds
VITE_CHART_ANIMATION=true
VITE_REAL_TIME_UPDATES=true
```

## Collection Modes

### Top 1000 Mode (Default)
- **Target**: Top 1000 cryptocurrencies by market cap
- **Coverage**: ~99% of total market capitalization
- **API Calls**: 4 requests per cycle
- **Collection Time**: 4-16 minutes
- **Recommended For**: Most use cases

```bash
COLLECTION_MODE=top1000
COINS_PER_BATCH=250
BATCH_DELAY_SECONDS=6
```

### Top 5000 Mode
- **Target**: Top 5000 cryptocurrencies by market cap
- **Coverage**: ~99.9% of total market capitalization
- **API Calls**: 20 requests per cycle
- **Collection Time**: 20-80 minutes
- **Recommended For**: Comprehensive analysis

```bash
COLLECTION_MODE=top5000
COINS_PER_BATCH=250
BATCH_DELAY_SECONDS=6
```

### Complete Mode
- **Target**: All available cryptocurrencies (18,000+)
- **Coverage**: 100% of available data
- **API Calls**: 75+ requests per cycle
- **Collection Time**: 75-300 minutes
- **Recommended For**: Research and complete datasets

```bash
COLLECTION_MODE=complete
COINS_PER_BATCH=250
BATCH_DELAY_SECONDS=6
```

## Rate Limiting Configuration

### CoinGecko API Limits

**Demo Plan (Recommended)**:
```bash
BATCH_DELAY_SECONDS=6               # Conservative approach
MAX_RETRIES=3
BACKOFF_MULTIPLIER=2
```

**Pro Plan**:
```bash
BATCH_DELAY_SECONDS=2               # Faster collection
MAX_RETRIES=5
BACKOFF_MULTIPLIER=1.5
```

### API Service Rate Limits

**Development**:
```bash
RATE_LIMIT_MAX_REQUESTS=200
RATE_LIMIT_WINDOW_MS=900000         # 15 minutes
```

**Production**:
```bash
RATE_LIMIT_MAX_REQUESTS=100
RATE_LIMIT_WINDOW_MS=900000         # 15 minutes
```

## Database Configuration

### Connection Settings

**Local Development**:
```bash
DATABASE_URL=postgresql://pulseguard_user:password@localhost:5432/crypto_db
```

**Production**:
```bash
DATABASE_URL=postgresql://username:password@prod-db-host:5432/crypto_db
```

**Docker**:
```bash
DATABASE_URL=postgresql://pulseguard_user:password@database:5432/crypto_db
```

### Performance Tuning

**PostgreSQL Configuration** (`postgresql.conf`):
```ini
# Memory
shared_buffers = 256MB              # 25% of RAM
effective_cache_size = 1GB          # 75% of RAM
work_mem = 4MB
maintenance_work_mem = 64MB

# Connections
max_connections = 100

# Write-ahead logging
wal_buffers = 16MB
checkpoint_completion_target = 0.9

# Query planner
random_page_cost = 1.1
effective_io_concurrency = 200
```

## Security Configuration

### API Key Management

**Generate Secure API Key**:
```bash
# Generate random API key
openssl rand -hex 32

# Or use Node.js
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

**Environment Security**:
```bash
# Set proper file permissions
chmod 600 .env
chmod 600 .env.api

# Never commit to version control
echo ".env*" >> .gitignore
```

### CORS Configuration

**Development**:
```bash
CORS_ORIGIN=http://localhost:3001,http://localhost:3000
CORS_CREDENTIALS=true
```

**Production**:
```bash
CORS_ORIGIN=https://your-dashboard-domain.com
CORS_CREDENTIALS=false
```

## Monitoring Configuration

### Dashboard Refresh Intervals

**Development**:
```bash
VITE_REFRESH_INTERVAL=10000         # 10 seconds for testing
VITE_DEBUG=true
```

**Production**:
```bash
VITE_REFRESH_INTERVAL=30000         # 30 seconds
VITE_DEBUG=false
```

### Alert Thresholds

Configure monitoring alerts in the dashboard:

```typescript
// src/config/monitoring.ts
export const alertThresholds = {
  apiUsageWarning: 80,              // 80% of rate limit
  missingDataThreshold: 5,          // 5% missing data triggers alert
  systemHealthTimeout: 10000,       // 10 seconds response timeout
  collectionDelayWarning: 300000,   // 5 minutes collection delay
  databaseConnectionTimeout: 5000   // 5 seconds DB timeout
};
```

## Logging Configuration

### Data Collector Logging

**Development**:
```bash
LOG_LEVEL=debug
LOG_TO_FILE=false
```

**Production**:
```bash
LOG_LEVEL=info
LOG_TO_FILE=true
LOG_FILE_PATH=/var/log/pulseguard/collector.log
LOG_MAX_SIZE=10MB
LOG_MAX_FILES=5
```

### API Service Logging

**Morgan Configuration**:
```bash
# Development
LOG_FORMAT=dev

# Production
LOG_FORMAT=combined
```

**Winston Configuration** (in code):
```javascript
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});
```

## Performance Configuration

### Database Optimization

**Connection Pooling**:
```javascript
// PostgreSQL pool configuration
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,                          // Maximum connections
  idleTimeoutMillis: 30000,         // 30 seconds
  connectionTimeoutMillis: 2000,    // 2 seconds
});
```

**Query Optimization**:
```sql
-- Essential indexes for performance
CREATE INDEX CONCURRENTLY idx_minute_close_coin_ts ON minute_close (coin_id, minute_ts DESC);
CREATE INDEX CONCURRENTLY idx_coin_priorities_rank ON coin_priorities (market_cap_rank);
CREATE INDEX CONCURRENTLY idx_collection_status_time ON collection_status (start_time DESC);
```

### Dashboard Performance

**React Query Configuration**:
```typescript
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30000,             // 30 seconds
      cacheTime: 300000,            // 5 minutes
      refetchOnWindowFocus: false,
      retry: 3,
      retryDelay: attemptIndex => Math.min(1000 * 2 ** attemptIndex, 30000)
    }
  }
});
```

## Environment-Specific Configurations

### Development Environment

**Optimized for development speed and debugging**:
```bash
# Data Collector
COLLECTION_MODE=top1000
BATCH_DELAY_SECONDS=6
LOG_LEVEL=debug

# API Service
NODE_ENV=development
LOG_LEVEL=debug
CORS_ORIGIN=*

# Dashboard
VITE_ENVIRONMENT=development
VITE_DEBUG=true
VITE_REFRESH_INTERVAL=10000
```

### Staging Environment

**Production-like with additional logging**:
```bash
# Data Collector
COLLECTION_MODE=top5000
BATCH_DELAY_SECONDS=4
LOG_LEVEL=info

# API Service
NODE_ENV=production
LOG_LEVEL=info
RATE_LIMIT_MAX_REQUESTS=150

# Dashboard
VITE_ENVIRONMENT=staging
VITE_DEBUG=false
VITE_REFRESH_INTERVAL=30000
```

### Production Environment

**Optimized for performance and stability**:
```bash
# Data Collector
COLLECTION_MODE=top5000
BATCH_DELAY_SECONDS=2.5
LOG_LEVEL=warn

# API Service
NODE_ENV=production
LOG_LEVEL=warn
RATE_LIMIT_MAX_REQUESTS=100

# Dashboard
VITE_ENVIRONMENT=production
VITE_DEBUG=false
VITE_REFRESH_INTERVAL=60000
```

## Configuration Validation

### Environment Validation Script

Create `scripts/validate-config.js`:

```javascript
const requiredEnvVars = {
  collector: [
    'DATABASE_URL',
    'COINGECKO_API_KEY',
    'COLLECTION_MODE'
  ],
  api: [
    'DATABASE_URL',
    'API_KEY_SECRET',
    'PORT'
  ],
  dashboard: [
    'VITE_API_BASE_URL',
    'VITE_API_KEY'
  ]
};

function validateConfig(component) {
  const missing = requiredEnvVars[component].filter(
    varName => !process.env[varName]
  );
  
  if (missing.length > 0) {
    console.error(`Missing required environment variables for ${component}:`);
    missing.forEach(varName => console.error(`  - ${varName}`));
    process.exit(1);
  }
  
  console.log(`✅ Configuration valid for ${component}`);
}
```

Run validation:
```bash
node scripts/validate-config.js collector
node scripts/validate-config.js api
node scripts/validate-config.js dashboard
```

## Best Practices

1. **Use separate environment files** for each component
2. **Never commit sensitive data** to version control
3. **Use strong, unique API keys** for each environment
4. **Monitor rate limits** and adjust delays accordingly
5. **Set appropriate log levels** for each environment
6. **Regularly rotate credentials** in production
7. **Use connection pooling** for database efficiency
8. **Configure proper CORS** for security
9. **Set reasonable timeouts** for external APIs
10. **Monitor configuration changes** in production