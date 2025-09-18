# Data Collector Architecture

## Overview

The PulseGuard Data Collector is a Node.js application designed to efficiently collect cryptocurrency price data from the CoinGecko API and store it in a PostgreSQL database with multiple time aggregations.

## Core Components

### 1. Priority Manager (`app/priority-manager.js`)

Manages market cap rankings and coin prioritization:

```javascript
class PriorityManager {
  // Fetches and caches daily market cap rankings
  async refreshPriorities()
  
  // Generates paginated coin batches based on collection mode
  getCoinBatches(mode)
  
  // Updates database with latest rankings
  updateCoinRanks()
}
```

**Key Features**:
- Daily market cap ranking updates
- Configurable collection modes (top1000, top5000, complete)
- Priority coin fallback (bitcoin, ethereum, cardano)
- Batch generation for API efficiency

### 2. Collection Engine (`app/complete-enhanced-collector.js`)

Main collection orchestrator with batch processing:

```javascript
class CollectionEngine {
  // Main collection orchestrator
  async collectPrices()
  
  // Processes single batch of coins
  async processBatch(coins)
  
  // Creates time-based aggregations
  async aggregateTimeframes()
  
  // Removes expired data
  async cleanupOldData()
}
```

**Key Features**:
- Batch processing with rate limiting
- Exponential backoff for API errors
- Comprehensive error handling
- Progress tracking and metrics

### 3. Continuous Collector (`app/continuous-collector.js`)

Handles minute-by-minute collection with queue management:

```javascript
class ContinuousCollector {
  // Minute-by-minute collection scheduler
  async startContinuousCollection()
  
  // Queue-based missed minute recovery
  async processQueue()
  
  // Gap detection and filling
  async detectAndFillGaps()
}
```

**Key Features**:
- Cron-based scheduling
- Queue system for missed minutes
- Gap detection and recovery
- Real-time health monitoring

### 4. Configuration Manager (`app/config.js`)

Centralized configuration management:

```javascript
class ConfigManager {
  // Loads and validates environment settings
  loadConfig()
  
  // Provides collection mode configurations
  getCollectionConfig(mode)
  
  // Handles priority cache loading
  loadPriorityCache()
}
```

## Data Flow Architecture

### 1. Collection Flow

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Scheduler     │───▶│  Priority Mgr   │───▶│ Collection Eng  │
│   (Cron/Manual) │    │  (Coin Rankings)│    │ (Batch Process) │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                       │
                                ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Priority Cache  │    │  CoinGecko API  │    │   Rate Limiter  │
│ (JSON File)     │    │  (REST Calls)   │    │ (Delay/Backoff) │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                       │
                                ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Data Validator │    │  Data Transform │    │  Database Store │
│  (Price Check)  │───▶│  (Normalize)    │───▶│  (PostgreSQL)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### 2. Aggregation Flow

```
minute_close (Raw Data)
       │
       ▼ (Every Hour)
hourly_close ──────────────┐
       │                   │
       ▼ (Every Day)       ▼ (Every Week)
daily_close ────────── weekly_close
       │                   │
       ▼ (Every Month)     ▼ (Every Year)
monthly_close ──────── yearly_close
```

### 3. Cleanup Flow

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Cleanup Job   │───▶│ Retention Check │───▶│  Data Removal   │
│   (Scheduled)   │    │ (Time-based)    │    │  (SQL DELETE)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │
         ▼
┌─────────────────┐
│ Retention Rules │
│ - minute: 60min │
│ - hourly: 24hrs │
│ - daily: 7 days │
│ - weekly: 52wks │
│ - monthly: ∞    │
└─────────────────┘
```

## Database Schema Architecture

### Time-Series Tables

```sql
-- Primary data storage with time-based partitioning
CREATE TABLE minute_close (
    coin_id TEXT NOT NULL,
    minute_ts TIMESTAMPTZ NOT NULL,
    price_usd NUMERIC(20,8),
    market_cap_usd NUMERIC(20,2),
    volume_24h_usd NUMERIC(20,2),
    coin_rank INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    PRIMARY KEY (coin_id, minute_ts)
) PARTITION BY RANGE (minute_ts);

-- Aggregated tables follow similar structure
CREATE TABLE hourly_close (...) PARTITION BY RANGE (hour_ts);
CREATE TABLE daily_close (...) PARTITION BY RANGE (day_ts);
CREATE TABLE weekly_close (...) PARTITION BY RANGE (week_ts);
CREATE TABLE monthly_close (...) PARTITION BY RANGE (month_ts);
```

### Management Tables

```sql
-- Market cap rankings and metadata
CREATE TABLE coin_priorities (
    coin_id TEXT PRIMARY KEY,
    market_cap_rank INTEGER NOT NULL,
    market_cap_usd NUMERIC(20,2),
    symbol TEXT,
    name TEXT,
    last_updated TIMESTAMPTZ DEFAULT NOW()
);

-- Collection monitoring and metrics
CREATE TABLE collection_status (
    id SERIAL PRIMARY KEY,
    collection_type TEXT NOT NULL, -- 'complete', 'continuous', 'priority_refresh'
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ,
    coins_processed INTEGER DEFAULT 0,
    coins_failed INTEGER DEFAULT 0,
    api_calls_made INTEGER DEFAULT 0,
    status TEXT DEFAULT 'running', -- 'running', 'completed', 'failed'
    error_message TEXT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);
```

## Rate Limiting Architecture

### 1. API Rate Limiter

```javascript
class RateLimiter {
  constructor(requestsPerMinute = 30) {
    this.requestsPerMinute = requestsPerMinute;
    this.requestTimes = [];
    this.baseDelay = 60000 / requestsPerMinute; // ms between requests
  }

  async waitForSlot() {
    const now = Date.now();
    const oneMinuteAgo = now - 60000;
    
    // Remove old requests
    this.requestTimes = this.requestTimes.filter(time => time > oneMinuteAgo);
    
    if (this.requestTimes.length >= this.requestsPerMinute) {
      const oldestRequest = Math.min(...this.requestTimes);
      const waitTime = 60000 - (now - oldestRequest);
      await this.sleep(waitTime);
    }
    
    this.requestTimes.push(Date.now());
  }
}
```

### 2. Backoff Strategy

```javascript
class BackoffStrategy {
  constructor(baseDelay = 1000, maxDelay = 30000, multiplier = 2) {
    this.baseDelay = baseDelay;
    this.maxDelay = maxDelay;
    this.multiplier = multiplier;
  }

  calculateDelay(attemptNumber) {
    const delay = this.baseDelay * Math.pow(this.multiplier, attemptNumber - 1);
    return Math.min(delay, this.maxDelay);
  }

  async executeWithBackoff(operation, maxAttempts = 3) {
    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
      try {
        return await operation();
      } catch (error) {
        if (attempt === maxAttempts) throw error;
        
        const delay = this.calculateDelay(attempt);
        console.log(`Attempt ${attempt} failed, retrying in ${delay}ms`);
        await this.sleep(delay);
      }
    }
  }
}
```

## Error Handling Architecture

### 1. Error Classification

```javascript
class ErrorHandler {
  classifyError(error) {
    if (error.response?.status === 429) {
      return 'RATE_LIMIT';
    } else if (error.response?.status >= 500) {
      return 'SERVER_ERROR';
    } else if (error.code === 'ECONNRESET' || error.code === 'ETIMEDOUT') {
      return 'NETWORK_ERROR';
    } else if (error.response?.status === 401) {
      return 'AUTH_ERROR';
    } else {
      return 'UNKNOWN_ERROR';
    }
  }

  async handleError(error, context) {
    const errorType = this.classifyError(error);
    
    switch (errorType) {
      case 'RATE_LIMIT':
        return this.handleRateLimit(error, context);
      case 'SERVER_ERROR':
      case 'NETWORK_ERROR':
        return this.handleRetryableError(error, context);
      case 'AUTH_ERROR':
        return this.handleAuthError(error, context);
      default:
        return this.handleUnknownError(error, context);
    }
  }
}
```

### 2. Recovery Strategies

```javascript
class RecoveryStrategy {
  async recoverFromFailure(failureType, context) {
    switch (failureType) {
      case 'BATCH_FAILURE':
        return this.retryIndividualCoins(context.coins);
        
      case 'DATABASE_FAILURE':
        return this.bufferDataAndRetry(context.data);
        
      case 'API_FAILURE':
        return this.fallbackToPriorityCoins(context.priorityCoins);
        
      case 'NETWORK_FAILURE':
        return this.scheduleRetryWithBackoff(context.operation);
    }
  }
}
```

## Performance Optimizations

### 1. Database Optimizations

```sql
-- Partitioning for time-series data
CREATE TABLE minute_close_y2025m01 PARTITION OF minute_close
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

-- Composite indexes for efficient queries
CREATE INDEX CONCURRENTLY idx_minute_close_coin_time 
    ON minute_close (coin_id, minute_ts DESC);

-- Partial indexes for active data
CREATE INDEX CONCURRENTLY idx_recent_minute_close 
    ON minute_close (minute_ts DESC, coin_id)
    WHERE minute_ts > NOW() - INTERVAL '1 hour';
```

### 2. Memory Management

```javascript
class MemoryManager {
  constructor(maxBatchSize = 1000) {
    this.maxBatchSize = maxBatchSize;
    this.processedCount = 0;
  }

  async processBatchesInChunks(allData) {
    for (let i = 0; i < allData.length; i += this.maxBatchSize) {
      const chunk = allData.slice(i, i + this.maxBatchSize);
      await this.processBatch(chunk);
      
      // Force garbage collection periodically
      if (++this.processedCount % 10 === 0) {
        if (global.gc) global.gc();
      }
    }
  }
}
```

### 3. Connection Pooling

```javascript
// PostgreSQL connection pool configuration
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,                    // Maximum connections
  idleTimeoutMillis: 30000,   // Close idle connections after 30s
  connectionTimeoutMillis: 2000, // Timeout connection attempts after 2s
  statement_timeout: 10000,   // Timeout statements after 10s
  query_timeout: 10000,       // Timeout queries after 10s
});
```

## Monitoring and Observability

### 1. Metrics Collection

```javascript
class MetricsCollector {
  constructor() {
    this.metrics = {
      totalCollections: 0,
      successfulCollections: 0,
      failedCollections: 0,
      avgCollectionTime: 0,
      apiCallsToday: 0,
      coinsProcessedToday: 0,
      lastCollectionTime: null
    };
  }

  recordCollection(success, duration, coinsProcessed) {
    this.metrics.totalCollections++;
    if (success) {
      this.metrics.successfulCollections++;
    } else {
      this.metrics.failedCollections++;
    }
    this.updateAvgCollectionTime(duration);
    this.metrics.coinsProcessedToday += coinsProcessed;
    this.metrics.lastCollectionTime = new Date();
  }
}
```

### 2. Health Checks

```javascript
class HealthChecker {
  async checkSystemHealth() {
    return {
      database: await this.checkDatabaseHealth(),
      api: await this.checkApiHealth(),
      collection: await this.checkCollectionHealth(),
      storage: await this.checkStorageHealth()
    };
  }

  async checkDatabaseHealth() {
    try {
      const result = await pool.query('SELECT NOW()');
      return { status: 'healthy', latency: Date.now() - startTime };
    } catch (error) {
      return { status: 'unhealthy', error: error.message };
    }
  }
}
```

This architecture provides a robust, scalable foundation for cryptocurrency data collection with comprehensive error handling, performance optimization, and monitoring capabilities.